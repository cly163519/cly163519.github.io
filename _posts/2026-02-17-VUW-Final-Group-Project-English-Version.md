--
layout: post
title: Times-7 RFID System - Complete Architecture Design Document
date: 2024-01-28
categories: [System Design, Architecture]
tags: [RFID, FastAPI, Supabase, React Native]
lang: en
---

# Times-7 RFID System - Complete Architecture Design Document

## Part 1: System Architecture Overview

### 1.1 Overall Architecture

```
Data Source Layer (RFID Reader / Simulators)
    ↓
Processing Layer (FastAPI Gateway)
    ↓
Storage Layer (Supabase PostgreSQL + Storage)
    ↓
Display Layer (Expo React Native Frontend)
```

### 1.2 Detailed Component Explanation

#### Data Source Layer: Three Types of Readers

| Source | Type | Interface | Data Format |
|--------|------|-----------|-------------|
| **Real Reader** | Impinj Reader (Hardware) | `GET {READER_BASE_URL}/data/stream` | NDJSON stream (tagInventory events) |
| **Simulator 1** | Stream Simulator (Software) | `GET /data/stream` | NDJSON stream (read from t7datastream.ndjson) |
| **Simulator 2** | Terminal Simulator (CLI) | `POST /api/sim/reader/events {tagIds:[...]}` | HTTP POST + JSON body |

**Key Point**: All three data sources feed into the FastAPI Gateway with unified `tagInventory` event format.

---

#### Processing Layer: FastAPI Gateway

**Startup Flow**:
```
FastAPI application starts
    ↓
app.on_event("startup") triggers run_reader_stream()
    ↓
Connect to Reader data stream (any or all three)
    ↓
Process tagInventory events in real-time
```

**Core Modules**:

1. **ActiveTags (3-second sliding window)**
   - Function: Maintain recently active tags in the last 3 seconds
   - Implementation: Call `sync_seen(tag_id, now)` whenever a tag is detected
   - Purpose: Dashboard polls this 3-second window of active tags

2. **TagInfoCache (24-hour TTL)**
   - Function: Cache IAS query results
   - Reason: A tag's authenticity result remains stable for 24 hours, avoid repeated queries
   - Workflow:
     - Tag first appears → Call IAS Service → Cache result
     - Same tag appears again → Read from cache directly (skip IAS)
     - After 24h → Cache expires, next appearance triggers new IAS query

3. **IAS Lookup (Authenticity verification)**
   - Type: Optional mock or real service (controlled by IAS_MODE env var)
   - Input: tag_id (tidHex)
   - Output: `{authentic: bool, brand: str, model: str, confidence: float, ...}`

4. **DB Writer (Data persistence)**
   - Trigger: First time a tag appears (or cache expires)
   - Operation: `upsert_latest_tag()` → Write to Supabase data table
   - Strategy: Upsert by tag_id (keep only the latest record per tag)

5. **External API Endpoints**
   - `GET /api/active-tags` → Return active tags from last 3 seconds `[ScanResult, ...]`
   - `POST /api/sim/reader/events` → Terminal Simulator inject events (testing)
   - `/debug/*` → Debug endpoints (optional, e.g., clear cache)

---

#### Storage Layer: Supabase (PostgreSQL + Storage)

**Database Tables**:

| Table Name | Fields | Description |
|------------|--------|-------------|
| **data** | id (PK), date, auth, info | Scan result snapshot. id=tag_id (upsert strategy), keeps only the latest record per tag |
| **product_info** | tid (PK), epc, description, origin, produced_on | Product metadata. Maintained by frontend NewProduct/ViewItem modals |
| **product_photo** | id, tid (FK), photo_url, created_at | Product image metadata. One product can have multiple photos, stores Storage public URLs |

**Object Storage** (Storage Bucket):

| Bucket Name | Path Structure | Purpose |
|-------------|----------------|---------|
| **product-photos** | `{tid}/{timestamp}.jpg` | Store product image files, generate public_url for frontend use |

---

#### Display Layer: Expo React Native Frontend

**5 Main Screens**:

