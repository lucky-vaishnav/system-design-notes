Indexing is one of those topics that almost every backend engineer uses daily, but many developers only know the surface-level answer:

> "Index makes queries faster."
> 

 deeper:

- How does an index actually work?
- Why can an index make queries slower?
- Why doesn't the DB use an index sometimes?
- What's a clustered index?
- What is a covering index?
- Why is `LIKE '%abc'` slow?
- Why does index order matter?

# Question: What is an Index?

An index is a separate data structure maintained by the database to find rows quickly without scanning the entire table.

Without index:

```
Users Table

ID    Name
1     John
2     Bob
3     Lucky
4     Alice
...
10,000,000 rows
```

Query:

```
SELECT*
FROM users
WHERE id=4;
```

Without index:

```
Check row 1
Check row 2
Check row 3
Check row 4
Found
```

Potentially:

```
10 million checks
```

This is called:

```
Full Table Scan
```

---

# Question: What does the database do with an index?

Instead of searching the table directly:

```
Table
```

Database searches:

```
Index
```

then jumps directly to the row.

```
Index
  |
  +--> Row Location
```

---

# Question: What is the most common index type?

Almost every major database uses:

```
B-Tree
```

or

```
B+ Tree
```

Examples:

- PostgreSQL
- MySQL
- SQL Server
- Oracle

Default index type:

```
B-Tree/B+Tree
```

---

# Question: Why not use a HashMap internally?

Many developers think:

```
Index = HashMap
```

Not usually.

Because databases need:

```
=
>
<
BETWEEN
ORDERBY
```

Hash maps only excel at:

```
=
```

Example:

```
WHERE id=5
```

Fast.

But:

```
WHERE id>5
```

Hash indexes become useless.

---

# Question: What does a B-Tree look like?

Suppose values:

```
1 2 3 4 5 6 7 8 9
```

B-Tree:

```
          5
        /   \
      2       8
     / \     / \
    1  3   6   9
         \
          4
```

Real databases use much wider trees.

Example:

```
Root
  |
100 children
  |
100 children
```

Not binary.

---

# Question: Why are B-Trees fast?

Because search complexity is:

```
O(log n)
```

Example:

10 million rows.

Without index:

```
O(n)
```

Potentially:

```
10,000,000 checks
```

With B-Tree:

```
log2(10,000,000)
≈ 24
```

Only around:

```
24 comparisons
```

Huge difference.

---

# Question: What is a B+ Tree?

Most modern databases use:

```
B+ Tree
```

instead of pure B-Tree.

In B+ Tree:

Internal nodes:

```
Only keys
```

Leaf nodes:

```
Actual pointers to rows
```

Leaf nodes are linked:

```
1 -> 2 -> 3 -> 4 -> 5
```

This makes:

```
ORDERBY
BETWEEN
RANGE SCANS
```

very efficient.

---

# Question: Why are indexes stored separately from the table?

Because:

```
Table Storage
```

is optimized differently.

Example:

```
Users Table
```

contains:

```
id
name
email
address
phone
...
```

Index only stores:

```
id
row_location
```

Much smaller.

Much faster.

---

# Question: What is Row Location?

Index does not usually store full row.

Example:

```
Index

ID     Pointer
1      Row#500
2      Row#900
3      Row#1200
```

Database:

1. Finds pointer
2. Goes to row

---

# Question: Why do indexes speed up SELECT but slow down INSERT?

Imagine:

```
INSERTINTO users
```

Without index:

```
Add row
Done
```

With index:

```
Add row
Update index
Rebalance B-Tree
```

Extra work.

---

# Question: Why do indexes slow UPDATE?

Suppose:

```
UPDATE users
SET email='abc'
```

If email indexed:

```
Update row
Update index
```

More work.

---

# Question: Why do indexes slow DELETE?

Delete requires:

```
Remove row
Remove index entry
Rebalance tree
```

Again extra work.

---

# Interview Question: Why not index every column?

Many juniors answer:

```
More indexes = More speed
```

Wrong.

Each index causes:

```
Extra storage
Extra writes
Extra maintenance
```

Too many indexes can hurt performance.

---

# Question: When does DB ignore an index?

Example:

```
SELECT*
FROM users
WHERE age>0;
```

Suppose:

```
99% rows match
```

Using index:

```
Read index
Jump to rows
Fetch almost whole table
```

Costly.

Database chooses:

```
Full Table Scan
```

instead.

---

# Question: What is Selectivity?

Very important interview topic.

High selectivity:

```
SSN
Email
User ID
```

Example:

```
1 row returned
```

