# Occupify Payment - API Specification

For business and system error codes, refer to [error_code.md](error_code.md).

> [!NOTE]
> **Base Port & Gateway Routing:**
> - Direct Service Port: `8183` (Internal)
> - Gateway Port: `8180` (External entry point)
> - Gateway Routes: `/contracts/**`, `/wallets/**`, `/payments/**`
> - Database Port: PostgreSQL `5532` (`occupify_payment`)
> - Status: *Scaffolding / Architecture Specification Phase*

---

## 1. Domain Responsibilities

The **Payment Service** (`occupify-payment`) governs the financial and legal transactions of the Occupify marketplace:
1. **Contract Management (`/contracts/**`):** Creation, digital signing, milestones, terms agreement between client and freelancer.
2. **Escrow Management:** Holding client funds in escrow until milestones or deliverables are approved.
3. **Wallet Management (`/wallets/**`):** User balance tracking, credit, debit, withdrawal requests.
4. **Payment Processing (`/payments/**`):** Integration with payment gateways (e.g., VNPay, Stripe), transaction histories, invoice generation.

---

## 2. Planned API Endpoints

### 2.1 Contracts (`/contracts`)
| Method | Endpoint | Authorization | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/contracts` | User (`@Authenticated`) | Initiates a draft contract from an accepted project bid or offer. |
| `GET` | `/contracts/{id}` | User / Admin | Retrieves contract details, milestone schedule, and signing state. |
| `PATCH`| `/contracts/{id}/sign` | User | Submits digital signature or acceptance of contract terms. |
| `POST` | `/contracts/{id}/milestones/{milestoneId}/release` | User (Employer) | Releases escrow payment to freelancer upon milestone completion. |

### 2.2 Wallets (`/wallets`)
| Method | Endpoint | Authorization | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/wallets/me` | User (`@Authenticated`) | Retrieves authenticated user's wallet balance, pending funds, and currency. |
| `GET` | `/wallets/me/transactions` | User | Retrieves paginated ledger transaction history. |
| `POST`| `/wallets/me/withdraw` | User | Requests a withdrawal to a linked bank account. |

### 2.3 Payments & Top-up (`/payments`)
| Method | Endpoint | Authorization | Description |
| :--- | :--- | :--- | :--- |
| `POST` | `/payments/checkout` | User | Creates a payment checkout session with external gateway. |
| `POST` | `/payments/webhook` | Public (Signed) | Receives signed transaction callbacks from payment gateway provider. |
