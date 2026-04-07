# Bynry Backend Engineering Intern – Case Study: StockFlow

**Candidate:** Siddhesh Pawar
**Date:** April 8, 2026

---

## Part 1: Code Review & Debugging

### Original Code

```python
def add_product(name, sku, price, warehouse_id):
    if not name or not sku:
        return "Error: Missing info"
    
    # Check if product exists
    product = db.query("SELECT * FROM products WHERE sku = ?", (sku,))
    if product:
        db.execute("UPDATE products SET price = ? WHERE sku = ?", (price, sku))
    else:
        db.execute("INSERT INTO products (name, sku, price) VALUES (?, ?, ?)", (name, sku, price))
    
    # Add to warehouse
    db.execute("INSERT INTO inventory (sku, warehouse_id, quantity) VALUES (?, ?, ?)", (sku, warehouse_id, 0))
    return "Success"
```

---

### Issues Identified

#### Issue 1 – Missing Input Validation for `price` and `warehouse_id`
- **Problem:** `price` and `warehouse_id` are never validated. A negative price or null `warehouse_id` can be inserted without error.
- **Impact in Production:** Products with invalid prices or missing warehouse assignments can corrupt inventory data and cause downstream errors in pricing, invoicing, and stock tracking.

#### Issue 2 – Duplicate Inventory Entry on Every Call
- **Problem:** The `INSERT INTO inventory` statement runs unconditionally every time the function is called —even when the product already exists in that warehouse. There's no check for an existing `(sku, warehouse_id)` pair.
- **Impact in Production:** Calling this function multiple times for the same product + warehouse creates duplicate rows in the `inventory` table, leading to inflated stock counts, reporting errors, and data inconsistency.

#### Issue 3 – `warehouse_id` Not Stored in `products` Table, But Also Not Validated Against a Warehouses Table
- **Problem:** The product is inserted without verifying that the given `warehouse_id` actually exists in the database. Foreign key integrity is not enforced at the application level.
- **Impact in Production:** Orphan inventory records pointing to non-existent warehouses cause broken joins and incorrect stock reports.

#### Issue 4 – No Transaction Management
- **Problem:** The `UPDATE/INSERT` for products and the `INSERT` for inventory are two separate database calls with no wrapping transaction. If the second statement fails, the first is already committed.
- **Impact in Production:** Partial writes leave the database in an inconsistent state — a product is updated/inserted but not linked to any warehouse, or vice versa.

#### Issue 5 – Silent Failure on DB Errors
- **Problem:** There's no try/except block. Any database exception will propagate as an unhandled crash, returning no meaningful error to the caller.
- **Impact in Production:** API callers receive no structured error response; the system fails silently or with an uncaught exception, making it impossible to debug without logs.

#### Issue 6 – Using `SELECT *` is Wasteful
- **Problem:** `SELECT *` fetches all product columns just to check existence.
- **Impact in Production:** Unnecessary data transfer and memory use, especially on large product tables.

#### Issue 7 – Missing `warehouse_id` column in the `products` INSERT
- **Problem:** If `products` has a `warehouse_id` column or FK relationship, it's not being populated.
- **Impact in Production:** Incomplete data records and potential NOT NULL constraint violations.

---

### Corrected Version