Excellent index candidate.

---

Low selectivity:

```
gender
status
is_active
```

Example:

```
50% rows match
```

Poor index candidate.

---

# Question: Why is LIKE '%abc' slow?

Example:

```
WHERE nameLIKE'%john'
```

Database cannot know where string starts.

Must scan.

Index unusable.

---

Good:

```
WHERE nameLIKE'john%'
```

Database can use index.

---

# Question: Why does Composite Index order matter?

Example:

```
CREATE INDEX idx_user
ON users(first_name,last_name);
```

Index order:

```
first_name
then last_name
```

Works:

```
WHERE first_name='Lucky'
```

Works:

```
WHERE first_name='Lucky'
AND last_name='Vaishnav'
```

Does NOT efficiently work:

```
WHERE last_name='Vaishnav'
```

because first column missing.

This is called:

```
Left-most Prefix Rule
```

Very common interview question.

---

# Question: What is a Covering Index?

Suppose:

```
SELECT id,name
FROM users
WHERE id=1;
```

Index contains:

```
id
name
```

Database can answer query from index alone.

No table lookup needed.

Called:

```
Covering Index
```

Very fast.

---

# Question: What is a Clustered Index?

Important distinction.

---

## Clustered Index

Table data itself stored in index order.

```
1
2
3
4
5
```

Rows physically arranged.

Usually only one.

Examples:

- SQL Server
- InnoDB Primary Key

---

## Non-Clustered Index

Separate structure.

```
Index -> Pointer -> Row
```

Can have many.

---

# Question: How does PostgreSQL differ?

PostgreSQL tables are heap tables.

Meaning:

```
Rows stored separately
```

Primary key index:

```
Not clustered by default
```

This surprises many MySQL developers.

---

# Question: What index types exist besides B-Tree?

## Hash Index

Good:

```
=
```

Bad:

```
>
<
BETWEEN
```

---

## Full Text Index

Used for:

```
search
```

Examples:

```
Google-like searching
```

PostgreSQL:

```
GIN
```

MySQL:

```
FULLTEXT
```

---

## GiST

PostgreSQL special index.

Used for:

```
Geospatial
Ranges
Custom operators
```

---

## GIN

PostgreSQL special index.

Used for:

```
JSONB
Arrays
Full-text search
```

Very common in modern apps.

---

# Question: How does indexing work for JSON fields?

PostgreSQL:

```
data JSONB
```

Normal B-Tree not enough.

Use:

```
CREATE INDEX idx_data
ONtable
USING GIN(data);
```

Then:

```
WHEREdata->>'status'='ACTIVE'
```

becomes fast.

---

# Question: What are the most common index mistakes?

### 1. Too many indexes

Causes slow writes.

---

### 2. Indexing low-cardinality columns

Example:

```
gender
status
```

Little benefit.

---

### 3. Wrong composite index order

Very common.

---

### 4. Using functions

Example:

```
WHERE LOWER(email)='abc'
```

May prevent index usage.

---

### 5. Leading wildcard

```
LIKE'%abc'
```

Usually kills index usage.

# Question: How can an index be used for `LIKE 'abc%'`?

Suppose you have an index on:

```
name
```

and data:

```
Aaron
Alice
Andy
abc
abcde
abcfgh
abczzz
bobby
charlie
```

The index (B+ Tree) is stored in sorted order:

```
Aaron
Alice
Andy
abc
abcde
abcfgh
abczzz
bobby
charlie
```

Query:

```
WHERE nameLIKE'abc%'
```

The database can think of this as:

```
WHERE name>='abc'
AND name<'abd'
```

because every value starting with:

```
abc
```

must lie between:

```
abc
```

and

```
abd
```

in sorted order.

So DB:

1. Finds first occurrence of `"abc"` in index.
2. Walks forward through leaf nodes.
3. Stops when values no longer start with `"abc"`.

Very efficient.

---

# Question: Why can't the index be used efficiently for `LIKE '%abc'`?

Example:

```
myabc
testabc
helloabc
abc
```

Query:

```
WHERE nameLIKE'%abc'
```

The database only knows:

```
ends with abc
```

but the beginning could be:

```
aabc
babc
zabc
xyzabc
```

In sorted index order these rows are scattered everywhere:

```
aabc
alice
babc
charlie
xyzabc
```

No continuous range exists.

Database cannot jump to one location.

Usually it must scan everything.

---

# Question: Why doesn't a HashMap-based index support `>` or `<`?

Because of how hashing works.

Suppose:

