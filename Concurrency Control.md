# Introduction

Any DBMS stores data using two essential files:
1. **Master Data File** (MDF): Actual Database that contains: Tables, Records and Indexes. This is what users think the database is.
2. **Log Data File** (LDF): Stores everything transactions do, Every change is written here first. Transactions never modify MDF directly.
    - Start Transaction
    - Read / Write Operations
    - Old Value & New Value
    - Commit / Abort

What happens when a transaction executes?
1.  **Transaction starts**: Operations are written to LDF first this is called **Write-Ahead Logging** (WAL)
2. **Commit happens**: Only after COMMIT changes are confirmed and moved to MDF.

- How to guarantee database consistency?
    -  If two transactions run together → **Concurrency Control**, It prevents Dirty Read, Lost Update, Incorrect Summary, ...etc
- What if some changes are written and others aren’t?
    - Without LDF → **database corruption** as LDF keeps full history → DB can decide: undo? redo?
- What if a failure happens mid-execution?
     -   DB uses **Recovery Techniques**: 
        -  Undo uncommitted transactions
        - Redo committed ones
- Recovery depends on LDF. If system crashes: 
    - DB reads **LDF** and determines:
        -  Transactions with COMMIT → **REDO**
        - Transactions without COMMIT → **UNDO**
    MDF alone is never trusted after failure, DF is the source of truth.

- Why are transaction updates written to the Log Data File before the Master Data File?
    - Transaction updates are written to the Log Data File to record all actions of a transaction from start to commit, so that atomicity and consistency are guaranteed, and the database can recover from failures using undo and redo operations.

- Due to Concurrency problems (lost update, dirt read, ...etc) before writing a data item, a transaction must lock it to prevent concurrent writes, similar to file locking, allowing multiple reads but only one write to ensure isolation and database consistency.

- Two operations in a schedule `S` are conflicting if and only if
    1. Belong to different transactions
    2. Access (read / write) the same data item
    3. At least one of them is WRITE operation
    Example: `W1(X) R2(X)` ➔ Conflict Operations, `R1(X) W2(Y)` ➔ are not conflict operations.

- Purpose of Concurrency Control:
    - To enforce **Isolation** (through mutual exclusion) among conflicting transactions.
    - To Protect **Database Consistency** through consistency preserving execution of transactions.
    - To enforce **Serializability**.

# Concurrency Control Techniques

## Locking

A **lock** is a flag that is associated with a data item. A data item could be a cell, row, column, table, or a whole database.

- Any serializable schedule is acceptable.

### Binary Locks

- It has Two Operations `Lock_item()` and `Unlock_item()`, `Lock_item(X) = L(X)` and `Unlock_item(X) = U(X)`
- `T1` issues `Lock_item(X)` = Data item `X` is locked to transaction `T1`
    - Only operations in `T1` can read and/or write item `X` until it unlocks that item.
- `T1` issues `Unlock_item(x)` = Data item `X` is released.
    - Any other transaction can lock `X`, and use `X`.
- Before `R(X)` and/or `W(X)` → `Lock_Item(X)`
- After All `R(X)` and/or `W(X)` → `Unlock_Item(X)`
- If `X` is already locked by Transaction `Ti` → No need to lock it again.
- If `X` is not locked by a transaction `Ti` → No need to `Unlock_Item(X)`

- Without locking, concurrent transactions may produce a non-serializable schedule, but using locking forces a safe order of conflicting operations, making the schedule serializable.
- Locking enforces an order on conflicting operations.

- Why is Binary Locking not practical?
    - It is OK for many transactions to access the same data item, if all the operations are READ operations. binary locking does NOT allow that.
    - Binary lock = **all or nothing**
    - If `T1` locks `X` for reading: `T2` cannot even read `X`
    - This causes: Unnecessary blocking and very low concurrency
    **Shared / Exclusive Locks** fixes this problem.
- Binary locking is just a **teaching step**, not used in real DBMSs.
#### Implementing Binary Locking Protocol

- We have a **lock table** with three columns:
    -  Data Item, LOCK (Boolean), Transaction ID.
    - A queue of waiting transactions.
- A **lock manager** manages the table and grants access to data items to locking transactions.
     - Checks who owns locks
     - Grants or denies lock requests
     - Puts transactions in a waiting queue if needed