```python
def add_product(name: str, sku: str, price: float, warehouse_id: int) -> dict:
    """
    Adds a new product or updates an existing one, then links it to a warehouse.

    Returns a dict: {"success": True/False, "message": str}
    """

    # --- Input Validation ---
    if not name or not isinstance(name, str):
        return {"success": False, "message": "Error: 'name' is required and must be a string."}
    if not sku or not isinstance(sku, str):
        return {"success": False, "message": "Error: 'sku' is required and must be a string."}
    if price is None or not isinstance(price, (int, float)) or price < 0:
        return {"success": False, "message": "Error: 'price' must be a non-negative number."}
    if not warehouse_id or not isinstance(warehouse_id, int):
        return {"success": False, "message": "Error: 'warehouse_id' is required and must be an integer."}

    try:
        with db.transaction():  # Wrap all operations in a single atomic transaction

            # --- Validate Warehouse Exists ---
            warehouse = db.query_one(
                "SELECT id FROM warehouses WHERE id = ?", (warehouse_id,)
            )
            if not warehouse:
                return {"success": False, "message": f"Error: Warehouse with id={warehouse_id} does not exist."}

            # --- Upsert Product (Check existence efficiently) ---
            existing_product = db.query_one(
                "SELECT id FROM products WHERE sku = ?", (sku,)
            )
            if existing_product:
                db.execute(
                    "UPDATE products SET name = ?, price = ?, updated_at = NOW() WHERE sku = ?",
                    (name, price, sku)
                )
            else:
                db.execute(
                    "INSERT INTO products (name, sku, price, created_at, updated_at) VALUES (?, ?, ?, NOW(), NOW())",
                    (name, sku, price)
                )

            # --- Upsert Inventory (Avoid duplicate entries) ---
            existing_inventory = db.query_one(
                "SELECT id FROM inventory WHERE sku = ? AND warehouse_id = ?",
                (sku, warehouse_id)
            )
            if not existing_inventory:
                db.execute(
                    "INSERT INTO inventory (sku, warehouse_id, quantity, created_at) VALUES (?, ?, 0, NOW())",
                    (sku, warehouse_id)
                )
            # If already exists, we do NOT reset quantity to 0 — preserve existing stock count

    except Exception as e:
        # Log the error for observability
        logger.error(f"add_product failed for sku={sku}, warehouse_id={warehouse_id}: {str(e)}")
        return {"success": False, "message": "Internal error: could not add product. Please try again."}

    return {"success": True, "message": f"Product '{sku}' successfully added/updated in warehouse {warehouse_id}."}
```

---

### Summary of Fixes

| # | Issue | Fix Applied |
|---|-------|-------------|
| 1 | Missing price/warehouse_id validation | Added type + value checks for all inputs |
| 2 | Duplicate inventory entry on re-run | Check existence before inserting into inventory |
| 3 | No warehouse FK validation | Query `warehouses` table before insert |
| 4 | No transaction | Wrapped in `db.transaction()` context manager |
| 5 | No error handling | Added `try/except` with structured error response |
| 6 | `SELECT *` inefficiency | Changed to `SELECT id` — fetch minimum needed |
| 7 | Quantity reset risk | Skip inventory insert if already present; preserve stock |

---

## Part 2: Database Design

### Schema

