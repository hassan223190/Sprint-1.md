Sprint 2 Manual: Catalog Data Foundation
Project: TechSphere Catalog Data Foundation & Administrative System
Course: E-Commerce
Sprint 2: Catalog Data Foundation & Domain Implementation

1. Sprint Goal and Scope Boundary

1.1 Sprint Goal
Given a product catalog administrator, the system must persist categories, products, variants, and SKUs without losing identity, relationship, price, or inventory meaning.

1.2 In Scope
Category Tree Management: Parent-child category hierarchies with unique slugs and cycle prevention.
Product Lifecycle: Creation, editing, status tracking (draft/published/archived), and category assignment.
Variants & SKUs: Unique SKU codes, price precision (DECIMAL/integer minor unit), stock validation, and availability rules.
Authenticated Administration: Secure CRUD endpoints restricted to authorized admin roles via JWT/Session security.
Data Integrity & Schema Rules: Database constraints (PK, FK, unique keys, check constraints on stock >= 0), migrations, seed data fixtures, and automated test suite.

1.3 Out of Scope
Public storefront catalog search and dynamic faceted filters (deferred to Sprint 3).
Asset binary uploading or S3 integration (stubbed metadata/URLs used).
Payment gateway integrations, shopper cart checkout, and order execution (deferred to Sprint 3/4).

