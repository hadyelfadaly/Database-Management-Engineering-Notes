# Introduction

- A transaction (TXN) is a sequence of of one or more operations (reads or writes) which reflects a single real world transition.
- DB actions: read (R(X)), write (W(X)), commit, abort
- In the real world, a TXN either happened completely or not at all. Ex: Transfer money between accounts.
- In SQL Default: each statement = one transaction
- In a program, multiple statements can be grouped together as a transaction

## Why we study Transactions?

- First, the meaning of “consistent"
	- “Consistent” means: **The database satisfies all rules, constraints, and data integrity conditions.**
	- Examples: Salary ≥ 0, Account balances never go negative, Sum of debits = sum of credits, ...etc.
- A consistent database means: The data makes sense and follows the rules.
- We Study Transactions Because a **transaction must transform the database FROM one consistent state TO another consistent state**.
- **Why is this important?**
	 - Because during a transaction: It may **not finish** or crash or multiple transactions at same time, ..etc
	 - Without transaction control, the database can become **inconsistent**
- Motivation for Transactions
     - Recoverability: Safe from system crashes
     - Concurrency: Multi-user database access
## Desirable Properties of a Transaction (ACID properties)

### Atomicity

- A transaction is an atomic unit of processing; it is either performed in its entirety or not performed at all. (all or nothing)
- Two possible outcomes for a TXN
	1. It commits: all the changes are made
	2. It aborts: no changes are made
- A DBMS ensures atomicity by undoing the actions of partial transactions
- To enable this, the DBMS maintains a record, called a log, of all writes to the database, The component of a DBMS responsible for this is called the recovery manager.

### Consistency

- A correct execution of the transaction must take the database from one consistent state to another.
- The tables must always satisfy user-specified integrity constraints
- How consistency is achieved:
	- Programmer makes sure a TXN takes a consistent state to a consistent state
	- System makes sure that the TXN is atomic

### Isolation

- A transaction should not make its updates visible to other transactions until it is committed.
- A transaction executes concurrently with other transactions

### Durability

- Durability or permanency: Once a transaction changes the database and the changes are committed, these changes must never be lost because of subsequent failure.
- The effect of a TXN must continue to exist (“persist”) after the TXN And after the whole program has terminated And even if there are power failures, crashes, etc.
- Means: Write data to disk

## Transaction Components

1. Begin Transaction
2. TSQL or DML
3. End Transaction
	- Ends Successfully: All changes applied on the data items through write operations are reflected on the database. (Ends by COMMIT)
	- Ends Unsuccessfully: Failure occurred during transaction processing. So, None of the changes applied on the data items through write operations should be reflected on the database. (Ends by ABORT) 

# Transaction Processing Data Model

- Database = Number of Data Items
- Data Item could be one record or one disk block
- Basic operations on data items are Read and Write

## Read Operation

- `read_item(X)`: reads a database item named X into the program.
     1. Find the address of the disk block that contains item X
     2. Copy that disk block into a buffer in main memory (if that disk block is not already in some main memory buffer)
     3. Copy item X from the buffer to the program variable named X
    Disk -> Memory -> Program
## Write Operation

- `write_item(X)`: Writes the value of program variable X into the database item named X.
     1. Find the address of the disk block that contains item X
     2. Copy that disk block into a buffer in main memory (if that disk block is not already in some main memory buffer)
     3. Copy item X from the program variable named X into its correct location in the buffer
     4. Store the updated block from the buffer back to disk (either immediately or at some later point in time)
    Program -> Memory -> Disk

# Transaction Lifecycle

- For recovery purposes, the system needs to keep track of when the transaction starts, terminates, and commits or aborts.
- Transaction states:
	- Active state
	- Partially committed state
	- Committed state
	- Failed state
	- Terminated State

![Image](Imgs/Pasted%20image%2020251118013539.png)

# Recoverability

- Why recovery needed? 
	1. A computer failure (system crash)
	2. A transaction or system error
	3. Local error or exception condition detected by the transaction
	4. Concurrency control enforcement
	5. Disk Failure
	6. Physical problems and catastrophes
	From 2 to 5 are most common

# Concurrency

- What is the difference between concurrency and parallel processing?
	- Concurrency (Interleaving): 
		- Multiple transactions **appear** to run at the same time But they are actually **interleaved** on **one CPU**, The CPU switches rapidly between them (Context Switching)
		- Only **one** instruction runs at a time
	- Parallel Processing (Simultaneous):
		- Multiple transactions run **truly at the same time** Because the system has **multiple CPUs** Each CPU executes one transaction independently

- Why Concurrency?
	- Allowing only serial execution of user programs may cause poor system performance
		- Low throughput, long response time
		- Poor resource utilization (CPU, disks)

- Advantages of Concurrency:
     - Decreasing waiting time
     - Decreasing Response time
     - Increasing Resource Utilization
     - Increasing Efficiency

- A schedule S of n transactions T1, T2, …, Tn is ordering of execution of operations of the transactions.

![Image](Imgs/Pasted%20image%2020251118020607.png)
- Serial Schedules ensure the consistency of the database, while non serial schedules might have conflicting operations

## Concurrency Problems

- Concurrency is more dangerous because transactions interleave on the same CPU, so they access shared data at overlapping times, which can easily cause anomalies like lost updates, dirty reads, and unrepeatable reads. Parallel processing uses different CPUs, so each transaction typically works on independent execution units — fewer conflicts.

### Lost Update Problem