- How a request is handled
    1. Transaction `T1` requests `Lock(X)`
    2. Lock Manager checks Lock Table
    3. If `X` is free:
        - Grant lock and update table
    4. If `X` is locked:
        - Block `T1` and put it in the queue

### Shared/Exclusive Locks

- It has three operations: 
    -  **`Read_Lock(x)` / Shared Lock / SL**: Many different transactions can have a read lock on the same data item at the same time.
    - **`Write_lock(x)`/ Exclusive Lock / XL**: Only one transaction can have a write lock on the same data item at the same time.
    - **`Unlock(x)`/UL**: Release the data item once finished accessing it, even before the transaction finishes.
- **Shared Lock**: Many Transactions Can READ the same data item but none of them can WRITE.
- **Exclusive Lock**: Only one transaction can READ and WRITE the data item.

- Shared/Exclusive Locking Protocol:
    1. Before `R(X)` → `Shared_Lock(X)`
    2. Before `W(X)` → `Exclusive_lock(X)`
    3. After All `R(X)` and/or `W(X)` → `Unlock(X)`
    4. If `X` is already locked for `T` → No need to lock it again but may upgrade (from Read ➔ Write) or downgrade (from Write ➔ Read). (**Lock Conversion**)
    5. If `X` is not locked for T → No need to `Unlock(x)`.

- Lock-compatibility Matrix
![Pasted image 20260109024037.png](./Imgs/Pasted%20image%2020260109024037.png)

- S + S → **true**
    - Many transactions can read the same item
    - No data change → safe
    **Multiple readers allowed**
- S + X → **false**
    - Someone is reading
    - Another wants to write
    - Writing may change what is being read
    **Writer must wait**
- X + S → **false**
    - Someone is writing
    - Another wants to read
    - Reader may see **dirty data**
    **Reader must wait**
-  X + X → **false** 
    - Two writers = **lost update**
    - Data corruption
    **Only one writer allowed**

- Does locking really guarantee serializability?
    -  Locking alone does not guarantee serializability; serializability is guaranteed only if transactions follow the **Two-Phase Locking** (2PL) protocol.
    - Once a transaction releases a lock, it must not acquire any new locks

#### Locks Conversion

A transaction that already holds a lock on item X is allowed under certain conditions to convert the lock from one locked state to another.

- Read ➔ Write. (**Lock upgrading**)
- Write ➔ Read. (**Lock downgrading**)

#### Implementation

- We have a **Lock Table** With Four Columns:
    -  Data Item, LOCK, ReadsCount, Transactions IDs
    - A queue of waiting transactions
- A lock manager that the table and grants access to data items to locking transactions
- Step-by-step example:
    1. `T1` requests `S(X)` -> Granted, Lock = S, ReadsCount = 1
    2. `T2` requests `S(X)` -> Granted, Lock = S, ReadsCount = 2
    3. `T3` requests `X(X)` -> Not compatible (Because there are multiple transactions reading)  ➜ `T3` goes to Queue
    4. `T1` and `T2` unlock X , ReadsCount → 0
    5. `T3` gets`X(X)`  -> Granted

- Why do we need ReadsCount?
    -  Binary locking was too strict
    - Shared locks allow high concurrency
    - ReadsCount tracks how many readers exist
    Without ReadsCount → cannot support multiple readers safely.

### Two-Phase Locking Protocol (2PL)

All locking operations (`read_lock()`, `write_lock()`) precede the first unlock operation in the transaction. **CAUTION**: ONCE STARTED UNLOCKING, YOU CANNOT AND IT IS NOT ALLOWED TO REQUEST A LOCK → This is the most important principal in 2PL.

- If every transaction in a schedule follows the two-phase locking protocol 2PL, the schedule is guaranteed to maintain serializability.
- A transaction cannot read a value that is later changed by another transaction in the same schedule.
- 2PL guarantees serializability but does NOT guarantee deadlock freedom.
- Why does basic 2PL guarantee serializability but may cause deadlock?
    - Because transactions may hold locks while waiting for other locks, creating circular wait conditions.
- Two Phases:
    1. **Growing phase**: Locks Growing
         - transaction may obtain locks
         - transaction may not release locks
    2. **Shrinking phase**: Locks Shrinking
        - transaction may release locks
        - transaction may not obtain locks
