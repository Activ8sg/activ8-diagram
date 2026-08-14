# Instant Payout — Payment Flow (Trusted Organizers)

> **Status:** Design / pre-implementation spec.
> **Scope:** Backend only. **No Flutter changes required** — players and organizers
> use the app exactly as today; only the *destination* of the money changes server-side.
>
> Diagrams below are [Mermaid](https://mermaid.js.org/) (renders on GitHub & VS Code).
> The rest of this folder uses `.drawio`; this flow is text-based so it diffs cleanly
> and stays in sync with the code.

---

## 1. Why this exists

Today Activ8 runs **one** money model — *escrow*: a player pays, the money is held in the
organizer's `pending_earnings`, matures into cashable `earnings_balance` when the game
ends, and the organizer cashes out. The platform never fronts money.

**Instant Payout** adds a **second, parallel model** for organizers who are both
`is_organizer` **and** `is_trusted`. The platform effectively **buys the game's inventory
upfront and resells the seats**:

- The platform **pays the organizer in advance** (before/independent of bookings).
- When players later book, their money is **kept by the platform** (to recoup the advance)
  and is **never credited to any organizer wallet** — the organizer was already paid.

The two models coexist. Which one a game uses is decided **once, at creation**, and
snapshotted on the game row so later trust changes never affect money already in flight.

---

## 2. Accounts & buckets

| Account | Where | Holds |
|---|---|---|
| Player **spendable** | `wallets.balance` | Top-ups; debited on booking |
| Organizer **pending** (escrow only) | `wallets.pending_earnings` | Escrowed earnings until game ends |
| Organizer **cashable** | `wallets.earnings_balance` | Matured earnings **and instant advances** — cashable now |
| **Platform house account** | `platform_ledger` *(new table)* | Advances paid out (−), player money recouped (+), refunds absorbed (−) |

> The platform account is a **ledger, not a wallet** — this is the "held in our system,
> not credited to any wallet" money you described. Per-game P&L = `SUM(amount)` for that
> `game_id`; negative = the platform lost money on that game (under-booked).

---

## 3. Per-organizer configuration

`is_trusted` (migration `101`, already live) gates **whether** a game is instant.
A new `config JSONB` column on `profiles` (migration `102`) controls **how much** and
**how soon**, per organizer:

```jsonc
{
  "payout": {
    "advance": "full",          // "full" = 100% upfront | "percentage" = split
    "advancePercentage": 60,     // used when advance = "percentage"
    "delayDays": 3,              // optional per-organizer override of the global X
    "paused": false,             // temporary OFF switch (migration 115)
    "pausedUntil": null          // optional auto-resume instant (ISO); null = until an admin lifts it
  }
}
```

- Global default delay `X` → `admin_config.instant_payout_delay_days`.
- Empty `config` (`{}`) + trusted ⇒ sane defaults (treat as `full`, global `X`, not paused).
- `config` is a generic settings bag — future organizer settings go here, no schema churn.

### Pause (migration `115`)

`paused` is the temporary OFF switch for the auto-transfer — the lever to use when the
answer isn't "pay later" but "don't pay for now" (disputed game, fraud check, cash-flow
freeze). It is deliberately **not** `is_trusted = false`, which would silently flip every
future game to escrow.

- Paused = `paused === true` AND (`pausedUntil` absent OR still in the future) —
  `fn_payout_is_paused(config)`; an elapsed `pausedUntil` auto-resumes with no admin write.
- New games still snapshot `payout_type = 'instant'`, so **players' money keeps being
  collected by the app** and is still not credited to the organizer. Only the outbound leg
  stops.
- Advances and percentage remainders queue up. Whatever is still queued when the pause
  lifts is paid by the first sweep, honouring the original `delayDays` cutoffs — so a game
  paid by bank transfer in the meantime **must** be recorded (see §3b) or it is paid twice.
- Enforced at the leaf functions (`fn_pay_game_advance`, `fn_settle_instant_game` → reason
  `PAUSED`), so the cron sweeps, the lazy on-read fallback and manual RPC calls all obey it.
- Admin UI: Users → user → Payout tab → "Pause auto-transfer" (+ optional resume date).
  API: `PATCH /admin/users/:userId/payout-config` `{ paused, pausedUntil }`, audited as
  `user.payout_pause` / `user.payout_resume`.
- The organizer's earnings forecast shows the queued amounts with `onHold: true` and a null
  `payoutDate` instead of a date it cannot promise.

### 3b. Offline settlement (migration `116`)

