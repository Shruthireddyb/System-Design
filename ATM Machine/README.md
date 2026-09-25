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

## 3. Design Patterns Used
- **Singleton:** ATM instance - only 1 ATM
- **State:** ATM flow changes state
- **Factory:** TransactionFactory creates TransactionType
- **Strategy:** Cash dispense algorithm (minimum notes)
- **Observer:** SMS notification after transaction
- **Decorator:** Receipt printer

## 4. Flow
1. IdleState -> HasCardState (insert card)
2. HasCardState -> authenticate via BankService -> SelectOperationState
3. User selects: BALANCE_CHECK / WITHDRAW / DEPOSIT
4. WITHDRAW: check balance -> canDispense? -> dispense -> update account -> SUCCESS
5. Failure: Insufficient funds / cash -> rollback + FAILED status

## 5. Concurrency
- `synchronized` on Account.withdraw() - prevents double withdraw
- One ATM lock - only one user session active
- Idempotency: txnId for each transaction