![Pasted image 20260109033213.png](./Imgs/Pasted%20image%2020260109033213.png)

- This is the Basic 2PL but there are other versions that solve the deadlock problem.

#### Basic 2PL

- Guarantees Serializability as it 2PL Algorithm but not deadlock free as locks/unlocks and operations are interleaving. 
- Does any schedules which follows basic 2PL is guaranteed to be {Recoverable, Cascadeless, or Strict}?
    - No, Because {Recoverable, Cascadeless, or Strict} schedules’ checks, search for either Commit or Abort of the participating and with conflict operations transactions, But Basic 2PL is not conditioning neither of commit nor abort. So no guarantee
#### Conservative/Static 2PL

A transaction should lock all the items it accesses before the transaction begins execution. All Locks Before {Execution}

- Guarantees Serializability as it 2PL Algorithm.
- Deadlock free, as each transaction acquires all of its locks before getting started, so no probability of deadlocks.
- Growing Phase -> All Locks then Execution of Operations then Shrinking Phase -> Unlocking
- Does any schedules which follows static 2PL is guaranteed to be {Recoverable, Cascadeless, or Strict}?
    - No, Because {Recoverable, Cascadeless, or Strict} schedules’ checks, search for either Commit or Abort of the participating and with conflict operations transactions, But Basic 2PL is not conditioning neither of commit nor abort. So no guarantee

#### Strict 2PL

**Strict 2PL** does not allow unlock of exclusive locks unless a commit or abort happens.

- Guarantees Serializability & Not deadlock free.
- for each W → R && W → W for different transactions on the same data item, a commit should appear between both.
- Does any schedules which follows Strict 2PL is guaranteed to be {Recoverable, Cascadeless, or Strict}?
    - yes, To unlock W then transaction should be committed or aborted. This means any coming read or write of another transaction accessing the same data item should acquire lock on this item, can not be acquired unless the first transaction committed or aborted.

#### Rigorous 2PL

A transaction does NOT release any lock (shared or exclusive) until it commits or aborts.

- **Strict 2PL** → holds X (write) locks till commit/abort
- **Rigorous 2PL** → holds ALL locks (S + X) till commit/abort
- Rigorous 2PL = Strict 2PL + shared locks held till C/A
- Guarantees Serializability & Not deadlock free
- Does any schedules which follows Rigorous 2PL is guaranteed to be {Recoverable, Cascadeless, or Strict}?
    - Yes, As strict 2PL Guarantees, then rigorous MUST.

#### Summary

| **Protocol**         | **Serializable** | **Recoverable** | **Cascadeless** | **Strict** | **Deadlock-free** |
| -------------------- | ---------------- | --------------- | --------------- | ---------- | ----------------- |
| **Basic 2PL**        | ✅                | ❌               | ❌               | ❌          | ❌                 |
| **Strict 2PL**       | ✅                | ✅               | ✅               | ✅          | ❌                 |
| **Rigorous 2PL**     | ✅                | ✅               | ✅               | ✅          | ❌                 |
| **Conservative 2PL** | ✅                | ❌               | ❌               | ❌          | ✅                 |

- What is the most practical 2PL Algorithm?
    - Strict 2PL, Because it has best balance between Correctness, Recovery simplicity and Reasonable concurrency.
    - Not Rigorous, because Rigorous is too restrictive, It delays even harmless reads and Lower concurrency.
    That’s why real DBMSs use Strict 2PL, not Basic and not Rigorous.

Rigorous 2PL ⊂ Strict 2PL ⊂ Basic 2PL, while Conservative 2PL is a subset of Basic 2PL and is the only deadlock-free variant.

- Concurrency Control Problems Solved by each:

| **2PL Variant**  | **Lost Update** | **Unrepeatable Read** | **Dirty Read** | **Incorrect Summary** |
| ---------------- | --------------- | --------------------- | -------------- | --------------------- |
| **No 2PL**       | ❌               | ❌                     | ❌              | ❌                     |
| **Basic 2PL**    | ✅               | ✅                     | ❌              | ❌                     |
| **Strict 2PL**   | ✅               | ✅                     | ✅              | ✅                     |
| **Rigorous 2PL** | ✅               | ✅                     | ✅              | ✅                     |
Basic 2PL prevents lost updates and unrepeatable reads but does not prevent dirty reads, while Strict and Rigorous 2PL prevent all concurrency anomalies by holding write locks until commit or abort.