1. **Dashboard (Home)**
   - Display: Real-time detected tags list (from Gateway /api/active-tags)
   - Functions:
     - Poll Gateway every 1 second for active tags
     - Query product_info for each tag_id to check if registered
     - Show product info if registered, else show "Not Registered"
     - Click tag card to open ViewItem or NewProduct modal

2. **Logs Page (History)**
   - Display: Latest scan records of all tags (from Supabase data table)
   - Query: `SELECT * FROM data ORDER BY date DESC`
   - Click a record to open ViewItem modal

3. **Search Page (Product Search)**
   - Input: tid or epc (exact match)
   - Query Method: **Direct query to Supabase product_info (bypass Gateway)**
     1. First search `WHERE tid == input`
     2. If not found, then search `WHERE epc == input`
   - Display: Five fields - tid, epc, description, origin, produced_on

4. **NewProduct Modal (Register New Product)**
   - Trigger: When unregistered tag is detected on Dashboard
   - Operations:
     1. Write to product_info (tid, epc, description, origin, produced_on)
     2. Optional: Upload image to Storage bucket → Get public_url
     3. Optional: Insert product_photo record (tid, photo_url)

5. **ViewItem Modal (View/Edit Product)**
   - Trigger: Click registered tag or Logs page record
   - Operations:
     - Query product_info to read product data
     - Query latest product_photo (`WHERE tid=... ORDER BY created_at DESC LIMIT 1`)
     - Can edit any field → Update product_info
     - Can upload new image → Insert new product_photo record

---

### 1.3 Data Flow Diagram (Text Version)

```
Reader Data Stream
    ↓
FastAPI Gateway
    ├─ ActiveTags Module (maintain 3-second window)
    ├─ TagInfoCache Module (24h TTL cache)
    │   └─ First query IAS Service
    │   └─ Store result in cache
    ├─ DB Writer (first time or cache expires)
    │   └─ Upsert to Supabase data table
    └─ API Endpoint
        └─ GET /api/active-tags (for frontend polling)
                    ↓
Frontend Dashboard
    ├─ Poll Gateway /api/active-tags every 1s
    ├─ Get latest 3s active tags
    ├─ Query Supabase product_info for each tag_id
    └─ Render tag cards (registered/unregistered)

Frontend Logs Page
    └─ Direct query Supabase data table (ORDER BY date DESC)
        └─ Show all tags' latest records

Frontend Search Page
    └─ Direct query Supabase product_info table (exact match)
        └─ Display product details

Frontend NewProduct/ViewItem Modal
    └─ Query/Write Supabase product_info
    └─ Upload images to Storage
    └─ Write product_photo records
```

---

## Part 2: Core Data Flows

### 2.1 Data Flow A: Real-time Detection Flow (Dashboard → Detected Items)

#### A1. Timeline

```
Time 0ms    : Reader detects tag
     0-100ms : Data streams to Gateway through network
     100-200ms: Gateway updates ActiveTags
     200-500ms: Cache check + IAS query (first time) / or cache read (repeat)
     500-800ms: DB write to Supabase data table
     1000ms   : Frontend polls GET /api/active-tags
     1000-1500ms: Frontend queries product_info to check registration status
     1500ms   : Dashboard displays tag result

Total Latency: ~1-2 seconds (user sees detection result)
```

#### A2. Detailed Process

**Step 1: Tag Detected by Reader**
```
Reader (Impinj / Simulator)
    ↓
Send NDJSON event stream
    ↓
Each event contains:
  {
    "eventType": "tagInventory",
    "tagInventoryEvent": {
      "tidHex": "3015E2C1A000000000000001",
      "epcHex": "3034257BF411E4000000001E",
      "rssi": -45
    }
  }
```

**Step 2: Gateway Receives and Processes**
```
FastAPI's run_reader_stream() function
    ↓
Monitor Reader data stream (loop read NDJSON)
    ↓
Filter eventType == "tagInventory"
    ↓
Extract tag_id = tidHex
    ↓
Sync to ActiveTags (mark as "just seen")
```

