# Introduction

Database starts in a consistent state then a failure happens, makes it inconsistent and our objective is to make the database always in a consistent state even after failures.

- Types of Failures:

|**Transaction Failure**|**System Failure**|**Media Failure**|
|---|---|---|
|• Incorrect input|• Addressing error|• Disk head crash|
|• Deadlock|• Application error|• Power disruption|
|• Cascade rollback|• Operating system fault|• Fire|
|• User interruption|• RAM failure|• Sabotage|
|• Etc...|• Etc....|• Etc...|
- What needs to be done when failure Happens?
    -  When transaction is interrupted during execution, revert to the last consistent state of the database. (**Recovery**)
    - When transaction is interrupted after it finished the execution but before its effects are reflected on the DB, confirm that transaction changes were reflected on the DB. (**Recovery**)

- **System log** (Log Data File (LDF)) keeps track of all transaction operations that are performed on the DB. They are used to recover the database to the most recent consistent state when failure occurred. System log is a sequential file that is stored on disk.
- Log Record Types:
	- (`start_transaction`, `T`). Indicates that transaction `T` has started execution.
	- (`write_item`, T, `X`, `old_value`, `new_value`). Indicates that transaction `T` has changed the value of database item `X` from `old_value` to `new_value`.
	- (`read_item`, `T`, `X`). Indicates that transaction `T` has read the value of database item `X`. Does not need to log old/new values.
	- (`commit`, `T`). Indicates that transaction `T` has completed successfully and affirms that its effect can be committed (recorded permanently) to the database.
	- (`abort`, `T`). Indicates that transaction `T` has been aborted.
- `old_value` = `BFIM` = Before Image
- `new_value` = `AFMG` = After Image
- `Back P` = Previous log record for this transaction
- `Next P` = Next log record for this transaction

- **Caching** is the process of bringing a DB block from disk to the DBMS Main memory.
    - In-memory buffers. Memory buffer = file block
    - When a block is needed, it is looked up in the cache:
	    - If available, use it.
	    - If not, load the file block to a memory buffer (known as cache the block)