#### 2PL Concurrency Control Technique Pros & Cons:

- Pros:
    - Guarantee Serializability of Schedules
    - Prevents Lost Update
    - Prevents Dirty Read
    - Prevents Incorrect Summary
    - Prevents Unrepeatable Read
    - Prevents Cascade Rollback
- Cons:
    - Might Cause deadlock
    - Might lead to starvation

### Deadlock Problem

**Deadlock** occurs when each transaction T in a set of two or more transactions is waiting for some item that is locked by some other transaction T’ in the set.

- We can prevent deadlocks by:
    - Conservative 2PL
    - Data Item Ordering
    - Timestamp Based: Wait-die and Wound-wait
    - No-Wait
    - Cautious-Wait
- We can detect deadlocks by:
    -  Wait-for graph
    - Timeout
- We can resolve deadlocks by:
    - Victim Selection
- Prevent → Detect → Resolve

#### Deadlock Detection (Wait-For Graph)

- **Nodes** → transactions
- Directed edge `Ti` → `Tj` → `Ti` is waiting for `Tj` to release a lock
- **Deadlock condition**: If the wait-for graph has a cycle → deadlock exists

#### Deadlock Resolution (Victim Selection)

**Victim Selection** is the process of choosing which transactions to abort when a deadlock presents.

- Avoid Old Transactions.
- Avoid Transactions that have performed many updates.
- Select Younger Transactions, ones that most likely performed less updates.

#### Deadlock Detection & Resolution

If a transaction waits for a time longer than a system-defined timeout threshold, the system assumes that the transaction may be deadlocked and aborts it.

- What if a transaction is enforced to abort multiple times and never gets a chance to commit?
    - That is called starvation.

#### Starvation

- When a certain transaction is not able to proceed:
    - Same transaction becomes a deadlock abortion victim many times.
    - The locking queue is not using a fair strategy.
- Solution:
    -  Use a fair waiting scheme, Ex: using a first-come-first-served queue.
    - Give higher priorities to previously aborted transactions.

#### Deadlock Prevention Protocols

##### Data Item Ordering

Data items are ordered, then locks should be granted in the same order.

- How it works?
     1. **Assign Unique IDs**: Give every data item a unique number
     2. **Enforce Order**: A transaction must request locks for resources in ascending order of their assigned numbers.
- Example: `T1` needs `A` and `C`, `T2` needs `B` and `A` 
	- `T1` Locks `A` then `C` and `T2` Locks `A` and `B`
	- Even though `T2` conceptually needs `B` first, it is forced to lock `A` first.
	    1. `T1` locks `A`
	    2. `T2` tries to lock `A` → waits
	    3. `T1` locks `C`, finishes, releases locks
	    4. `T2` gets `A`, then `B`, finishes
	    T2 never holds B while waiting for A, No cycle -> No deadlock
- Pros:
    - Deadlock free
    - Simple idea
- Cons:
    - Low Flexibility
    - Transactions may lock items earlier than needed
    - Reduced concurrency
##### Wait-Die

- Prioritize active transactions

If `Ti` wants to lock item `X` while `Tj` is already locking X then: if Timestamp `TS(Ti)` < Timestamp `TS(Tj)`,  `Ti` is allowed to wait as it is the older transaction else `Ti` die (abort) as it is the younger transaction and restart it later with the same timestamp to avoid starvation.

- Older Transaction Waits, Younger Transactions Dies.

##### Wound Wait

- Prioritize older transactions

If `Ti` wants to lock item `X` while `Tj` is already locking X then: if Timestamp `TS(Ti)` < Timestamp `TS(Tj)`,  `Ti` wounds `Tj` ( `Ti` forces `Tj` to abort) as it is the older transaction and restart `Tj` later with the same timestamp else `Ti`waits as it is the younger transaction.

- Older Transaction Wounds (Aborts Younger), Younger Transaction Waits

##### No-Wait

- ONLY Abort Protocol

If a transaction is unable to obtain a lock, it is immediately aborted and then restarted after a certain time delay without checking whether a deadlock will actually occur or not.

- Transactions abort and restart needlessly

##### Cautious-Wait

No transaction is allowed to wait for another (Waiting) blocked transaction.