The reason to pause is usually **"pay this organizer by hand, outside the app"** — we still
collect from players in-app (which is what caps our exposure when an organizer cancels a
game), but the money to the organizer goes out by bank transfer so it never becomes
cashable wallet balance.

```
pause organizer ─▶ they keep posting (payout_type still 'instant', players still pay in-app)
we bank-transfer ─▶ outside the app entirely
record it here  ─▶ game closed for payout, wallet/earnings untouched
```

`instant_offline_payouts` (game_id, organizer_id, amount, reference, note, recorded_by,
paid_at) is the record. `fn_mark_game_advance_offline(game, amount?, admin?, ref?, note?)`:

- stamps `advance_paid_at` + `advance_settled_at` so **no sweep ever pays that game**,
- leaves `advance_amount = 0` — nothing entered the wallet, nothing is cashable, the
  organizer never sees it as earnings,
- defaults `amount` to what the app *would* have advanced (capacity, or the percentage),
- refuses with `ALREADY_PAID_IN_APP` if the wallet advance already went out,
- allows several tranches per game; `fn_unmark_game_advance_offline` undoes a mis-entry
  (safe — there is no wallet credit to claw back).

Deliberately **not** a `wallet_transaction`: that ledger is the organizer's in-app money.
Offline cash is folded into `v_instant_payout_pnl` instead — `paid_offline` plus its share
of `advanced_to_organizer` — so per-game P&L stays honest.

- Admin UI: **Payouts → "Instant — unpaid"** lists every instant game still owing, with
  what we owe vs what players have already paid us, and a "Mark paid offline" action.
- API: `GET /admin/payouts/instant-held[?organizerId=]`,
  `POST|DELETE /admin/payouts/games/:gameId/offline`, audited as `payout.offline_record` /
  `payout.offline_revert`.

---

## 4. Decision at game creation

```mermaid
flowchart TD
    A["POST /games (createGame)"] --> B{"is_organizer AND is_trusted?"}
    B -- No --> C["payout_type = 'escrow'<br/>(existing flow, unchanged)"]
    B -- Yes --> D["payout_type = 'instant'<br/>snapshot on game row"]
    D --> E{"config.payout.advance"}
    E -- full --> F["100% of capacity paid at cutoff"]
    E -- percentage --> G["pct% at cutoff<br/>+ remaining (100 − pct)% at game end"]
```

`capacity = max_slots × price_per_player` (the full potential revenue).
`payout_type` is **snapshotted** at creation — flipping trust later does not retro-change it.

---

## 5. Instant payout — end-to-end sequence

```mermaid
sequenceDiagram
    actor O as Trusted Organizer
    participant API as Backend API
    participant DB as Postgres (RPC fns)
    participant Cron as Cron (every 15 min)
    actor P as Player
    participant PL as platform_ledger

    O->>API: POST /games
    API->>DB: is_organizer && is_trusted → payout_type='instant'
    Note over DB: advance_paid_at = NULL, advance_settled_at = NULL

    rect rgb(224,242,255)
    Note over Cron,DB: ADVANCE #1 — at LEAST(created_at + X days, start_datetime)
    Cron->>DB: fn_pay_all_due_advances()
    DB->>DB: credit organizer earnings_balance (60% or 100% of capacity)
    DB->>PL: 'advance' row (− amount)
    DB->>DB: stamp games.advance_paid_at, advance_amount
    DB-->>O: notify "SGD X paid in advance — ready to cash out"
    end

    rect rgb(230,247,230)
    Note over P,PL: PLAYERS BOOK (anytime, before or after advance)
    P->>API: POST /bookings
    API->>DB: fn_debit_wallet (player balance) — unchanged
    API->>PL: fn_recoup_booking_payment → 'recoupment' row (+ amount)
    Note over API: organizer wallet NOT touched (already paid)
    end

    rect rgb(255,248,225)
    Note over Cron,DB: ADVANCE #2 — percentage mode only, at end_datetime
    Cron->>DB: fn_settle_instant_game()
    DB->>DB: credit organizer earnings_balance (remaining 40% of capacity)
    DB->>PL: 'advance' row (− remaining)
    DB->>DB: stamp games.advance_settled_at
    end

    O->>API: POST /wallet/cash-out (existing flow)
    API->>DB: fn_cash_out_earnings → 'payout' row (admin disburses)
```

> **"Instant" = the advance lands in the organizer's cashable `earnings_balance`
> immediately at the cutoff** (instead of waiting for the game to end). Actual
> money-to-bank still runs through the existing cash-out → `admin/payouts` disbursement.
> Fully-automated bank disbursement (HitPay Payouts API) is a separate future item.