3. Link to Sprint 1 Decisions (Reused & Modified)
Architecture: Reused Node.js / Express backend with PostgreSQL database and React administration interface.
Authentication: Reused JWT token-based auth mechanism from Sprint 1 for securing /api/v1/admin/* routes.
Schema Evolution: Extended Sprint 1 ERD entities (USERS, PRODUCTS, CATEGORIES, ORDERS, CARTS) by adding normalized VARIANTS, SKUS, ASSETS, and SPECIFICATIONS (JSONB) tables.

4. Updated ERD and Data Dictionary

3.1 Entity-Relationship Structure
+-----------------------------------------------------------------------------------+

|                                  CATEGORIES                                       |

|---|

| PK  | id         | INT          | Auto-increment primary key                      |

| FK  | parent_id  | INT          | Ref: CATEGORIES(id) ON DELETE SET NULL          |

|     | name       | VARCHAR(255) | NOT NULL                                        |

|     | slug       | VARCHAR(255) | UNIQUE, NOT NULL                                |

|     | is_active  | BOOLEAN      | DEFAULT true                                    |

|     | created_at | TIMESTAMP    | NOT NULL                                        |

|     | updated_at | TIMESTAMP    | NOT NULL                                        |

+-----+------------+--------------+-------------------------------------------------+

       |          |

       | (parent) | (categorizes)

       v          v

+-----------------------------------------------------------------------------------+

|                                   PRODUCTS                                        |

+-----------------------------------------------------------------------------------+

| PK  | id             | INT          | Auto-increment primary key                  |

| FK  | category_id    | INT          | Ref: CATEGORIES(id) ON DELETE RESTRICT      |

|     | name           | VARCHAR(255) | NOT NULL                                    |

|     | slug           | VARCHAR(255) | UNIQUE, NOT NULL                            |

|     | description    | TEXT         | NOT NULL                                    |

|     | status         | VARCHAR(20)  | CHECK (status IN ('draft','published',...)) |

|     | specifications | JSONB        | DEFAULT '{}'                                |

|     | created_at     | TIMESTAMP    | NOT NULL                                    |

|     | updated_at     | TIMESTAMP    | NOT NULL                                    |

+-----------------------------------------------------------------------------------+

       |                  |

       | (has)            | (displays)

       v                  v

+------------------+    +-----------------------------------------------------------+

|     VARIANTS     |    |                          ASSETS                           |

+------------------+    +-----------------------------------------------------------+

| PK | id          |    | PK | id          | INT          | Auto-increment primary PK |

| FK | product_id  |    | FK | product_id  | INT          | Ref: PRODUCTS(id) CASCADE |

|    | name        |    |    | storage_url | VARCHAR(512) | NOT NULL                  |

|    | option_val  |    |    | role        | VARCHAR(20)  | CHECK ('primary', ...)    |

+------------------+    |    | alt_text    | VARCHAR(255) | DEFAULT ''                |

       |                |    | sort_order  | INT          | DEFAULT 0                 |

       | (materializes) +-----------------------------------------------------------+

       v

+-----------------------------------------------------------------------------------+

|                                     SKUS                                          |

+-----------------------------------------------------------------------------------+

| PK  | id             | INT          | Auto-increment primary key                  |

| FK  | variant_id     | INT          | Ref: VARIANTS(id) ON DELETE CASCADE         |

|     | sku_code       | VARCHAR(100) | UNIQUE, NOT NULL                            |

|     | price          | NUMERIC(12,2)| CHECK (price >= 0)                          |

|     | stock_quantity | INT          | CHECK (stock_quantity >= 0)                 |

|     | is_active      | BOOLEAN      | DEFAULT true                                |

|     | created_at     | TIMESTAMP    | NOT NULL                                    |

|     | updated_at     | TIMESTAMP    | NOT NULL                                    |

+-----------------------------------------------------------------------------------+

3.2 Data Dictionary & Integrity Rules
Entity
Field
Type
Constraints & Policies
CATEGORIES
 id
 INT (PK)
 Auto-increment primary key


 parent_id
 INT (FK)
 References CATEGORIES(id), ON DELETE SET NULL, cycle prevention check


 slug
 VARCHAR(255)
 UNIQUE, NOT NULL
PRODUCTS
 id
 INT (PK)
 Auto-increment primary key


 category_id
 INT (FK)
 References CATEGORIES(id), ON DELETE RESTRICT


 slug
 VARCHAR(255)
 UNIQUE, NOT NULL


 status
 VARCHAR(20)
 CHECK (status IN ('draft', 'published', 'archived'))
SKUS
 id
 INT (PK)
 Auto-increment primary key


 variant_id
 INT (FK)
 References VARIANTS(id), ON DELETE CASCADE


 sku_code
 VARCHAR(100)
 UNIQUE, NOT NULL


 price
 DECIMAL(12,2)
 NUMERIC(12,2), CHECK (price >= 0)


 stock_quantity
 INT
 CHECK (stock_quantity >= 0)

4. Administration Route Table with Examples
Method
Route
Purpose
Auth Required
 POST
 /api/v1/admin/categories
Create category
 Admin Bearer Token
 GET
 /api/v1/admin/categories
Fetch category tree
 Admin Bearer Token
 POST
 /api/v1/admin/products
Create draft product
 Admin Bearer Token
 PATCH
 /api/v1/admin/products/:id
Update product content/status
 Admin Bearer Token
 POST
 /api/v1/admin/products/:id/skus
Create validated SKU for product
 Admin Bearer Token
 PATCH
 /api/v1/admin/skus/:id
Update price/stock/status
 Admin Bearer Token
 GET
 /api/v1/admin/products
Return admin product listing
 Admin Bearer Token

Sample Endpoint Payload & Response
POST /api/v1/admin/products/101/skus
Request Payload:{

  "variant_id": 12,

  "sku_code": "TS-LPT-PRO-16GB",

  "price": 1299.99,

  "stock_quantity": 25,

  "is_active": true

}

Response (201 Created):{

  "success": true,

  "data": {

    "id": 401,

    "product_id": 101,

    "variant_id": 12,

    "sku_code": "TS-LPT-PRO-16GB",

    "price": 1299.99,

    "stock_quantity": 25,

    "is_active": true,

    "created_at": "2026-10-02T18:00:00Z"

  }

}

5. Data Integrity and Business Rule Decisions
Draft vs Published SKU Rules: A draft product can exist without SKUs during creation. However, a product cannot transition to published status without at least one active, sellable SKU.
Category Assignment: Each product belongs to exactly one canonical primary category (category_id FK).
Category Deactivation: Deactivating a parent category recursively disables all child categories and hides associated products from public catalog queries while keeping data intact.
Out-of-Stock SKUs: Represented with stock_quantity: 0. Public APIs mark items as "Out of Stock" while preventing order creation.
Price Overrides: Multiple SKUs can share the same base price, but each SKU retains its explicit price record in the database.
Negative Stock & Duplicate Prevention: Enforced via DB CHECK constraint (stock_quantity >= 0) and UNIQUE(sku_code) index.
Deactivating Ordered Products: Historical orders preserve immutable line-item snapshots (unit_price, sku_code). Deactivation affects future carts/checkout validation only.

6. Seed Data and Demonstration Instructions

6.1 Sample Seed Fixtures
Category Tree:
Electronics (Root Category)
Laptops & Computers (Child Category)
Smartphones & Accessories (Child Category)
Products:
ProBook Laptop 15 (Multi-variant: 16GB RAM / 32GB RAM)
Wireless Noise-Canceling Headphones
Ultra-Fast USB-C Charger
SKUs:
PBL15-16GB-512GB (Price: 1299.99, Stock: 15)
PBL15-32GB-1TB (Price: 1699.99, Stock: 8)
ANC-HD-BLK (Price: 199.99, Stock: 40)
USBC-CHG-65W (Price: 29.99, Stock: 0 - Out of stock demo)

6.2 Local Execution Command
npm run seed:catalog

7. Test Strategy, Command, and Results

7.1 Test Suites
Unit Tests: Category hierarchy validation, cycle detection (e.g., Cat A -> Cat B -> Cat A).
Integration Tests: Duplicate slug rejection (400 Bad Request), duplicate SKU rejection (409 Conflict).
Security Tests: Unauthorized 401/403 block on admin endpoints.

7.2 Execution Command & Results
npm run test:catalog

Test Results Output:PASS tests/catalog.test.js

  ✓ Should create parent and child categories (42ms)

  ✓ Should prevent category parent cycle dependency (18ms)

  ✓ Should reject duplicate product slug with HTTP 400 (15ms)

  ✓ Should reject negative stock updates with DB constraint error (22ms)

  ✓ Should reject unauthenticated requests to POST /api/v1/admin/products (12ms)

Test Suites: 1 passed, 1 total

Tests:       5 passed, 5 total

8. Known Limitations and Sprint 3 Backlog
Limitations: Binary asset uploading (S3) and dynamic spec indexing are currently stubbed.
Sprint 3 Backlog Items: Public catalog search, faceted filter indexing, publication state workflows, and Cart-to-SKU binding validation.

