# **Concurrency Control**

## **Scenario: Many Concurrent Requests Trying to Book the Same Movie Theatre Seat**

### **Critical Section**

```pseudo
{
    Read Seat row with ID
    if Free:
        Logic to change the status from free to booked
        Update the DB
}
```

### **What Will Happen**

- All 3 requests will see the seat as free
- All 3 will change the status to booked
- All 3 would receive success
- The same seat is allocated to 3 users

---

## **Approach 1: Synchronized Block**

```pseudo
synchronized()
{
    Read Seat row with ID
    if Free:
        Logic to change the status from free to booked
        Update the DB
}
```

### **What Will Happen**

- First, User 1’s request is processed, and the seat status changes from free → booked
- User 2 and User 3 will not be able to book the seat

### **When Will It Work**

- Centralized system

### **When Will It Not Work**

- Distributed system

---

## **More Detailed Scenario**

### **Difference Between a Process and a Thread**

- One **process** can have multiple **threads**
- In a centralized system with a single process, `synchronized` will work (shared memory space)
- In a distributed system (e.g., load balancer with 2 servers), there are 2 separate **processes**, so both can run in parallel and bypass the synchronization of each other

---

## **Approach 2: Distributed Concurrency Control**

### **Prerequisites**

#### **1. What Is the Usage of Transactions?**

Transactions help achieve **data integrity**. They ensure the database remains **consistent** even in failure scenarios.

**Example:**

Initial state:
A has 150 Rs
B has 0 Rs

Goal: Debit 50 Rs from A and credit it to B

**Transaction Block:**

```sql
BEGIN TRANSACTION:
    Debit 50 Rs from A
    Credit 50 Rs to B   -- suppose this fails
IF all success:
    COMMIT
ELSE:
    ROLLBACK
END TRANSACTION
```

- If there is any failure, it **rolls back** all previous steps to keep the DB in a consistent state
- Without transactions, a partial update could leave the DB **inconsistent**

---

#### **2. What Is DB Locking?**

Locking ensures **concurrent transactions** don't interfere with each other by protecting data from conflicting reads/writes.

**Types of Locks:**

1. **Shared Lock (S)** – Read lock

   - Allows multiple transactions to read
   - Prevents write

2. **Exclusive Lock (X)** – Write lock

   - Prevents both read and write from other transactions

**Important Notes:**

- If there is a **shared lock**, an **exclusive lock** cannot be granted until the shared lock is removed
- If there is an **exclusive lock**, no other transaction can be granted either a shared or exclusive lock

---

#### **3. What Are the Isolation Levels?**

ACID property **I - Isolation** refers to how/if transactions are isolated from each other.

| **Isolation Level**      | **Dirty Read Possible** | **Non-repeatable Read Possible** | **Phantom Read Possible** |
| ------------------------ | ----------------------- | -------------------------------- | ------------------------- |
| **Read Uncommitted (0)** | Yes                     | Yes                              | Yes                       |
| **Read Committed (1)**   | No                      | Yes                              | Yes                       |
| **Repeatable Read (2)**  | No                      | No                               | Yes                       |
| **Serializable (3)**     | No                      | No                               | No                        |

**Note:**

- **Consistency** increases from top to bottom
- **Concurrency** decreases from top to bottom

---

### **Locking Strategy by Isolation Level**

- **Read Uncommitted**

  - No lock required

- **Read Committed**

  - **Read:** Shared lock acquired and released immediately after the read
  - **Write:** Exclusive lock acquired and held till end of transaction

- **Repeatable Read**

  - **Read:** Shared lock acquired and released at the end of the transaction
  - **Write:** Exclusive lock acquired and held till the end of the transaction

- **Serializable**

  - Same as **Repeatable Read**
  - Additionally applies **range locks** (locks on sets of rows)
  - All locks are released only at the end of the transaction

---

### **Examples of Isolation Issues**

---

#### **Dirty Read**

A transaction may read data that another uncommitted transaction has modified.

**Example:**

```text
T1
{
    Read status of seat → sees "booked"
}

T2
{
    Update status to "booked"
    Not yet committed
    Eventually commits
}
```

- `T1` reads a status modified by `T2` before it is committed
- If `T2` fails, `T1` has read an invalid state

---

#### **Non-repeatable Read**

A transaction reads the same data twice and gets different values due to concurrent modifications.

**Example:**

```text
T1
{
    Read status of seat → sees "free"
    ...
    Read status again → sees "booked"
}

T2
{
    Updates status from "free" to "booked"
    Commits in between T1's two reads
}
```

- The second read by `T1` sees a different value due to `T2`'s commit

---

#### **Phantom Read**

A transaction re-executes a query and finds rows that weren’t there before.

**Example:**

DB State:

```text
ID:1, Status: Free
ID:3, Status: Booked
```

```text
T2
{
    Query: SELECT * FROM seats WHERE ID > 0 AND ID < 5
    → Returns rows: 1, 3
}
```

Meanwhile:

```text
T3
{
    INSERT INTO seats (ID, Status) VALUES (2, 'Booked')
}
```

Then:

```text
T2
{
    Query: SELECT * FROM seats WHERE ID > 0 AND ID < 5
    → Returns rows: 1, 2, 3 (phantom row appears)
}
```

---

### **Important Note**

- If **only reads** are required and **high consistency** is needed, **locking may not be necessary**, depending on the use case

---

## **Distributed Concurrency Control**

---

### **1. Optimistic Concurrency Control (OCC)**

- Isolation Level: **Read Uncommitted**, **Read Committed**
- Provides high concurrency
- Assumes conflict is rare
- If conflict occurs, transaction is **rolled back and retried**
- No locks held for long durations

---

### **2. Pessimistic Concurrency Control (PCC)**

- Isolation Level: **Repeatable Read**, **Serializable**
- Acquires locks early and holds until transaction ends
- Prevents conflicts by blocking access
- **Can lead to deadlocks**
- Used when high integrity is critical and conflicts are likely

---

## **Difference Between Optimistic and Pessimistic Concurrency Control**

| **Aspect**               | **Optimistic Concurrency Control (OCC)** | **Pessimistic Concurrency Control (PCC)**  |
| ------------------------ | ---------------------------------------- | ------------------------------------------ |
| **Isolation Level Used** | Below **Repeatable Read**                | **Repeatable Read** and **Serializable**   |
| **Concurrency**          | High                                     | Lower                                      |
| **Deadlock Possibility** | No                                       | Possible                                   |
| **Conflict Handling**    | Overhead of rollback and retry logic     | Transactions may rollback due to deadlocks |
| **Locking Strategy**     | No locks until commit                    | Long locks held                            |
| **Timeout Issues**       | Rare                                     | Possible due to long locks                 |

---