---

## 6. Game payout lifecycle (state)

```mermaid
stateDiagram-v2
    [*] --> Created: POST /games

    Created --> Escrow: not trusted → payout_type='escrow'
    Created --> InstantPending: trusted → payout_type='instant'

    InstantPending --> Advanced: cron cutoff — credit 60% or 100%
    Advanced --> Settled: game end — credit remaining 40% (percentage)
    Advanced --> Settled: full mode — no-op

    Escrow --> Matured: game end — pending becomes cashable
    Matured --> [*]
    Settled --> [*]
```

---

## 7. Worked example (percentage mode, 60/40)

Capacity: `10 slots × SGD 10 = SGD 100`. Organizer config: `advance: percentage, 60%`.

| Event | Organizer `earnings_balance` | `platform_ledger` | Platform net (this game) |
|---|---|---|---|
| Advance #1 (cutoff) | +60 | −60 (advance) | −60 |
| Player books 7 seats | — | +70 (recoupment) | +10 |
| Advance #2 (game end) | +40 | −40 (advance) | −30 |
| **Totals** | **100** | **−30** | **−30 (loss, under-booked)** |

If the game sells out (10 × 10 = 100 recouped) the platform nets **0**. The platform
absorbs the shortfall on under-booked games — this is the risk it takes on for trusted
organizers, settled/monitored via `platform_ledger`.

---

## 8. Edge cases & policy

| Case | Behaviour |
|---|---|
| **Player cancels** (game still on) | Player refunded **from the platform** (`platform_ledger` 'refund', −). Organizer keeps the advance. Platform absorbs. No clawback. |
| **Organizer cancels the whole game** *after* advance paid | Platform **keeps** all recouped player money, refunds players from platform funds, and **settles the advance with the organizer offline** (manual). No automated clawback. Repeat offenders → future blacklist. |
| **Organizer cancels** *before* advance paid | No advance yet → nothing to reverse; normal cancellation. |
| **Under-booking** | Platform absorbs the gap (see §7). Visible as negative net in `platform_ledger`. |
| **Trust revoked after creation** | In-flight games keep `payout_type` from their snapshot; only *new* games become escrow. |
| **Offline ('pay organiser') bookings** | Unchanged — no money moves through Activ8, no recoupment row. |

---

## 9. Backend implementation checklist

**Migrations** (`backend/migrations/`)
- `102_add_config_to_profiles.sql` — `config JSONB NOT NULL DEFAULT '{}'` (+ rollback).
- `103_add_payout_type_to_games.sql` — `payout_type`, `advance_paid_at`, `advance_amount`, `advance_settled_at` + partial index (+ rollback).
- `104_create_platform_ledger.sql` — table, indexes, service-only RLS (+ rollback).
- `105_instant_payout_functions.sql` — `fn_pay_game_advance`, `fn_pay_all_due_advances`, `fn_recoup_booking_payment`, `fn_settle_instant_game`; extend `fn_cancel_game_refunds` for the instant path; add `'game_advance'` to the `wallet_transactions` type check (+ rollback).

**Services**
- `services/wallet-ops.service.js` — thin wrappers: `payGameAdvance`, `recoupBookingPayment`, `settleInstantGame`.
- `plugins/games/service.js`
  - `createGame`: stamp `payout_type = (is_organizer && is_trusted) ? 'instant' : 'escrow'`.
  - `createBooking`: for `payout_type='instant'`, call `recoupBookingPayment` **instead of** `creditPendingEarning`; player debit unchanged.
  - cancellation/refund paths: route instant-game refunds through the platform.

**Cron** (`backend/src/app.js`)
- Extend the 15-min job: pay due advances (`fn_pay_all_due_advances`, notify) and, at game end, top up percentage-mode remainders (`fn_settle_instant_game`). Escrow settlement untouched.

**Admin**
- Endpoint to set `is_trusted` and write `config.payout`.
- Surface `platform_ledger` net exposure (per game + aggregate) in the admin payouts view.

**Flutter** — **no changes.**

---

## 10. Settled decisions

1. **Percentage split** = *X% of capacity now, remaining (100−X)% of capacity at game end* — a fixed split on capacity, **not** tied to how many players actually booked.
2. **Whole-game cancellation after advance** = platform keeps the money, **settles with the organizer offline**; no automated clawback (future: blacklist).
3. **Player refunds** = platform absorbs.
4. **Timing** = `LEAST(created_at + X days, start_datetime)`.
5. **Scope** = automatic for every game by a trusted organizer.
6. **Frontend** = unchanged.
