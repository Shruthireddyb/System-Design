# ATM Machine - LLD 

## 1. Requirements
**Functional:**
- Insert card, enter PIN, authenticate
- Operations: Balance Check, Cash Withdrawal, Deposit
- Dispense cash with minimum notes (2000, 500, 100)
- Print receipt / SMS notification
- Block card after 3 wrong PIN attempts

**Non-Functional:**
- One user per ATM at a time (thread-safe)
- Low latency, handle cash shortage, network failure
- Secure PIN handling

## 2. Classes (LLD)

**User:** card, pin
**ATM:** atmId, location, cashDispenser, cardReader, keypad, screen, bankService
    - Methods: authenticateUser(), executeTransaction()
**BankService:** authenticate(), getBalance(), withdraw(), deposit()
**Bank:** Map<cardNo, Account>
**Account:** accNo, balance, synchronized withdraw/deposit
**Card:** cardNo, linkedAccount, isBlocked
**CashDispenser:** Map<Denomination, Count> + dispenseCash(amount) [Strategy Pattern - Greedy]
**Transaction:** id, type, amount, status
**ATMState (State Pattern):** IdleState, HasCardState, SelectOperationState, CheckBalanceState, CashWithdrawalState, DepositState

## 3. Design Decisions

| Decision Area | Decision Taken | Reason | Alternative Rejected |
| :--- | :--- | :--- | :--- |
| **Architecture** | 3-Layer - Presentation + Business Logic + Data Access | Simple, Easy to test, Good for ATM single machine | Microservices - Overkill for single ATM |
| **Language** | Java + Spring Boot | Strong OOP, Banking uses Java, Thread safe | Python - Slower, Less used in banking |
| **Database** | PostgreSQL - Relational DB | ACID needed for money, Strong consistency | NoSQL / MongoDB - No ACID, Risk of money loss |
| **Caching** | No Cache for Balance | Balance must be real-time from DB, Cache can show wrong balance | Redis - Dangerous for money, Stale data risk |
| **Authentication** | Card Number + PIN (4-digit), PIN hashed with BCrypt | PIN never stored in plain text, Secure | OTP Only - Slower, Needs network |
| **PIN Attempts** | Max 3 attempts, Then card blocked for 24 hours | Prevents brute force attack | Unlimited attempts - Security risk |
| **Session** | Session timeout after 30 seconds idle | Prevents next user accessing previous user account | No timeout - Security risk |
| **Cash Dispenser** | Greedy Algorithm for denominations | Gives minimum notes, Fast, Simple | DP Algorithm - Slower, Not needed |
| **Transaction** | DB Transaction - @Transactional, 2-Phase Commit | If cash dispense fails, money must be rolled back | No transaction - Money deducted but cash not given |
| **Concurrency** | DB Row-Level Lock on Account + Pessimistic Lock | Prevents double withdrawal at same time from 2 ATMs | Optimistic Lock - Risk of race condition |
| **Communication** | ATM -> Bank Server via HTTPS / TLS 1.3 + ISO 8583 Protocol | Banking standard, Encrypted | Plain HTTP - Insecure |
| **Logging** | Every transaction logged in Transaction Table + Audit Log | For dispute, For mini statement, For bank audit | No log - Cannot resolve customer complaints |
| **Receipt** | Print + SMS + Email | User proof, Bank proof | Print only - Paper lost = No proof |
| **Cash Inventory** | ATM checks cash available before withdrawal | Avoid partial dispense | No check - ATM tries to dispense but fails mid-way |
| **Card Reader** | Chip + Magnetic Stripe, Read Card Number Only | Card number is key, No sensitive data on card | Store CVV on ATM - Security violation |
| **Scalability** | Stateless Bank Server + Load Balancer | 1000s of ATMs can connect, Horizontal scaling | Stateful Server - Single point of failure |
| **Failure Handling** | If network fails, Transaction rolled back, Show error | No money loss, User can retry | Allow offline withdrawal - Double spend risk |
| **Denominations** | 2000, 500, 200, 100 - Priority to higher notes first | Less notes to dispense, Faster | Random - More notes, Slower |