**Step 3: Check TagInfoCache**
```
if tag_id in cache and not expired:
    ├─ Cache hit ✓
    ├─ Read {auth, info} from cache
    └─ Only update ActiveTags' last_seen time (skip IAS, skip DB)

else:
    ├─ Cache miss (first time or expired)
    ├─ Call IAS Service
    │  └─ Input: tag_id
    │  └─ Output: {authentic: bool, info: {...}}
    ├─ Store in TagInfoCache (TTL 24h)
    └─ Write to Supabase data table
       └─ Operation: UPSERT data (id, date, auth, info) ON CONFLICT(id) DO UPDATE
```

**Step 4: Frontend Polling**
```
Frontend Dashboard every 1 second:
    ↓
GET /api/active-tags
    ↓
Gateway returns active tags from last 3 seconds
    [
      {
        "id": "3015E2C1A000000000000001",
        "date": "2024-01-28T10:30:45Z",
        "auth": true,
        "info": {...}
      },
      ...
    ]
    ↓
Frontend queries product_info for each tag
    ├─ If exists → Tag card shows product info + "Registered" badge
    └─ If not exists → Tag card shows "Not Registered" + [Register] button
    ↓
Render to Dashboard
```

**Step 5: User Interaction (Optional)**
```
User clicks "Not Registered" tag
    ↓
Open NewProduct Modal
    ↓
Fill form + Save
    ↓
Write to product_info
    ↓
(Optional) Upload image → Storage → product_photo
    ↓
Next polling cycle, this tag displays as "Registered"
```

---

### 2.2 Data Flow B: Product Registration/Edit Flow (NewProduct / ViewItem Modal)

#### B1. New Product Registration Flow (NewProduct Modal)

```
Scenario: Dashboard finds unregistered tag, user clicks "Register"

Flow:
  ┌─────────────────────────────────────────┐
  │ 1. Modal Opens                          │
  │    Display form:                        │
  │    - tid (read-only, auto-filled)       │
  │    - epc (user input)                   │
  │    - description (user input)           │
  │    - origin (user input)                │
  │    - produced_on (date picker)          │
  │    - [Select Image] (optional)          │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 2. User Fills Form                      │
  │    - Input all fields                   │
  │    - (Optional) Select product image    │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 3. Click [Save]                         │
  │    Frontend validates input not empty   │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 4. Upsert product_info                  │
  │    INSERT INTO product_info (           │
  │      tid, epc, description, origin,     │
  │      produced_on                        │
  │    ) VALUES (...) ON CONFLICT DO UPDATE │
  │                                          │
  │    → Supabase returns success            │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 5. (If user selected image)             │
  │    Upload to Supabase Storage           │
  │    - Path: product-photos/{tid}/{ts}.jpg│
  │    - Get public_url                     │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 6. (If upload successful)               │
  │    Insert product_photo record          │
  │    INSERT INTO product_photo (          │
  │      tid, photo_url, created_at         │
  │    ) VALUES (...)                       │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 7. Modal Closes                         │
  │    Next Dashboard polling or refresh,   │
  │    tag displays as "Registered" with    │
  │    product information                  │
  └─────────────────────────────────────────┘
```

#### B2. View/Edit Product Flow (ViewItem Modal)

