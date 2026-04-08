# StockFlow – B2B Inventory Management System
### Backend Intern Case Study | Bynry Inc. Pune
**Language:** Java | **Database:** MySQL (SQL)

---

## Solution to the Case Study: 

This is my solution for the **StockFlow backend intern Case Study** given by Bynry Inc. Pune

Introduction: StockFlow is a B2B inventory management platform where small businesses can track/history of  products across multiple warehouses and manage supplier bussiness.

This case study had 3 parts:
1. Code Review & Debugging
2. Database Design (SQL Schema)
3. API Implementation – Low Stock Alerts

> I had good  knowledge  about **Java and Database** so I rewrote all logic in Java (JDBC-based). 

---

## Tech Stack

| What | Which |
|------|-------|
| Language | Java |
| Database | MySQL |
| DB Connection | JDBC (Java Database Connectivity) |
| Concepts used | OOP, Exception Handling, Collections, Generics, Strings, JDBC, basic Multithreading idea |

---

## About This Project

StockFlow is a B2B inventory management platform for small businesses to track products across multiple warehouses and manage supplier relationships.

This case study has 3 parts — code debugging, database design, and API implementation. I used **Java + SQL only** since that is what I know. All logic is written using core Java with JDBC for database connectivity.

---

## Project Structure

```
StockFlow/
├── Part1_Debugging/
│   ├── ProductRequest.java        # Input DTO with validation
│   ├── ApiResponse.java           # Response wrapper
│   └── ProductController.java     # Fixed product creation with transaction
│
├── Part2_Database/
│   └── schema.sql                 # All 9 tables with indexes and constraints
│
├── Part3_LowStockAlerts/
│   ├── LowStockAlert.java         # Alert DTO with inner SupplierInfo class
│   ├── AlertResponse.java         # API response wrapper
│   ├── AlertService.java          # Business logic + JDBC query
│   └── AlertController.java       # REST endpoint
│
└── README.md
```

---

## Part 1 – Code Review & Debugging

### Problems Found in Original Code

| # | Problem | Impact |
|---|---------|--------|
| 1 | Two separate `commit()` calls | Product saves but inventory may not — data becomes inconsistent |
| 2 | No try-catch block | Any DB error crashes the whole request with no proper message |
| 3 | No input validation | Null name, empty SKU, negative price can all get saved silently |
| 4 | No SKU uniqueness check | Duplicate SKUs allowed — breaks inventory tracking |
| 5 | No warehouse existence check | Product gets linked to a warehouse that does not exist |
| 6 | `price` has no type check | String like `"abc"` can be passed — causes type error later |
| 7 | No proper HTTP status codes | Always returns 200 even when something goes wrong |

### How I Fixed It

The biggest issue was **two separate commits**. If the server crashes between them, product is saved but inventory record is never created. The fix is to wrap both inserts in a **single JDBC transaction**.

```java
try {
    conn.setAutoCommit(false);    // turn off auto-commit, start manual transaction

    // Step 1: validate input fields
    // Step 2: check SKU is not duplicate
    // Step 3: check warehouse exists
    // Step 4: insert product
    // Step 5: insert inventory using same productId

    conn.commit();                // save BOTH inserts together
    conn.setAutoCommit(true);

} catch (SQLException e) {
    conn.rollback();              // if anything fails, cancel everything
    return new ApiResponse(500, "Database error: " + e.getMessage());
}
```

Other fixes applied:
- Added null and empty checks for all required fields before any DB call
- Used `PreparedStatement` with `?` placeholders — prevents SQL injection
- Used `BigDecimal` for price — handles decimal values accurately
- Checked SKU uniqueness with `SELECT COUNT(*)` before insert
- Checked warehouse exists before linking product to it
- Returned proper HTTP codes — 201 for created, 400 for bad input, 404 for not found, 409 for duplicate, 500 for server error

---

## Part 2 – Database Design

### Schema Overview

```
companies
    ├── warehouses          (one company → many warehouses)
    └── products            (one company → many products)
            ├── inventory           (one product → qty in each warehouse)
            │       └── inventory_logs    (every stock change is recorded)
            ├── product_suppliers   (many-to-many: product ↔ supplier)
            ├── bundle_items        (bundle product → its component products)
            └── sales_activity      (sales records used for alert calculation)

suppliers
    └── product_suppliers
```

### Key Tables

