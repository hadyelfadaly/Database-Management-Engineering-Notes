# Introduction

- The purpose of The entire purpose of **query processing and optimization** is to solve one main problem: How can the DBMS answer a user’s query _efficiently_?
- DBMS has algorithms to implement relational algebra expressions
- SQL is a different kind of high level language (declarative); specify what is wanted, not how it is obtained while A relational algebra (RA) expression is procedural Describes “How” to achieve the result step by step.
- Query Optimization: The process of choosing a suitable execution strategy for processing query.
- Query Optimization Techniques:
     - Heuristic Optimization: Is based on heuristic rules for ordering the operations in a query execution strategy
     - Cost Based Optimization: Involves systematically estimating the cost of different execution strategies and choosing the execution plan with the lowest cost estimate.
 ![Image](Pasted%20image%2020251113161656.png)
- So in Short queries go in this sequence: English->SQL->RA (Relational Algebra)->Query Tree

# Query Tree

- Immediate form of query
- Corresponds to a relational algebra expression
- Input relations (tables) → leaf nodes
- RA operations → internal nodes
- Execution (evaluation):
    - Starts at leaf nodes
    - Internal nodes are replaced by the result relation
    - Ends at the root node

# Relational Algebra

## Select Operator

- Produces Table containing subset of rows of given argument table satisfying a certain condition

### Selection Condition

- Operators: <. <=, >=, >, =, !=
- Simple selection condition:
     - "attribute" operator "constant"
     - "attribute" operator "attribute"
- Another Conditions:
     - "condition" AND/OR "Condition"
     - NOT "Condition"

![Image](Imgs/Pasted%20image%2020251113170130.png)

### Comparisons

- Equality comparison like: Pnumber = 10
- Nonequality comparisons like: salary > 30000 or salary < 30000 or salary != 30000.

### Selection Algorithms

#### Simple Condition (one comparison)

| Comparison Type           | Comparison Attribute                | How                                                                                                                                   |
| ------------------------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Equality                  | Key or non-key with secondary index | Navigate the index to locate the<br>index record with the comparison value.                                                           |
| Equality                  | Nonkey with clustering index        | Navigate the index to locate the first data block satisfying the condition and then retrieve subsequent blocks until all is retrieved |
| Equality                  | Key with primary index              | Navigate the index to locate the index record with the comparison value.                                                              |
| Non-equality              | Key with primary index              | Navigate the index to locate the data block with the corresponding equality condition then retrieve preceding or subsequent records.  |
| Equality and non-equality | Any (without any access structure)  | Scan the data file, block by block looking for records satisfying the comparison. aka: linear search/brute force.                     |
| Equality                  | Ordering key without an index       | Navigate the data file using binary search to locate the data block. aka: binary search                                               |

#### Conjunctive condition (multiple AND comparisons)

| When                                                                                    | How                                                                                                                                                                                             |
| --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A comparison attribute with an index                                                    | Use the index to retrieve satisfying records then apply the remaining comparisons                                                                                                               |
| A group of comparison attributes with a composite index appears in equality comparisons | Use the composite index to retrieve satisfying records then Apply the remaining comparisons                                                                                                     |
| Comparison attribute(s) with secondary index(es) with record pointers.                  | Use the secondary index(es) to retrieve the record pointers then Intersect the record pointers Retrieve the records and apply the rest of the comparisons. aka: Intersection of record pointers |
Example on last one: 

we have this query

```sql
SELECT * FROM Employee WHERE Dno = 5 AND Salary > 3000 AND Gender = 'F';
```
 And there is a secondary index on Dno, Salary and gender

1. Use each index to retrieve list of record pointers 
     Dno = 5 → returns {5, 8, 10, 13}
     Salary > 3000 → returns {3, 8, 10, 13, 15}
     Gender = F → returns {2, 8, 10, 11, 13}
2. Intersect the record pointer sets Because we want **AND**, we only want records that appear in _all_ sets.
     Intersection = {8, 10, 13} These are the **only records** that satisfy _all_ conditions.
3. Retrieve those records: bring their blocks into memory and apply any remaining comparisons if needed

- Why is this efficient?
Because:
- You avoid scanning the whole file
- You use **multiple indexes at the same time**
- You only access the records that are guaranteed to match
#### Disjunctive condition (multiple OR comparisons)

- Use simple condition algorithms on the individual conditions, then union the results and remove duplicates.


## Project Operator

-  Produces Table containing subset of columns of given argument table.
- Project operator **removes duplicates automatically** because:
     - In relational algebra, a relation is a set and a set cannot contain duplicate tuples so projection must eliminate duplicates
- Example: 
![Image](Imgs/Pasted%20image%2020251113173439.png)

- Keep only **Name** and **Address** Remove **Id** and **Hobby** and remove duplicates
- Same **Name**, same **Address**, different **Id**, But since **Id** is removed, the two rows become identical. so one of them is removed.

