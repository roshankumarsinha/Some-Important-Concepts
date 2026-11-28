# Some Important Concepts

A collection of notes on important concepts related to LLD, HLD etc.

## Table of Contents

- [Bloom Filter](#bloom-filter)
- [Indexing, B-Trees and B+ Trees](#indexing-b-trees-and-b-trees)

---

## Bloom Filter

A **Bloom filter** is a space-efficient, probabilistic data structure used to test whether an element **is possibly in a set** or **is definitely not in a set**.

> 🔗 Reference discussion: [claude.ai/share/5c296f2e-72f9-47f1-8681-171f0e8d965c](https://claude.ai/share/5c296f2e-72f9-47f1-8681-171f0e8d965c)

### Key Properties

- **No false negatives**: If the filter says an element is _not present_, it is guaranteed to not be in the set.
- **Possible false positives**: If the filter says an element _is present_, it might actually not be — there's a tunable probability of error.
- **Space efficient**: Uses a bit array instead of storing actual elements, making it much smaller than the underlying set.
- **No deletions** (in the basic version): You cannot remove an element once added, since bits may be shared between multiple elements. (A _Counting Bloom Filter_ variant supports deletion.)
- - **Constant-time operations**: Both insert and lookup take O(k) time, where k is the number of hash functions, no matter how many elements are stored.

### How It Works

1. Start with a bit array of size `m`, all bits initialized to `0`.
2. Use `k` independent hash functions, each mapping an element to one of the `m` positions.
3. **To add an element**: Hash it with all `k` hash functions and set the corresponding `k` bits to `1`.
4. **To check membership**: Hash the element with all `k` hash functions.
   - If **any** of the `k` bits is `0` → the element is **definitely not** in the set.
   - If **all** `k` bits are `1` → the element is **probably** in the set (could be a false positive).

### Example

```
Bit array (m = 10): [0 0 0 0 0 0 0 0 0 0]

Add "apple" -> hash1=1, hash2=4, hash3=7
Bit array:          [0 1 0 0 1 0 0 1 0 0]

Add "banana" -> hash1=3, hash2=4, hash3=9
Bit array:          [0 1 0 1 1 0 0 1 0 1]

Check "apple"  -> bits 1,4,7 are all 1 -> possibly present ✅
Check "grape"  -> hash1=2 -> bit 2 is 0 -> definitely not present ❌
```

### Complexity

| Operation | Time Complexity |
| --------- | --------------- |
| Insert    | O(k)            |
| Lookup    | O(k)            |
| Space     | O(m) bits       |

Where `k` is the number of hash functions and `m` is the bit array size (both constants relative to input size).

---

## Indexing, B-Trees and B+ Trees

### 1. How a Disk Stores Data

Think of a disk like a **dartboard**.

```
            Sector 0
          ┌─────────┐
     S5  /  ┌─────┐  \  S1
        │  /  ┌─┐  \  │         Rings  = TRACKS  (track 0, 1, 2 ...)
        │ │   │ │   │ │         Slices = SECTORS (sector 0, 1, 2 ...)
        │  \  └─┘  /  │
     S4  \  └─────┘  /  S2      Ring ∩ Slice = one BLOCK
          └─────────┘
            Sector 3
```

| Term           | Simple meaning                    | Example                     |
| -------------- | --------------------------------- | --------------------------- |
| **Track**      | One ring on the disk              | Track 2                     |
| **Sector**     | One pie slice                     | Sector 1                    |
| **Block**      | Where a ring and a slice meet     | Block = (Track 2, Sector 1) |
| **Block size** | How much one block holds          | Assume **512 bytes**        |
| **Offset**     | Position of a byte inside a block | 0 to 511                    |

**Address of one byte = Track + Sector + Offset**

**How the disk reaches a block:**

- **Spin the disk** to change the **sector**.
- **Move the arm (head)** to change the **track**.

#### Key rule

The disk always reads and writes **one whole block at a time**, never one byte.

> **So the cost of any search = number of blocks we read.**
> Everything in these notes is about **reducing block reads**.

### 2. Disk vs RAM

```
   ┌──────────┐   bring block in    ┌──────────┐
   │   DISK   │ ──────────────────► │   RAM    │ ◄── program works here
   │ (big,    │ ◄────────────────── │ (small,  │
   │  slow)   │   write result back │  fast)   │
   └──────────┘                     └──────────┘
```

- A program **cannot** use data directly on disk. Data must first come into RAM.
- **Data Structures** = organising data **in RAM**.
- **DBMS (databases)** = organising data **on disk**.

B-Trees belong to the second group: they organise data on disk.

### 3. How a Table Sits on Disk

**Example: an `Employee` table with 100 rows.**

| Field             | Size          |
| ----------------- | ------------- |
| Employee ID       | 10 bytes      |
| Name              | 50 bytes      |
| Department        | 10 bytes      |
| (two more fields) | ...           |
| **Total per row** | **128 bytes** |

Now some easy maths:

```
Assume each row takes 128 bytes and each block is 512 bytes.
Rows per block  = 512 / 128 = 4 rows
Blocks needed   = 100 / 4   = 25 blocks
```

```
 Block 1        Block 2        Block 3              Block 25
┌─────────┐   ┌─────────┐   ┌─────────┐          ┌──────────┐
│ Row 1   │   │ Row 5   │   │ Row 9   │          │ Row 97   │
│ Row 2   │   │ Row 6   │   │ Row 10  │   ....   │ Row 98   │
│ Row 3   │   │ Row 7   │   │ Row 11  │          │ Row 99   │
│ Row 4   │   │ Row 8   │   │ Row 12  │          │ Row 100  │
└─────────┘   └─────────┘   └─────────┘          └──────────┘
```

### 4. Searching Without an Index

Query: _"Find the employee with ID = 87"_

We don't know where it is, so we check block 1, block 2, block 3 ... in the worst case **all 25 blocks**.

> **Without an index: up to 25 block reads.**

Real-life version: finding a name in a register with no order, page by page.

### 5. Index: The Fix

An **index** is a small separate list stored on disk:

```
   INDEX (on disk)                       TABLE (on disk)
┌──────────┬─────────────┐
│ Emp ID   │ Row pointer │
├──────────┼─────────────┤            ┌──────────────────┐
│ 1        │ ────────────┼──────────► │ Row 1 (Block 1)  │
│ 2        │ ────────────┼──────────► │ Row 2 (Block 1)  │
│ ...      │             │            │ ...              │
│ 87       │ ────────────┼──────────► │ Row 87 (Block 22)│
│ ...      │             │            └──────────────────┘
└──────────┴─────────────┘
```

Each entry = **key + row pointer** (where that row lives on disk).

**Dense index** = one index entry **for every row**.

#### Size of the index

```
Entry size        = Key (10 bytes) + Pointer (6 bytes) = 16 bytes
Entries per block = 512 / 16 = 32
Blocks for index  = 100 / 32 = 3.2  →  4 blocks
```

#### Cost of a search now

```
Read index (max 4 blocks)  +  read the 1 block that has the row
        = 4 + 1 = 5 block reads
```

> **25 reads → 5 reads.** That's the power of an index.

Real-life version: the **index page at the back of a textbook**. Find the word there, jump straight to the page.

### 6. Multi-Level Index: An Index on the Index

**New problem:** the table grows to **1,000 rows**.

```
Table blocks  = 1000 / 4  = 250 blocks
Index blocks  = 1000 / 32 ≈ 32 blocks
```

(The video rounds the index up to about 40 blocks. The idea is the same.)

Now even **searching the index** is slow. **Fix:** build a smaller index **above** it.

The upper index does **not** store every key. It stores **one entry per block** of the lower index:

- Entry `1` → points to the index block holding keys 1–32
- Entry `33` → points to the index block holding keys 33–64
- and so on.

**Sparse index** = one entry **per block**, not per row.

```
 Level 2 (sparse)     Level 1 (dense)              Table
 ┌──────────┐        ┌───────────────┐          ┌────────────┐
 │  1   ────┼──────► │ 1, 2, ... 32  │ ───────► │  Row data  │
 │  33  ────┼──────► │ 33, ... 64    │ ───────► │  Row data  │
 │  65  ────┼──────► │ 65, ... 96    │ ───────► │  Row data  │
 │  ...     │        │ ...           │          │  ...       │
 └──────────┘        └───────────────┘          └────────────┘
   1–2 blocks           ~32 blocks                250 blocks
```

**Search cost:**

```
Level 2 index  →  1 block of Level 1 index  →  1 block of table
   ~1-2 reads   +        1 read              +     1 read
              ≈ 3 to 4 block reads   (instead of 250!)
```

> **Each new level cuts the number of block reads.**

#### Turn it sideways, and it's a TREE

```
                 ┌───────────┐
                 │  Level 2  │            ◄── top (root)
                 └─────┬─────┘
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     ┌─────────┐  ┌─────────┐  ┌─────────┐
     │ Level 1 │  │ Level 1 │  │ Level 1 │  ◄── bottom (leaves)
     └────┬────┘  └────┬────┘  └────┬────┘
          ▼            ▼            ▼
       table rows   table rows   table rows
```

### 7. What We Really Want

We don't want a person to watch the table and add index levels by hand.

We want a **self-managing multi-level index**:

- Table **grows** → a new top level is **added automatically**.
- Table **shrinks** → the top level is **removed automatically**.

> **This is exactly what a B-Tree does.**

### 8. From Binary Search Tree to M-Way Search Tree

#### Binary Search Tree (BST): 1 key, 2 children

```
          [20]
         /    \
      [10]    [30]        smaller → left,  bigger → right
```

#### M-Way Search Tree: many keys, many children

```
               [ 20 | 50 ]
             /      |      \
     [10 | 15]  [30 | 35]  [60 | 90]
      < 20       20..50       > 50
```

**Search for 30:** 30 > 20 and 30 < 50, so go to the **middle** child. Found.

| Term               | Meaning                                    |
| ------------------ | ------------------------------------------ |
| **M (degree)**     | Max number of **children** a node can have |
| Max keys in a node | **M − 1**                                  |
| Example above      | 3 children, 2 keys → **3-way** tree        |
| BST                | Just an M-way tree with **M = 2**          |

#### Node layout (4-way example)

```
┌─────┬──────┬─────┬──────┬─────┬──────┬─────┐
│ C1  │ Key1 │ C2  │ Key2 │ C3  │ Key3 │ C4  │      C = child pointer
└─────┴──────┴─────┴──────┴─────┴──────┴─────┘
   4 child pointers, 3 keys
```

#### Used as a database index: add a row pointer per key

```
┌────┬────────────┬────┬────────────┬────┬────────────┬────┐
│ C1 │ Key1 + R1  │ C2 │ Key2 + R2  │ C3 │ Key3 + R3  │ C4 │    R = row pointer
└────┴────────────┴────┴────────────┴────┴────────────┴────┘
```

### 9. The Problem with a Plain M-Way Tree

There are **no rules** for building it.

**Example:** a 10-way tree (each node can hold up to 9 keys). Insert 10, 20, 30 ...

Nothing stops you from making a **new node for every key**:

```
 [10]
    \
    [20]
       \
       [30]
          \
          [40]      ◄── a long chain, like a linked list
```

- Each node could hold 9 keys, but holds only 1.
- Height ≈ number of keys.
- Searching becomes as slow as **linear search**.

> Problem: no control over **height**. Fix: add **rules**, which gives a **B-Tree**.

### 10. B-Tree = M-Way Search Tree + Rules

| #   | Rule                                                                   | Why                                                              |
| --- | ---------------------------------------------------------------------- | ---------------------------------------------------------------- |
| 1   | Every node must be **at least half full**: at least **⌈M/2⌉ children** | Stops the tree from becoming tall and empty                      |
| 2   | **Root** is the exception: it can have just **2 children** (1 key)     | Otherwise we couldn't even insert the first key                  |
| 3   | **All leaves are at the same level**                                   | Every search takes the same number of steps (balanced)           |
| 4   | The tree is **built bottom-up**                                        | It grows **upward** from the leaves, so leaves always stay level |

Example: **M = 10** means each non-root node needs at least **5 children** (4 keys).

### 11. Building a B-Tree Step by Step (Degree 4)

**Degree 4** means max **4 children** and **3 keys** per node.

**Split rule:** when a full node gets a 4th key:

1. Line up all 4 keys in order.
2. Split into **left** and **right** nodes.
3. Push **one middle key up** to the parent (make a new root if there's no parent).

#### Insert: 10, 20, 40

```
[10 | 20 | 40]          ◄── full
```

#### Insert 50 → Split #1

```
Line up: 10, 20, 40, 50   →   40 goes UP

            [40]                 ◄── new root (tree grew UPWARD)
           /    \
    [10 | 20]   [50]
```

#### Insert 60, 70

```
            [40]
           /    \
    [10 | 20]   [50 | 60 | 70]   ◄── right leaf now full
```

#### Insert 80 → Split #2

```
Line up: 50, 60, 70, 80   →   70 goes UP

             [40 | 70]
           /     |     \
   [10 | 20] [50 | 60]  [80]
```

#### Insert 30

```
             [40 | 70]
           /     |     \
 [10|20|30]  [50 | 60]  [80]      ◄── left leaf now full
```

#### Insert 35 → Split #3

```
Line up: 10, 20, 30, 35   →   30 goes UP

              [30 | 40 | 70]               ◄── root now full
           /     |      |     \
    [10|20]   [35]   [50|60]   [80]
```

#### Insert 5

```
              [30 | 40 | 70]
           /     |      |     \
   [5|10|20]   [35]   [50|60]   [80]       ◄── left leaf full again
```

#### Insert 15 → Split #4, and it CASCADES upward

**Step A:** leaf overflows. Line up 5, 10, 15, 20, and **15 goes up**.

**Step B:** the root becomes 15, 30, 40, 70, which is too many! It splits too, and **40 goes up** as the new root.

```
                       [40]                    ◄── NEW ROOT (level 1)
                    /        \
            [15 | 30]          [70]            ◄── level 2
           /    |    \        /    \
      [5|10]  [20]  [35]  [50|60]  [80]        ◄── leaves (level 3)
```

**What happened:**

- The tree is now **3 levels** tall.
- It grew **upward**, never downward.
- **All leaves are still on the same level.**
- The new levels were created **automatically**. This is the self-managing multi-level index from Section 7.

> **Which key goes up?** When 4 keys split, you can send up the 2nd or the 3rd. Either is fine. It only decides whether the tree leans left or right.

#### Insertion flowchart

```
            ┌──────────────────┐
            │  Insert key K    │
            └────────┬─────────┘
                     ▼
        ┌──────────────────────────┐
        │ Go down to the right leaf│
        └────────────┬─────────────┘
                     ▼
            ┌──────────────────┐   yes   ┌───────────────────┐
            │ Leaf has space?  │ ──────► │ Put K in sorted   │──► Done
            └────────┬─────────┘         │ position          │
                     │ no                └───────────────────┘
                     ▼
        ┌──────────────────────────┐
        │ SPLIT: middle key goes UP│
        │ to the parent            │
        └────────────┬─────────────┘
                     ▼
            ┌──────────────────┐   yes
            │ Parent has space?│ ──────► Done
            └────────┬─────────┘
                     │ no
                     ▼
         Split the parent too (repeat upward).
         If the ROOT splits → new root → tree grows 1 level taller.
```

### 12. B-Tree Node as a Database Index

In a B-Tree, **every key in every node** has its own **row pointer**:

```
             ┌─────────────────────┐
             │ 40 (+ row pointer)  │
             └─────────────────────┘
               /                \
 ┌──────────────────────┐   ┌──────────────────┐
 │15(+ptr) | 30(+ptr)   │   │ 70 (+ptr)        │
 └──────────────────────┘   └──────────────────┘
          ...                       ...
```

Each node holds 3 things:

1. **Keys**
2. **Child pointers**: point to child nodes (disk blocks)
3. **Row pointers**: one per key, pointing to the real row on disk

### 13. B+ Tree: The Version Databases Actually Use

Make **3 changes** to the B-Tree:

| #   | Change                                                                                |
| --- | ------------------------------------------------------------------------------------- |
| 1   | **Row pointers only in leaf nodes.** Upper nodes hold only keys, as signposts.        |
| 2   | **Every key has a copy in a leaf.** So 15, 30, 40, 70 also appear down in the leaves. |
| 3   | **Leaves are linked** left to right, like a linked list.                              |

#### Same keys, as a B+ Tree

```
                          [40]                        ◄── signposts only
                       /        \                         (sparse index)
               [15 | 30]          [70]
              /    |    \        /    \
             ▼     ▼     ▼      ▼      ▼
         [5|10]→[15|20]→[30|35]→[40|50|60]→[70|80]    ◄── ALL keys + row pointers
                                                           (dense index)
```

Notice:

- 15, 30, 40, 70 appear **twice**: once as a signpost above, once in a leaf.
- Every key is in the leaf row, so the leaf level is a **dense index**.
- The upper levels are **sparse indexes**.
- This is **exactly the multi-level index from Section 6**, which is why databases love B+ Trees.

#### Why the linked leaves matter: range queries

Query: _"Give me all employees with ID between 20 and 60."_

```
1. Go down the tree to find 20               (a few block reads)
2. Then just walk RIGHT along the leaves:

   [15|20] → [30|35] → [40|50|60] → stop (next leaf starts at 70)
       ▲
     start
```

No need to go back up the tree. It's like reading a sorted list.

### 14. B-Tree vs B+ Tree

| Point          | B-Tree                                  | B+ Tree                                                       |
| -------------- | --------------------------------------- | ------------------------------------------------------------- |
| Row pointers   | In **every** node                       | **Only in leaves**                                            |
| Keys           | Each key appears **once**, at any level | **Every key is in a leaf**. Upper nodes may repeat some keys. |
| Leaves linked? | No                                      | **Yes**, like a linked list                                   |
| Range queries  | Harder (jump up and down the tree)      | **Easy** (walk along the leaves)                              |
| As an index    | –                                       | Leaves = **dense**, upper levels = **sparse**                 |
| Used by        | Less common for DB indexes              | **MySQL, PostgreSQL, SQL Server, Oracle**                     |

### 15. Why Not Index Every Column?

If indexes make search fast, why not put one on **every** column? Because every index is **another B+ Tree** that the database must store and keep up to date.

```
 employee table + 1 index on every column

            ┌──────────────┐
            │  TABLE ROWS  │
            └──────┬───────┘
   ┌──────────┬────┴─────┬───────────┬───────────┐
   ▼          ▼          ▼           ▼           ▼
 B+ Tree   B+ Tree    B+ Tree     B+ Tree     B+ Tree
 (emp_id)  (name)     (dept)      (salary)    (city)

 1 INSERT  =  1 table write  +  5 tree updates
```

#### 1. Writes become slow

Every `INSERT`, `UPDATE` and `DELETE` must also change **every index** on the affected columns.

```
INSERT one employee
  → write the row                        (1 write)
  → insert key into emp_id tree          (may split nodes, Section 11)
  → insert key into name tree            (may split nodes)
  → insert key into dept tree            (may split nodes)
  → ... one more for each index
```

More indexes = more block reads and writes per change. Tables with heavy writes (orders, logs, payments) suffer the most.

#### 2. Extra disk space

Each index stores **key + row pointer** for every row (Section 5). With 10 indexes, the indexes alone can be **bigger than the table**.

#### 3. Less memory for what matters

The database keeps hot index blocks in RAM (the **buffer pool**). Many indexes fight for the same RAM, so the useful ones get pushed out and more reads go to disk.

> **Rule of thumb:** an index speeds up **reads** but slows down **writes** and costs **space**. Add one only when a real, frequent query needs it.