```
Scenario 1: User clicks registered tag on Dashboard
Scenario 2: User clicks record on Logs page

Flow:
  ┌─────────────────────────────────────────┐
  │ 1. Modal Opens, Auto-load              │
  │                                        │
  │    Query 1:                            │
  │    SELECT * FROM product_info          │
  │    WHERE tid = {tag_id}                │
  │    → Return: tid, epc, description,    │
  │      origin, produced_on               │
  │                                        │
  │    Query 2:                            │
  │    SELECT * FROM product_photo         │
  │    WHERE tid = {tag_id}                │
  │    ORDER BY created_at DESC LIMIT 1    │
  │    → Return: latest photo's photo_url  │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 2. Modal Shows Product Details          │
  │    - tid, epc, description, origin      │
  │    - produced_on                        │
  │    - Latest image                       │
  │    - [Edit] button                      │
  │    - [Upload New Image] button          │
  │    - [Delete Product] button (optional) │
  └─────────────────────────────────────────┘
                    ↓
         User chooses one action:
                    ↓
  ┌──────────────────────────┬──────────────────────────┐
  │ Action A: Edit Field     │ Action B: Upload Image    │
  ├──────────────────────────┼──────────────────────────┤
  │ Click field to modify    │ Click [Upload New Image] │
  │ (e.g., epc/description)  │                          │
  │       ↓                  │ Select image file         │
  │ UPDATE product_info      │       ↓                  │
  │ SET {field} = ...        │ POST to Storage bucket   │
  │ WHERE tid = ...          │ product-photos/{tid}     │
  │       ↓                  │       ↓                  │
  │ Supabase update success  │ Get public_url           │
  │       ↓                  │       ↓                  │
  │ Modal refreshes, shows   │ INSERT product_photo     │
  │ updated content          │ (tid, photo_url, ...)    │
  │                          │       ↓                  │
  │                          │ Modal shows new image    │
  └──────────────────────────┴──────────────────────────┘
```

---

### 2.3 Data Flow C: History Page (Logs Page)

```
Scenario: User opens Logs page to view scan history

Flow:
  ┌────────────────────────────────────────┐
  │ 1. LogsPage Component Loads            │
  │    Trigger useEffect or onMount event  │
  └────────────────────────────────────────┘
                    ↓
  ┌────────────────────────────────────────┐
  │ 2. Query Supabase data Table           │
  │                                        │
  │    SELECT *                            │
  │    FROM data                           │
  │    ORDER BY date DESC                  │
  │    LIMIT 100                           │
  │                                        │
  │    Return: Latest 100 scan records     │
  │    (each is latest snapshot per tag)   │
  └────────────────────────────────────────┘
                    ↓
  ┌────────────────────────────────────────┐
  │ 3. Frontend Renders List                │
  │                                        │
  │    For each row display:               │
  │    - tid (tag ID)                      │
  │    - date (latest scan time)           │
  │    - auth (✓ Authentic / ✗ Suspicious)│
  │    - info (complete IAS info)          │
  │                                        │
  │    List items are clickable            │
  └────────────────────────────────────────┘
                    ↓
  ┌────────────────────────────────────────┐
  │ 4. User Interaction (Optional)          │
  │                                        │
  │    Click a record                      │
  │         ↓                              │
  │    Open ViewItem Modal                 │
  │    (view/edit product info for tag)    │
  │    (see Data Flow B2)                  │
  └────────────────────────────────────────┘

Important Note:
  data table uses upsert strategy (latest only)
  So Logs page shows snapshot of each tag's latest scan
  Not complete historical trajectory
  
  For complete history, create separate
  scan_history table with INSERT (not upsert)
```

---

### 2.4 Data Flow D: Search Function (Search Page)

```
Scenario: User enters tid or epc on Search page to find product

Flow:
  ┌──────────────────────────────────────┐
  │ 1. SearchPage Shows Search Box        │
  │    User enters search term q          │
  │    (can be tid or epc)                │
  └──────────────────────────────────────┘
                    ↓
  ┌──────────────────────────────────────┐
  │ 2. User Clicks [Search]               │
  │    Frontend validates input not empty │
  └──────────────────────────────────────┘
                    ↓
  ┌──────────────────────────────────────┐
  │ 3. Query 1: Exact Match on tid        │
  │                                      │
  │    SELECT tid, epc, description,     │
  │           origin, produced_on        │
  │    FROM product_info                 │
  │    WHERE tid = q                     │
  │    (exact match, not fuzzy)          │
  │                                      │
  │    [Found?]                          │
  │    ├─ Yes → Jump to Step 5 display   │
  │    └─ No → Continue to Step 4        │
  └──────────────────────────────────────┘
                    ↓
  ┌──────────────────────────────────────┐
  │ 4. Query 2: Exact Match on epc        │
  │                                      │
  │    SELECT tid, epc, description,     │
  │           origin, produced_on        │
  │    FROM product_info                 │
  │    WHERE epc = q                     │
  │    (exact match)                     │
  │                                      │
  │    [Found?]                          │
  │    ├─ Yes → Jump to Step 5 display   │
  │    └─ No → Go to Step 6 not found    │
  └──────────────────────────────────────┘
                    ↓
  ┌──────────────────────────────────────┐
  │ 5. Display Search Result Card         │
  │                                      │
  │    Show five fields:                 │
  │    ✓ tid                             │
  │    ✓ epc                             │
  │    ✓ description                     │
  │    ✓ origin                          │
  │    ✓ produced_on                     │
  │                                      │
  │    User can click card to open       │
  │    ViewItem Modal for editing        │
  └──────────────────────────────────────┘
                    ↓
                  Done

  ┌──────────────────────────────────────┐
  │ 6. Display "Item not found"           │
  │                                      │
  │    Neither tid nor epc match         │
  │    Suggest user try again or         │
  │    register new product on Dashboard │
  └──────────────────────────────────────┘

Key Features:
  ✓ Search bypasses Gateway (direct Supabase query)
  ✓ Exact match (no fuzzy search)
  ✓ Try tid first, then epc
  ✓ Return complete 5-field product info
```