- Example: `T1` wants to lock item `X` while `T2` is already locking it, If `T2` is blocked (`T2` is waiting for another item to be released by another transaction). then ➔ `T1` Aborts Else, `T1` Waits.

##### Summary

- Which of the FIVE deadlock prevention protocols/schemes avoid starvation?
    - Only Wait die and Wound Wait.

## Timestamping

- **Timestamp**: A monotonically increasing variable (integer) indicating the age of a transaction. A larger timestamp value indicates a more recent event or operation.
- Timestamp based algorithm uses timestamp to serialize the execution of concurrent transactions.
- `TS(T1)` < `TS(T2)` -> `T1` is Older Than `T2`.
    - If `TS(T1)` < `TS(T2)` Then the only serializable schedule acceptable is the one equivalent to the serial schedule `T1`→`T2`, How?
        -  Order of Conflicting Operations preserves the Timestamp Ordering.
- **Transaction Timestamp** (TS): For each transaction assign a timestamp.
- `read_TS(X)`: The read timestamp of item `X` is the largest timestamp among all the timestamps of transactions that have successfully read item `X`,  `read_TS(X)` = `TS(T)`, where `T` is the youngest transaction that has read `X` successfully.
- `write_TS(X)`: The write timestamp of item `X` is the largest of all the timestamps of transactions that have successfully written item `X`, `write_TS(X)` = `TS(T)`, where `T` is the youngest transaction that has written `X` successfully.

### Timestamp ordering algorithm

If transaction `T` issues `WRITE(X)`

- **Case 1**: If `Read_TS(X)` > `TS(T)` or  `Write_TS(X)` > `TS(T)`, Means:
    -  A younger transaction already read or wrote `X`
    - Letting `T` write now would violate timestamp order
    - So we Abort `T`
- **Case 2**: If `Read_TS(X)` ≤ `TS(T)` and `Write_TS(X)` ≤ `TS(T)`, Then:
    -  Execute the write
    - Set `Write_TS(X)` = `TS(T)`

If transaction `T` issues `READ(X)`

- **Case 1**: If `Write_TS(X)` > `TS(T)`, Means:
    - A younger transaction already wrote `X`
    - `T`is trying to read a value from the “future”
    - So we Abort `T`
- **Case 2**:If `Write_TS(X)` ≤ `TS(T)`, Then:
    -  Execute the read
    - Update `Read_TS(X)` = `max(Read_TS(X), TS(T))` 

 Timestamp Ordering rule: **Old transactions must never see effects of younger ones.**

- Pros:
    - Serializability
    - Deadlock Free
- Cons:
    - Recoverability
    - Cascade Abort
    - Starvation

### Strict Timestamp Ordering

A variation of basic TO called **strict TO** ensures that the schedules are both strict (for easy recoverability) and (conflict) serializable.

- `T` Issues `READ(X)` or `WRITE(X)`
    -  If `Write_TS(X)` < `TS(T)`, Means:
	    -  A younger transaction `T` wants to access X
	    - `X` was last written by an older transaction `T′`
	    - And `T′` has not committed yet
	    DELAY `R/W(X)` until `T′` commits or aborts, Where `TS(T’)` = `Write_TS(x)`
- No Deadlocks, Since always `T` waits for `T’` when `TS(T)`>`TS(T’)` 

- Easy check for Strict Timestamp Ordering Algorithm is, is the schedule strict?
    - If No, then it will not be approved by the strict Timestamp ordering
- Originally, if the schedule is not approved by the basic TS Ordering, then it will NOT be approved by the strict TS Ordering Algorithm as Strict TS Ordering Algorithm assumes the basic one.

### Thomas’s Write Rule

A modification of the basic TO algorithm does not enforce conflict serializability, but it rejects fewer write operations by modifying the checks for the `write_item(X)` operation while read check remains the same.

 - `T` Issues `WRITE(X)`
    -  If `Read_TS(X)` > `TS(T)`, Means:
	    -  A younger transaction already read `X`
	    Abort `T`
	- If `Write_TS(X)` > `TS(T)`, Means:
	    -   A younger transaction already Wrote `X`
	    Continue without performing W(X)
	- If `Write_TS(x)` or `Read_TS(x)` =<`TS(T)`
	    -  Execute `Write_TS(X)`
	    - Set `Write_TS(X)` = `TS(T)`