```sql
-- Products table with threshold for low stock alerts
CREATE TABLE products (
    id                  INT AUTO_INCREMENT PRIMARY KEY,
    company_id          INT NOT NULL,
    name                VARCHAR(255) NOT NULL,
    sku                 VARCHAR(100) NOT NULL,
    price               DECIMAL(10, 2) NOT NULL DEFAULT 0.00,
    product_type        ENUM('standard', 'bundle') DEFAULT 'standard',
    low_stock_threshold INT DEFAULT 10,
    created_at          DATETIME DEFAULT NOW(),
    updated_at          DATETIME DEFAULT NOW() ON UPDATE NOW(),
    UNIQUE KEY unique_sku_per_company (sku, company_id),
    FOREIGN KEY (company_id) REFERENCES companies(id)
);

-- Inventory tracks quantity per product per warehouse
CREATE TABLE inventory (
    id           INT AUTO_INCREMENT PRIMARY KEY,
    product_id   INT NOT NULL,
    warehouse_id INT NOT NULL,
    quantity     INT NOT NULL DEFAULT 0,
    UNIQUE KEY unique_product_warehouse (product_id, warehouse_id),
    FOREIGN KEY (product_id)   REFERENCES products(id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id)
);

-- Every stock change is logged here
CREATE TABLE inventory_logs (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    inventory_id    INT NOT NULL,
    change_type     ENUM('add', 'remove', 'adjustment', 'sale') NOT NULL,
    quantity_before INT NOT NULL,
    quantity_change INT NOT NULL,
    quantity_after  INT NOT NULL,
    changed_at      DATETIME DEFAULT NOW(),
    FOREIGN KEY (inventory_id) REFERENCES inventory(id)
);

-- Sales activity used to calculate days_until_stockout
CREATE TABLE sales_activity (
    id            INT AUTO_INCREMENT PRIMARY KEY,
    product_id    INT NOT NULL,
    warehouse_id  INT NOT NULL,
    quantity_sold INT NOT NULL,
    sold_at       DATETIME DEFAULT NOW(),
    FOREIGN KEY (product_id)   REFERENCES products(id),
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(id)
);
CREATE INDEX idx_sales_date ON sales_activity(sold_at);
```

Full schema with all 9 tables is in `Part2_Database/schema.sql`

### Design Decisions Explained

- **`DECIMAL(10,2)` for price** — avoids floating point errors that happen with `double` or `float`
- **`UNIQUE KEY (sku, company_id)`** — SKU is unique per company, not globally (different companies can reuse same SKU)
- **`inventory_logs` separate table** — we never lose history of stock changes, useful for audit and debugging
- **`ON DELETE CASCADE`** — if company is deleted, warehouses auto-delete to avoid orphan records
- **Indexes on date columns and foreign keys** — without this, JOIN queries on large data become very slow
- **`is_primary` flag in `product_suppliers`** — tells which supplier to contact first when reordering

### Questions I Would Ask the Product Team

1. Is SKU unique per company or globally across the platform?
2. What counts as "recent" sales — last 7 days, 30 days?
3. Can a product have multiple suppliers? Which one should show in alerts?
4. Do we need soft delete (mark as deleted) or actual hard delete?
5. Should low stock threshold be per product or can it differ per warehouse?
6. Do we need user login and roles like Admin vs Staff?
7. How should bundle stock be calculated — from component stock?

---

## Part 3 – Low Stock Alerts API

### Endpoint

```
GET /api/companies/{company_id}/alerts/low-stock
```

### Assumptions Made

- "Recent sales" means last 30 days — would confirm with team
- `days_until_stockout` = `current_stock ÷ average daily sales (30 days)`
- Products with zero sales in last 30 days are excluded as per the given business rule
- If a product is in 3 warehouses, it gives 3 separate alert rows (one per warehouse)
- Show primary supplier; if no supplier is linked, show null in response

### SQL Query

```sql
SELECT
    p.id, p.name, p.sku,
    w.id AS warehouse_id, w.name AS warehouse_name,
    inv.quantity AS current_stock,
    p.low_stock_threshold AS threshold,
    s.id AS supplier_id, s.name AS supplier_name, s.contact_email,

    CASE
        WHEN COALESCE(SUM(sa.quantity_sold), 0) = 0 THEN NULL
        ELSE FLOOR(inv.quantity / (SUM(sa.quantity_sold) / 30.0))
    END AS days_until_stockout

FROM products p
JOIN inventory inv
    ON inv.product_id = p.id
JOIN warehouses w
    ON w.id = inv.warehouse_id AND w.company_id = ?
LEFT JOIN product_suppliers ps
    ON ps.product_id = p.id AND ps.is_primary = TRUE
LEFT JOIN suppliers s
    ON s.id = ps.supplier_id
JOIN sales_activity sa
    ON sa.product_id = p.id
    AND sa.warehouse_id = inv.warehouse_id
    AND sa.sold_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)

WHERE p.company_id = ?
  AND inv.quantity < p.low_stock_threshold

GROUP BY p.id, p.name, p.sku, w.id, w.name,
         inv.quantity, p.low_stock_threshold,
         s.id, s.name, s.contact_email

ORDER BY inv.quantity ASC;
```

Why `LEFT JOIN` for suppliers — so products without a supplier still appear in alerts.
Why `INNER JOIN` for sales\_activity — so products with no recent sales are automatically excluded.
`CASE WHEN` on days\_until\_stockout — handles division by zero, returns null instead of crash.

### Java Service Layer (AlertService.java)