---

## Part 3: Database Table Structure Details

### 3.1 data Table (Scan Result Snapshot Table)

**Purpose**: Store latest scan result per tag, backend writes, frontend Logs reads

**Table Structure**:
```
data (
  id         VARCHAR(255) PRIMARY KEY,      -- tag_id (tidHex)
  date       TIMESTAMP NOT NULL,             -- Last scan time
  auth       BOOLEAN NOT NULL,               -- IAS verification result
  info       JSONB NOT NULL                  -- Complete IAS info object
)
```

**Field Details**:

| Field | Data Type | Description | Example |
|-------|-----------|-------------|---------|
| id | VARCHAR(PK) | Tag's tidHex as primary key | `"3015E2C1A000000000000001"` |
| date | TIMESTAMP | Last scan time (auto-updated) | `2024-01-28T10:30:45.123Z` |
| auth | BOOLEAN | IAS result (true=authentic, false=suspicious) | `true` or `false` |
| info | JSONB | Complete info from IAS service | `{"authentic": true, "brand": "Nike", "confidence": 0.99, ...}` |

**Key Characteristics**:
- **Upsert strategy**: By `id` (tag_id) as primary key, INSERT for new tags, UPDATE for existing
- **Latest only**: Each tag has one record (newest), not complete history
- **Usage**: Logs page `SELECT * FROM data ORDER BY date DESC` query

---

### 3.2 product_info Table (Product Metadata Table)

**Purpose**: Store registered product info, frontend NewProduct/ViewItem Modal maintains

**Table Structure**:
```
product_info (
  tid          VARCHAR(255) PRIMARY KEY,    -- Tag ID
  epc          VARCHAR(255) UNIQUE,         -- EPC code
  description  TEXT,                        -- Product description
  origin       VARCHAR(255),                -- Origin country
  produced_on  DATE,                        -- Production date
  created_at   TIMESTAMP DEFAULT NOW(),     -- Creation time
  updated_at   TIMESTAMP DEFAULT NOW()      -- Last update time
)
```

**Field Details**:

