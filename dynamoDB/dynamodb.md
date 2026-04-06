# DynamoDB Expert Roadmap & SQL Migration Guide
## DynamoDB Architectural Internals & Advanced Modeling

This guide focuses on high-scale architecture, internal mechanics, and the strategic shift from relational to non-relational modeling.

To master DynamoDB at a 10-year experience level, you must move beyond CRUD and understand the "Distributed Systems" nature of the service.

---

##  Table of Contents

1. [Architectural Internals](#1-architectural-internals)
   - [Global Admission Control](#11-global-admission-control-gac)
   - [Adaptive Capacity](#12-adaptive-capacity--bursting)
   - [Partitioning](#13-partitioning--consistent-hashing)
   - [B-Tree Indexing](#14-b-tree-indexing)
2. [Advanced Data Modeling](#2-advanced-data-modeling-the-no-join-philosophy)
3. [Operations & Performance](#3-operations--performance-at-scale)
4. [Security & Governance](#4-advanced-security--governance)
5. [TTL (Time to Live)](#5-advanced-ttl-time-to-live-mechanics)
6. [Filter Expression Trap](#6-the-filter-expression-efficiency-trap)
7. [Write Sharding](#7-write-sharding-for-ultra-hot-keys)
8. [Import / Export](#8-enterprise-importexport-patterns)
9. [Zero-Downtime Migration](#9-zero-downtime-migration-the-drift)
10. [SQL -> DynamoDB Migration](#10-sql-to-dynamodb-migration-strategy)
11. [Performance Optimization](#11-dynamodb-performance-optimization-100k-records)
12. [Consistency Models](#12-consistency-models)
13. [Conditional Writes](#13-conditional-writes--optimistic-locking)
14. [Error Handling](#14-error-handling--retry-strategy)
15. [Pagination](#pagination-in-dynamodb)
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
26. [SDK Best Practices](#26-sdk-best-practices)
27. [Security](#27-security-overview)
28. [Idempotency Keys](#28-idempotency-keys)
29. [Data Modeling Patterns](#29-data-modeling-patterns)
30. [Global Tables](#30-global-tables-conflict-resolution)
31. [Latency Optimization](#31-latency-optimization)
32. [Schema Evolution](#32-schema-evolution)
33. [Disaster Recovery](#33-disaster-recovery-dr)
34. [Data Migration](#34-data-migration-strategy)
35. [Write & Read Amplification](#35-write--read-amplification)
36. [Index Projections](#36-secondary-index-projections)
37. [Distributed Systems Thinking](#37-distributed-system-thinking)

---

## 1. Architectural Internals

### 1.1 Global Admission Control (GAC)

GAC is the entry point of the DynamoDB fleet. It acts like a gatekeeper for internal requests before they reach the storage layer.

- **Function** : It validates IAM permissions, checks if the table exists, and ensures the request doesn't exceed the total account or table-level throughput.
- **Placement** : It lives in the **Request Router** layer.
- **Why it's first** : By throttling at this layer, DynamoDB protects the underlying storage partitions from being "flooded," preserving system stability.
- **Workflow:**

    ```text
    Your API Request
        ->
    Global Admission Control (GAC) (Step 1)
        ->
    Partitioning / Hashing (Step 2)
        ->
  Physical Nodes (Step 3)
    ```

### 1.2 Adaptive Capacity & Bursting

DynamoDB manages "Hot Partitions" by rebalancing throughput dynamically across the partition fleet.

- **Adaptive Capacity:** If one partition is under heavy load while others are idle, DynamoDB reallocates unused throughput to the busy partition.
  - **Scenario:** A table with 3,000 total WCU spread across 3 partitions (A, B, C).
  - **Normal Allocation:** A: 1,000 | B: 1,000 | C: 1,000.
  - **Traffic Spike on A:** A receives 1,500 writes/sec.
  - **Adaptive Result:** DynamoDB may shift allocation to A: 1,500 | B: 750 | C: 750 to prevent throttling on A.

- **Bursting:** For short-lived traffic spikes, DynamoDB retains a "burst buffer" of unused capacity (up to 5 minutes) to handle sudden surges.

### 1.3 Partitioning & Consistent Hashing

Once a request passes GAC, DynamoDB must find the physical location of the data.

1. **The "Mailbox" Analogy (Consistent Hashing):**
    The same key always produces the same hash, ensuring DynamoDB knows exactly where the data is stored without looking it up in a giant table.
    - **Workflow:** `Partition Key` -> `Hash Function` -> `Correct Storage Location` (e.g., `UserID:123` -> `Hash` -> `Partition B`).

2. **Physical Storage Nodes:**
    DynamoDB maintains **three copies** of each partition across three different **Availability Zones (AZs)**.
    - **Leader:** Handles all WRITES.
    - **Followers:** Handle READS (eventually consistent).

3. **The 10GB Limit:**
    Each physical partition is capped at **10GB**. High-cardinality keys are required to ensure data is spread across many 10GB chunks rather than piling up in one.

4. **Hardware Throughput Limits:**
    - **Write:** 1,000 WCU/sec per partition.
    - **Read:** 3,000 RCU/sec per partition.

### 1.4 B-Tree Indexing

- Within each partition, the **Sort Key (SK)** is physically stored in a B-Tree structure. This enables high-performance range queries using operators like `begins_with`, `between`, and comparison operators (`>`, `<`).
- DynamoDB uses B-Tree on Sort Key to quickly find ordered data inside a partition without scanning

[Read More](./topic-1-arch-internals.svg)

---

## 2. Advanced Data Modeling (The "No-Join" Philosophy)

- **Single-Table Design:** The practice of storing multiple distinct entity types (e.g., Users, Orders, Products) in a single table. This allows you to retrieve all related data for a complex access pattern in a single atomic `Query` call.
- **Adjacency Lists:** A technique for modeling many-to-many ($N:M$) relationships. By using a "Global Secondary Index (GSI) Flip, where the original SK becomes the PK, you can query relationships from both directions.
- **GSI Overloading:** A strategy to stay under the GSI limit (20 per table) by using generic attribute names (e.g., `GSI1_PK`, `GSI1_SK`). This allows a single index to serve different purposes for different entity types.

- **Sparse Indexes:** By only populating a GSI attribute for specific items, you create a "Sparse Index." This is ideal for "Needle in a Haystack" queries, such as finding only the "In-Progress" orders out of millions of "Completed" ones.

[Read More](./topic-2-advanced-modeling.svg)

---

## 3. Operations & Performance at Scale

### 3.1 DynamoDB Streams vs. Kinesis (Event Processing)

When data changes in DynamoDB (insert/update/delete), you can trigger downstream actions via event-driven architectures.

- **DynamoDB Streams:** Use for Lambda triggers and short-term event processing.
  - *Example:* User creates order -> Stream records event -> Lambda processes event -> Sends confirmation email.
  ```json
  {
    "eventName": "INSERT",
    "Keys": { "userId": "123" },
    "OldImage": null,
    "NewImage": { "name": "Pradeep" }
  }
  ```

- **Amazon Kinesis Data Streams:** Better for long-term retention, real-time analytics, and multiple consumers.
  - *Example:* Orders -> Kinesis -> Analytics system + Fraud detection + Real-time Dashboard.

### 3.2 Transactions (ACID)

DynamoDB supports atomic transactions for complex operations.

- **Limits:** Maximum of 100 items per transaction.

- **Cost:** Approximately 2x the cost of a normal write because DynamoDB performs extra coordination checks to guarantee consistency.

- **How DynamoDB Handles It Internally** When you run a transaction:
  - DynamoDB locks/checks items
    - Verifies no conflicts
    - Performs all operations
    - Commits only if everything is safe
    - If anything fails -> rollback

### 3.3 Global Tables (Active-Active)

Provides multi-region replication and conflict resolution.

- **Mechanism:** The same table exists in multiple AWS regions with automatic replication.
- **Benefits:** Low latency for global users and high availability/disaster recovery.
- **Conflict Resolution:** Uses the "Last Writer Wins" rule where the latest timestamped update overwrites previous ones.

### 3.4 FinOps (Cost Optimization)

1. **On-Demand:** You pay per request. Best for unpredictable traffic or new applications.
2. **Provisioned Capacity:** You define specific RCU/WCU limits. More cost-effective for predictable, steady-state traffic.
3. **Reserved Capacity:** Commit to a baseline usage over a 1 or 3-year term for a **50â€"70% cost reduction**.

### 3.5 DAX (DynamoDB Accelerator)

DAX is a fully managed, highly available, in-memory cache for DynamoDB that delivers up to a 10x performance improvementâ€"reducing response times from milliseconds to **microseconds**, even at millions of requests per second.

#### Architectural Integration

DAX is a **write-through** caching service. Your application points to the DAX cluster endpoint instead of the standard DynamoDB endpoint.

- **Read-Through:** If the requested data is in the cache (a cache hit), DAX returns it immediately. If not (a cache miss), DAX fetches it from DynamoDB, stores it, and then returns it.
- **Write-Through:** When you write data, it is written to the DAX cache and the DynamoDB table simultaneously. This ensures the cache remains consistent with the underlying table.

#### Primary Use Cases

DAX is specifically engineered for:

- **Read-Heavy Workloads:** Applications with high read-to-write ratios, such as product catalogs, social media feeds, or gaming leaderboards.
- **"Hot" Key Mitigation:** Offloading traffic from frequently accessed items (like a viral news story or a trending product) to prevent RCU throttling on the base table.
- **Extreme Low Latency:** Real-time applications that require sub-millisecond response times for a smooth user experience.

#### When NOT to Use DAX

While powerful, DAX is not a "one-size-fits-all" solution. Avoid DAX if:

- **Write-Heavy Operations:** DAX provides no performance benefit for writes; it may actually add a slight overhead due to the write-through requirement.
- **Strict Strong Consistency:** DAX is optimized for **eventually consistent** reads. While it supports strongly consistent reads, they must be issued to the DynamoDB endpoint directly or they will bypass the cache gain.
- **Cost Sensitivity:** DAX is an additional provisioned resource (clusters of nodes). If millisecond latency (standard DynamoDB) is sufficient for your SLA, the extra cost of a DAX cluster may not be justified.
- **Frequently Changing Data:** If your data expires or changes every few seconds, the cache churn (constant invalidation and reloading) can negate the performance benefits.

[Read More](./topic-3-operations-streams.svg)

---

## 4. Advanced Security & Governance

### 4.1 Resource-Based Policies

Resource-based policies allow you to grant permissions directly to a specific resource. In this architecture, we are enabling **Account B** to read data from a DynamoDB table located within **Account A**.

- **Location:** This policy is written and attached directly to the **DynamoDB Table** (on the resource itself) in Account A.
- **Key Distinction:** Unlike standard identity-based policies, this is **NOT** configured in *IAM -> Policies*. It lives on the DynamoDB service side attached to the table.
- **Involved Accounts:**
  - **Account A (111111111111):** The primary owner of the DynamoDB table (`UsersTable`).
  - **Account B (222222222222):** The external/third-party account requiring cross-account read access.

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
### 4.2 Client-Side Encryption

Client-side encryption ensures that sensitive user data, such as Personally Identifiable Information (PII), is encrypted **locally within your application** before it is ever transmitted over the network or stored in DynamoDB. This provides the highest tier of data protection; since the data is already ciphertext when it reaches AWS, it remains secure even in the unlikely event that the database layer is compromised.

### Workflow using AWS Database Encryption SDK

The implementation involves a secure orchestration between your application, AWS Key Management Service (KMS), and DynamoDB.

1. **Key Retrieval:** Your application requests a unique data key or references a customer master key (CMK) from **AWS KMS**.
2. **Local Encryption:** Before the data leaves the application's memory space, the **AWS Database Encryption SDK** encrypts the specified sensitive fields using the retrieved key.
3. **Encrypted Storage:** The application sends the ciphertext (encrypted data) to **DynamoDB**. From the database's perspective, this is just an opaque blob or encrypted string.
4. **Decryption on Read:** When the application retrieves the data, the SDK automatically handles the decryption process using the authorized KMS key, allowing the application to work with the original plaintext.

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

[Read More](./topic-4-resource-policies.svg)

---

## 5. Advanced TTL (Time to Live) Mechanics

DynamoDB **Time to Live (TTL)** allows items to "self-destruct" automatically after a specified timestamp. This background process helps manage storage costs by removing stale data without consuming your provisioned throughput.

### The 48-Hour Deletion Window

TTL is a background process and does not guarantee that an item will be deleted the exact second it expires.

- **Timeline:** DynamoDB typically removes expired items within a **48-hour window** from the expiration time.
- **Constraints:** You can designate only **one** TTL attribute per table.
- **Configuration Details:**
  - â›" The 48-hour window is **NOT** configurable by the user.
  - â›" Deletion is **NOT** guaranteed to happen exactly at the 48-hour mark; it depends on table traffic and system resources.
  - It is a **best-effort** background process managed internally by AWS.

> **Expert Tip:** To ensure users never see expired data during the 48-hour deletion lag, always include a filter in your application queries (e.g., using a Filter Expression) to ignore items where `ExpirationTime < CurrentTime`.
### Enabling TTL on a Table

To use TTL, you must explicitly enable it and designate a specific attribute formatted as a Unix epoch timestamp in seconds for DynamoDB to monitor.

**How to Enable (AWS Console):**

1. Navigate to the **DynamoDB Console** and select your **Table**.
2. Go to the **Additional settings** tab.
3. Locate the **Time to Live (TTL)** section.
4. Click **Enable**.
5. Enter the **Attribute name** (e.g., `ttl` or `expiration_time`).
6. Click **Save**.
### The Archiving Pattern (TTL + Streams)

When the internal TTL worker deletes an item, the event is captured by **DynamoDB Streams**. This is a powerful architectural pattern used to preserve data for compliance or analytics before it is permanently purged.

**Archiving Workflow:**

1. **Item Expires:** The Unix timestamp in the designated TTL attribute is reached.
2. **DynamoDB Deletes:** The background process identifies and removes the item.
3. **Stream Capture:** A `REMOVE` event is pushed to the DynamoDB Stream.
    - *Identification:* These specific events have a `userIdentity` of `DynamoDB Service`, allowing you to distinguish them from manual deletions performed by users or applications.
4. **Lambda Trigger:** An AWS Lambda function subscribed to the stream is triggered by the deletion event.
5. **S3 Archive:** The Lambda function processes the item data and writes it to **Amazon S3** for long-term storage.

**Summary Flow:**
>Item expires -> DynamoDB deletes it -> Stream records event -> Lambda triggers -> Save to S3
### Understanding DynamoDB Streams

It is important to understand the nature of the streaming capability to use it effectively:

- **Integration:** It is **NOT** a separate AWS service (like Kinesis Data Streams); it is a built-in **feature** of the DynamoDB service.
- **Activation:** It must be enabled at the table level, where you can choose what information is written to the stream (e.g., Keys only, New Image, Old Image, or both).
- **Functionality:** Once enabled, it automatically records a time-ordered sequence of every item-level modification (**Create**, **Update**, **Delete**) in the table for up to 24 hours.

[Read More](./topic-5-ttl-workflow.svg)

---

## 6. The "Filter Expression" Efficiency Trap

A common misconception in DynamoDB is that a `FilterExpression` reduces the cost of a query. In reality, filtering happens **after** the data has been read from the disk and metered for cost.

### Filter vs. Query: The Execution Order

When you execute a query with a `FilterExpression`, DynamoDB follows this specific sequence:

1. **Read:** DynamoDB reads data from the partition based on the Partition Key (PK) and any optional Sort Key (SK) conditions defined in the `KeyConditionExpression`.
2. **Filter:** It then applies your `FilterExpression` to the retrieved items in memory.
3. **Return:** Only the items that pass the filter are returned to your application.


### The RCUs (Read Capacity Units) Trap

To understand the cost impact, consider this scenario involving an `order_table`:

- **Dataset:** `user_123` has **100 orders** stored in a single partition.
- **Request:** You want to retrieve only orders where `status = "DELIVERED"`.
- **FilterExpression:** `status = :s` (where `:s` is "DELIVERED").
- **Result:** Only **1 item** matches the criteria; 99 items are filtered out.

| Perspective | Result |
| :--- | :--- |
| **Expectation** | "I received 1 item, so I should only pay for 1 read." |
| **Reality** | **You pay for 100 reads.** |

Because DynamoDB had to pull all 100 items into memory to evaluate the `status` field, you are billed for the full 100-item read operation.


### Optimization Strategies

If your queries are consistently filtering out more than **20% of the data**, your current schema is inefficient. You should move the filtered attribute into the primary schema structure.

#### 1. Redesign the Sort Key (SK)

Incorporate the filter criteria directly into the Sort Key to allow DynamoDB to perform a **Key Condition Expression** instead of a post-read filter.

- **Original Schema:** `PK = user_123`, `SK = order_id`
- **Optimized Schema:** `PK = user_123`, `SK = status#order_id`

By using `BeginsWith(DELIVERED#)`, DynamoDB only reads the matching items from the disk, significantly reducing RCU consumption.

#### 2. Global Secondary Index (GSI)

If you need to query by status across many different users simultaneously, create a GSI to target the data precisely.

- **GSI Partition Key:** `status`
- **GSI Sort Key:** `order_date`

**Result:** When you query `status = DELIVERED` on the GSI, DynamoDB only touches the data that matches that status. There is zero waste, and you only pay for the items you actually retrieve.

[Read More](./topic-6-filter-trap.svg)

---

## 7. Write Sharding (For Ultra-Hot Keys)

Write sharding is a high-scale design pattern used to handle "hot" partition keys that exceed DynamoDB's native throughput limits.

### The Problem: Throughput Limits

In DynamoDB, a single partition key has a hard performance ceiling. If your application attempts to write to the same partition key too frequently, you will encounter **ProvisionedThroughputExceededException**.

- **The Limit:** A single partition key can handle a maximum of **1,000 Write Capacity Units (WCU)** per second.
- **The "Hot Key" Scenario:** If thousands of items share the same Partition Key (PK) simultaneously, they are all routed to the same physical partition.

**Example: High-Concurrency Status Updates**
Imagine storing users with a generic status as the Partition Key:

- **PK:** `ACTIVE`
- **Scenario:** 10,000 users become `ACTIVE` at the exact same moment.
- **Execution:** All 10,000 writes target the single key `ACTIVE`.
-  **Result:** The 1,000 WCU/sec limit is breached, leading to massive throttling and application failures.


### The Strategy: Adding Entropy (Sharding)

To solve this, you distribute the load by appending a random suffix a shard to the Partition Key. This forces DynamoDB to spread the data across multiple underlying physical partitions.

#### Implementation Logic

Instead of using a static key like `ACTIVE`, your application logic should append a shard number: `ACTIVE#N` (where N is a random number between 1 and your total shard count).

**Optimized Distribution:**
Instead of one overloaded key, the load is split across multiple shards:

- **ACTIVE#1:** 2,500 writes
- **ACTIVE#2:** 2,500 writes
- **ACTIVE#3:** 2,500 writes
- **ACTIVE#4:** 2,500 writes

**Result:** The total load of 10,000 writes is successfully processed because no individual shard exceeds the 1,000 WCU limit.


### Key Considerations

- **Reading Sharded Data:** When you need to retrieve all "ACTIVE" users, your application must perform parallel queries across all shards (`ACTIVE#1` through `ACTIVE#N`) and merge the results in the application layer.

- **Shard Count:** Choose a shard count that comfortably covers your peak traffic. If you expect 5,000 writes/sec, use at least 6â€"10 shards to provide a safety buffer against unexpected spikes.

[Read More](./topic-7-write-sharding.svg)

---

## 8. Enterprise Import/Export Patterns

Modern AWS architectures leverage native integrations to move data into and out of DynamoDB without the overhead of custom application logic or performance degradation.

### 6.1 S3 Import (Zero RCU/WCU)

You can perform bulk data imports from **Amazon S3** (CSV, JSON, or Amazon Ion formats) directly into a **new** DynamoDB table.

- **Cost Efficiency:** This method does not consume any provisioned Read or Write Capacity Units (RCU/WCU). It is significantly cheaper than traditional `BatchWriteItem` scripts or custom Lambda loaders.
- **Infrastructure:** The S3 file acts as the source storage. DynamoDB manages the import job internally to populate the table.

#### How it works

1. **Preparation:** Store your data file (e.g., `data.json`) in an S3 bucket.
2. **Trigger:** Initiate the "Import from S3" job via the DynamoDB console or CLI.
3. **Processing:** DynamoDB reads the file directly from S3.
4. **Creation:** DynamoDB automatically creates the target table and populates the items.

#### Configuration Steps

In the AWS Console:

- Navigate to **DynamoDB**.
- Select **Import from S3** in the sidebar.
- Provide the **S3 URL** (bucket and path).
- Define the **New Table Name**.
- Specify the **Partition Key** (and optional Sort Key).


### 6.2 Zero-ETL Integration with Amazon OpenSearch

**Zero-ETL** refers to the ability to sync data between systems automatically without writing, maintaining, or scaling custom data transfer logic (like Lambda functions or Kinesis streams).

- **Synchronization:** Data in DynamoDB is automatically mirrored and kept in sync with **Amazon OpenSearch Service**.
- **Advanced Search:** Enables "Google-like" search capabilities, including fuzzy matching (handling typos), partial word matches, and complex filtering.
- **No Code:** Eliminates the need for custom sync scripts or intermediate services.

#### Search Comparison

| Feature | Without OpenSearch (DynamoDB Only) | With OpenSearch (Zero-ETL) |
| :--- | :--- | :--- |
| **Example Query** | Searching for "pradip" | Searching for "pradip" |
| **Logic** | Exact match only | Fuzzy/Text matching |
| **Result** |  No result ("pradip" â‰  "Pradeep") | Matches "Pradeep" |

#### Internal Sync Mechanism

1. **Source of Truth:** Data is stored in DynamoDB (e.g., `{"name": "Pradeep"}`).
2. **Auto-Sync:** Every `Insert`, `Update`, or `Delete` in DynamoDB is automatically propagated to OpenSearch.
3. **Search Index:** OpenSearch maintains a highly optimized index for fast, complex text queries.

#### Execution Flow

1. **User Input:** User types "pradip" into the application UI.
2. **Backend API:** The application backend receives the request.
3. **Search Query:** The backend queries **OpenSearch** instead of DynamoDB for the search operation.
4. **Retrieval:** OpenSearch returns the matching record ("Pradeep") based on relevance.
5. **Response:** The UI displays the correct result to the user.


[Read More](./topic-8-import-export.svg)

---

## 9. Zero-Downtime Migration: "The Drift"

- When you migrate data from SQL -> DynamoDB without stopping your app, you write to both systems (dual write).

- **Idempotency & Versioning:** During "Dual Writes," network failures can cause retries. Use a `version` attribute or the `attribute_not_exists(PK)` check in your TypeScript code to ensure you don't overwrite newer data with older data during the migration window.
- **Dual Write + Retry Flow: (Request 1)**

    ```text
   User updates name = "Pradeep"
        ->
    Write to SQL 
            ->
    Write to DynamoDB  (network fails)
            ->
    System retries later...
    ```

- **Another Api Request (Request 2)**

    ```text
   User updates name = "Pradeep kumar"
        ->
    Write to SQL 
            ->
    Write to DynamoDB 
    
    Note : As per second Request both db has name = "Pradeep kumar"
    ```

- **Problem (late retry comes back)**

    ```text
    Retry old request (Request 1):
    name = "Pradeep"
        ->
    Write to DynamoDB (with old data, name="Pradeep")
    ```

- **Final Solution with Versioning Flow**

    ```text
    1. Read current data from DB
    -> name = "Pradeep", version = 1

    2. User sends update
    -> name = "Pradeep Kumar"

    3. App prepares update
    -> new version = 2

    4. Send update with condition
    -> Update name = "Pradeep Kumar"
    -> Set version = 2
    -> ONLY IF current version = 1

    5. DB checks condition:
    âœ" If version = 1 -> Update succeeds
     If version â‰  1 -> Reject update
    
    Note : We update data only if the version matches, so outdated requests can't overwrite newer data.
    ```

- **Data Drift:** Use a comparison script (Dark Reading) to periodically check if the data in your new DynamoDB table matches the source SQL database before performing the final cutover.
  - During migration (SQL -> DynamoDB), both DBs are running.
  - Sometimes data becomes different bw both database (data drift)
  - Data Drift Check (Dark Reading Flow)
        ```text
    Script execution timing (Every few minutes / hours):
                ->
        Run comparison script
                ->
        1. Read batch from SQL
        2. Read same batch from DynamoDB
        3. Compare both
                ->
        Check:
            - Missing records
            - Incorrect values
                ->
        Log / Fix mismatches
            - Insert missing records
            - Update incorrect data
                ->
        Once everything matches -> safe to switch (cutover)
        ```


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

[Read More](./topic-9-migration-drift.svg)

---

## 10. SQL to DynamoDB Migration Strategy

Migrating to NoSQL is a **functional rewrite**, not a simple lift-and-shift.

### Step 1: Document Access Patterns

In SQL, you model data. In DynamoDB, you **model queries**.

1. Identify every API endpoint and UI view.
2. List the required data for each (e.g., "Get User by ID," "Get all Orders for User in the last 30 days").
3. Determine the "velocity" (read/write frequency) of each pattern.
    1. Which API is used a lot? (high traffic)
    2. Which is rare?

```text
Handle this in 3 steps:
1.JOIN data in SQL
2.Group it in Node.js (reduce)
3.Insert structured JSON into DynamoDB
```

### Step 2: Denormalization & Schema Design

- **Flatten the Data:** In SQL, `Orders` and `Products` are separate tables joined by an ID. In DynamoDB, you store the `Product_Name` and `Price` directly inside the `Order` item. This "duplicates" data but eliminates the need for a JOIN.
- **Primary Key Strategy:**
  - **Partition Key (PK):** Must be high-cardinality (unique) to spread data across partitions (e.g., `USER_ID`).
  - **Sort Key (SK):** Used to group and filter data (e.g., `TIMESTAMP` or `STATUS#DATE`).
- **Move Calculation to "Write Time":** SQL uses `SELECT COUNT(*)`. This is too expensive for DynamoDB. Instead, when a user clicks "Like," your code should increment a `LikeCount` attribute on the post item immediately.

### Step 3: Migration Execution (The Zero-Downtime Pattern)

| Phase | Action | Purpose |
| :--- | :--- | :--- |
| **1. Dual Write** | Application writes to **both** SQL and DynamoDB. | Ensures new data is captured in both places. |
| **2. Backfill** | Use AWS Glue or DMS to migrate historical data. | Move the "cold" data without affecting production. |
| **3. Verification** | Run "Dark Reads" (query both, compare results). | Validate that the NoSQL schema returns correct data. |
| **4. Cutover** | Switch the application "Source of Truth" to DynamoDB. | Fully retire the SQL database. |


### Summary Comparison: The Mindset Shift

| Feature | SQL (Relational) | DynamoDB (NoSQL) |
| :--- | :--- | :--- |
| **Data Goal** | Remove redundancy (Normalized). | Performance at scale (Denormalized). |
| **Joins** | Handled by the Database engine. | Handled by your Schema Design (No Joins). |
| **Scaling** | Vertical (Bigger Servers). | Horizontal (More Partitions). |
| **New Queries** | Easy to add anytime. | Requires careful planning or a New Index (GSI). |

[Read More](./topic-10-sql-to-ddb.svg)

---

## 11. DynamoDB Performance Optimization (100K Records)

| # | Topic | Do (Best Practice) |  Don't Do |
| --- | ------- | ---------------------- | ------------ |
| 1 | Hot Partitions | Use high-cardinality keys / sharding | Use same partition key for many requests |
| 2 | API Calls | Use BatchWriteItem / BatchGetItem | Make one API call per item |
| 3 | Processing | Use parallel execution (Promise.all) | Process requests sequentially |
| 4 | Data Access | Use Query with proper keys | Use Scan on large tables |
| 5 | Indexing | Create GSIs for access patterns | Rely on primary key for all queries |
| 6 | Filtering | Design keys to avoid filtering | Use FilterExpression on large data |
| 7 | Capacity | Use On-Demand / Auto Scaling | Keep fixed low capacity under high load |
| 8 | Caching | Use DAX for frequent reads | Hit DB for every request |
| 9 | Item Size | Keep items small, use S3 for large data | Store large blobs in DynamoDB |
| 10 | Processing Logic | Use async (Streams + Lambda) | Do heavy work inside API |
| 11 | Data Fetching | Use pagination (LastEvaluatedKey) | Fetch huge data (100k) in one request |

[Read More](./topic-11-performance-optimization.svg)

---

## 12. Consistency Models

DynamoDB replicates data across three facilities (Availability Zones). How you read that data affects both cost and accuracy.

- **Eventually Consistent (Default):**
  - **Speed:** Provides the lowest latency.
  - **Cost:** 0.5 RCU per 4KB.
  - **Risk:** May return slightly stale data if a write just occurred.
- **Strongly Consistent:**
  - **Accuracy:** Always returns the most up-to-date data by querying the "Leader" node.
  - **Cost:** **2x RCU** (1 RCU per 4KB).
  - **Constraint:** Not supported on Global Secondary Indexes (GSIs).

**Expert Advice:** Use strongly consistent reads only for critical paths like financial ledger balances or inventory locks.

[Read More](./topic-12-consistency-models.svg)

---

## 13. Conditional Writes & Optimistic Locking

To prevent data corruption when multiple updates happen at the same time (concurrent writes).

- **Mechanism:** Include a `version` attribute in your item.
- **Execution:** Before updating, check if the version in the database matches the version you read.
- **ConditionExpression:** `attribute_not_exists(PK) OR version = :expectedVersion`
- **Result:** Ensures only the latest version is updated and prevents race conditions.

### What is Optimistic Locking?

In a distributed system, many people or programs might try to change the same piece of data at the same time. This can cause a problem called a "Lost Update," where one person's changes get erased by someone else. Optimistic Locking is a way to stop this from happening.

### How it Works

You add a small piece of info called a **version** to your item. Think of it like a counter that goes up every time the data changes.

- **Step 1:** You read the data and see that it is on version 1.
- **Step 2:** You get ready to make your changes.
- **Step 3:** Before you save, the system checks the database. If the version is still 1, your change is allowed, and the version becomes 2.
- **Step 4:** If someone else changed it while you were working, the version might already be 2. In this case, the system will stop your update so you do not overwrite their work.

- **With Optimistic Locking**

    ```text
    Initial: version = 1

    User A reads version = 1
    User B reads version = 1

    User A updates -> version becomes 2 
    User B tries update -> FAIL  (because version != 1)
    ```

### Using Condition Expressions

To make this work in a database like DynamoDB, you use a rule called a **ConditionExpression**.

The rule looks like this: `attribute_not_exists(PK) OR version = :expectedVersion`

- **attribute_not_exists(PK):** This allows the save if the item is brand new.
- **version = :expectedVersion:** This allows the update only if the version number in the database is exactly what you expected to see.

### Why Use This?

- It prevents "race conditions" where two updates collide.

- It ensures that only the latest and most correct version of the data is saved.
- It keeps your data safe without slowing down the system too much.

[Read More](./topic-13-conditional-writes.svg)

---

## 14. Error Handling & Retry Strategy

### What happens when you use too much power?

When you try to read or write data faster than your database can handle, DynamoDB gives you an error called **ProvisionedThroughputExceededException**. This just means you have hit your limit for a moment.

### The Best Way to Fix It: Exponential Backoff with Jitter

Instead of giving up or trying again immediately, you should use a smart retry plan.

- **Exponential Backoff:** This means you wait a little bit longer after each failed try. For example, wait 1 second, then 2 seconds, then 4 seconds.
    Example

    ```text
    1000 requests fail at the same time
        ->
    All retry after 1 second
        ->
    DB gets 1000 requests again
        ->
    Fails again -> repeat cycle
    ```

- **Jitter:** This adds a tiny bit of random time to each wait. Instead of exactly 2 seconds, you might wait 1.8 or 2.1 seconds.
    Example

    ```text
    1000 requests fail
    ->
    Retry times become random:
    1.1s, 1.3s, 1.8s, 2.2s, ...
    ->
    Load spreads out
    ->
    System recovers smoothly 
    ```

### Why is this better?

If a thousand computers all try to fix the error at the exact same second, they will crash the system again. This is called a **"thundering herd."** Adding Jitter spreads out the retries so they don't all hit the database at the same time.

### Why You Need a Retry Strategy

- **Without a Retry:** Your system will just fail even if the database is ready to work again a half-second later.

- **With a Retry:** Your system stays strong and keeps working even when things get busy.

[Read More](./topic-14-error-handling.svg)

---

## 15. Pagination in DynamoDB

### The 1MB Limit

When you ask DynamoDB for data using a **Query** or a **Scan**, it can only give you up to **1MB** of data at one time. If your search matches more than 1MB of data, DynamoDB stops at the limit and sends you what it has. This is called a "page" of data.


### How to Get the Rest of the Data

To get the next set of data, you use two special labels:

- **LastEvaluatedKey:** When DynamoDB stops at the 1MB limit, it sends this key back to you. It is like a "bookmark" that tells you exactly where the database stopped reading.
- **ExclusiveStartKey:** To get the next page, you send a new request to DynamoDB. You include the `LastEvaluatedKey` from the first step and put it into a field called `ExclusiveStartKey`.


### The Step-by-Step Flow

1. **First Request:** You ask for data.
2. **The Response:** DynamoDB sends you the first 1MB of data and a `LastEvaluatedKey`.
3. **Second Request:** You ask for data again, but you tell DynamoDB to start at the `ExclusiveStartKey` (the bookmark).
4. **Repeat:** You keep doing this until DynamoDB does **not** send a `LastEvaluatedKey` back. That means you have reached the end of the data.
5. The **LastEvaluatedKey** is a small piece of data that contains the **Primary Key** of the last item that DynamoDB successfully read before it hit the 1MB limit.

### Why is this helpful?

- **Saves Memory:** Your application does not get overwhelmed by too much data at once.

- **Faster Response:** Small "pages" of data travel faster over the internet than one giant file.
- **Better Control:** You can show the user 10 items at a time instead of 1,000.

[Read More](./topic-15-pagination.svg)

---

## 16. Data Types & Limits

- It mean basically telling you what kind of data you can store in one item (row) in Amazon DynamoDB and the limits.

- **Max Item Size:** One item (like a row in SQL) can be maximum 400 KB (includes all attribute names and values).
- **Scalars:** String, Number, Boolean, Null, Binary.
- **Documents:** **Map** (JSON-like objects) and **List** (ordered arrays).
- **Sets:** String Set, Number Set, Binary Set (unique values only).

[Read More](./topic-16-data-types.svg)

---

## 17. Secondary Index Comparison (LSI vs GSI)

- Two Types of Secondary Index
    1. LSI (Local) : Same PK, different SK

        ```text
        PK = userId
        SK = orderDate   (main table)

        LSI:
        PK = userId
        SK = orderAmount
        ```

    2. GSI (Global) : Completely new key

        ```text
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
| **Max count** | 5 | 20 |

> Important Notes

- GSI limit (20) can be increased via AWS support (soft limit).
  - AWS has set 20 as the default maximum, BUT you can request AWS to increase it
- LSI limit (5) is hard limit -> cannot be increased.
- Too many GSIs = higher cost + write amplification
- To increase GSI  `Go to AWS Service Quotas >
Request increase to 25 (or more)`

[Read More](./topic-17-lsi-gsi-comparison.svg)

---

## 18. PartiQL (SQL-like Queries)

DynamoDB supports **PartiQL**, a SQL-compatible query language.

- **Usage:** `SELECT * FROM "Users" WHERE "UserID" = '123'`

- **Useful for:** Quick debugging in the AWS Console and developers coming from a relational background.

**Where to run PartiQL?**
> AWS Console > DynamoDB > PartiQL editor
[Read More](./topic-18-partiql.svg)

---

## 19. Backup & Restore

- **On-Demand:** Full backups for long-term archiving.
  - Take a full photo of the table at this exact moment
  - **How it works**

    - **Start the backup:** Manually tell the system to create a backup whenever you need one.
    - **The system takes a snapshot:** DynamoDB makes a full copy of all the data in your table at that exact moment.
    - **AWS keeps it safe:** Amazon Web Services stores this copy safely. Taking a backup does not slow down your app or affect your users.

  - **Features**

    - **Point-in-time snapshot:** The backup only contains the data that was in the table at the specific second you started it.
    - **Stored until you delete it:** These backups stay in your account forever unless you choose to delete them yourself.
    - **No performance impact:** Your database table will keep running at full speed while the backup is being made.

  - **Use cases**

    - **Before big updates:** It is smart to take a backup before you deploy major changes to your code.
    - **Before design changes:** Create a copy before you change how your table or data is organized.
    - **Compliance and audits:** Use these backups to keep long-term records for legal or official business reviews.
  - **What is a Snapshot?** : A snapshot is like a frozen copy of your data. It captures everything in your table at one exact moment.

    > Simple Example

    **1. Data at 2:00 PM (Backup Taken)**

    | User | Balance |
    | :--- | :--- |
    | User A | 100 |
    | User B | 200 |

    **2. Data at 3:00 PM (Changes Made)**

    - User A now has 0.
    - User B is deleted.

    **3. The Result**

    The backup still has the **original** data (User A with 100 and User B with 200). It does not change when your live table changes.

- **Point-in-Time Recovery (PITR):** Continuous backups for 35 days. Allows you to restore to any second in that window.
**Protection:** Guards against accidental deletions or application-level data corruption.

  - **How it works**: When you turn this feature on, DynamoDB automatically saves every change made to your data. It does this all the time without you needing to do anything.
  - **Time history**
    - **35-day window:** The system keeps a continuous record of your data for the last 35 days.
    - **Rolling backup:** As a new day starts, the oldest day is removed, so you always have the most recent 5 weeks of history.
  - **Restoring data**
    - **Total control:** Restore table to any exact second within that 35-day window.
    - **Safety net:** If someone accidentally deletes data or a bug breaks your database, you can simply "roll back" to a time right before the mistake happened.
    - **New table:** When you restore, DynamoDB creates a brand new table with your old data so your current table stays safe.

> Example of Point-in-Time Recovery (PITR)

### Recovery Timeline

| Time | Action | Status |
| :--- | :--- | :--- |
| 1:00 PM | Data is correct and healthy | Safe |
| 2:00 PM | A bug accidentally deletes 1 million records |  Data Loss |
| 2:10 PM | You find the issue and need to fix it | Issue Found |

**How to Fix this** : With Point-in-Time Recovery, you can go back to the exact second before the mistake happened.

- **The Solution:** You tell DynamoDB to restore the table to **1:59:59 PM**.
- **The Result:** Your data is fully recovered and the deleted records are back.

## Feature Comparison Table

| Feature | On-Demand Backup | Point-in-Time Recovery (PITR) |
| :--- | :--- | :--- |
| **Type** | Manual snapshot | Continuous backup |
| **Time range** | One single point in time | The last 35 days |
| **Restore detail** | Only at the exact backup time | To any second in the window |
| **Best for** | Long-term storage and audits | Recovery from sudden mistakes |
| **Automation** | You must do it manually | It works automatically |

[Read More](./topic-19-backup-restore.svg)

---

## 20. IAM & Access Control

- **Row-Level Security:**
  - Row-level security ensures that a user can only access items (rows) that belong to them.
  - In DynamoDB: Each item has a Partition Key, LeadingKeys refers to that key
  - Where to use IAM Policy
    ```text
      User logs in -> gets identity
              ->
      User calls API / DynamoDB
              ->
      IAM policy attached to role is checked
              ->
      "dynamodb:LeadingKeys" condition evaluated
              ->
      Allow / Deny
    ```
  - **IAM Policy Example (RLS)**

    ```json
    {
        "Effect": "Allow",
        "Action": "dynamodb:GetItem",
        "Resource": "arn:aws:dynamodb:region:account-id:table/Orders",
        "Condition": {
            "ForAllValues:StringEquals": {
            "dynamodb:LeadingKeys": "${aws:userid}"
            }
        }
    }
    ```

  - **What's happening here**:
    - ```${aws:userid} -> logged-in user```
    - DynamoDB checks: if Requested partition key == user ID
    - If mismatch ->  Access denied
  - **Works best when**:
    - UserID is partition key
    - If your design is wrong -> RLS becomes hard
    - This is why data modeling + security are linked

- **Attribute-Level Security:** User can access item, but not all fields (attributes)
    This table shows how user information and sensitive data like credit cards are stored in DynamoDB.
    > Example : User cannot see CreditCardNumber

    | UserID | Name | Email | CreditCardNumber |
    | :--- | :--- | :--- | :--- |
    | 101 | A | <a@mail.com> | 1234-XXXX |

  - What happens:  
    - If query tries:
```CreditCardNumber ->  denied```, it will allowed attributes returned
    - If app ```requests * -> may fail```
Must explicitly request allowed attributes
    - Best practice = Row + Attribute security together
      - Only access their data (RLS)
      - Only see safe fields (ALS)

  - **IAM Policy Example (Attribute-Level)**

    ```json
    {
        "Effect": "Allow",
        "Action": "dynamodb:GetItem",
        "Resource": "arn:aws:dynamodb:region:account-id:table/Users",
        "Condition": {
            "ForAllValues:StringEquals": {
            "dynamodb:Attributes": [
                "UserID",
                "Name",
                "Email"
            ]
            }
        }
    }
    ```

[Read More](./topic-20-iam-access.svg)

---

## 21. Monitoring & Observability

Use **Amazon CloudWatch** to track health:

- **ConsumedRead/WriteCapacity:** This metric tells you if you are using too much of your database's power at once. It helps you decide if you need to change your settings.

  - **Example**
    - If your table has: 100 Read Capacity Units (RCU)
    - And metrics show: ConsumedReadCapacity = 95 (You are using 95% of capacity)
  - **What to do**

      | Situation | Action |
      | :--- | :--- |
      | **High usage (>80%)** | Increase capacity or enable auto-scaling to prevent slowdowns |
      | **Low usage (<30%)** | Reduce capacity to save money and avoid wasting resources |

- **ThrottledRequests:** number of requests DynamoDB rejected
  - Actually tests this understanding:
    - DynamoDB scaling = not just increasing capacity
    - It's about:
      - data distribution (partition key)
      - traffic patterns
      - adaptive capacity
  - Why does throttling happen? : DynamoDB rejects requests when you exceed limits like:
      - Capacity limit
        - Provisioned mode: RCU/WCU exhausted
        - On-demand: sudden spike beyond adaptive capacity
      - Hot partition
        - One partition gets too much traffic All users hitting same ```PK -> "USER#1"```
  - Possible solutions

    | Problem | Solution |
    | :--- | :--- |
    | **Capacity limit** | Increase your **RCU** (Read) or **WCU** (Write) settings manually. |
    | **Traffic spikes** | Turn on **Auto Scaling** or switch to **On-Demand** mode to handle sudden jumps. |
    | **Hot key** | **Redesign your partition key** to spread data more evenly across the system. |
    | **Read-heavy traffic** | **Add a cache (DAX)** to handle frequent reads without hitting the database. |
- **Action:** Set alarms to notify your team before users experience latency.
  
  - When Alarm comes in action ```Normal -> High usage -> Throttling starts -> Latency increases -> Errors -> Users complain```
  - How Alert works
    ```text
    CloudWatch detects high usage
            ->
    Alarm triggers
            ->
    SNS sends alert to team
            ->
    Auto scaling increases capacity
            ->
    System stabilizes BEFORE users notice
    ```
  - **Monitoring Thresholds**

    | Usage Level | Stage | Impact |
    | :--- | :--- | :--- |
    | **60% usage** | Normal | Everything is working fine. **OK** |
    | **85% usage** | Warning | You are getting close to your limit. **Risk** |
    | **100% usage** | Limit Reached | Throttling begins. Requests are queued or slowed.  **Slow** |
    | **>100% demand** | Overload | Requests are rejected. Users see errors. ðŸš¨ **Failures** |
  - What Alarm does
    - Early detection 
    - Automatic recovery 
    - Zero / minimal downtime 

[Read More](./topic-21-monitoring.svg)

---

## 22. Cost Optimization

- **Avoid Scans:** Scans read every item in the table; always prefer `Query`.
  - Why not to use Scan:
    - Table has 1 million records: You want 10 records
    - Scan will: Read all 1M items  Then return 10
    - You pay for 1M reads, not 10
  - Query: Always prefer `Query` because it jumps directly to the data you need using an index.
- **Short Attribute Names:** Since attribute names count toward the 400KB limit, use `u_id` instead of `User_Identification_Number`.
    - Each item can be max 400KB This includes: Keys, Attribute names, Attribute values
    - Cost impact
      - More storage = more cost
      - Larger items = more RCU/WCU consumption
    - Performance impact
      - DynamoDB reads in 4KB chunks
      - Larger item size:
        - More chunks needed
        - More RCU consumed
      - Example
        - Item size	RCU used
        - 1KB	1 RCU
        - 5KB	2 RCUs
- **Optimize GSI:** Only project attributes you actually need into your index.
  - Every GSI:
    - Stores data separately
    - Gets updated on every write
    - Costs extra storage + extra write capacity
    - More data in GSI = more cost Exp 
        - Write to table + Write to GSI  + Write to GSI2 (if exists)
        - If GSI has more attributes: More data written, More WCU consumed
        - 3 GSIs + Large attributes = Cost multiplies 3x+
  
[Read More](./topic-22-cost-optimization.svg)

---

## 23. Multi-Tenant Design (SaaS) 
  - One system serves multiple customers (tenants)
  - Example:
    - Netflix -> millions of users
    - Shopify -> many stores (tenants)
  **Pattern:** `PK = TENANT#<ID>`, `SK = USER#<ID>`
  - All data of a tenant is grouped together: Example
    - PK = TENANT#1, SK = USER#101
    - PK = TENANT#1, SK = ORDER#500
  - Data Isolation
    - Each tenant only accesses their own data
    - No mixing between tenants
  - Fast Queries
    - Fetch all tenant data using:
    - Query PK = TENANT#1
    - No need for Scan
  - Easy Security
    - Use IAM:
    - dynamodb:LeadingKeys = TENANT#<tenantId>
    - Ensures tenant-level access control
  - Possible Problem : Hot Partition Risk
    - Large tenant → heavy traffic on one PK
    - Can cause throttling
  - Solution
    - Use sharding:
    - TENANT#1#SHARD#1
  
- **Result:** Ensures data isolation and allows for efficient querying of all data belonging to a specific customer.

[Read More](./topic-23-multi-tenant.svg)

---

## 24. Time-Series Pattern: 
  - Data that changes over time (with timestamps)
  - Examples:
    - IoT sensor readings
    - Logs
    - User activity
    - Stock price

- **Design:** `PK = SENSOR_ID`, `SK = TIMESTAMP`
  - High-velocity ingestion (fast writes)
    - New data always comes with new timestamp
    - No overwriting
    - Appends smoothly
    - Scales well
  - Efficient range queries (e.g., "Get data from the last 2 hours") Exp ``` Query PK = "S1"
AND SK BETWEEN "10:00" AND "10:10" ```.
  - Why DynamoDb, why no SQL
    - This can be done in SQL as well.

      | Aspect | 🟦 SQL | 🟩 DynamoDB |
      |--------|--------|------------|
      | **1. Storage & Access Pattern** | Data stored in tables (rows) <br> Index is a separate structure | Data stored using PK + SK <br> Physically organized by PK and sorted by SK <br>  No extra index needed for many queries |
      | **2. Performance at Scale** | Works great at normal scale <br> At huge scale: <br> • Index scans <br> • Disk I/O <br> • Sharding complexity | Built for massive scale: <br> • Millions of writes/sec <br> • Massive time-series data <br>  Direct partition lookup + sorted range scan (very fast) |
      | **3. Write Pattern (VERY IMPORTANT)** | Writes can cause: <br> • Locks <br> • Index updates <br> • Contention | Append-only pattern: <br> • S1 → 10:00 <br> • S1 → 10:05 <br> • S1 → 10:10 <br> ✔️ No locks <br> ✔️ No joins <br> ✔️ Highly scalable |
      | **4. Infrastructure Complexity** | You handle: <br> • Scaling <br> • Partitioning <br> • Replication | Fully managed: <br> • Auto scaling |

[Read More](./topic-24-time-series.svg)

---



## 25. Large-Scale Delete Strategy

- **Batch Delete:** Use `BatchWriteItem` (25 items per call).

- **Parallel Processing:** Run multiple threads to scan and delete.
  - DynamoDB does NOT allow bulk delete in one go
  - You can only delete max 25 items per API call using BatchWriteItem
  - Run multiple workers/threads
  - Each worker deletes different batches
    ```
    Example:
    Worker 1 → deletes items 1–1000
    Worker 2 → deletes items 1001–2000 
    Worker 3 → deletes items 2001–3000  
    ```
- **Recreate Table:** If deleting nearly all data, it's often faster/cheaper to export the small portion you need and drop the table.

[Read More](./topic-25-delete-strategy.svg)

---

## 26. SDK Best Practices

- **Connection Reuse:** Use a persistent HTTP connection to avoid TLS handshake overhead.
  - What is "Connection Reuse"?
    - When your Node.js app calls DynamoDB, it communicates over HTTPS.
    - Every request needs:
      - TCP connection
      - TLS handshake (security setup )
      - Then actual API call

  - Without connection reuse:
    - New connection for every request
    - Slow
    - Wastes resources

    - **Request Example**
      - Request 1 → Open connection → TLS handshake → Close
      - Request 2 → Open connection → TLS handshake → Close
      - Request 3 → Open connection → TLS handshake → Close
    - **Problem**
      - TLS handshake is expensive (time + CPU)
      - Adds latency (~50–200ms per request)
      - Wastes resources

  - With connection reuse:
    - Same connection for multiple requests
    - Fast
    - Efficient

    - **Request Example**
      - Request 1 → Use connection → Keep open
      - Request 2 → Use same connection
      - Request 3 → Use same connection
    - **Benefit**
      - No repeated TLS handshakes
      - Lower latency
      - Saves CPU
      - Open connection once → reuse same connection for multiple requests → no repeated handshake
      
    - Example
    ```javascript
    import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
    import { NodeHttpHandler } from "@smithy/node-http-handler";
    import https from "https";

    // Create keep-alive agent
    const agent = new https.Agent({keepAlive: true, maxSockets: 50});
    const client = new DynamoDBClient({
      region: "ap-south-1",
      requestHandler: new NodeHttpHandler({httpsAgent: agent})
    });
    ```

[Read More](./topic-26-sdk-best-practices.svg)

---

## 27. Security Overview

1. **IAM:** Always follow least privilege to reduce blast radius if compromised.
2. **Encryption:** **At Rest** (KMS) and **In Transit** (TLS/HTTPS) are default. 
  - At Rest: Encrypts data on disk (KMS)
    - AWS managed (default )
    - Customer managed (more control)
    
    Example

    ```text
    Data → encrypted → stored
    Read → decrypted → returned
    ```
  - In Transit: Encrypts data in motion (TLS/HTTPS)
    - Nobody can: sniff network, read packets
    Example
    ```text
    App → (HTTPS) → DynamoDB
    ```
   - **Question:** I'm not encrypting data manually, so how is it still encrypted.
   - **Answer:**:DynamoDB protects data in 2 ways: 
      1. Encryption At Rest (Storage level)
          - Exp : If we save like
            > putItem({ name: "Pradeep", phone: "9876543210" });
            
          - What DynamoDB does automatically:
            - Before saving to disk
            - It encrypts your data using AWS Key Management Service (KMS)
            - So internally it becomes something like:
              > Encrypted blob (not readable)
          
      2. Encryption In Transit (Network level)
          - When your app talks to DynamoDB:
            - It uses HTTPS
            - Which uses Transport Layer Security (TLS)
              > Your App → (TLS encrypted) → DynamoDB
            - Example > ``` { "phone": "9876543210" }```
            - It is encrypted while traveling over the network
          
      
3. **VPC Endpoints: (Private Network Access)** Keep traffic within the AWS private network for higher security.
  - Without VPC Endpoint: ```EC2 → Internet → DynamoDB```
    - Goes over public internet
    - Needs NAT Gateway / Internet Gateway
  - With VPC Endpoint: ```EC2 → VPC → DynamoDB```
    - No internet involved
    - Traffic stays inside AWS private network
    - No NAT Gateway needed
    - More secure
  - VPC Flow
    ```text
    App (IAM Role attached)
      ↓
    IAM checks permission
      ↓
    HTTPS (encrypted in transit)
      ↓
    DynamoDB (encrypted at rest via KMS)
      ↓
    Access Granted via VPC Endpoint (private network)
    ```
  

[Read More](./topic-28-security.svg)

---

## 28. Idempotency Keys

Essential for critical operations like payments.

- **Problem:** If a network timeout occurs, the client might retry a request that actually succeeded, leading to duplicate data.
  - **WITHOUT idempotency**
  - Example
  ```text
  Imagine a payment API: User clicks  "Pay ₹1000"
  Client → API → DynamoDB → Payment done 
  Client sees timeout → Retries → DynamoDB → Payment done again ❌
  Result: User charged twice!
  ```
- **Solution: Idempotency Key** Store a unique `requestId` with the item and use a `ConditionExpression` to ensure the write only happens once.
  - Only insert if requestId does NOT already exist
  - Example
  ```json
  {
    "requestId": "txn-12345",
    "userId": "u1",
    "amount": 1000
  }
  ```
  - How DynamoDB helps ```ConditionExpression: "attribute_not_exists(requestId)"``` means: Insert only if this requestId is not already used
  - Result
    ```text
    First request → Success 
    Retry → Fails (requestId already exists) ❌
    No duplicate payment!
    ```

---

## 29. Data Modeling Patterns

Mastering these core patterns is the key to Single-Table Design:

- **One-to-Many:** One user → many orders, Use same PK, different SK
  - Example
    ```text
    PK = USER#1
    SK = PROFILE        → user info  
    SK = ORDER#101      → order 1  
    SK = ORDER#102      → order 2  
    SK = ORDER#103      → order 3  
    ```
  - Query: ```Query PK = USER#1``` → Get all orders for this user
  - Conclusion
    - DynamoDB stores items with same PK together
    - Sorted by SK
- **Many-to-Many:** Using an Adjacency List pattern.
  - Example
    ```text
    Students ↔ Courses
    - One student → many courses
    - One course → many students

    PK = STUDENT#1   SK = COURSE#101
    PK = STUDENT#1   SK = COURSE#102

    PK = COURSE#101  SK = STUDENT#1
    PK = COURSE#101  SK = STUDENT#2
    ```
  - We can query both ways:
    - Get all courses for a student ```PK = STUDENT#1```
    - Get all students for a course ```PK = COURSE#101```
- **Inverted Index:** Swapping PK and SK in a GSI to query relationships from both ends.
  - Problem: Sometimes you want to query from the other side
  - Table Given
    ```text
    Orders stored like:

    PK = USER#1
    SK = ORDER#101
    ```
  - But what if you want to query by order ID?
    - You can't do it with the current table
    - You need a GSI
    - Create a GSI with:
      ```text
      PK = ORDER#101
      SK = USER#1
      ```
    - Now you can query by order ID!

---

## 30. Global Tables Conflict Resolution

When data is updated in multiple regions simultaneously, DynamoDB must decide which update wins.
  - What are Global Tables
    - They allow: ```Same table → multiple AWS regions```
    - Example:
      ```text
      India (ap-south-1)
      US (us-east-1)
      ```
    - Both regions:
      ```text
      Read & write data
      Automatically sync
      ```
  - **The Problem (Conflict)**: What if same item is updated in 2 regions at the same time? 😬
    - At same time:
      ```text
      India updates:
      name = "Pradeep Kumar"
      US updates:
      name = "Pradeep R"
      ```
    - Now conflict: Which value should win? 

- **Mechanism:** **Last Writer Wins (LWW)** based on a system-level timestamp (automatic by aws).
  - Solution: 
    - Last Writer Wins (LWW)
    - DynamoDB adds a hidden timestamp to every item
    - When conflict occurs:
      - The update with the latest timestamp wins
      - No manual intervention needed!
    - How it works
      - Each write has a hidden timestamp:
      ```text
      India write → 10:00:01.100  
      US write → 10:00:02.200
      US write wins 
      ```
- **Alternate Solution**: 
  - Option 1: Avoid multi-region writes
    - ```Only 1 region writes → others read```
  - Option 2: Add versioning:  Reject outdated updates
    ```text
    {
      "version": 2
    }
    ``` 
  - Option 3: Custom conflict handling
    ```text
    Store both versions
    Resolve in application logic
    ```
  - **Final Mental Model**
    ```text
    Multiple regions write → conflict possible  
    DynamoDB → picks latest timestamp → winner
    ```

---

## 31. Latency Optimization 

  - Latency=time taken to get response
  >User clicks → API responds → time taken = latency
  - Goal: Reduce response time as much as possible

- **Connection Reuse & SDK Configuration::** (See [SDK Best Practices](#26-sdk-best-practices) for implementation details).
- **Lambda Optimization:** Use "Provisioned Concurrency" to eliminate cold starts.

---

## 32. Schema Evolution

Since DynamoDB is schemaless, you can add attributes at any time.

- **Best Practice:** Use a `version` attribute or a `type` attribute to help your application code handle items created under different schema versions.

---

## 33. Disaster Recovery (DR)

- **PITR:** Restores your table to any point in time within the last 35 days.

- **Multi-Region:** Use Global Tables to ensure your application can failover to a different region instantly.
- **RTO/RPO:** Define your Recovery Time and Recovery Point objectives early.

---

## 34. Data Migration Strategy

For large-scale migrations (SQL to DynamoDB):

- **Batching:** Don't move everything at once; use batch processing.
- **Parallelism:** Use multiple workers to increase throughput.
- **Monitoring:** Track every failure and implement a dead-letter queue (DLQ) for failed records.

---

## 35. Write & Read Amplification

- **Write Amplification:** Every GSI you add increases the cost of a write, as DynamoDB must update the index as well.

- **Read Amplification:** Fetching a 400KB item when you only need one field is wasteful.
**Solution:** Use **Projection Expressions** to retrieve only the data you need.

---

## 36. Secondary Index Projections

When creating a GSI, choose what data to copy into it:

- **KEYS_ONLY:** Smallest index size, lowest cost.
- **INCLUDE:** Only the specific attributes you query frequently.
- **ALL:** Most flexible, but most expensive.

---

## 37. Distributed System Thinking

DynamoDB is not a traditional SQL server; it is a massive distributed fleet.

- **Design for failure:** Assume retries will happen.
- **Embrace Eventual Consistency:** It provides the highest availability and lowest cost.
- **Focus on Throughput:** Success in DynamoDB is measured by how efficiently you use your RCU and WCU.

---