- **Flushing** is the process of writing a modified page from the “in memory buffer” to the disk.
    - To free-up space
    - For recovery purposes
    - Dirty bit
        - 0: unmodified buffer
        - 1: modified buffer
    - Pin-unpin bit
	    - 0: the buffer could be flushed
	    - 1: the buffer can’t be flushed (if database is working on the block we sit pin-unpin bit = 1 so it doesn't get flushed while being worked on)

## Write-Ahead-Logging (WAL)

**Write Ahead Logging** is the recovery mechanism used to ensure that the `BFIM` is recorded in the appropriate log entry and the log entry is flushed to disk before the `BFIM` is overwritten with the `AFIM` on disk.

- WAL Actions:
	1. A log entry is flushed to the disk before the data buffers.
	2. All log entries must be written to the disk before a commit is considered complete


## When the data is allowed to be flushed to disk?

Database has updated a cache buffer, when to flush?

- **Steal** → Permits data to be flushed anytime (before or after the commit) ”you allowed to flush the updated blocks to disk any time”. (Has some freedom compared to other techniques)
- **No-Steal** → Prohibits changed data to be written to disk before the transaction commits “you allowed to flush the updated blocks after commit”.
- **Force** → Forces all pages updated by a transaction to be written to disk before the transaction commits.” “flush to disk after updates”. (Once an update occurs its written to disk)
- **No Force** → Flushing the updated pages of a transaction before commit are **not mandatory**. “flush to disk when the system decide”.

- Most common Technique: Steal/No-Force
- Periodically, the DBMS flushes its buffer to the disk (**Checkpoint**)
- **Checkpointing** is used as an optimization method with this technique to speed up the recovery process after a system failure.
- Checkpointing steps:
	1. Suspend execution of transactions temporarily.
	2. Force modified buffer data to disk (i.e., buffers with dirty bit =1)
	3. Write a (checkpoint) record to the log, save the log to disk.
	4. Resume normal transaction execution
- Checkpoint record include list of active transactions.

## Data Update

- Where?
	- **In-place Update**: Writes a modified buffer to the same original file block.
	- **Shadow Update**: Writes a modified buffer to a different disk block.
- When?
    - **Immediate Update**: Once a data item is modified in cache, the disk copy can be updated➔ Steal/No-Force and Steal/Force.
    - **Deferred Update**: All modified data items in the cache are written after a transaction ends its execution/ its commit (No-Steal/No-Force)

# Recovery Techniques

- When transaction fails **during** execution, Transaction **Rollback/Undo**.
    - Restore all BFIMs to disk.
    - Check for cascade rollback (If the CC mechanism doesn’t guarantee a Cascade less schedule)
	    - If other transactions read its dirty data, They must also be rolled back unless CC guarantees cascade less schedules
	Result Database returns to the **last consistent state**
- When transaction fails **After** it finishes execution, Transaction **Roll-forward/Redo**.
    - Restore all AFIMs on to disk (Reapply the committed changes)
    Result: Database reaches the **committed consistent state**

- Does the existence of the Checkpoint make any difference during recovery?
    - Yes, checkpoints reduce recovery time by allowing the DBMS to ignore log records before the checkpoint, since their effects are already reflected on disk

- Is applying Undo → Redo different from applying Redo → Undo?
    -  YES, the order matters, UNDO first, then REDO
    - **UNDO** removes effects of **uncommitted** transactions
    - **REDO** reapplies effects of **committed** transactions
    - If you do **Redo first**, you might:
	    - Reapply updates
	    - Then immediately undo them → wasted work or even inconsistency

- If we have multiple checkpoints only last checkpoint matters

- How to Handle Catastrophic Failure?!
    -  Backup Database and Log on external Secondary Device(s) / Archives.

## Deferred Update

Using **deferred update** schemes, the DBMS records all updates in the log but postpones (defers) all the updates to the database on disk until the transaction commits.

- No-Undo/Redo (No-Steal/No-Force)

Deferred Update Actions:
- During Transaction Execution:
    - Updates are recorded in the log and the cache
    - Dirty buffers are pinned(pin=1)
- During Commit Execution: 
    - Log updates are forced to the disk (WAL)
    - Dirty blocks are unpinned (pin=0)
- During Recovery:
    - All operations of all transactions with commit entry in the log, should be redone.

- All committed transactions must be redone.
- Recovery is Idempotent (recovery always gives the same result even if it was applied multiple times)
	- Prevents the database from corruption if failure occurred during the recovery process. 
- All committed transactions till the last checkpoint must be redone.
- The `BFIM` (Undo-log Entries) is not required to be stored in the log, Since no undo will take place.
- Only `AFIM` (Redo-log Entries) are needed in the redo process.
- Read log entries are unnecessary for recovery when the concurrency control prevents cascading aborts, since reads do not modify data and no transaction can read uncommitted updates.

- Procedure:
	1. Navigate the log file from bottom till the latest checkpoint.
	2. Prepare Commit List (List of all committed transactions till the latest checkpoint)
	3. Prepare Active List (List of all running transactions)
	4. For each log entry that belongs to a transactions in the Commit list: Write the `AFIM` in the database on disk in the order in which they were written into the log (top to bottom)
	5. For transactions in Active List: No action will be taken during recovery, but the transactions needs to be canceled and restarted.

![Pasted image 20260110005606.png](./Imgs/Pasted%20image%2020260110005606.png)

- Undo List is always empty in the deferred update.

- When recovery and concurrency control meet?
	- If the concurrency control mechanism used does not prevent cascading rollback → the recovery protocol MUST check for cascade rollbacks, thus read log entries must be stored in the log.

- Improved Deferred Update Recovery Algorithm or Improved no-undo/redo Recovery Protocol
	    - we go from bottom to top and check only committed transactions after checkpoint, once we find an update we add the update to the UDIL (User Defined Item List) if it's not there, if it's there it wont be added, this ensures we get the last updated value for each committed item.

- Pros:
    - It is successful when transactions are usually short and apply limited number of updates
    - It never needs Undo
- Cons: 
    - Not suitable for Long transactions (might occupy buffer), as it is No-Steal / No–Force, as it only moves the logs of the dirty blocks from the buffer to the Hard Disk (HD) when commit point is reached.
    - Therefore, only the transactions with commit records moved from the buffer to the HD, and other active transactions remain at the buffer until reaching their commit.

## Immediate Update

In these techniques, when a transaction issues an update command, the database on disk (log file part) can be updated immediately, without any need to wait for the transaction to reach its commit point.

### Undo/Redo

Transactions are allowed to commit before their changes are recorded to disk and they are allowed
to apply their updates on disk before commit.

- Steal/No-Force
- Once committed, update the log blocks (not necessarily after commit ➔ which called no force)

Immediate Update Actions:
- During transaction execution:
	- Updates are recorded in the log and the cache
	- Dirty buffers are flushed
- During commit:
	- Commit is written to log
	- Log updates are forced to the disk (WAL)
- During recovery:
	- All operations of all transactions with commit entry in the log, should be redone (same as deferred) -> Write After Image `AFIM` to disk
	- All operations of all transactions without commit entry in the log, should be undone -> Write Before Image `BFIM` to disk 

- Procedure:
	 1. Prepare Commit List (List of all committed transactions till the latest checkpoint)
	 2. Prepare Active List (List of all running transactions)
	 3. For transactions in Active List
		- Write the `BFIM` in the database on disk in the REVERSE order in which they were written into the log (bottom to top)
	4. For each write (redo) log entry that belongs to a transactions in the Commit list
		-  Write the `AFIM` in the database on disk in the order in which they were written into the log (top to bottom)

### Undo/No-Redo

All updates of a transaction are recorded on disk before the transaction commits

- Steal/Force