```java
public AlertResponse getLowStockAlerts(int companyId) {

    if (companyId <= 0) {
        throw new IllegalArgumentException("Invalid company ID");
    }

    List<LowStockAlert> alerts = new ArrayList<>();

    try {
        PreparedStatement ps = conn.prepareStatement(sql);
        ps.setInt(1, companyId);
        ps.setInt(2, companyId);
        ResultSet rs = ps.executeQuery();

        while (rs.next()) {

            // supplier is null if not linked (LEFT JOIN)
            LowStockAlert.SupplierInfo supplier = null;
            if (rs.getObject("supplier_id") != null) {
                supplier = new LowStockAlert.SupplierInfo(
                    rs.getInt("supplier_id"),
                    rs.getString("supplier_name"),
                    rs.getString("contact_email")
                );
            }

            // days can be null if no sales data
            Integer days = null;
            if (rs.getObject("days_until_stockout") != null) {
                days = rs.getInt("days_until_stockout");
            }

            alerts.add(new LowStockAlert(
                rs.getInt("id"), rs.getString("name"), rs.getString("sku"),
                rs.getInt("warehouse_id"), rs.getString("warehouse_name"),
                rs.getInt("current_stock"), rs.getInt("threshold"),
                days, supplier
            ));
        }

    } catch (SQLException e) {
        System.err.println("DB Error: " + e.getMessage());
        throw new RuntimeException("Failed to fetch alerts", e);
    }

    return new AlertResponse(alerts, alerts.size());
}
```

### Edge Cases Handled

| Edge Case | How Handled |
|-----------|-------------|
| Invalid company ID | `IllegalArgumentException` thrown before any DB call |
| Company has no warehouses | Query returns empty list, `totalAlerts = 0` |
| Product has no supplier linked | `LEFT JOIN` — supplier is `null` in response, no crash |
| Product has no sales in 30 days | `INNER JOIN` on `sales_activity` — auto excluded from results |
| Division by zero in days calc | `CASE WHEN` — returns `null` safely |
| SQL exception | Caught, logged, `RuntimeException` thrown, returns 500 |
| Product in multiple warehouses | Each warehouse appears as separate alert row |

---

## JDBC – How the DB Connection Works

JDBC (Java Database Connectivity) is a Java API to connect and run SQL queries from Java code.

```java
// 1. Open connection
Connection conn = DriverManager.getConnection(
    "jdbc:mysql://localhost:3306/stockflow", "username", "password");

// 2. Use PreparedStatement — safe from SQL injection
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM products WHERE company_id = ?");
ps.setInt(1, companyId);   // ? is replaced safely

// 3. Read the result row by row
ResultSet rs = ps.executeQuery();
while (rs.next()) {
    System.out.println(rs.getInt("id") + " - " + rs.getString("name"));
}

// 4. Always close resources
rs.close();
ps.close();
conn.close();
```

**Transaction in JDBC:**

```java
conn.setAutoCommit(false);   // start transaction manually

// run insert 1
// run insert 2

conn.commit();               // save all together
// OR
conn.rollback();             // cancel all on error
```

---

## Java Concepts Used

| Java Topic | Where Used in This Project |
|------------|---------------------------|
| OOP – Classes | Separate class for each responsibility: controller, service, DTO |
| OOP – Encapsulation | All fields are `private` with public getters/setters |
| OOP – Inner Class | `SupplierInfo` is a static inner class inside `LowStockAlert` |
| Strings | `getName().isEmpty()`, null checks, SQL string concatenation |
| Exception Handling | `try-catch` in every DB method, `rollback()` on failure |
| Collections | `List<LowStockAlert>` declared as interface, `new ArrayList<>()` as implementation |
| Generics | `List<LowStockAlert>`, `ResponseEntity<?>` for type safety |
| JDBC | `Connection`, `PreparedStatement`, `ResultSet`, manual transactions |
| Memory Management | `PreparedStatement` reused, resources closed after use |

---

## How to Run

```bash
# 1. Clone the repo
git clone https://github.com/yourusername/stockflow-casestudy.git
cd stockflow-casestudy

# 2. Create DB and run schema
mysql -u root -p < Part2_Database/schema.sql

# 3. Download MySQL JDBC driver
# https://dev.mysql.com/downloads/connector/j/

# 4. Compile and run
javac -cp .:mysql-connector-j-8.x.x.jar src/*.java
java  -cp .:mysql-connector-j-8.x.x.jar AlertController
```

---

## What I Would Improve With More Time

- Use `try-with-resources` so JDBC connections close automatically
- Add JUnit 5 unit tests for service layer
- Use Spring Boot properly with `@Repository` and HikariCP connection pool
- Add pagination so large alert lists don't slow down the response
- Add proper logging with SLF4J instead of `System.err.println`

---

*Submitted for: Backend Intern – Bynry Inc., Pune &nbsp;|&nbsp; Language: Java + SQL*
