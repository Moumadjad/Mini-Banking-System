# Mini Banking System

## Part 1: Transaction Management

### Scenario

Two transfers happen at the same time and both touch Account A.

- T1: transfer $200 from A to B
- T2: transfer $150 from C to A

Both transactions need to read and then update the balance of A.

### Concurrency issue

This is a lost update problem.

Account A starts at $1000.

1. T1 reads A = 1000
2. T2 reads A = 1000 (before T1 has written anything back)
3. T1 computes 1000 - 200 = 800 and writes A = 800
4. T2 computes 1000 + 150 = 1150 and writes A = 1150

Final balance is 1150, but it should be 950. T1's update got overwritten and just disappeared. That's the lost update.

### Locking mechanism

Use an exclusive lock on the row while a transaction is doing its read-modify-write, something like `SELECT balance FROM accounts WHERE id = A FOR UPDATE`. The other transaction has to wait until the lock is released (commit or rollback) before it can touch the same row. A shared lock wouldn't work here because both transactions need to write, not just read.

### Pessimistic or optimistic?

Pessimistic locking makes more sense for a banking system. Money transfers need to be correct every time, and if two transactions collide on the same account it's better to just make one wait than to let both proceed and risk losing an update. Optimistic locking (checking a version number and retrying on conflict) is fine for low-contention stuff like updating a profile, but for account balances, where conflicts on the same row are actually expected, pessimistic locking avoids retry loops and gives predictable behavior, which matters more for audits and correctness.

### Schedule table

Without locking (unsafe):

| Step | T1 | T2 | A |
|---|---|---|---|
| 1 | read A (1000) | | 1000 |
| 2 | | read A (1000) | 1000 |
| 3 | write A = 800 | | 800 |
| 4 | | write A = 1150 | 1150 |
| 5 | commit | | |
| 6 | | commit | wrong, should be 950 |

With locking (safe):

| Step | T1 | T2 | A |
|---|---|---|---|
| 1 | lock A | | 1000 |
| 2 | read A (1000) | waiting | 1000 |
| 3 | write A = 800, commit, unlock | waiting | 800 |
| 4 | | lock A, read A (800) | 800 |
| 5 | | write A = 950, commit, unlock | 950, correct |

## Part 2: Distributed Database

Branches: Tunis, Sousse, Sfax

### Horizontal fragmentation

Split the Customers table by branch_id, one fragment per branch (Customers_Tunis, Customers_Sousse, Customers_Sfax). Each branch mainly deals with its own customers so this keeps queries local and fast. If a customer from Tunis shows up in Sfax, that branch can still query the Tunis fragment or go through a small directory service that maps customer_id to home branch.

### Vertical fragmentation

Move things like email, phone number, and login info into a separate table (Customer_Contact), keeping name/id/branch/account in the main Customers table. This data isn't needed for most day to day operations and it's more sensitive, so separating it makes sense both for performance and access control.

### What to replicate

- Customer info (name, id, branch): replicate everywhere. Any branch might need to look up a customer quickly, and this data barely changes.
- Account balances: don't replicate, keep it at the owning branch only. Balances change all the time and need to stay consistent, replicating them everywhere would just create the same lost-update problem but across branches.
- Transaction history: keep the full detail at the branch where it happened, but replicate a summarized version centrally for reporting/audits. Since transactions are never modified after being written, this replication is cheap and safe.

### Static or dynamic allocation for transaction history

Static. Each transaction record stays permanently at the branch where it was created. It's write-once, read-many data, there's no real reason to move it around dynamically like you might with mutable data. Static allocation also keeps the audit trail simple, every transaction has one clear origin.