```
1 -> hash(1) -> bucket 500
2 -> hash(2) -> bucket 12
3 -> hash(3) -> bucket 900
4 -> hash(4) -> bucket 44
```

Notice:

```
1,2,3,4
```

are not stored in order.

After hashing:

```
500
12
900
44
```

there is no relationship anymore.

---

Hash index is excellent for:

```
WHERE id=3
```

because:

```
hash(3)
```

immediately tells us the bucket.

---

But:

```
WHERE id>3
```

means:

```
4
5
6
7
8
...
```

Hash table has no idea where values greater than 3 are located.

It only knows:

```
exact bucket for exact value
```

Therefore:

```
Hash Index
=
Excellent for equality

Bad for ranges
```

---

# Question: In a B+ Tree, what exactly is the "key"?

Yes, the key is normally the indexed column value.

Example:

```
CREATE INDEX idx_email
ON users(email);
```

Data:

```
id    email
1     a@test.com
2     c@test.com
3     b@test.com
```

Index stores keys:

```
a@test.com
b@test.com
c@test.com
```

sorted.

The key is:

```
email value
```

---

# Question: Then what is the pointer?

Pointer means:

```
Where is the actual row?
```

Example:

Table:

```
Row#100 -> a@test.com
Row#250 -> b@test.com
Row#500 -> c@test.com
```

Index:

```
a@test.com -> Row#100

b@test.com -> Row#250

c@test.com -> Row#500
```

When searching:

```
WHERE email='b@test.com'
```

DB:

1. Finds key in index.
2. Gets pointer.
3. Jumps directly to Row#250.

---

# Question: Then what's the difference between B-Tree and B+ Tree?

This part confuses many developers.

---

## B-Tree

Data can exist in:

```
Root
Intermediate nodes
Leaf nodes
```

Example:

```
         [50]
        /    \
     [20]   [80]
```

If searching for:

```
50
```

DB can stop at root.

Data may exist there.

---

## B+ Tree

Internal nodes contain:

```
Only keys
```

Example:

```
         [50]
        /    \
     [20]   [80]
```

These are navigation entries only.

Actual row pointers are stored only in leaf nodes.

Example:

```
Leaf:

20 -> Row#100
30 -> Row#200
50 -> Row#300
80 -> Row#400
```

---

# Question: Why is B+ Tree preferred?

Because leaf nodes are linked.

Example:

```
Leaf1 -> Leaf2 -> Leaf3 -> Leaf4
```

Range query:

```
WHERE ageBETWEEN20AND30
```

DB:

1. Finds 20.
2. Walks leaf nodes sequentially.

Very fast.

---

B-Tree:

```
May need tree traversal repeatedly.
```

B+ Tree:

```
Sequential leaf scanning.
```

Much better for databases.

That's why:

- PostgreSQL uses B+ Tree style indexes
- MySQL InnoDB uses B+ Tree
- SQL Server uses B+ Tree

---

# Mental Model for Interviews

Think:

```
Hash Index
```

Like:

```
Dictionary
```

You can instantly find:

```
"Apple"
```

but not:

```
All words after "Apple"
```

---

Think:

```
B+ Tree
```

Like:

```
A sorted phone book
```

You can quickly find:

```
Smith
```

or

```
All names from Smith to Taylor
```

because everything is ordered.

> **B+ Tree is almost always better for databases**, which is why nearly all modern relational databases use B+ Trees (or a variation of them) for their primary index implementation.
> 

---

# Question: Which is better, B-Tree or B+ Tree?

For databases:

```
B+ Tree wins
```

For general computer science:

```
Depends on use case
```

Databases care heavily about:

- Disk reads
- Range queries
- ORDER BY
- BETWEEN
- Sequential scanning

B+ Tree is optimized for all of these.

---

# Question: Which databases use B+ Tree today?

### PostgreSQL

Default index:

```
B-tree
```

But PostgreSQL's implementation behaves much closer to a **B+ Tree** than the textbook B-Tree.

---

### MySQL (InnoDB)

Uses:

```
B+ Tree
```

for primary and secondary indexes.

---

### SQL Server

Uses:

```
B+ Tree
```

---

### Oracle

Uses:

```
B+ Tree
```

---

### MariaDB

Uses:

```
B+ Tree
```

---

### DB2

Uses:

```
B+ Tree
```

---

So practically:

```
Almost every major RDBMS uses B+ Tree
```

---

# Question: Why did databases choose B+ Tree instead of B-Tree?

Because database workloads look like:

```
WHERE ageBETWEEN20AND30

ORDERBY created_at

WHERE emailLIKE'abc%'
```

