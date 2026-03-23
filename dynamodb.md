# DynamoDB Expert Roadmap & SQL Migration Guide

This guide focuses on high-scale architecture, internal mechanics, and the strategic shift from relational to non-relational modeling.

---

## 1. Deep-Dive Topics for Senior Architects

To master DynamoDB at a 10-year experience level, you must move beyond CRUD and understand the "Distributed Systems" nature of the service.

### 1.1 Architectural Internals

#### 1.1.1 Partitioning & Consistent Hashing
1. **The "Mailbox" Analogy (Consistent Hashing)**
    * The same key always produces the same hash, so DynamoDB always knows exactly where the data is stored.
    * Example: Partition Key → Hash Function → Correct Storage Location (123 → Hash → 845723 → Mailbox D)


2. **Physical Storage Nodes**
    * DynamoDB keeps 3 copies of each partition.
    * Copies are stored in 3 different Availability Zones.
    * One server is Leader (writes).
    * Others are Followers (backups).
    * If a server fails → another server automatically takes over.
    * Your application never notices the failure.


3. **The 10GB Limit & Schema Design**
    * Each partition can store up to 10GB of data.
    * Millions of users → all data goes to one partition → can exceed 10GB. Example: Partition Key = Country: USA
    * Use high-cardinality keys (many unique values). Example Partition Key = UserID or OrderID

4. **Throughput (RCU/WCU) and Partitions**
    * Write capacity 1000 WCU/sec
    * If you send: 1500 writes/sec where **(Max = 1000 WCU)** then 1000 writes succeed✅ | 500 throttled ❌
    * Read capacity 3000 RCU/sec

#### 1.1.2 Adaptive Capacity & Bursting
* Research how DynamoDB handles "Hot Partitions" by rebalancing throughput dynamically across the fleet.
* A Hot Partition = one partition getting much more traffic than others.
* DynamoDB monitors traffic across all partitions.
* If some partitions are idle, DynamoDB can temporarily give their unused capacity to the busy partition.

**Example:** Suppose a table has 3000 WCU and 3 partitions:
* Partition A → 1000 WCU
* Partition B → 1000 WCU
* Partition C → 1000 WCU

**Traffic:**
* Partition A → 1500 writes/sec
* Partition B → 50
* Partition C → 50

*Normally A would throttle at 1000 WCU, but Adaptive Capacity may rebalance:*
* Partition A → ~1500 WCU
* Partition B → ~750 WCU
* Partition C → ~750 WCU

* **Bursting:** If traffic suddenly jumps for a short time, DynamoDB may allow temporary extra capacity using previously unused capacity.
    
    Example:
    * Normal traffic → 500 writes/sec
    * Sudden spike → 1200 writes/sec

#### 1.1.3 Global Admission Control (GAC)
The mechanism that prevents a single heavy user from impacting the performance of other tenants in the multi-tenant architecture.

#### 1.1.4 B-Tree Indexing
How the Sort Key (SK) is physically stored to allow for efficient range queries (`begins_with`, `between`, etc.).

---

### 1.2 Advanced Data Modeling (The "No-Join" Philosophy)

* **Single-Table Design:** Mastering the art of storing multiple entities (Users, Orders, Products) in one table to satisfy complex access patterns in a single `Query` call.
* **Adjacency Lists:** Modeling Many-to-Many ($N:M$) relationships using a single table by flipping PKs and SKs in a GSI.
* **GSI Overloading:** Reducing costs and staying under the GSI quota by using generic attributes (e.g., `GSI1_PK`, `GSI1_SK`) for different entity types.
* **Sparse Indexes:** Creating GSIs that only include items where a specific attribute is present—ideal for "Needle in a Haystack" searches (e.g., finding only "Processing" orders).

---

### 1.3 Operations & Performance at Scale

#### 1.3.1 DynamoDB Streams vs. Kinesis (Event Processing)
* Building event-driven architectures. Use Streams for Lambda triggers or Kinesis for long-term retention and advanced analytics.
* When data changes in DynamoDB (insert/update/delete), you may want to trigger other actions.
* **Use Streams when:**
    - Trigger AWS Lambda
    - Short-term event processing
    - Simple pipelines
    * *Example:* User creates order → DynamoDB stores order → DynamoDB Stream records event → Lambda processes event → Send email / update analytics