```sql
-- Companies (multi-tenant top-level entity)
CREATE TABLE companies (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Warehouses (each company can have many)
CREATE TABLE warehouses (
    id          SERIAL PRIMARY KEY,
    company_id  INT NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        VARCHAR(255) NOT NULL,
    location    TEXT,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Suppliers
CREATE TABLE suppliers (
    id          SERIAL PRIMARY KEY,
    company_id  INT NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        VARCHAR(255) NOT NULL,
    email       VARCHAR(255),
    phone       VARCHAR(50),
    address     TEXT,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Products (base catalog)
CREATE TABLE products (
    id              SERIAL PRIMARY KEY,
    company_id      INT NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    sku             VARCHAR(100) NOT NULL,
    product_type    VARCHAR(50) NOT NULL DEFAULT 'simple',  -- 'simple' | 'bundle'
    price           NUMERIC(12, 2) NOT NULL CHECK (price >= 0),
    low_stock_threshold INT NOT NULL DEFAULT 10,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (company_id, sku)  -- SKU unique per company
);

-- Product <-> Supplier mapping (a product can have multiple suppliers)
CREATE TABLE product_suppliers (
    id          SERIAL PRIMARY KEY,
    product_id  INT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    supplier_id INT NOT NULL REFERENCES suppliers(id) ON DELETE CASCADE,
    unit_cost   NUMERIC(12, 2),
    lead_time_days INT,
    is_preferred BOOLEAN DEFAULT FALSE,
    UNIQUE (product_id, supplier_id)
);

-- Bundle composition (which products make up a bundle)
CREATE TABLE bundle_items (
    id              SERIAL PRIMARY KEY,
    bundle_id       INT NOT NULL REFERENCES products(id) ON DELETE CASCADE,  -- must be product_type='bundle'
    component_id    INT NOT NULL REFERENCES products(id) ON DELETE RESTRICT,  -- component product
    quantity        INT NOT NULL CHECK (quantity > 0),
    UNIQUE (bundle_id, component_id),
    CHECK (bundle_id <> component_id)  -- prevent self-referential bundle
);

-- Inventory: stock levels per product per warehouse
CREATE TABLE inventory (
    id              SERIAL PRIMARY KEY,
    product_id      INT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    warehouse_id    INT NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    quantity        INT NOT NULL DEFAULT 0 CHECK (quantity >= 0),
    last_updated    TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (product_id, warehouse_id)
);

-- Inventory Audit Log (tracks every change)
CREATE TABLE inventory_audit (
    id              SERIAL PRIMARY KEY,
    product_id      INT NOT NULL REFERENCES products(id),
    warehouse_id    INT NOT NULL REFERENCES warehouses(id),
    changed_by      INT REFERENCES users(id),       -- who made the change
    change_type     VARCHAR(50) NOT NULL,            -- 'restock', 'sale', 'transfer', 'adjustment'
    quantity_before INT NOT NULL,
    quantity_after  INT NOT NULL,
    delta           INT NOT NULL,                   -- quantity_after - quantity_before
    note            TEXT,
    changed_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Users (for audit trail)
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    company_id  INT NOT NULL REFERENCES companies(id) ON DELETE CASCADE,
    name        VARCHAR(255) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    role        VARCHAR(50) NOT NULL DEFAULT 'staff',  -- 'admin', 'manager', 'staff'
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Sales activity (to determine "recent sales" for low-stock alerts)
CREATE TABLE sales_transactions (
    id              SERIAL PRIMARY KEY,
    product_id      INT NOT NULL REFERENCES products(id),
    warehouse_id    INT NOT NULL REFERENCES warehouses(id),
    quantity_sold   INT NOT NULL CHECK (quantity_sold > 0),
    sold_at         TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

### Indexes

```sql
-- Fast lookups and common query patterns
CREATE INDEX idx_products_company ON products(company_id);
CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_warehouse ON inventory(warehouse_id);
CREATE INDEX idx_inventory_audit_product ON inventory_audit(product_id);
CREATE INDEX idx_inventory_audit_changed_at ON inventory_audit(changed_at);
CREATE INDEX idx_sales_product_time ON sales_transactions(product_id, sold_at);
CREATE INDEX idx_warehouses_company ON warehouses(company_id);
```

---

### Entity-Relationship Summary

```
companies  ──< warehouses
companies  ──< products  ──< inventory >──  warehouses
companies  ──< suppliers
products   ──< product_suppliers >──  suppliers
products   ──< bundle_items >──  products  (self-referential for bundles)
inventory  ──< inventory_audit
products   ──< sales_transactions
```

---

### Design Decisions & Justifications

| Decision | Reason |
|----------|--------|
| `UNIQUE(company_id, sku)` on products | Allows different companies to reuse SKUs while preventing duplicates within a company |
| `CHECK (quantity >= 0)` on inventory | Prevents negative stock (business rule enforcement at DB level) |
| Separate `inventory_audit` table | Provides full audit history without bloating the main inventory table; supports compliance and debugging |
| `low_stock_threshold` per product | Each product type has a different threshold (raw materials vs. finished goods) |
| `product_type` field + `bundle_items` table | Cleanly handles product bundles without over-complicating the base product schema |
| `is_preferred` on `product_suppliers` | Enables the API to recommend the preferred supplier on low-stock alerts |
| `sales_transactions` table | Needed to determine "recent sales activity" without relying on an external system |

---

### Clarifying Questions for the Product Team

1. **Bundle stock calculation:** When querying inventory for a bundle, should we calculate available quantity from component stock, or do bundles have their own inventory rows?
2. **Threshold type:** Is `low_stock_threshold` a global minimum, or should it be per-warehouse-per-product (e.g., each warehouse has its own threshold)?
3. **"Recent sales activity" definition:** What time window defines "recent"? Last 7 days? 30 days? Should this be configurable per company?
4. **Multi-currency support:** Is `price` always in a single currency, or do we need `currency_code`?
5. **Soft deletes:** Should products/warehouses/suppliers be soft-deleted (add `deleted_at` column) rather than hard-deleted to preserve audit history?
6. **Supplier lead times:** Are lead times fixed or do they vary by order quantity?
7. **User roles & permissions:** Do different roles (admin, manager, staff) have different access to certain warehouse data?

---

## Part 3: API Implementation – Low Stock Alerts

### Endpoint
```
GET /alerts/low-stock?company_id={id}
```

---

### Business Rules Implemented
- Low-stock threshold is **per-product** (from `products.low_stock_threshold`)
- Only alerts for products with **sales in the last 30 days** (configurable)
- Handles **multiple warehouses** per company
- Includes **preferred supplier** information for reordering

---

### Python Implementation (FastAPI + SQLAlchemy / raw SQL)

```python
from fastapi import APIRouter, Query, HTTPException, Depends
from typing import Optional
from datetime import datetime, timedelta
from pydantic import BaseModel
import logging

router = APIRouter()
logger = logging.getLogger(__name__)