```sql
SELECT DISTINCT ID, name FROM R
```

Where R have (ID, name, sal, dob)

- The implementation requires the following:
     - Remove unwanted columns (the easy step)
     - Eliminate any duplicate tuples produces. (the difficult step)


### Project Algorithms

- No DISTINCT in query -> no need to remove duplicates
- DISTINCT in query -> 
     - Key in attribute list then there will be no duplicates
     - if key not in attribute list we have to eliminate duplicates

- We sort the table then loop on records one by one we output the first instance of a record into result table and if it appears again (duplicated) we pass until a new record is found.

## Set Operations

- Set operations: UNION, INTERSECTION, SET DIFFERENCE and CARTESIAN PRODUCT
- CARTESIAN PRODUCT of relations R and S include all possible combinations of records from R and S. The attribute of the result include all attributes of R and S. It is a very expensive operation and should be avoided if possible.
- Set operations apply to type compatible relations
     - Same number of attributes
     - Names of attributes are the same in both
     - Same attributes domains

### Union

- Sort the two relations on the same attributes
- Scan and merge both sorted files concurrently, whenever the same tuple exists in both relations, only one is kept in the merged results. (Elimination of duplicates)

### Intersection

- Sort the two relations on the same attributes.
- Scan and merge both sorted files concurrently, keep in the merged results only those tuples that appear in both relations.

### Set Difference (R-S)

- Keep in the merged results only those tuples that appear in relation R but not in relation S.


## Aggregate Operators

- Functions that operate on sets: COUNT, SUM, AVG, MAX, MIN, they return scalar (numbers) not tables
- Index available -> Index scan
- No index but table sorted on the attribute we want -> the Min and Max is readily available
- No index and table is not sorted -> table scan

### Aggregate Operators With B-Tree

- MAX ->Traverse till the right-most index key
- MIN -> Traverse till the left-most index key
- SUM -> Visit all keys
- AVERAGE -> Visit all keys
- COUNT -> Is usually part of the catalog so it can be found directly


# Group By

## How DBMS performs GROUP BY (Sorting method)

1. Sort on grouping attributes
2. 2. Partition, while scanning the sorted table: Identify the start and end of each group
3. Apply aggregate functions

- If there is a clustering index on grouping attribute then groups are already together -> no sorting needed -> GROUP BY becomes much faster.
- Clustering index → Already grouped

# Condition Selectivity (sl)

- Selectivity = **the fraction (percentage) of records that satisfy a condition**.
- It is always between **0 and 1**.
- selectivity is used by the optimizer to estimate:
     - number of qualifying rows
     - cost of selections
     - cost of joins
     - pushing selections down in query trees

## Equality on a KEY attribute

- A key attribute has **unique values** — no duplicates.
- Example: `Name` is a key → every employee has a unique name. so the condition `σ_Name = 'Allen'` can match **AT MOST 1 record** since name is unique it can be found only once (1 record or not found)
- table size = n rows, selectivity = 1/n, Because exactly **one** record satisfies the condition.

## Equality on a NON-KEY attribute

- A non-key attribute **can repeat**. Like `dept_number` (1,1,1,2,2,2,3,3,3,4,4,4,5,5,5)
- `dept_number` is uniformly distributed  → each value appears equally often. In this case: 
    - `i` = number of distinct values, so `i` here = 5 (distinct values are 1, 2, 3, 4, 5)
    - selectivity = 1/`i`
    - Because if values are uniformly distributed:
         - 1/5 of the records are dept 1 and 1/5 are dept 2 and etc.

## Why does the optimizer need selectivity?

- Because before performing a selection, the DBMS estimates: Result size = sₗ × Relation Size

# Algorithms for Join Operations

## Sort Merge Join (J3)

- both files are ordered on join attribute or They **can be sorted** efficiently
- When:
     -  R is ordered by A
     - S is ordered by B 
     - R join S
     - join condition A = B
    we do sort merge  join.
- Why sorting is useful?
     - Because once both relations are sorted, the join becomes **linear**
     - We scan each relation only once.
     - No nested loops needed.
### How Does the Algorithm Works?

1. If not sorted, Sort both relations.
2. Initialize 2 pointers, `i` for R and `j` for S
3. Compare `R[i].A` with `S[j].B`
     1. Case 1: `R[i].A` < `S[j].B` -> move pointer in R forward (`i++`)
     2. Case 2: `R[i].A` > `S[j].B` -> move pointer in S forward (`j++`)
     3. Case 3: `R[i].A` == `S[j].B` -> Match! -> Output all combinations of matching rows, If duplicates exist on both sides, handle the entire matching group