* **Amazon Kinesis is better when:**
    - You need long data retention
    - Real-time analytics
    - Multiple consumers
    * *Example:* Orders → Kinesis → Analytics system → Fraud detection → Dashboard

#### 1.3.2 Transactions (ACID)
Deep dive into the 100-item limit and the 2x cost overhead of `TransactWriteItems`.
* Limits:
    - Max items per transaction = 100
    - Cost ≈ 2x normal write (DynamoDB does extra checks to guarantee consistency.)

#### 1.3.3 Global Tables (Active-Active)
Multi-region replication and conflict resolution (Last Writer Wins).
* With DynamoDB Global Tables, the same table exists in multiple regions.
* Low latency for global users
* High availability
* *Example:* US user writes in US region | Asia user writes in Asia region.
* Replication happens automatically.
* Conflict rule: Last Writer Wins (The latest update overwrites previous ones).

#### 1.3.4 FinOps (Cost Optimization)
1. **On-Demand:** You pay per request. Best for low traffic apps or traffic that changes a lot. No capacity planning needed.
2. **Provisioned Capacity:** You define specific limits (e.g., 2000 reads/sec, 500 writes/sec). Cheaper if traffic is predictable.
3. **Reserved Capacity:** If you know baseline traffic for a long time, reserve capacity for a **50–70% cost reduction**.

#### 1.3.5 DAX (DynamoDB Accelerator)
DAX is an in-memory cache in front of DynamoDB that speeds up reads from milliseconds to microseconds.
* Best for: High read traffic, product catalogs, user sessions.
* **Architecture:** Users → Application → DAX Cache → DynamoDB.
* **Write process:** App → DAX → DAX writes to DynamoDB → DAX updates cache.
* **When NOT to use DAX:** Mostly write operations, very frequently changing data, or when caching is not helpful (e.g., real-time trading systems).

---

## 2. Advanced Security & Governance

* **2.1 Resource-Based Policies:** Allow **Account B** to read data from a DynamoDB table in **Account A**.

    - You write this policy inside DynamoDB (on the table itself) in Account A
    - This policy is NOT written in IAM → Policies, It is directly attached to the DynamoDB table
    - **Account A (111111111111)**  `Owner of the DynamoDB table (`UsersTable`)`
    - **Account B (222222222222)** `External account that needs read access`
    - Policy (How Resource-Based Policies works)

        ```json
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                "Sid": "AllowAccountBReadAccess",
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::222222222222:root"
                },
                "Action": [
                    "dynamodb:GetItem",
                    "dynamodb:Scan",
                    "dynamodb:Query"
                ],
                "Resource": "arn:aws:dynamodb:ap-south-1:111111111111:table/UsersTable"
                }
            ]
        }
        ```

* **2.2 Client-Side Encryption:** Encrypt sensitive user data before storing it in DynamoDB using AWS Database Encryption SDK.
    ```
    How You Do This (Using AWS Database Encryption SDK)
    Step-by-step:
        1. Your app gets a secret key (KMS key)
        2. Before saving:
            - SDK encrypts fields
        3. Store encrypted data in DynamoDB
        4. When reading:
            - SDK decrypts it back
    ```
 

**Example**

```javascript
    const { DynamoDBClient } = require("@aws-sdk/client-dynamodb");
    const { encrypt, decrypt } = require("@aws-crypto/client-node");
    // Step 1: Create encryption config (KMS Key ARN)
    const keyArn = "arn:aws:kms:ap-south-1:111111111111:key/your-key-id";
    // Initialize DynamoDB client
    const dynamo = new DynamoDBClient({ region: "ap-south-1" });
    // Example data (PII)
    const user = {name: "Pradeep",phone: "9876543210"};
    // Step 2: Encrypt data before saving
    const encryptedUser = encrypt(keyArn, user);
    // Step 3: Store encrypted data in DynamoDB
    await dynamo.put({TableName: "UsersTable",Item: encryptedUser});
    // Step 4: Read data from DynamoDB
    const data = await dynamo.get({TableName: "UsersTable"});
    // Step 5: Decrypt data after reading
    const decryptedUser = decrypt(keyArn, data.Item);
    console.log(decryptedUser);
```
---

## 3. Advanced TTL (Time to Live) Mechanics

