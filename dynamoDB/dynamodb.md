# DynamoDB Expert Roadmap & SQL Migration Guide
## DynamoDB Architectural Internals & Advanced Modeling

This guide focuses on high-scale architecture, internal mechanics, and the strategic shift from relational to non-relational modeling.

To master DynamoDB at a 10-year experience level, you must move beyond CRUD and understand the "Distributed Systems" nature of the service.
# 📑 Table of Contents

1. [Architectural Internals](#1-architectural-internals)
   - [Partitioning](#11-partitioning--consistent-hashing)
   - [Adaptive Capacity](#12-adaptive-capacity--bursting)
   - [Global Admission Control](#13-global-admission-control-gac)
   - [B-Tree Indexing](#14-b-tree-indexing)
2. [Advanced Data Modeling](#2-advanced-data-modeling-the-no-join-philosophy)
3. [Operations & Performance](#3-operations--performance-at-scale)
4. [Security & Governance](#4-advanced-security--governance)
5. [TTL (Time to Live)](#5-advanced-ttl-time-to-live-mechanics)
6. [Filter Expression Trap](#6-the-filter-expression-efficiency-trap)
7. [Write Sharding](#7-write-sharding)
8. [Import / Export](#8-enterprise-importexport-patterns)
9. [Zero-Downtime Migration](#9-zero-downtime-migration-the-drift)
10. [SQL → DynamoDB Migration](#10-sql-to-dynamodb-migration-strategy)
11. [Performance Optimization](#11-dynamodb-performance-optimization-100k-records)
12. [Consistency Models](#12-consistency-models)
13. [Conditional Writes](#13-conditional-writes--optimistic-locking)
14. [Error Handling](#14-error-handling--retry-strategy)
15. [Pagination](#15-pagination)
16. [Data Types & Limits](#16-data-types--limits)
17. [LSI vs GSI](#17-secondary-index-comparison-lsi-vs-gsi)
18. [PartiQL](#18-partiql-sql-like-queries)
19. [Backup & Restore](#19-backup--restore)
20. [IAM & Access](#20-iam--access-control)
21. [Monitoring](#21-monitoring--observability)
22. [Cost Optimization](#22-cost-optimization)
23. [Multi-Tenant Design](#23-multi-tenant-design-saas)
24. [Time-Series](#24-time-series-pattern)
25. [Large Delete](#25-large-scale-delete-strategy)
26. [Anti-Patterns](#26-anti-patterns-avoid-these)
27. [SDK Best Practices](#27-sdk-best-practices)
28. [Security](#28-security-overview)
29. [Adaptive Capacity](#29-adaptive-capacity)
30. [Hot Partition Detection](#30-hot-partition-detection)
31. [Retry Strategy](#31-retry-strategy-exponential-backoff--jitter)
32. [Idempotency Keys](#32-idempotency-keys)
33. [Data Modeling Patterns](#33-data-modeling-patterns)
34. [Event-Driven Architecture](#34-event-driven-architecture-streams)
35. [Global Tables](#35-global-tables-conflict-resolution)
36. [Capacity Planning](#36-capacity-planning)
37. [Latency Optimization](#37-latency-optimization)
38. [Schema Evolution](#38-schema-evolution)
39. [Disaster Recovery](#39-disaster-recovery-dr)
40. [Data Migration](#40-data-migration-strategy)
41. [Write & Read Amplification](#41-write--read-amplification)
42. [Index Projections](#42-secondary-index-projections)
43. [Distributed Systems Thinking](#43-distributed-system-thinking)
## 1. Architectural Internals

### 1.1 Partitioning & Consistent Hashing

1.  **The "Mailbox" Analogy (Consistent Hashing):**
    The same key always produces the same hash, ensuring DynamoDB knows exactly where the data is stored.
    * **Workflow:** `Partition Key` → `Hash Function` → `Correct Storage Location` (e.g., `123` → `Hash` → `845723` → `Mailbox D`).

2.  **Physical Storage Nodes:**
    DynamoDB maintains **three copies** of each partition, stored across three different **Availability Zones (AZs)** to ensure durability and availability.
    * **Leader:** One server acts as the primary for all write operations.
    * **Followers:** Two servers act as backups and serve eventually consistent reads.
    * **High Availability:** If a node fails, another automatically takes over; this failover process is transparent to the application.

3.  **The 10GB Limit & Schema Design:**
    Each physical partition has a hard storage limit of **10GB**.
    * **Hot Partitions:** If millions of users are mapped to a single partition (e.g., `Partition Key = Country: USA`), you risk exceeding the 10GB limit and exhausting throughput.
    * **High-Cardinality Keys:** It is best practice to use unique values like `UserID` or `OrderID` to distribute data evenly across the keyspace.

        > **Note:** If a partition key collection (all items with the same PK) reaches 10GB, inserting a new record with that same key will fail with an `ItemCollectionSizeLimitExceededException`.

4.  **Throughput (RCU/WCU):**
    Throughput is allocated at the partition level:
    * **Write Capacity:** 1,000 WCU/sec per partition.
    * **Read Capacity:** 3,000 RCU/sec per partition.
    * **Example:** If you send 1,500 writes/sec to a single partition capped at 1,000 WCU:
        * 1,000 writes succeed ✅
        * 500 writes are throttled ❌

### 1.2 Adaptive Capacity & Bursting
DynamoDB manages "Hot Partitions" by rebalancing throughput dynamically across the partition fleet.

* **Adaptive Capacity:** If one partition is under heavy load while others are idle, DynamoDB reallocates unused throughput to the busy partition.
    * **Scenario:** A table with 3,000 total WCU spread across 3 partitions (A, B, C).
    * **Normal Allocation:** A: 1,000 | B: 1,000 | C: 1,000.
    * **Traffic Spike on A:** A receives 1,500 writes/sec.
    * **Adaptive Result:** DynamoDB may shift allocation to A: 1,500 | B: 750 | C: 750 to prevent throttling on A.


* **Bursting:** For short-lived traffic spikes, DynamoDB retains a "burst buffer" of unused capacity (up to 5 minutes) to handle sudden surges.

### 1.3 Global Admission Control (GAC)

GAC is a system that helps manage how requests are handled. Here is a simple breakdown of how it works:

- **What is GAC?** : It acts like a gatekeeper for internal requests. It makes sure everything stays organized.
- **Where does it work?** :It works at the very beginning, before the data reaches the partition.
- **Can we configure it?** : No. You cannot change the settings for GAC yourself.
- **Why does throttling happen?** : Throttling occurs when the system gets too much work at once. This usually happens because of:
    * Too many requests (overload)
    * Poor system design
- **How to avoid issues?** To keep things running smoothly, you should:
    * Use a good Partition Key (PK)
    * Scale your system properly
    * Use a retry plan if a request fails
- Where GAC works (internally)
    ```
    Your API Request
        ↓
    Global Admission Control (GAC)
        ↓
    Partition (where data lives)
        ↓
    Response
    ```
    
### 1.4 B-Tree Indexing
- Within each partition, the **Sort Key (SK)** is physically stored in a B-Tree structure. This enables high-performance range queries using operators like `begins_with`, `between`, and comparison operators (`>`, `<`).
- DynamoDB uses B-Tree on Sort Key to quickly find ordered data inside a partition without scanning

---

## 2. Advanced Data Modeling (The "No-Join" Philosophy)

* **Single-Table Design:** The practice of storing multiple distinct entity types (e.g., Users, Orders, Products) in a single table. This allows you to retrieve all related data for a complex access pattern in a single atomic `Query` call.
* **Adjacency Lists:** A technique for modeling many-to-many ($N:M$) relationships. By using a "Global Secondary Index (GSI) Flip"—where the original SK becomes the PK—you can query relationships from both directions.
* **GSI Overloading:** A strategy to stay under the GSI limit (20 per table) by using generic attribute names (e.g., `GSI1_PK`, `GSI1_SK`). This allows a single index to serve different purposes for different entity types.
* **Sparse Indexes:** By only populating a GSI attribute for specific items, you create a "Sparse Index." This is ideal for "Needle in a Haystack" queries, such as finding only the "In-Progress" orders out of millions of "Completed" ones.
---

## 3. Operations & Performance at Scale

### 3.1 DynamoDB Streams vs. Kinesis (Event Processing)
When data changes in DynamoDB (insert/update/delete), you can trigger downstream actions via event-driven architectures.

* **DynamoDB Streams:** Use for Lambda triggers and short-term event processing.
    * *Example:* User creates order → Stream records event → Lambda processes event → Sends confirmation email.
* **Amazon Kinesis Data Streams:** Better for long-term retention, real-time analytics, and multiple consumers.
    * *Example:* Orders → Kinesis → Analytics system + Fraud detection + Real-time Dashboard.

### 3.2 Transactions (ACID)
DynamoDB supports atomic transactions for complex operations.
* **Limits:** Maximum of 100 items per transaction.
* **Cost:** Approximately 2x the cost of a normal write because DynamoDB performs extra coordination checks to guarantee consistency.
* **How DynamoDB Handles It Internally** When you run a transaction:
    - DynamoDB locks/checks items
    - Verifies no conflicts
    - Performs all operations
    - Commits only if everything is safe
    - If anything fails → rollback

### 3.3 Global Tables (Active-Active)
Provides multi-region replication and conflict resolution.
* **Mechanism:** The same table exists in multiple AWS regions with automatic replication.
* **Benefits:** Low latency for global users and high availability/disaster recovery.
* **Conflict Resolution:** Uses the "Last Writer Wins" rule where the latest timestamped update overwrites previous ones.

### 3.4 FinOps (Cost Optimization)
1.  **On-Demand:** You pay per request. Best for unpredictable traffic or new applications.
2.  **Provisioned Capacity:** You define specific RCU/WCU limits. More cost-effective for predictable, steady-state traffic.
3.  **Reserved Capacity:** Commit to a baseline usage over a 1 or 3-year term for a **50–70% cost reduction**.

### 3.5 DAX (DynamoDB Accelerator)
DAX is a fully managed, highly available, in-memory cache for DynamoDB that delivers up to a 10x performance improvement—reducing response times from milliseconds to **microseconds**, even at millions of requests per second.

#### Architectural Integration
DAX is a **write-through** caching service. Your application points to the DAX cluster endpoint instead of the standard DynamoDB endpoint. 
* **Read-Through:** If the requested data is in the cache (a cache hit), DAX returns it immediately. If not (a cache miss), DAX fetches it from DynamoDB, stores it, and then returns it.
* **Write-Through:** When you write data, it is written to the DAX cache and the DynamoDB table simultaneously. This ensures the cache remains consistent with the underlying table.

#### Primary Use Cases
DAX is specifically engineered for:
* **Read-Heavy Workloads:** Applications with high read-to-write ratios, such as product catalogs, social media feeds, or gaming leaderboards.
* **"Hot" Key Mitigation:** Offloading traffic from frequently accessed items (like a viral news story or a trending product) to prevent RCU throttling on the base table.
* **Extreme Low Latency:** Real-time applications that require sub-millisecond response times for a smooth user experience.

#### When NOT to Use DAX
While powerful, DAX is not a "one-size-fits-all" solution. Avoid DAX if:
* **Write-Heavy Operations:** DAX provides no performance benefit for writes; it may actually add a slight overhead due to the write-through requirement.
* **Strict Strong Consistency:** DAX is optimized for **eventually consistent** reads. While it supports strongly consistent reads, they must be issued to the DynamoDB endpoint directly or they will bypass the cache gain.
* **Cost Sensitivity:** DAX is an additional provisioned resource (clusters of nodes). If millisecond latency (standard DynamoDB) is sufficient for your SLA, the extra cost of a DAX cluster may not be justified.
* **Frequently Changing Data:** If your data expires or changes every few seconds, the cache churn (constant invalidation and reloading) can negate the performance benefits.

---

## 4. Advanced Security & Governance

### 2.1 Resource-Based Policies
Resource-based policies allow you to grant permissions directly to a specific resource. In this architecture, we are enabling **Account B** to read data from a DynamoDB table located within **Account A**.

* **Location:** This policy is written and attached directly to the **DynamoDB Table** (on the resource itself) in Account A.
* **Key Distinction:** Unlike standard identity-based policies, this is **NOT** configured in *IAM → Policies*. It lives on the DynamoDB service side attached to the table.
* **Involved Accounts:**
    * **Account A (111111111111):** The primary owner of the DynamoDB table (`UsersTable`).
    * **Account B (222222222222):** The external/third-party account requiring cross-account read access.

    #### Policy Configuration
    The following JSON policy demonstrates how resource-based permissions are structured to permit secure cross-account access:

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

### 2.2 Client-Side Encryption
Client-side encryption ensures that sensitive user data, such as Personally Identifiable Information (PII), is encrypted **locally within your application** before it is ever transmitted over the network or stored in DynamoDB. This provides the highest tier of data protection; since the data is already ciphertext when it reaches AWS, it remains secure even in the unlikely event that the database layer is compromised.

---

### Workflow using AWS Database Encryption SDK
The implementation involves a secure orchestration between your application, AWS Key Management Service (KMS), and DynamoDB. 


1. **Key Retrieval:** Your application requests a unique data key or references a customer master key (CMK) from **AWS KMS**.
2. **Local Encryption:** Before the data leaves the application's memory space, the **AWS Database Encryption SDK** encrypts the specified sensitive fields using the retrieved key.
3. **Encrypted Storage:** The application sends the ciphertext (encrypted data) to **DynamoDB**. From the database's perspective, this is just an opaque blob or encrypted string.
4. **Decryption on Read:** When the application retrieves the data, the SDK automatically handles the decryption process using the authorized KMS key, allowing the application to work with the original plaintext.

---

### Step-by-Step Implementation (Node.js)

The following example demonstrates how to use the `@aws-crypto/client-node` alongside the standard AWS SDK for JavaScript (v3) to protect sensitive user fields.

```javascript
const { DynamoDBClient } = require("@aws-sdk/client-dynamodb");
const { encrypt, decrypt } = require("@aws-crypto/client-node");
// Step 1: Define the KMS Key ARN (The root of trust)
const keyArn = "arn:aws:kms:ap-south-1:111111111111:key/your-key-id";
// Initialize the standard DynamoDB client
const dynamo = new DynamoDBClient({ region: "ap-south-1" });
// Example data containing Personally Identifiable Information (PII)
const user = {
    name: "Pradeep",
    phone: "9876543210"
};

async function secureDatabaseOperations() {
    try {
        // Step 2: Encrypt data before it leaves the application
        // The SDK handles the interaction with KMS to secure the payload
        const encryptedUser = await encrypt(keyArn, user);
        // Step 3: Store the encrypted "blob" or fields in DynamoDB
        await dynamo.put({TableName: "UsersTable", Item: encryptedUser});
        // Step 4: Retrieve the data from DynamoDB
        const result = await dynamo.get({TableName: "UsersTable",Key: { name: "Pradeep" }});
        // Step 5: Decrypt the data after reading so the app can use it
        const { decryptedUser } = await decrypt(keyArn, result.Item);
        console.log("Decrypted User Data:", decryptedUser);
    } catch (error) {
        console.error("Encryption/Decryption Error:", error);
    }
}
```
---

## 5. Advanced TTL (Time to Live) Mechanics

DynamoDB **Time to Live (TTL)** allows items to "self-destruct" automatically after a specified timestamp. This background process helps manage storage costs by removing stale data without consuming your provisioned throughput.

### The 48-Hour Deletion Window
TTL is a background process and does not guarantee that an item will be deleted the exact second it expires.

* **Timeline:** DynamoDB typically removes expired items within a **48-hour window** from the expiration time.
* **Constraints:** You can designate only **one** TTL attribute per table.
* **Configuration Details:**
    * ⛔ The 48-hour window is **NOT** configurable by the user.
    * ⛔ Deletion is **NOT** guaranteed to happen exactly at the 48-hour mark; it depends on table traffic and system resources.
    * ✅ It is a **best-effort** background process managed internally by AWS.

> **Expert Tip:** To ensure users never see expired data during the 48-hour deletion lag, always include a filter in your application queries (e.g., using a Filter Expression) to ignore items where `ExpirationTime < CurrentTime`.

---

### Enabling TTL on a Table
To use TTL, you must explicitly enable it and designate a specific attribute—formatted as a **Unix Epoch timestamp** in seconds—for DynamoDB to monitor.



**How to Enable (AWS Console):**
1.  Navigate to the **DynamoDB Console** and select your **Table**.
2.  Go to the **Additional settings** tab.
3.  Locate the **Time to Live (TTL)** section.
4.  Click **Enable**.
5.  Enter the **Attribute name** (e.g., `ttl` or `expiration_time`).
6.  Click **Save**.

---

### The Archiving Pattern (TTL + Streams)
When the internal TTL worker deletes an item, the event is captured by **DynamoDB Streams**. This is a powerful architectural pattern used to preserve data for compliance or analytics before it is permanently purged.


**Archiving Workflow:**
1.  **Item Expires:** The Unix timestamp in the designated TTL attribute is reached.
2.  **DynamoDB Deletes:** The background process identifies and removes the item.
3.  **Stream Capture:** A `REMOVE` event is pushed to the DynamoDB Stream.
    * *Identification:* These specific events have a `userIdentity` of `DynamoDB Service`, allowing you to distinguish them from manual deletions performed by users or applications.
4.  **Lambda Trigger:** An AWS Lambda function subscribed to the stream is triggered by the deletion event.
5.  **S3 Archive:** The Lambda function processes the item data and writes it to **Amazon S3** for long-term storage.

**Summary Flow:**
>Item expires → DynamoDB deletes it → Stream records event → Lambda triggers → Save to S3

---

### Understanding DynamoDB Streams
It is important to understand the nature of the streaming capability to use it effectively:

* **Integration:** It is **NOT** a separate AWS service (like Kinesis Data Streams); it is a built-in **feature** of the DynamoDB service.
* **Activation:** It must be enabled at the table level, where you can choose what information is written to the stream (e.g., Keys only, New Image, Old Image, or both).
* **Functionality:** Once enabled, it automatically records a time-ordered sequence of every item-level modification (**Create**, **Update**, **Delete**) in the table for up to 24 hours.
---

## 6. The "Filter Expression" Efficiency Trap

A common misconception in DynamoDB is that a `FilterExpression` reduces the cost of a query. In reality, filtering happens **after** the data has been read from the disk and metered for cost.

### Filter vs. Query: The Execution Order
When you execute a query with a `FilterExpression`, DynamoDB follows this specific sequence:

1.  **Read:** DynamoDB reads data from the partition based on the Partition Key (PK) and any optional Sort Key (SK) conditions defined in the `KeyConditionExpression`.
2.  **Filter:** It then applies your `FilterExpression` to the retrieved items in memory.
3.  **Return:** Only the items that pass the filter are returned to your application.


---

### The RCUs (Read Capacity Units) Trap
To understand the cost impact, consider this scenario involving an `order_table`:

* **Dataset:** `user_123` has **100 orders** stored in a single partition.
* **Request:** You want to retrieve only orders where `status = "DELIVERED"`.
* **FilterExpression:** `status = :s` (where `:s` is "DELIVERED").
* **Result:** Only **1 item** matches the criteria; 99 items are filtered out.

| Perspective | Result |
| :--- | :--- |
| **Expectation** | "I received 1 item, so I should only pay for 1 read." |
| **Reality** | **You pay for 100 reads.** |

Because DynamoDB had to pull all 100 items into memory to evaluate the `status` field, you are billed for the full 100-item read operation.

---

### Optimization Strategies
If your queries are consistently filtering out more than **20% of the data**, your current schema is inefficient. You should move the filtered attribute into the primary schema structure.

#### 1. Redesign the Sort Key (SK)
Incorporate the filter criteria directly into the Sort Key to allow DynamoDB to perform a **Key Condition Expression** instead of a post-read filter.

* **Original Schema:** `PK = user_123`, `SK = order_id`
* **Optimized Schema:** `PK = user_123`, `SK = status#order_id`

By using `BeginsWith(DELIVERED#)`, DynamoDB only reads the matching items from the disk, significantly reducing RCU consumption.

#### 2. Global Secondary Index (GSI)
If you need to query by status across many different users simultaneously, create a GSI to target the data precisely.



* **GSI Partition Key:** `status`
* **GSI Sort Key:** `order_date`

**Result:** When you query `status = DELIVERED` on the GSI, DynamoDB only touches the data that matches that status. There is zero waste, and you only pay for the items you actually retrieve.

---

## 7. Write Sharding (For Ultra-Hot Keys)

Write sharding is a high-scale design pattern used to handle "hot" partition keys that exceed DynamoDB's native throughput limits.

### The Problem: Throughput Limits
In DynamoDB, a single partition key has a hard performance ceiling. If your application attempts to write to the same partition key too frequently, you will encounter **ProvisionedThroughputExceededException**.

* **The Limit:** A single partition key can handle a maximum of **1,000 Write Capacity Units (WCU)** per second.
* **The "Hot Key" Scenario:** If thousands of items share the same Partition Key (PK) simultaneously, they are all routed to the same physical partition.



**Example: High-Concurrency Status Updates**
Imagine storing users with a generic status as the Partition Key:
* **PK:** `ACTIVE`
* **Scenario:** 10,000 users become `ACTIVE` at the exact same moment.
* **Execution:** All 10,000 writes target the single key `ACTIVE`.
* ❌ **Result:** The 1,000 WCU/sec limit is breached, leading to massive throttling and application failures.


---

### The Strategy: Adding Entropy (Sharding)
To solve this, you distribute the load by appending a random suffix—a **shard**—to the Partition Key. This forces DynamoDB to spread the data across multiple underlying physical partitions.



#### Implementation Logic
Instead of using a static key like `ACTIVE`, your application logic should append a shard number: `ACTIVE#N` (where N is a random number between 1 and your total shard count).

**Optimized Distribution:**
Instead of one overloaded key, the load is split across multiple shards:
* **ACTIVE#1:** 2,500 writes
* **ACTIVE#2:** 2,500 writes
* **ACTIVE#3:** 2,500 writes
* **ACTIVE#4:** 2,500 writes

✅ **Result:** The total load of 10,000 writes is successfully processed because no individual shard exceeds the 1,000 WCU limit.

---

### Key Considerations
* **Reading Sharded Data:** When you need to retrieve all "ACTIVE" users, your application must perform parallel queries across all shards (`ACTIVE#1` through `ACTIVE#N`) and merge the results in the application layer.
* **Shard Count:** Choose a shard count that comfortably covers your peak traffic. If you expect 5,000 writes/sec, use at least 6–10 shards to provide a safety buffer against unexpected spikes.
---

## 8. Enterprise Import/Export Patterns

Modern AWS architectures leverage native integrations to move data into and out of DynamoDB without the overhead of custom application logic or performance degradation.

### 6.1 S3 Import (Zero RCU/WCU)
You can perform bulk data imports from **Amazon S3** (CSV, JSON, or Amazon Ion formats) directly into a **new** DynamoDB table. 

* **Cost Efficiency:** This method does not consume any provisioned Read or Write Capacity Units (RCU/WCU). It is significantly cheaper than traditional `BatchWriteItem` scripts or custom Lambda loaders.
* **Infrastructure:** The S3 file acts as the source storage. DynamoDB manages the import job internally to populate the table.



#### How it works:
1.  **Preparation:** Store your data file (e.g., `data.json`) in an S3 bucket.
2.  **Trigger:** Initiate the "Import from S3" job via the DynamoDB console or CLI.
3.  **Processing:** DynamoDB reads the file directly from S3.
4.  **Creation:** DynamoDB automatically creates the target table and populates the items.

#### Configuration Steps:
In the AWS Console:
* Navigate to **DynamoDB**.
* Select **Import from S3** in the sidebar.
* Provide the **S3 URL** (bucket and path).
* Define the **New Table Name**.
* Specify the **Partition Key** (and optional Sort Key).

---

### 6.2 Zero-ETL Integration with Amazon OpenSearch
**Zero-ETL** refers to the ability to sync data between systems automatically without writing, maintaining, or scaling custom data transfer logic (like Lambda functions or Kinesis streams).

* **Synchronization:** Data in DynamoDB is automatically mirrored and kept in sync with **Amazon OpenSearch Service**.
* **Advanced Search:** Enables "Google-like" search capabilities, including fuzzy matching (handling typos), partial word matches, and complex filtering.
* **No Code:** Eliminates the need for custom sync scripts or intermediate services.

#### Search Comparison:
| Feature | Without OpenSearch (DynamoDB Only) | With OpenSearch (Zero-ETL) |
| :--- | :--- | :--- |
| **Example Query** | Searching for "pradip" | Searching for "pradip" |
| **Logic** | Exact match only | Fuzzy/Text matching |
| **Result** | ❌ No result ("pradip" ≠ "Pradeep") | ✅ Matches "Pradeep" |

#### Internal Sync Mechanism:


1.  **Source of Truth:** Data is stored in DynamoDB (e.g., `{"name": "Pradeep"}`).
2.  **Auto-Sync:** Every `Insert`, `Update`, or `Delete` in DynamoDB is automatically propagated to OpenSearch.
3.  **Search Index:** OpenSearch maintains a highly optimized index for fast, complex text queries.

#### Execution Flow:
1.  **User Input:** User types "pradip" into the application UI.
2.  **Backend API:** The application backend receives the request.
3.  **Search Query:** The backend queries **OpenSearch** instead of DynamoDB for the search operation.
4.  **Retrieval:** OpenSearch returns the matching record ("Pradeep") based on relevance.
5.  **Response:** The UI displays the correct result to the user.
---

## 9. Zero-Downtime Migration: "The Drift"

* When you migrate data from SQL → DynamoDB without stopping your app, you write to both systems (dual write).
    

* **Idempotency & Versioning:** During "Dual Writes," network failures can cause retries. Use a `version` attribute or the `attribute_not_exists(PK)` check in your TypeScript code to ensure you don't overwrite newer data with older data during the migration window.
* **Dual Write + Retry Flow: (Request 1)**
    ```
   User updates name = "Pradeep"
        ↓
    Write to SQL ✅
            ↓
    Write to DynamoDB ❌ (network fails)
            ↓
    System retries later...
    ```
* **Another Api Request (Request 2)**
    ```
   User updates name = "Pradeep kumar"
        ↓
    Write to SQL ✅
            ↓
    Write to DynamoDB ✅
    
    Note : As per second Request both db has name = "Pradeep kumar"
    ```
* **❌ Problem (late retry comes back)**
    ```
    Retry old request (Request 1):
    name = "Pradeep"
        ↓
    Write to DynamoDB ✅ (with old data, name="Pradeep")
    ```
* **Final Solution with Versioning Flow**
    ```
    1. Read current data from DB
    → name = "Pradeep", version = 1

    2. User sends update
    → name = "Pradeep Kumar"

    3. App prepares update
    → new version = 2

    4. Send update with condition
    → Update name = "Pradeep Kumar"
    → Set version = 2
    → ONLY IF current version = 1

    5. DB checks condition:
    ✔ If version = 1 → Update succeeds
    ❌ If version ≠ 1 → Reject update
    
    Note : We update data only if the version matches, so outdated requests can’t overwrite newer data.
    ```
* **Data Drift:** Use a comparison script (Dark Reading) to periodically check if the data in your new DynamoDB table matches the source SQL database before performing the final cutover.
    - During migration (SQL → DynamoDB), both DBs are running.
    - Sometimes data becomes different bw both database (data drift)
    - Data Drift Check (Dark Reading Flow)
        ```
        Script execution timing (Every few minutes / hours):
                ↓
        Run comparison script
                ↓
        1. Read batch from SQL
        2. Read same batch from DynamoDB
        3. Compare both
                ↓
        Check:
            - Missing records
            - Incorrect values
                ↓
        Log / Fix mismatches
            - Insert missing records
            - Update incorrect data
                ↓
        Once everything matches → safe to switch (cutover)
        ```

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
```

## 10. SQL to DynamoDB Migration Strategy

Migrating to NoSQL is a **functional rewrite**, not a simple lift-and-shift.

### Step 1: Document Access Patterns
In SQL, you model data. In DynamoDB, you **model queries**.
1. Identify every API endpoint and UI view.
2. List the required data for each (e.g., "Get User by ID," "Get all Orders for User in the last 30 days").
3. Determine the "velocity" (read/write frequency) of each pattern.
    1. Which API is used a lot? (high traffic)
    2. Which is rare?
```
Handle this in 3 steps:
1.JOIN data in SQL
2.Group it in Node.js (reduce)
3.Insert structured JSON into DynamoDB
```

### Step 2: Denormalization & Schema Design

* **Flatten the Data:** In SQL, `Orders` and `Products` are separate tables joined by an ID. In DynamoDB, you store the `Product_Name` and `Price` directly inside the `Order` item. This "duplicates" data but eliminates the need for a JOIN.
* **Primary Key Strategy:** 
    * **Partition Key (PK):** Must be high-cardinality (unique) to spread data across partitions (e.g., `USER_ID`).
    * **Sort Key (SK):** Used to group and filter data (e.g., `TIMESTAMP` or `STATUS#DATE`).
* **Move Calculation to "Write Time":** SQL uses `SELECT COUNT(*)`. This is too expensive for DynamoDB. Instead, when a user clicks "Like," your code should increment a `LikeCount` attribute on the post item immediately.

### Step 3: Migration Execution (The Zero-Downtime Pattern)

| Phase | Action | Purpose |
| :--- | :--- | :--- |
| **1. Dual Write** | Application writes to **both** SQL and DynamoDB. | Ensures new data is captured in both places. |
| **2. Backfill** | Use AWS Glue or DMS to migrate historical data. | Move the "cold" data without affecting production. |
| **3. Verification** | Run "Dark Reads" (query both, compare results). | Validate that the NoSQL schema returns correct data. |
| **4. Cutover** | Switch the application "Source of Truth" to DynamoDB. | Fully retire the SQL database. |

---
### Summary Comparison: The Mindset Shift

| Feature | SQL (Relational) | DynamoDB (NoSQL) |
| :--- | :--- | :--- |
| **Data Goal** | Remove redundancy (Normalized). | Performance at scale (Denormalized). |
| **Joins** | Handled by the Database engine. | Handled by your Schema Design (No Joins). |
| **Scaling** | Vertical (Bigger Servers). | Horizontal (More Partitions). |
| **New Queries** | Easy to add anytime. | Requires careful planning or a New Index (GSI). |

## 11. DynamoDB Performance Optimization (100K Records)

| # | Topic | ✅ Do (Best Practice) | ❌ Don’t Do |
|---|------|----------------------|------------|
| 1 | Hot Partitions | Use high-cardinality keys / sharding | Use same partition key for many requests |
| 2 | API Calls | Use BatchWriteItem / BatchGetItem | Make one API call per item |
| 3 | Processing | Use parallel execution (Promise.all) | Process requests sequentially |
| 4 | Data Access | Use Query with proper keys | Use Scan on large tables |
| 5 | Indexing | Create GSIs for access patterns | Rely on primary key for all queries |
| 6 | Filtering | Design keys to avoid filtering | Use FilterExpression on large data |
| 7 | Capacity | Use On-Demand / Auto Scaling | Keep fixed low capacity under high load |
| 8 | Caching | Use DAX for frequent reads | Hit DB for every request |
| 9 | Item Size | Keep items small, use S3 for large data | Store large blobs in DynamoDB |
|10 | Processing Logic | Use async (Streams + Lambda) | Do heavy work inside API |
|11 | Data Fetching | Use pagination (LastEvaluatedKey) | Fetch huge data (100k) in one request |


## 12. Consistency Models
DynamoDB replicates data across three facilities (Availability Zones). How you read that data affects both cost and accuracy.

* **Eventually Consistent (Default):**
    * **Speed:** Provides the lowest latency.
    * **Cost:** 0.5 RCU per 4KB.
    * **Risk:** May return slightly stale data if a write just occurred.
* **Strongly Consistent:**
    * **Accuracy:** Always returns the most up-to-date data by querying the "Leader" node.
    * **Cost:** **2x RCU** (1 RCU per 4KB).
    * **Constraint:** Not supported on Global Secondary Indexes (GSIs).

👉 **Expert Advice:** Use strongly consistent reads only for critical paths like financial ledger balances or inventory locks.

---

## 13. Conditional Writes & Optimistic Locking
To prevent data corruption when multiple updates happen at the same time (concurrent writes).
* **Mechanism:** Include a `version` attribute in your item.
* **Execution:** Before updating, check if the version in the database matches the version you read.
* **ConditionExpression:** `attribute_not_exists(PK) OR version = :expectedVersion`
* **Result:** Ensures only the latest version is updated and prevents race conditions.

### What is Optimistic Locking?
In a distributed system, many people or programs might try to change the same piece of data at the same time. This can cause a problem called a "Lost Update," where one person's changes get erased by someone else. Optimistic Locking is a way to stop this from happening.

### How it Works
You add a small piece of info called a **version** to your item. Think of it like a counter that goes up every time the data changes.

* **Step 1:** You read the data and see that it is on version 1.
* **Step 2:** You get ready to make your changes.
* **Step 3:** Before you save, the system checks the database. If the version is still 1, your change is allowed, and the version becomes 2.
* **Step 4:** If someone else changed it while you were working, the version might already be 2. In this case, the system will stop your update so you do not overwrite their work.
- **With Optimistic Locking**

    ```
    Initial: version = 1

    User A reads version = 1
    User B reads version = 1

    User A updates → version becomes 2 ✅
    User B tries update → FAIL ❌ (because version != 1)
    ```

### Using Condition Expressions
To make this work in a database like DynamoDB, you use a rule called a **ConditionExpression**. 

The rule looks like this: `attribute_not_exists(PK) OR version = :expectedVersion`

* **attribute_not_exists(PK):** This allows the save if the item is brand new.
* **version = :expectedVersion:** This allows the update only if the version number in the database is exactly what you expected to see.

### Why Use This?
* It prevents "race conditions" where two updates collide.
* It ensures that only the latest and most correct version of the data is saved.
* It keeps your data safe without slowing down the system too much.
---

## 14. Error Handling & Retry Strategy

### What happens when you use too much power?
When you try to read or write data faster than your database can handle, DynamoDB gives you an error called **ProvisionedThroughputExceededException**. This just means you have hit your limit for a moment.

### The Best Way to Fix It: Exponential Backoff with Jitter
Instead of giving up or trying again immediately, you should use a smart retry plan.

* **Exponential Backoff:** This means you wait a little bit longer after each failed try. For example, wait 1 second, then 2 seconds, then 4 seconds.
    Example
    ```
    1000 requests fail at same time
    ↓
    All retry after 1 second
    ↓
    DB gets 1000 requests again 💥
    ↓
    Fails again → repeat cycle
    ```
* **Jitter:** This adds a tiny bit of random time to each wait. Instead of exactly 2 seconds, you might wait 1.8 or 2.1 seconds.
    Example
    ```
    1000 requests fail
    ↓
    Retry times become random:
    1.1s, 1.3s, 1.8s, 2.2s, ...
    ↓
    Load spreads out
    ↓
    System recovers smoothly ✅
    ```
### Why is this better?
If a thousand computers all try to fix the error at the exact same second, they will crash the system again. This is called a **"thundering herd."** Adding Jitter spreads out the retries so they don't all hit the database at the same time.



### Why You Need a Retry Strategy
* **Without a Retry:** Your system will just fail even if the database is ready to work again a half-second later.
* **With a Retry:** Your system stays strong and keeps working even when things get busy.

---

# Pagination in DynamoDB

### The 1MB Limit
When you ask DynamoDB for data using a **Query** or a **Scan**, it can only give you up to **1MB** of data at one time. If your search matches more than 1MB of data, DynamoDB stops at the limit and sends you what it has. This is called a "page" of data.



---

### How to Get the Rest of the Data
To get the next set of data, you use two special labels:

* **LastEvaluatedKey:** When DynamoDB stops at the 1MB limit, it sends this key back to you. It is like a "bookmark" that tells you exactly where the database stopped reading.
* **ExclusiveStartKey:** To get the next page, you send a new request to DynamoDB. You include the `LastEvaluatedKey` from the first step and put it into a field called `ExclusiveStartKey`.

---

### The Step-by-Step Flow
1. **First Request:** You ask for data. 
2. **The Response:** DynamoDB sends you the first 1MB of data and a `LastEvaluatedKey`.
3. **Second Request:** You ask for data again, but you tell DynamoDB to start at the `ExclusiveStartKey` (the bookmark).
4. **Repeat:** You keep doing this until DynamoDB does **not** send a `LastEvaluatedKey` back. That means you have reached the end of the data.
5. The **LastEvaluatedKey** is a small piece of data that contains the **Primary Key** of the last item that DynamoDB successfully read before it hit the 1MB limit.

### Why is this helpful?
* **Saves Memory:** Your application does not get overwhelmed by too much data at once.
* **Faster Response:** Small "pages" of data travel faster over the internet than one giant file.
* **Better Control:** You can show the user 10 items at a time instead of 1,000.


---

## 16. Data Types & Limits
* It mean basically telling you what kind of data you can store in one item (row) in Amazon DynamoDB and the limits.
* **Max Item Size:** One item (like a row in SQL) can be maximum 400 KB (includes all attribute names and values).
* **Scalars:** String, Number, Boolean, Null, Binary.
* **Documents:** **Map** (JSON-like objects) and **List** (ordered arrays).
* **Sets:** String Set, Number Set, Binary Set (unique values only).

---

## 17. Secondary Index Comparison (LSI vs GSI)

- Two Types of Secondary Index
    1. LSI (Local) : Same PK, different SK
        ```
        PK = userId
        SK = orderDate   (main table)

        LSI:
        PK = userId
        SK = orderAmount
        ```
    2. GSI (Global) : Completely new key
        ```
        Main:
        PK = userId

        GSI:
        PK = email
        ```
- Secondary index = extra cost + extra storage

| Feature | LSI (Local Secondary Index) | GSI (Global Secondary Index) |
| :--- | :--- | :--- |
| **Partition Key** | Must be the same as the table. | Can be different. |
| **Sort Key** | Must be different. | Can be different. |
| **Consistency** | Supports Strongly Consistent reads. | Eventually Consistent only. |
| **Creation** | Only at table creation time. | Can be added or deleted anytime. |

---

## 18. PartiQL (SQL-like Queries)
DynamoDB supports **PartiQL**, a SQL-compatible query language.
* **Usage:** `SELECT * FROM "Users" WHERE "UserID" = '123'`

* **Useful for:** Quick debugging in the AWS Console and developers coming from a relational background.

Where to run PartiQL?
>AWS Console > DynamoDB >  “PartiQL editor > Write query like:
---

## 19. Backup & Restore
* **On-Demand:** Full backups for long-term archiving.
* **Point-in-Time Recovery (PITR):** Continuous backups for 35 days. Allows you to restore to any second in that window.
👉 **Protection:** Guards against accidental deletions or application-level data corruption.

---

## 20. IAM & Access Control
* **Row-Level Security:** Restrict users to only their own data using `dynamodb:LeadingKeys`.
* **Attribute-Level Security:** Prevent access to sensitive columns (like `CreditCardNumber`) using IAM `Condition` keys.

---

## 21. Monitoring & Observability
Use **Amazon CloudWatch** to track health:
* **ConsumedRead/WriteCapacity:** Are you hitting your limits?
* **ThrottledRequests:** Do you need more partitions or GAC adjustment?
👉 **Action:** Set alarms to notify your team before users experience latency.

---

## 22. Cost Optimization
* **Avoid Scans:** Scans read every item in the table; always prefer `Query`.
* **Short Attribute Names:** Since attribute names count toward the 400KB limit, use `u_id` instead of `User_Identification_Number`.
* **Optimize GSI:** Only project attributes you actually need into your index.

---

## 23. Multi-Tenant Design (SaaS)
* **Pattern:** `PK = TENANT#<ID>`, `SK = USER#<ID>`
- **Result:** Ensures data isolation and allows for efficient querying of all data belonging to a specific customer.

---

## 24. Time-Series Pattern
* **Design:** `PK = SENSOR_ID`, `SK = TIMESTAMP`
- **Enables:** High-velocity ingestion and efficient range queries (e.g., "Get data from the last 2 hours").

---

## 25. Large-Scale Delete Strategy
* **Batch Delete:** Use `BatchWriteItem` (25 items per call).
* **Parallel Processing:** Run multiple threads to scan and delete.
* **Recreate Table:** If deleting nearly all data, it’s often faster/cheaper to export the small portion you need and drop the table.

---

## 26. Anti-Patterns (Avoid These)
* ❌ **Full Table Scans:** Extremely expensive and slow at scale.
* ❌ **Low Cardinality PKs:** Using "Status" as a PK leads to **Hot Partitions**.
* ❌ **Large Items:** Frequently reading/writing 400KB items will exhaust your WCU/RCU quickly.

---

## 27. SDK Best Practices
* **Connection Reuse:** Use a persistent HTTP connection to avoid TLS handshake overhead.
* **Batch Operations:** Maximize efficiency by grouping requests (up to 25 writes or 100 reads per call).

---

## 28. Security Overview
1.  **IAM:** Fine-grained policies following Least Privilege.
2.  **Encryption:** **At Rest** (KMS) and **In Transit** (TLS/HTTPS) are default.
3.  **VPC Endpoints:** Keep traffic within the AWS private network for higher security.
4.  **Data Protection:** Hash passwords (Argon2/BCrypt) before storage; never store raw PII if avoidable.

---
## 29. Adaptive Capacity
DynamoDB is designed to handle "unbalanced" workloads automatically.
* **Mechanism:** It redistributes throughput from idle partitions to "hot" partitions that are experiencing high traffic.
* **Constraint:** This is a reactive measure; it works best when the access pattern is mostly balanced. It cannot bypass the physical hardware limit of a single partition (3000 RCU / 1000 WCU).

---

## 30. Hot Partition Detection
Monitoring for uneven traffic is essential for maintaining low latency.
* **CloudWatch Metrics:** Monitor `ThrottledRequests` and `ConsumedReadCapacityUnits`.
* **Contributor Insights:** Use this tool to see which specific Partition Keys are causing the most traffic in real-time.


---

## 31. Retry Strategy (Exponential Backoff & Jitter)
In distributed systems, failures are inevitable.
* **Exponential Backoff:** Progressively increase the wait time between retries to avoid overwhelming the database.
* **Jitter:** Add random variance to the retry timing to prevent "the thundering herd" effect.

---

## 32. Idempotency Keys
Essential for critical operations like payments.
* **Problem:** If a network timeout occurs, the client might retry a request that actually succeeded, leading to duplicate data.
* **Solution:** Store a unique `requestId` with the item and use a `ConditionExpression` to ensure the write only happens once.

---

## 33. Data Modeling Patterns
Mastering these core patterns is the key to Single-Table Design:
* **One-to-Many:** Using Item Collections (same PK, different SK).
* **Many-to-Many:** Using an Adjacency List pattern.
* **Inverted Index:** Swapping PK and SK in a GSI to query relationships from both ends.

---

## 34. Event-Driven Architecture (Streams)
DynamoDB Streams provide a time-ordered sequence of every change in your table.

* **Use Cases:**
    * Generating **Audit Logs**.
    * Triggering **Analytics** pipelines.
    * Synchronizing data with **OpenSearch** for fuzzy searching.

---

## 35. Global Tables Conflict Resolution
When data is updated in multiple regions simultaneously, DynamoDB must decide which update wins.
* **Mechanism:** **Last Writer Wins (LWW)** based on a system-level timestamp.
👉 **Note:** Ensure your application can handle eventual consistency across regions.

---

## 36. Capacity Planning
Throughput is calculated based on item size (rounded up to the nearest 4KB for reads and 1KB for writes).
* **RCU:** Depends on item size and whether you use Strong or Eventual consistency.
* **WCU:** Depends strictly on item size.

---

## 37. Latency Optimization
* **Connection Reuse:** Keep TCP connections open to avoid handshake overhead.
* **SDK Configuration:** Tune timeout and retry settings.
* **Lambda Optimization:** Use "Provisioned Concurrency" to eliminate cold starts.

---

## 38. Schema Evolution
Since DynamoDB is schemaless, you can add attributes at any time.
* **Best Practice:** Use a `version` attribute or a `type` attribute to help your application code handle items created under different schema versions.

---

## 39. Disaster Recovery (DR)
* **PITR:** Restores your table to any point in time within the last 35 days.
* **Multi-Region:** Use Global Tables to ensure your application can failover to a different region instantly.
* **RTO/RPO:** Define your Recovery Time and Recovery Point objectives early.

---

## 40. Data Migration Strategy
For large-scale migrations (SQL to DynamoDB):
* **Batching:** Don't move everything at once; use batch processing.
* **Parallelism:** Use multiple workers to increase throughput.
* **Monitoring:** Track every failure and implement a dead-letter queue (DLQ) for failed records.

---

## 41. Write & Read Amplification
* **Write Amplification:** Every GSI you add increases the cost of a write, as DynamoDB must update the index as well.
* **Read Amplification:** Fetching a 400KB item when you only need one field is wasteful. 
👉 **Solution:** Use **Projection Expressions** to retrieve only the data you need.

---

## 42. Secondary Index Projections
When creating a GSI, choose what data to copy into it:
* **KEYS_ONLY:** Smallest index size, lowest cost.
* **INCLUDE:** Only the specific attributes you query frequently.
* **ALL:** Most flexible, but most expensive.

---

## 43. Distributed System Thinking
DynamoDB is not a traditional SQL server; it is a massive distributed fleet.
* **Design for failure:** Assume retries will happen.
* **Embrace Eventual Consistency:** It provides the highest availability and lowest cost.
* **Focus on Throughput:** Success in DynamoDB is measured by how efficiently you use your RCU and WCU.
---