- How to handle duplicates? 
     - After a match happens, initialize a new pointers `l` and `k` and set them to the rows where the match happened: `l = j` and `k = i`
     - Start with `l` move it forward to check if the next row in S is still equal to the current row in R (a duplicate), while `R[i].A` = `S[l].B` output the matching row into result. (R is standing still while s is moving forward to check for duplicates)
     - Now same with `k` after finishing with `l`, while `R[k].A` = `S[j].B` output the matching row into result. (S is standing still while R is moving forward to check for duplicates)
    in simpler words, It first scans forward in **S** (keeping R fixed) to output all S-duplicates matching `R[i]`, then scans forward in **R** (keeping S fixed) to output all R-duplicates matching `S[j]`.

## Single-loop Join (J2)

- Also called: Index Nested Loop Join and Index-based join
- When:
     - One table has an index on its join attribute
- Example: **S has an index on B**.
### How Does the Algorithm Works?

1. Scan R block by block
2. For each record r in the block: Look at `r.A`
3. Use the index on `S.B`

- This avoids scanning S block-by-block (like nested-loop join). Instead, you use the index to retrieve matching tuples instantly.
- Process:
	- Look at DeptID=2 → use index → find block B1 → output 2 rows
	- Look at DeptID=3 → use index → find block B1 → output 1 row
	- Look at DeptID=4 → use index → find block B2 → output 2 rows
    S is never scanned sequentially — **the index gives direct access**.
- Single-Loop Join is more efficient because instead of scanning S for every tuple of R (like nested-loop join), it uses the index on S to directly jump to the matching records, eliminating the inner scan.
 - Nested-loop join:  
    **R block → S block → compare each pair** → very expensive.
- Single-loop join (index join):  
    **R record → use index to find all matching S records instantly**  → no scanning of S → much faster.

- The outer relation should be the one _without_ the index, because we must scan it anyway; for each of its tuples, we then use the index on the inner relation to directly retrieve the matching records.

## Nested-loop Join (J1)

- Also called: Nested-block join
- When:
     - No indexes is available for the joining attributes

### How Does the Algorithm Works?

![Image](Imgs/Pasted%20image%2020251117200436.png)

## Factors Affecting the Join

- Available buffer space[Memory buffers] 
- Choice of inner VS outer relation

### Choice of inner VS outer relation

- in **Nested-Loop Join**, the cost formula is: Cost = b(outer)​+[b(outer) / buffers​​]×b(inner​)
- The **outer relation should be the smaller one** to minimize the number of times we scan the inner relation, because the outer relation determines how many times the inner relation is scanned.
- If R is large and S is small:
	- Outer = S, Inner = R, → small outer → fewer passes → much cheaper.


# Query Evaluation

- A query block contains a single SELECT-FROM-WHERE expression (along with ORDER BY, GROUP BY AND HAVING, if any).
- Convert the SQL query into **an equivalent relational algebra expression**, then choose an execution plan to evaluate it efficiently. But there can be many equivalent RA expressions…
- Goal: Rewrite a query into an **equivalent** form that produces the **same result**, but is **cheaper** to execute.
	- Equivalent = same result set
	- Better = lower cost (disk I/O, CPU, memory, communication)

# Heuristic Optimization

- Rules for ordering the operations in query optimization.
- To transform a relational expression into another equivalent expression we need transformation rules that preserve equivalence
- They are Techniques that that don’t have proofs.
- Reached by trial-and-error, or Common-sense
- The task of heuristic optimization of query trees is to find a final query tree that is efficient to execute.
- Steps in Converting a Query Tree During Heuristic Optimization:
     1. Initial (canonical) query tree for SQL query Q.
     2. Moving SELECT operations down the query tree. (early in RA)
     3. Applying the more restrictive SELECT operation first.
     4. Replacing CARTESIAN PRODUCT and SELECT with JOIN operations
     5. Moving PROJECT operations down the query tree. (early in RA) 

## RA Transformation Rules

1. Cascade of σ: σ c1 AND c2 AND ... AND cn(R) ≡ σ c1 (σc2 (...(σcn(R))...) )
2. Commutativity of σ : σc1 (σc2(R)) ≡ σc2 (σc1(R))
3. Cascade of π: πList1 (πList2 (...(πListn(R))...) ) ≡ πList1(R)
4. Commuting σ with π: πA1, A2, ..., An (σc(R)) ≡ σc (πA1, A2, ..., An (R))
5. Commutativity of ⨝ (and x): R ⨝(C S ≡ S ⨝(C) R; R x S ≡ S x R
6. Commuting σ with ⨝ (or x ): (σc ( R ⨝ S ) ≡ (σc (R))) ⨝ S OR (σc ( R ⨝ S ) ≡ (σc1 (R)) ⨝ (σc2 (S)))
7. Commuting π with ⨝ (or x): πL (R⨝C S ) ≡ (πA1, ..., An(R)) ⨝C (πB1, ..., Bm(S))
8. Commutativity of set operations: The set operations υ and ∩ are commutative but “–” is not
9. Converting a (σ, x) sequence into ⨝: (σC (R x S)) = (R ⨝C S)
# Cost Estimation

- It estimates cost of different execution strategies and chooses the execution plan with lowest execution cost