* **The 48-Hour Deletion Window:** TTL does not delete items the second they expire. DynamoDB typically removes them within **48 hours**.
    * *Expert Tip:* Always include a filter in your queries to ignore items where `ExpirationTime < CurrentTime` to avoid showing "stale" data to users.
* **The Archiving Pattern (TTL + Streams):** When TTL deletes an item, it is recorded in the DynamoDB Stream with a specific `userIdentity` of "DynamoDB Service." You can trigger a Lambda from this stream to archive the data to **Amazon S3** for long-term compliance before it vanishes forever.

---

## 4. The "Filter Expression" Efficiency Trap

* **Filter vs. Query:** A `FilterExpression` is applied **after** the data has been read from disk.
* **The RCU Trap:** You are charged for every item read from the disk *before* the filter is applied. If you read 100 items but the filter removes 99 of them, you still pay for all 100.
* **Optimization:** If you are filtering out more than 20% of your data, you should redesign your **Sort Key (SK)** or create a **Global Secondary Index (GSI)** to target the data more precisely.

---

## 5. Write Sharding (For Ultra-Hot Keys)

* **The Problem:** A single Partition Key is limited to 1,000 WCU (Writes/sec). If you use a low-cardinality key (e.g., `Status: ACTIVE`), you will hit this limit quickly.
* **The Strategy:** Append a random suffix (a "shard") to the key: `ACTIVE#1`, `ACTIVE#2`, `ACTIVE#N`. This spreads the write load across multiple physical partitions.

---

## 6. Enterprise Import/Export Patterns

* **S3 Import (Zero RCU/WCU):** You can import bulk data (CSV, JSON, Ion) directly from S3 into a *new* table. This does not consume any provisioned capacity and is significantly cheaper than running a `BatchWriteItem` script.
* **Zero-ETL with OpenSearch:** DynamoDB now integrates directly with **Amazon OpenSearch**. This allows for full-text search capabilities (like fuzzy matching) on your DynamoDB data without writing custom Lambda "glue" code to sync the two.

---

## 7. Zero-Downtime Migration: "The Drift"

* **Idempotency & Versioning:** During "Dual Writes," network failures can cause retries. Use a `version` attribute or the `attribute_not_exists(PK)` check in your TypeScript code to ensure you don't overwrite newer data with older data during the migration window.
* **Data Drift:** Use a comparison script (Dark Reading) to periodically check if the data in your new DynamoDB table matches the source SQL database before performing the final cutover.

---

### Implementation Snippet (Node.js/TypeScript)

```typescript
// Example: Conditional Write to prevent migration "Drift"
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

async function safeMigrate(userData: any) {
  const command = new PutCommand({
    TableName: "UsersTable",
    Item: userData,
    // Ensure we don't overwrite if a newer version was already written by the app
    ConditionExpression: "attribute_not_exists(PK) OR version < :newVersion",
    ExpressionAttributeValues: {
      ":newVersion": userData.version,
    },
  });
  return await docClient.send(command);
}

## 2. SQL to DynamoDB Migration Strategy

Migrating to NoSQL is a **functional rewrite**, not a simple lift-and-shift.

### Step 1: Document Access Patterns
In SQL, you model data. In DynamoDB, you **model queries**.
1. Identify every API endpoint and UI view.
2. List the required data for each (e.g., "Get User by ID," "Get all Orders for User in the last 30 days").
3. Determine the "velocity" (read/write frequency) of each pattern.

### Step 2: Denormalization & Schema Design

* **Flatten the data:** If you need "Product Name" whenever you view an "Order," store the Product Name inside the Order item.
* **Primary Key Design:** Define your Partition Key (high cardinality) and Sort Key (for range queries).
* **Pre-calculate:** Instead of `SUM()` or `COUNT()` at runtime, update a counter attribute every time an item is added.

### Step 3: Migration Execution (The Zero-Downtime Pattern)

| Phase | Action | Purpose |
| :--- | :--- | :--- |
| **1. Dual Write** | Application writes to **both** SQL and DynamoDB. | Ensures new data is captured in both places. |
| **2. Backfill** | Use AWS Glue or DMS to migrate historical data. | Move the "cold" data without affecting production. |
| **3. Verification** | Run "Dark Reads" (query both, compare results). | Validate that the NoSQL schema returns correct data. |
| **4. Cutover** | Switch the application "Source of Truth" to DynamoDB. | Fully retire the SQL database. |

---
