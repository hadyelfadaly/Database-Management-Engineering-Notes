- The purpose of The entire purpose of **query processing and optimization** is to solve one main problem: How can the DBMS answer a user’s query _efficiently_?
- DBMS has algorithms to implement relational algebra expressions
- SQL is a different kind of high level language; specify what is wanted, not how it is obtained while A relational algebra (RA) expression is procedural Describes “How” to achieve the result step by step.
- Query Optimization: The process of choosing a suitable execution strategy for processing query.
- Query Optimization Techniques:
     - Heuristic Optimization: Is based on heuristic rules for ordering the operations in a query execution strategy
     - Cost Based Optimization: Involves systematically estimating the cost of different execution strategies and choosing the execution plan with the lowest cost estimate.
 ![Image](Pasted%20image%2020251113161656.png)
- So in Short queries go in this sequence: English->SQL->RA (Relational Algebra)->Query Tree

# Query Tree

- Corresponds to a relational algebra expression
- Input relations (tables) → leaf nodes
- RA operations → internal nodes
- Execution (evaluation):
    - Starts at leaf nodes
    - Internal nodes are replaced by the result relation
    - Ends at the root node

# Query Evaluation

- A query block contains a single SELECT-FROM-WHERE expression (along with ORDER BY, GROUP BY AND HAVING, if any).

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