# Escrow Payment Flow (Current System)

> **Status:** As-built — reflects the code on `backend` today.
> **Model:** *Escrow-until-game-ends*. The platform never fronts money; a player pays,
> the organizer's earning is held in escrow until the game finishes, then becomes cashable.
> This is the default flow for **every organizer** and the counterpart to
> [Instant-Payout-Payment-Flow.md](./Instant-Payout-Payment-Flow.md) (trusted organizers).
>
> Diagrams are [Mermaid](https://mermaid.js.org/) (render in GitHub / VS Code).

---

## 1. The money model

```
top-up ──▶ balance ──(book)──▶ organizer pending_earnings ──(game ends)──▶ earnings_balance ──▶ payout
 (HitPay)  (spendable)          (escrow, not cashable)        (cron)        (cashable)          (admin)
```

Real cash always sits in the platform's payment processor (HitPay) / bank. The wallet
tables are an **internal liability ledger** — booking a game just moves a liability from
"owed to player" to "owed to organizer". `wallet_transactions` is append-only and is the
source of truth; the balance columns are atomically-maintained caches.

### Wallet buckets (`wallets` row per user)

| Bucket | Column | Meaning | Cashable? |
|---|---|---|---|
| Spendable | `balance` | Top-ups, refunds, redeemed points — used to book | No, spend-only |
| Pending earnings | `pending_earnings` | Organizer earnings held in escrow until game ends | No |
| Earnings | `earnings_balance` | Matured earnings, ready to cash out | Yes |

---

## 2. Top-up (HitPay)

```mermaid
sequenceDiagram
    actor P as Player
    participant API as Backend (wallet)
    participant HP as HitPay
    participant DB as Postgres

    P->>API: POST /wallet/top-up { amount }
    API->>HP: createPaymentRequest()
    HP-->>API: { id, url }
    API->>DB: insert hitpay_payments (status 'pending')
    API-->>P: { paymentUrl }
    P->>HP: pays on hosted page
    HP-->>API: POST /wallet/top-up/webhook (status 'completed')
    API->>DB: guard — already wallet_credited? → stop (idempotent)
    API->>DB: fn_credit_wallet → balance += amount ('top_up' tx)
    API->>DB: hitpay_payments.wallet_credited = true
```

- Idempotent on the `wallet_credited` flag — duplicate webhooks never double-credit.
- Legacy `POST /wallet/top-up` (dev/manual) credits directly via `fn_credit_wallet`.
- `redeemPoints` is a sibling: deduct points → `fn_credit_wallet` (`points_redemption`).

---

## 3. Booking — `createBooking`

```mermaid
flowchart TD
    A["POST /bookings { gameId, numSpots, paymentMethod }"] --> B{"Validations"}
    B -->|"game not published / own game /<br/>no slots / already booked"| X["Reject"]
    B -->|ok| C["totalAmount = price × numSpots"]
    C --> D{"payment path"}

    D -->|"totalAmount = 0"| E["FREE<br/>payment_status='free', method=null<br/>no debit, no earning"]
    D -->|"paymentMethod = 'offline'"| F["OFFLINE (pay organiser)<br/>payment_status='unpaid', method='offline'<br/>no debit, no earning — organiser confirms later"]
    D -->|"wallet (default)"| G["WALLET<br/>fn_debit_wallet (player balance)<br/>payment_status='paid', method='wallet'"]

    E --> H["insert booking (confirmed)<br/>+ game_participants + points + referral"]
    F --> H
    G --> H
    G --> I["fn_credit_pending_earning<br/>organizer pending_earnings += total"]
    I --> H
    H --> J["Telegram slot update"]
```

Key rules enforced before charging: game `published`, not the organizer's own game, slot
availability (public / community / hybrid pools), no duplicate confirmed booking, ≤10 spots.
If the booking `insert` fails **after** a wallet debit, the debit is auto-refunded.

**Only `wallet`-paid bookings move money through Activ8**, so only those credit the
organizer's escrow. Offline and free bookings credit nothing.

---

## 4. Escrow earnings lifecycle

```mermaid
stateDiagram-v2
    [*] --> Pending: booking paid (fn_credit_pending_earning)

    Pending --> Matured: game ends + booking still confirmed (cron settles)
    Pending --> Reversed: booking cancelled (non-refund path)
    Pending --> Resolved: refund request decided (kept portion → cashable)

    Matured --> CashedOut: organizer cash-out (fn_cash_out_earnings)
    CashedOut --> Disbursed: admin processes payout

    Reversed --> [*]
    Resolved --> [*]
    Disbursed --> [*]
```

- **Settlement** runs every 15 min (`fn_settle_all_matured_earnings`) and lazily on any
  earnings read (`fn_settle_matured_earnings`). It sweeps `game_earning` rows that are
  still `pending`, whose booking is `confirmed`, and whose `end_datetime < NOW()` —
  moving the amount `pending_earnings → earnings_balance`, then notifies the organizer.
- The cron runs **auto-approve of expired refund requests first**, so contested bookings
  are reversed, not matured.

---

## 5. Player cancels — refund-request lifecycle

```mermaid
sequenceDiagram
    actor P as Player
    participant API as Backend (games)
    participant DB as Postgres
    actor O as Organizer
    participant Cron as Cron

    P->>API: DELETE /bookings/:id (reason?)
    API->>DB: guard — booking confirmed & game not started
    alt wallet-paid (refund eligible)
        API->>DB: booking → cancelled, refund_status='pending'
        Note over DB: earning STAYS in pending_earnings (escrowed)
        API-->>O: notify "Refund requested"
    else offline / free (not eligible)
        API->>DB: booking → cancelled, refund_status='none'
        API->>DB: fn_reverse_pending_earning (claw back escrow)
    end
    API->>DB: remove participant, reverse booking points

    Note over O,Cron: Resolution — organizer decides, or cron auto-approves
    alt organizer decides
        O->>API: decideRefund(amount 0..total)
    else grace hours elapsed after game end
        Cron->>DB: fn_auto_approve_expired_refunds (full refund)
    end
    API->>DB: fn_resolve_refund_request
    Note over DB: refund → player balance (+)<br/>full pending earning reversed<br/>kept portion → organizer earnings_balance (cashable)
```

`refund_status`: `none` → `pending` → `approved` / `denied`.
`fn_resolve_refund_request` clamps the refund to `[0, total]`: the **refunded** part goes
back to the player's `balance`, the **kept** part (`total − refund`) is credited to the
organizer's cashable `earnings_balance`, and the original pending earning is reversed —
so the kept-vs-refunded split is decided atomically in one place.

---

## 6. Organizer cancels the whole game

```mermaid
flowchart LR
    A["DELETE /games/:id"] --> B["fn_cancel_game_refunds<br/>(one transaction)"]
    B --> C["each confirmed booking:<br/>refund total → player balance"]
    B --> D["each booking:<br/>reverse organizer earning (pending/matured)"]
    B --> E["reverse booking points"]
    B --> F["booking → cancelled, payment_status='refunded'"]
    B --> G["delete all participants"]
    B --> H["game → cancelled (soft delete)"]
```

Full refunds to every player and a full reversal of the organizer's earnings, atomically.

---

## 7. Cash-out & disbursement

```mermaid
sequenceDiagram
    actor O as Organizer
    participant API as Backend (wallet)
    participant DB as Postgres
    actor A as Admin

    O->>API: POST /wallet/cash-out { amount? }
    API->>DB: fn_settle_matured_earnings (sweep first)
    API->>DB: fn_cash_out_earnings
    Note over DB: earnings_balance -= amount<br/>'payout' tx status 'pending'
    API-->>O: { status: 'pending' }
    A->>API: POST /admin/payouts/:organizerId/process
    API->>DB: flip 'payout' tx → completed
    Note over A: actual bank/PayNow transfer done off-app
```

`earnings_balance` is debited at cash-out time; admin processing only flips the ledger
status. Real money-out (bank / PayNow) is manual.

---

## 8. Cron responsibilities (`app.js`, every 15 min)

1. **Auto-approve expired refunds** (`fn_auto_approve_expired_refunds`) — full refund for
   pending requests the organizer never decided, once the game has been over for
   `refund_request_grace_hours`. Runs **before** settlement.
2. **Settle matured earnings** (`fn_settle_all_matured_earnings`) — `pending → cashable`
   for finished games, then notify each organizer "earnings available".

(Daily jobs prune old notifications — not payment-related.)

---

## 9. Payment-method matrix

| Method | Player debit | `payment_status` | Organizer earning | Refund on cancel |
|---|---|---|---|---|
| **wallet** (default) | Yes (`balance`) | `paid` | `pending_earnings` (escrow) | Refund request → resolved |
| **offline** (pay organiser) | No | `unpaid` | None | None (seat freed) |
| **free** (price 0) | No | `free` | None | None (seat freed) |

---

## 10. Ledger transaction types (`wallet_transactions.type`)

| Type | Bucket touched | When |
|---|---|---|
| `top_up` | `balance` +| HitPay / manual top-up |
| `booking_payment` | `balance` − | Player books (wallet) |
| `refund` | `balance` + | Cancellation / failed booking / game cancelled |
| `points_redemption` | `balance` + | Redeem points for credit |
| `game_earning` | `pending_earnings` + (then `earnings_balance` on maturity) | Player books a paid game |
| `earning_reversal` | reverses `game_earning` | Booking/game cancelled |
| `payout` | `earnings_balance` − | Cash-out (awaiting disbursement) |