RECENT_SALES_DAYS = 30  # Configurable: window for "recent sales activity"


# --- Response Models ---

class WarehouseStock(BaseModel):
    warehouse_id: int
    warehouse_name: str
    location: Optional[str]
    quantity: int

class SupplierInfo(BaseModel):
    supplier_id: int
    supplier_name: str
    email: Optional[str]
    phone: Optional[str]
    unit_cost: Optional[float]
    lead_time_days: Optional[int]
    is_preferred: bool

class LowStockAlert(BaseModel):
    product_id: int
    product_name: str
    sku: str
    product_type: str
    total_quantity: int
    low_stock_threshold: int
    warehouse_breakdown: list[WarehouseStock]
    suppliers: list[SupplierInfo]

class LowStockResponse(BaseModel):
    company_id: int
    generated_at: datetime
    total_alerts: int
    alerts: list[LowStockAlert]


# --- Endpoint ---

@router.get("/alerts/low-stock", response_model=LowStockResponse)
def get_low_stock_alerts(
    company_id: int = Query(..., description="ID of the company to check"),
    db=Depends(get_db)
):
    """
    Returns a list of products that are below their low-stock threshold
    and have had recent sales activity, with warehouse-level breakdowns
    and supplier info for easy reordering.
    """

    # --- Validate Company Exists ---
    company = db.query_one("SELECT id FROM companies WHERE id = ?", (company_id,))
    if not company:
        raise HTTPException(status_code=404, detail=f"Company with id={company_id} not found.")

    recent_cutoff = datetime.utcnow() - timedelta(days=RECENT_SALES_DAYS)

    # --- Main Query ---
    # Step 1: Find products with recent sales activity, per company
    # Step 2: Sum inventory across warehouses
    # Step 3: Filter where total < threshold
    # Step 4: Join warehouse breakdown + supplier info

    low_stock_query = """
        WITH recent_active_products AS (
            -- Products that had at least one sale in the last N days
            SELECT DISTINCT st.product_id
            FROM sales_transactions st
            JOIN products p ON p.id = st.product_id
            WHERE p.company_id = ?
              AND st.sold_at >= ?
        ),
        product_totals AS (
            -- Total inventory per product across all warehouses
            SELECT
                p.id          AS product_id,
                p.name        AS product_name,
                p.sku,
                p.product_type,
                p.low_stock_threshold,
                SUM(i.quantity) AS total_quantity
            FROM products p
            JOIN inventory i ON i.product_id = p.id
            JOIN warehouses w ON w.id = i.warehouse_id AND w.company_id = p.company_id
            JOIN recent_active_products rap ON rap.product_id = p.id
            WHERE p.company_id = ?
            GROUP BY p.id, p.name, p.sku, p.product_type, p.low_stock_threshold
            HAVING SUM(i.quantity) < p.low_stock_threshold
        )
        SELECT * FROM product_totals
        ORDER BY total_quantity ASC;
    """

    low_stock_rows = db.query(low_stock_query, (company_id, recent_cutoff, company_id))

    if not low_stock_rows:
        return LowStockResponse(
            company_id=company_id,
            generated_at=datetime.utcnow(),
            total_alerts=0,
            alerts=[]
        )

    product_ids = [row["product_id"] for row in low_stock_rows]

    # --- Warehouse breakdown per product ---
    warehouse_query = """
        SELECT
            i.product_id,
            w.id   AS warehouse_id,
            w.name AS warehouse_name,
            w.location,
            i.quantity
        FROM inventory i
        JOIN warehouses w ON w.id = i.warehouse_id
        WHERE i.product_id = ANY(?)
          AND w.company_id = ?
        ORDER BY i.product_id, i.quantity ASC;
    """
    warehouse_rows = db.query(warehouse_query, (product_ids, company_id))

    # Index by product_id for O(1) lookup
    warehouse_map: dict[int, list[WarehouseStock]] = {}
    for row in warehouse_rows:
        pid = row["product_id"]
        warehouse_map.setdefault(pid, []).append(WarehouseStock(
            warehouse_id=row["warehouse_id"],
            warehouse_name=row["warehouse_name"],
            location=row["location"],
            quantity=row["quantity"]
        ))

    # --- Supplier info per product ---
    supplier_query = """
        SELECT
            ps.product_id,
            s.id         AS supplier_id,
            s.name       AS supplier_name,
            s.email,
            s.phone,
            ps.unit_cost,
            ps.lead_time_days,
            ps.is_preferred
        FROM product_suppliers ps
        JOIN suppliers s ON s.id = ps.supplier_id
        WHERE ps.product_id = ANY(?)
        ORDER BY ps.product_id, ps.is_preferred DESC;
    """
    supplier_rows = db.query(supplier_query, (product_ids,))

    supplier_map: dict[int, list[SupplierInfo]] = {}
    for row in supplier_rows:
        pid = row["product_id"]
        supplier_map.setdefault(pid, []).append(SupplierInfo(
            supplier_id=row["supplier_id"],
            supplier_name=row["supplier_name"],
            email=row["email"],
            phone=row["phone"],
            unit_cost=row["unit_cost"],
            lead_time_days=row["lead_time_days"],
            is_preferred=row["is_preferred"]
        ))

    # --- Assemble Response ---
    alerts = []
    for row in low_stock_rows:
        pid = row["product_id"]
        alerts.append(LowStockAlert(
            product_id=pid,
            product_name=row["product_name"],
            sku=row["sku"],
            product_type=row["product_type"],
            total_quantity=row["total_quantity"],
            low_stock_threshold=row["low_stock_threshold"],
            warehouse_breakdown=warehouse_map.get(pid, []),
            suppliers=supplier_map.get(pid, [])
        ))

    return LowStockResponse(
        company_id=company_id,
        generated_at=datetime.utcnow(),
        total_alerts=len(alerts),
        alerts=alerts
    )