| Field | Data Type | Description | Example |
|-------|-----------|-------------|---------|
| tid | VARCHAR(PK) | Tag ID (linked to data table's id) | `"3015E2C1A000000000000001"` |
| epc | VARCHAR(UNIQUE) | EPC code (unique, for Search exact query) | `"3034257BF411E4000000001E"` |
| description | TEXT | Product description | `"Nike Air Max 2024"` |
| origin | VARCHAR | Origin/manufacturer location | `"Vietnam"` |
| produced_on | DATE | Production date | `2024-01-15` |
| created_at | TIMESTAMP | Creation time | `2024-01-28T10:30:00Z` |
| updated_at | TIMESTAMP | Last update time | `2024-01-28T11:45:30Z` |

**Query Examples**:

```sql
-- Dashboard check if tag registered
SELECT * FROM product_info WHERE tid = 'tag_id' LIMIT 1;

-- Search exact match by tid
SELECT tid, epc, description, origin, produced_on 
FROM product_info WHERE tid = q LIMIT 1;

-- Search exact match by epc
SELECT tid, epc, description, origin, produced_on 
FROM product_info WHERE epc = q LIMIT 1;

-- ViewItem Modal read product info
SELECT * FROM product_info WHERE tid = 'tag_id';

-- Edit: update fields
UPDATE product_info SET epc = '...', description = '...' WHERE tid = 'tag_id';
```

---

### 3.3 product_photo Table (Product Image Metadata Table)

**Purpose**: Store product image URLs and metadata, linked to product_info

**Table Structure**:
```
product_photo (
  id         SERIAL PRIMARY KEY,            -- Auto-increment PK
  tid        VARCHAR(255) NOT NULL,         -- Tag ID (FK)
  photo_url  VARCHAR(500) NOT NULL,         -- Storage public URL
  created_at TIMESTAMP DEFAULT NOW()        -- Upload time
  
  FOREIGN KEY (tid) REFERENCES product_info(tid) ON DELETE CASCADE
)
```

**Field Details**:

| Field | Data Type | Description | Example |
|-------|-----------|-------------|---------|
| id | SERIAL(PK) | Auto-increment primary key | `1, 2, 3, ...` |
| tid | VARCHAR(FK) | Tag ID (links to product_info) | `"3015E2C1A000000000000001"` |
| photo_url | VARCHAR | Public URL from Supabase Storage | `"https://xyz.supabase.co/storage/v1/object/public/product-photos/3015E2C1A.../1705049445000.jpg"` |
| created_at | TIMESTAMP | Upload time | `2024-01-28T10:30:00Z` |

**Key Characteristics**:
- **One-to-many**: One product (tid) can have multiple images
- **Cascade delete**: Deleting product_info auto-deletes associated product_photo records
- **Get latest image**: `SELECT * FROM product_photo WHERE tid = ? ORDER BY created_at DESC LIMIT 1`

**Query Examples**:

```sql
-- ViewItem Modal get latest image
SELECT photo_url FROM product_photo 
WHERE tid = 'tag_id' 
ORDER BY created_at DESC LIMIT 1;

-- Get all images for a product (oldest to newest)
SELECT * FROM product_photo 
WHERE tid = 'tag_id' 
ORDER BY created_at ASC;

-- Insert new image
INSERT INTO product_photo (tid, photo_url) 
VALUES ('tag_id', 'https://...');

-- Delete specific image
DELETE FROM product_photo WHERE id = photo_id;
```

---

### 3.4 Supabase Storage - product-photos Bucket

**Purpose**: Store product image files

**Path Structure**:
```
product-photos/
├─ 3015E2C1A000000000000001/
│  ├─ 1705049445000.jpg        (timestamp in milliseconds)
│  ├─ 1705049512000.jpg
│  └─ ...
├─ 3015E2C1A000000000000002/
│  ├─ 1705049600000.jpg
│  └─ ...
└─ ...
```

**Public URL Example**:
```
https://xyz.supabase.co/storage/v1/object/public/product-photos/3015E2C1A000000000000001/1705049445000.jpg
```

**Permission Settings**:
- **Read**: **Public** (anyone can access, frontend uses URL directly)
- **Upload**: **Authenticated** (only logged-in users, via supabase-js SDK)

---

## Part 4: Complete API Reference

### 4.1 Gateway API (FastAPI)

#### Endpoint 1: GET /api/active-tags

**Function**: Get active tags from last 3 seconds

**Request**:
```http
GET http://localhost:3000/api/active-tags
```

**Response** (200 OK):
```json
[
  {
    "id": "3015E2C1A000000000000001",
    "date": "2024-01-28T10:30:45.123Z",
    "auth": true,
    "info": {
      "authentic": true,
      "brand": "Nike",
      "model": "Air Max 2024",
      "confidence": 0.99
    }
  },
  {
    "id": "3015E2C1A000000000000002",
    "date": "2024-01-28T10:30:47.456Z",
    "auth": false,
    "info": {
      "authentic": false,
      "brand": "Unknown",
      "confidence": 0.45
    }
  }
]
```

**Usage Scenario**: Frontend Dashboard polls every 1 second

**Frontend Code**:
```javascript
const response = await fetch('http://localhost:3000/api/active-tags');
const activeTags = await response.json();
```

---

#### Endpoint 2: POST /api/sim/reader/events

**Function**: Terminal Simulator inject mock tag events (testing only)

**Request**:
```http
POST http://localhost:3000/api/sim/reader/events
Content-Type: application/json

{
  "tagIds": [
    "3015E2C1A000000000000001",
    "3015E2C1A000000000000002",
    "3015E2C1A000000000000003"
  ]
}
```

**Response** (200 OK):
```json
{
  "status": "ok",
  "count": 3,
  "message": "3 tags injected successfully"
}
```

**Test Command**:
```bash
curl -X POST http://localhost:3000/api/sim/reader/events \
  -H "Content-Type: application/json" \
  -d '{"tagIds": ["tid1", "tid2", "tid3"]}'
```

---

### 4.2 Supabase JavaScript SDK Common Queries

#### Query 1: Check if Product is Registered

```javascript
const { data, error } = await supabase
  .from('product_info')
  .select('*')
  .eq('tid', tag_id)
  .single();

if (data) {
  // Registered
  console.log(`Product: ${data.epc}, ${data.description}`);
} else {
  // Not registered
  console.log('This tag is not registered');
}
```

---

#### Query 2: Insert or Update Product Info

```javascript
const { data, error } = await supabase
  .from('product_info')
  .upsert({
    tid: tag_id,
    epc: 'EPC_CODE',
    description: 'Nike Air Max 2024',
    origin: 'Vietnam',
    produced_on: '2024-01-15'
  })
  .select();

if (error) {
  console.error('Save failed:', error);
} else {
  console.log('Saved successfully:', data);
}
```

---

#### Query 3: Upload Image to Storage

```javascript
const file = selectedImage;  // File object
const fileName = `${tag_id}/${Date.now()}.jpg`;

const { data, error } = await supabase.storage
  .from('product-photos')
  .upload(fileName, file);

if (error) {
  console.error('Upload failed:', error);
} else {
  // Get public URL
  const { data: urlData } = supabase.storage
    .from('product-photos')
    .getPublicUrl(fileName);
  
  const public_url = urlData.publicUrl;
  console.log('Image URL:', public_url);
  
  // Insert product_photo record
  await supabase
    .from('product_photo')
    .insert({
      tid: tag_id,
      photo_url: public_url
    });
}
```

---

#### Query 4: Get Product History (Logs Page)

```javascript
const { data, error } = await supabase
  .from('data')
  .select('*')
  .order('date', { ascending: false })
  .limit(100);

if (error) {
  console.error('Query failed:', error);
} else {
  // data is history records array
  data.forEach(record => {
    console.log(
      `${record.id}: ${record.auth ? '✓ Authentic' : '✗ Suspicious'} @ ${record.date}`
    );
  });
}
```

---

#### Query 5: Search Products (Search Page)

```javascript
async function searchProduct(query) {
  // First search by tid (exact)
  let { data, error } = await supabase
    .from('product_info')
    .select('tid, epc, description, origin, produced_on')
    .eq('tid', query)
    .single();
  
  // If not found, search by epc (exact)
  if (!data && !error) {
    ({ data, error } = await supabase
      .from('product_info')
      .select('tid, epc, description, origin, produced_on')
      .eq('epc', query)
      .single());
  }
  
  return data;  // Return object if found, null if not
}

// Usage
const result = await searchProduct('3015E2C1A000000000000001');
if (result) {
  console.log('Found product:', result);
} else {
  console.log('Item not found');
}
```

---

#### Query 6: Get Latest Product Image (ViewItem Modal)

```javascript
const { data: photo, error } = await supabase
  .from('product_photo')
  .select('photo_url')
  .eq('tid', tag_id)
  .order('created_at', { ascending: false })
  .limit(1)
  .single();

if (photo) {
  // Display image
  console.log('Latest image:', photo.photo_url);
  // 
} else {
  // No image
  console.log('This product has no image');
}
```

---

## Part 5: Critical Design Decisions

### 5.1 Why is ActiveTags Window 3 Seconds?

```
Comparison of alternatives:

1-second window:
  ❌ Tags easily missed
  ❌ RFID detection has network latency, 1s too short causes flickering
  ❌ Reader scan frequency unstable

3-second window:
  ✓ Covers detection latency
  ✓ Relatively stable tag list for user view
  ✓ Industry standard for RFID readers
  ✓ Frontend 1s polling also fast enough

5+ second window:
  ❌ Already-left tags still displayed, confuses user
  ❌ Too poor real-time performance
```

---

### 5.2 Why is TagInfoCache TTL 24 Hours?

```
Why cache at all?
  IAS Service query is expensive:
  - Network round-trip latency
  - Remote server processing time
  - Possible database calls

Tag's authenticity stays stable for 24h:
  - Once verified, result very stable
  - Won't suddenly flip from authentic to suspicious
  - 24h is reasonable trust window

24-hour trade-off:
  ✓ Avoid repeated queries, improve performance
  ✓ Reduce IAS service load
  ✓ User experience fast (cache hit)
  ⚠ If tag info changes within 24h, need manual cache clear
```

---

### 5.3 Why Does data Table Use Upsert Instead of Insert?

```
If insert every time:

Problems:
  ❌ data table explodes in row count
     (e.g., 1000 tags × 10 scans = 10,000 rows)
  ❌ Wasted storage space
  ❌ Logs page queries slow (SELECT scans many rows)
  ❌ Can't quickly see "current state"

Using upsert (by tag_id):

Advantages:
  ✓ data table keeps only latest data
  ✓ Row count = tag count (controllable)
  ✓ Queries fast (1 row per tag)
  ✓ Logs page performance good

Trade-off:
  ⚠ Lose complete scan history
     (can't see full "when was this tag scanned" trajectory)

If complete history needed:
  Solution: Add separate scan_history table
  - scan_history: INSERT every time (no upsert)
  - data: UPSERT every time (latest only)
  - Logs can query scan_history for complete history
```

---

### 5.4 Why Doesn't Search Go Through Gateway?

```
If search went through Gateway:

Problems:
  ❌ Gateway has no product_info data (only scan data in data table)
  ❌ Need extra /api/search endpoint development
  ❌ Mixes data sources in Gateway
  ❌ Extra network hop (API instead of direct query)

Direct Supabase Query:

Advantages:
  ✓ product_info is frontend-maintained data, clear logic
  ✓ supabase-js SDK solves in one line
  ✓ Shortest network path, fast response
  ✓ Gateway focuses on scan data

Disadvantages:
  ⚠ Frontend directly accesses database
  ⚠ Needs proper RLS (Row Level Security) policy
  ⚠ Frontend has access to entire product_info table
```

---

## Summary Checklist

```
System Components:
  ✓ Three Reader types (real Impinj + two Simulators)
  ✓ FastAPI Gateway (ActiveTags + TagInfoCache + IAS + DB Writer)
  ✓ Supabase database (data / product_info / product_photo tables)
  ✓ Supabase Storage (product-photos bucket)
  ✓ Expo React Native frontend (5 main screens)

Core Functions:
  ✓ Real-time detection (3s window + 1s frontend poll)
  ✓ Product registration (NewProduct Modal + image upload)
  ✓ Product editing (ViewItem Modal + image management)
  ✓ History records (Logs page + time-ordered)
  ✓ Exact search (Search page + tid/epc)

Key Decisions:
  ✓ ActiveTags 3s - balance detection stability and real-time
  ✓ TagInfoCache 24h - avoid repeated IAS queries
  ✓ data table upsert - keep table size manageable
  ✓ Search direct Supabase - simplify architecture, fast response
```

---

## Disclaimer: 
## This system is part of a group project at VUW University. The entire project was completed collaboratively by group members Egan, Tane, Rebekka, and Laura. 
## This document is for personal study only and was completed using Claude + ChatGPT; it is not an original work. My main contribution was the front-end database connection and search mechanism. 
## Feel free to read and learn, but please do not forward it!