- A transaction's update gets overwritten by another transaction.
- Why it happens:
	- Both read the same old value, write different updates, last write wins.

### Dirty Read Problem (Temporary Update)

- A dirty read happens when a transaction reads a value written by another transaction that hasn't committed yet. If that other transaction aborts later, the reader has seen a value that never actually existed in the final database.
- Why it happens:
	- No protection from uncommitted writes.

### Unrepeatable Read Problem

- A transaction reads the same row twice and gets different values.
- Why it happens:
	- Another Transaction updates rows between the 2 reads.

### Incorrect Summary Problem

- A transaction computing an aggregate sees some old values and some new values
- Why it happens:
	- Another Transaction updates rows during the summary scan.

## Expressing Schedules

- r: read, w: write, c: commit, a: abort
- Reads and Writes are the most important ones
- `r1(x)` r -> Operation, 1 -> Transaction ID, X -> Data Item

## Conflicting Operations

- Two operations are conflicting, iff:
	- Belong to different transactions
	- Access the same data item
	- At least one of them is a write operation
- Conflict types:
	- Read-Write conflict -> `r1(x) w2(x)`
	- Write-Write conflict -> `w1(x) w2(x)`
	- Write-Read conflict -> `w1(x) r2(x)`

## Proper Schedule

- Properties of a proper[Complete] Schedule
	- Contains all operations from all transactions (Read/Write Operations)
	- All operations from the same transaction appear in the same order as in that transaction.

## Schedule Types

- From the **recoverability** perspective, There are three types of schedules
	- if transaction fails we need to undo the effect of this transaction to ensure the atomicity property

### Recoverable Schedules

- A schedule S is recoverable iff
	- No transaction T in S commits until all transactions T’ that have written some item X that T reads have committed.
	- Example: T1 writes X and T2 reads X, T2 Must wait for T1 to commit first, if T1 aborts, then T2 must abort and rollback too otherwise we fall into dirt read problem again

- Recoverability Test:
	1. Find conflicting operations of type Write-Read
	2. Find sequence of transaction termination (Abort/Commit)
	3. If the transaction that performed the write operation did not terminate first (Commit/Abort) then this schedule is not recoverable. Otherwise it is recoverable.

### Cascading Rollback Problem

- Cascading rollback (aka Cascading abort) occurs when an uncommitted transaction is forced to roll back because it read a data item written by another transaction that failed.
- Example: r1(X); w1(X); r2(X); a1; a2; At this point T2 was forced to rollback because it read data that is no longer valid

### Cascadeless Schedules (avoid cascading abort)

- A schedule S is cascadeless iff:
	- All transactions read only items that were written by committed transaction. (T1 must commit before T2 reads X) 

- Avoidance of Cascading Abort Test:
	1. Find conflicting operations of type Write-Read
	2. If the transaction that performed the write operation terminates (Commit/Abort) before the read operation takes place then this schedule is cascadeless. Otherwise it is not Cascadeless.

- If a schedule is cascadeless then it is recoverable also

### Strict Schedule

- A schedule S is strict iff
	- All transactions read/write only items that were written by committed/aborted transaction. (T1 must commit or abort before T2 reads/writes X)

- Strict Test:
	1. Find conflicting operations of type Write-Read and Write-Write
	2. If the transaction that performed the first Write operation terminates (Commit/Abort) before the read/write operation takes place from the other transaction on the same data item, then this schedule is Strict. Otherwise it is not Strict.

- if a schedule is strict then it is cascadeless and recoverable too

### Serial Schedule

- A schedule S is serial iff:
	- Transactions operations are not interleaving
- Example: r1(X); w1(X); r1(Y); w1(Y); c1;r2(X); w2(X); c2;
- Serial schedules don’t suffer lost updates, dirty reads
- Enforcing serial schedules reduces performances
	- T1 will have to wait T2 or vice versa

- Our goal now is to get valid non serial schedule to improve performance
- Idea: Check if the non serial schedule is equivalent to a serial schedule

### Serializable Schedule

- A schedule S of n transactions is serializable iff
	- It is equivalent to another schedule S’
	- S’ is a serial schedule of the same n transactions

- Being serializable is not the same as being serial
- Being serializable implies that the schedule is a correct schedule. Because:
	- It will leave the database in a consistent state.
	- The interleaving is appropriate and will result in a state as if the transactions were serially executed, yet will achieve efficiency due to concurrent execution.

## Schedule Equivalence Types

### Result Equivalence

- Two schedules are result-equivalent if:
	- They produce the same final database state

### Conflict Equivalence

- Two schedules are conflict-equivalent if:
	- the order of any two conflicting operations is the same in both schedules.

![Image](Pasted%20image%2020251118030807.png)
- If a schedule is serial then it is Conflict-Serializable and View-Serializable
- If a schedule is Conflict Serializable then its view serializable

### Serialization Graph

- A tool that is used to tell whether a given schedule S is conflict-serializable.
- A graph consists of nodes and edges
- Serialization Graph Construction
	1. Create a node, for each transaction.
	2. Create an edge (T1 →T2) if there is w1(x) before r2(x)
	3. Create an edge (T1 →T2) if there is r1(x) before w2(x)
	4. Create an edge (T1 →T2) if there is w1(x) before w2(x)

- Schedule S is conflict-serializable, iff the serialization graph has no cycles. between each pair
## Transaction Processer (Scheduler)

- Forces the order of operations execution to guarantee certain schedule type, Mostly, serializable schedule
	- How?
		- By Concurrency Control Protocols 