```

---

### Sample JSON Response

```json
{
  "company_id": 42,
  "generated_at": "2026-04-08T10:30:00Z",
  "total_alerts": 2,
  "alerts": [
    {
      "product_id": 101,
      "product_name": "Industrial Valve A",
      "sku": "IND-VALVE-A",
      "product_type": "simple",
      "total_quantity": 3,
      "low_stock_threshold": 15,
      "warehouse_breakdown": [
        {
          "warehouse_id": 1,
          "warehouse_name": "Mumbai Central",
          "location": "Mumbai, MH",
          "quantity": 2
        },
        {
          "warehouse_id": 2,
          "warehouse_name": "Pune Depot",
          "location": "Pune, MH",
          "quantity": 1
        }
      ],
      "suppliers": [
        {
          "supplier_id": 5,
          "supplier_name": "AlphaTech Supplies",
          "email": "orders@alphatech.com",
          "phone": "+91-9876543210",
          "unit_cost": 450.00,
          "lead_time_days": 7,
          "is_preferred": true
        },
        {
          "supplier_id": 9,
          "supplier_name": "BetaGlobal",
          "email": "supply@betaglobal.in",
          "phone": null,
          "unit_cost": 480.00,
          "lead_time_days": 14,
          "is_preferred": false
        }
      ]
    },
    {
      "product_id": 204,
      "product_name": "Safety Gloves Bundle",
      "sku": "SFTY-GLOVE-BDL",
      "product_type": "bundle",
      "total_quantity": 0,
      "low_stock_threshold": 20,
      "warehouse_breakdown": [
        {
          "warehouse_id": 1,
          "warehouse_name": "Mumbai Central",
          "location": "Mumbai, MH",
          "quantity": 0
        }
      ],
      "suppliers": []
    }
  ]
}
```

---

### Edge Cases Handled

| Edge Case | Handling |
|-----------|----------|
| Company not found | Returns HTTP 404 with clear message |
| No low-stock products | Returns empty `alerts` array with `total_alerts: 0` |
| Product has no supplier | `suppliers` array is empty (not an error) |
| Product in multiple warehouses | Each warehouse shown in `warehouse_breakdown`; total is summed across all |
| Bundle products | Included in the same alert structure; future enhancement can recursively check component stock |
| Recent sales window | Configurable via `RECENT_SALES_DAYS` constant |

---

### Performance Notes
- **CTEs** (`WITH` clauses) reduce subquery repetition and improve readability
- **Batch fetch** of warehouse breakdown and suppliers using `ANY(?)` avoids N+1 query problem
- **`HAVING` clause** filters low-stock products at the DB level, not in Python
- **`GROUP BY` + `SUM`** aggregates total quantity per product efficiently with the index on `inventory(product_id)`

---

## Summary

| Part | Key Deliverables |
|------|-----------------|
| Code Review | 7 issues identified with production impact + fully corrected function |
| Database Design | 8-table normalized schema with audit log, bundle support, supplier mapping, and FK constraints |
| API | Fully implemented FastAPI endpoint with Pydantic models, 3-query batch strategy, edge case handling, and sample JSON |