These are all:

```
Range operations
```

B+ Tree is designed specifically for these.

---

# Question: Let's compare searching in B-Tree and B+ Tree

Suppose indexed values:

```
10
20
30
40
50
60
70
80
90
```

---

## B-Tree

Data may exist in every level.

```
              [50]
             /    \
        [20,30]   [70,80]
         /   \      /   \
      [10] [40] [60] [90]
```

Searching:

```
WHERE id=50
```

Found immediately:

```
Root node
```

Done.

---

## B+ Tree

Internal nodes only help navigation.

```
              [50]
             /    \
         [30]      [70]
          |          |
--------------------------------
10 -> 20 -> 30 -> 40 -> 50 -> 60 -> 70 -> 80 -> 90
--------------------------------
```

All actual row references exist in leaves.

Searching:

```
WHERE id=50
```

Traversal:

```
Root
 ↓
Leaf
 ↓
Found
```

One extra step.

---

# Question: Doesn't B-Tree seem faster then?

For a single lookup:

Sometimes yes.

Example:

```
WHERE id=50
```

B-Tree might find it sooner.

But databases rarely care only about one-row lookups.

---

# Question: What happens for range queries?

Example:

```
WHERE idBETWEEN40AND80
```

---

## B-Tree

Need to repeatedly traverse.

```
Find 40

Go back up

Find 50

Go back up

Find 60

Go back up
```

Lots of tree navigation.

---

## B+ Tree

Leaf nodes are linked.

```
40 -> 50 -> 60 -> 70 -> 80
```

Database does:

```
Find 40
Walk right
Walk right
Walk right
Walk right
```

Done.

Extremely efficient.

---

# Visual Comparison

## B-Tree

```
              [50]
             /    \
        [20,30]   [70,80]
         /   \      /   \
      [10] [40] [60] [90]
```

Data exists everywhere.

---

## B+ Tree

```
             [50]
            /    \
         [30]    [70]

                |
                V

------------------------------------------------
10 -> 20 -> 30 -> 40 -> 50 -> 60 -> 70 -> 80 -> 90
------------------------------------------------
```

All real records live at leaf level.

Leaves linked together.

---

# Question: Why does B+ Tree use less memory?

Suppose keys:

```
1
2
3
4
5
```

B-Tree may store row information in:

```
Root
Intermediate nodes
Leaves
```

Data repeated across tree.

---

B+ Tree stores:

```
Internal nodes:
Only keys

Leaves:
Keys + row pointers
```

Internal nodes become much smaller.

Example:

```
Internal node

10
20
30
40
```

instead of:

```
10 + row info
20 + row info
30 + row info
40 + row info
```

Smaller nodes mean:

```
More keys per page
```

which means:

```
Shorter tree
```

which means:

```
Fewer disk reads
```

---

# Question: Why do databases care about disk reads?

Because disk access is expensive.

Imagine:

```
Memory read:
0.0001 ms

Disk read:
0.1 - 10 ms
```

Thousands of times slower.

Database designers optimize for:

```
Fewer disk page reads
```

not:

```
Fewer CPU operations
```

This is why B+ Tree became dominant.

---

# Question: What happens during ORDER BY?

Query:

```
SELECT*
FROM users
ORDERBY email;
```

---

With B+ Tree:

```
Leaf nodes already sorted
```

Database can simply walk:

```
Leaf1
 ↓
Leaf2
 ↓
Leaf3
```

No sorting needed.

---

With B-Tree:

More navigation required.

---

# Interview Question: Why are leaf nodes linked in B+ Tree?

Because of:

```
BETWEEN
ORDERBY
GROUPBY
LIKE'abc%'
```

Example:

```
WHERE idBETWEEN100AND1000
```

DB:

```
Find 100

Walk linked leaves

Until 1000
```

No additional tree traversals.

---

# Real Production Example

Suppose your parking system has:

```
SELECT*
FROM trips
WHERE created_atBETWEEN
'2025-01-01'
AND
'2025-01-31';
```

Millions of rows.

B+ Tree:

```
Find Jan 1

Walk linked pages

Until Jan 31
```

Extremely efficient.

This is one reason databases can return huge date-range reports quickly.

One small correction to something you may hear in interviews: when PostgreSQL says its default index type is **"B-tree"**, people often assume it means the classic textbook B-Tree. Internally, PostgreSQL stores tuples only in leaf pages and uses internal pages for navigation, making it much closer to a B+ Tree in behavior than many textbook diagrams suggest. That's why PostgreSQL gets the same range-scan benefits as MySQL and SQL Server.
