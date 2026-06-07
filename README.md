# 🎮 Analytical SQL Business Game — Summoning the Data Daemons

### Presentation Link: https://canva.link/uq0sizeh1cyesaq
### Video Link: https://drive.google.com/drive/u/0/folders/1G9vgZcCx4N2j0R9bW2Rcfsh4TtcWtZN-

> *Three Analytical Quests to Master Games World Egypt*

This repository is my submission for the **Analytical SQL Business Game** — a challenge by our Analytical SQL instructor to come up with creative, real-world use cases for advanced SQL analytical features. I chose **Games World Egypt**, a fictional video game retail store, as my business domain and framed the presentation as three "quests" — each one tackling a genuine business problem using a powerful SQL technique.

---

## 🗺️ The Business Context

**Games World Egypt** is a video game retail store that sells new and used consoles, accessories, digital cards, and collectibles. The store tracks customers, inventory, products, and sales — including trade-ins, returns, and resales of used items.

### Database Schema

| Table | Key Columns |
|-------|-------------|
| `customers` | `customer_id` (PK), `customer_name`, `email`, `phone`, `registration_date`, `loyalty_tier` |
| `products` | `product_id` (PK), `product_name`, `category`, `brand`, `list_price`, `cost_price`, `is_serialized` |
| `inventory` | `inventory_id` (PK), `product_id` (FK), `serial_number`, `condition`, `stock_quantity`, `reorder_level`, `current_location`, `last_updated` |
| `sales` | `sale_id` (PK), `customer_id` (FK), `product_id` (FK), `inventory_id` (FK), `sale_date`, `quantity`, `unit_price`, `sale_type`, `related_sale_id` (FK → sales) |

---

## ⚔️ The Three Quests

### Quest 1 — Ghosts in the Warehouse: Inventory Velocity

**The Problem:**
The store suffers from overstocked slow-movers sitting on shelves (*Phantoms*) while hot products sell out and opportunities are missed. Capital is tied up in dead stock and shelf space is wasted.

**The Magical Solution:**

Uses a multi-step CTE pipeline with a **PIVOT** and a **LAG** window function to classify every product by its inventory velocity status:

```sql
WITH
  -- 1. Summon the sales data for the last 3 months, pivoted by month
  SalesPivot AS (
    SELECT * FROM (
        SELECT p.product_id, p.product_name, p.category,
               TO_CHAR(s.sale_date, 'YYYY-MM') AS sale_month, s.quantity
        FROM products p
        JOIN sales s ON p.product_id = s.product_id
        WHERE s.sale_date >= ADD_MONTHS(SYSDATE, -3)
    )
    PIVOT (SUM(quantity) FOR sale_month IN (
        '2024-01' AS jan_sales,
        '2024-02' AS feb_sales,
        '2024-03' AS mar_sales
    ))
  ),
  -- 2. Conjure the current inventory levels
  CurrentInventory AS (
    SELECT product_id, stock_quantity, reorder_level FROM inventory
  )

-- 3. The final divination
SELECT ci.product_id, p.product_name, p.category, ci.stock_quantity,
       sp.jan_sales, sp.feb_sales, sp.mar_sales,
       LAG(sp.mar_sales, 1, 0) OVER (PARTITION BY p.category ORDER BY p.product_id) AS previous_month_sales,
       CASE
           WHEN sp.mar_sales > ci.stock_quantity * 0.5              THEN 'HOT'
           WHEN sp.mar_sales = 0 AND ci.stock_quantity > 10         THEN 'PHANTOM'
           WHEN sp.mar_sales < LAG(sp.mar_sales, 1, 0)
                               OVER (PARTITION BY p.category
                                     ORDER BY p.product_id) * 0.3  THEN 'COOLING'
           ELSE 'STABLE'
       END AS product_status
FROM CurrentInventory ci
JOIN products p ON ci.product_id = p.product_id
LEFT JOIN SalesPivot sp ON p.product_id = sp.product_id
ORDER BY product_status, ci.stock_quantity DESC;
```

**SQL Features:** `WITH` (CTEs) · `PIVOT` · `LAG` window function · `CASE WHEN`

**Sample Insight:**

| PRODUCT_NAME | CATEGORY | STOCK_QTY | MAR_SALES | STATUS |
|---|---|---|---|---|
| Nintendo Switch Lite (Purple) | Handheld Console | 3 | 18 | HOT |
| ASUS ROG Swift OLED PG27AQDM | Monitors | 5 | 7 | HOT |
| Logitech G102 Light Sync (White) | Accessories | 42 | 29 | COOLING |
| T-Dagger ARENA Keyboard (Brown) | Keyboards | 18 | 2 | COOLING |
| PlayStation Network Gift Card 10 USD | Digital Card | 120 | 110 | STABLE |
| Used – Red Dead Redemption 2 (PS4) | Used Games | 1 | 1 | PHANTOM |
| Funko Pop! The Mandalorian | Collectibles | 45 | 1 | PHANTOM |

---

### Quest 2 — The Hidden Journeys of Our Customers: Customer Pathways

**The Problem:**
The store knows customers buy, but doesn't understand *how* they become big spenders. What purchase sequences lead to loyalty? Which entry-level products act as a gateway to high-value console purchases?

**The Magical Solution:**

Classifies every purchase by tier and item class, then uses **MATCH_RECOGNIZE** to detect the exact sequence pattern where customers ascend from low-value items to high-value console purchases:

```sql
SELECT *
FROM (
    SELECT c.customer_id, c.customer_name, s.sale_date, s.product_id,
           p.product_name, p.category, p.list_price,
           CASE
               WHEN p.list_price < 1000              THEN 'LOW_VALUE'
               WHEN p.list_price BETWEEN 1000 AND 5000 THEN 'MID_VALUE'
               WHEN p.list_price > 5000              THEN 'HIGH_VALUE'
               ELSE 'OTHER'
           END AS purchase_tier,
           CASE
               WHEN p.category LIKE '%Used%' OR p.category = 'Used corner' THEN 'USED'
               WHEN p.category = 'Digital Card'                             THEN 'DIGITAL'
               WHEN p.category IN ('Nintendo Switch', 'PlayStation', 'XBOX') THEN 'CONSOLE'
               ELSE 'ACCESSORY'
           END AS item_class
    FROM sales s
    JOIN customers c ON s.customer_id = c.customer_id
    JOIN products p  ON s.product_id  = p.product_id
)
MATCH_RECOGNIZE (
    PARTITION BY customer_id ORDER BY sale_date
    MEASURES
        STARTING.product_name             AS "FIRST_PURCHASE",
        FINAL LAST(CONSOLE_PURCHASE.product_name) AS "ASCENDED_TO",
        COUNT(*)                          AS "STEPS_IN_CHAIN",
        MAX(sale_date)                    AS "ASCENSION_DATE"
    ONE ROW PER MATCH
    PATTERN (START_ITEM LOW_ITEM* CONSOLE_PURCHASE)
    DEFINE
        LOW_ITEM         AS (item_class IN ('USED', 'ACCESSORY') OR purchase_tier = 'LOW_VALUE'),
        CONSOLE_PURCHASE AS (item_class = 'CONSOLE' AND purchase_tier = 'HIGH_VALUE')
) MR
ORDER BY MR.customer_id;
```

**SQL Features:** `CASE WHEN` · `MATCH_RECOGNIZE` · `PARTITION BY` · `MEASURES` · `PATTERN` / `DEFINE`

**Sample Insight:**

| CUSTOMER_ID | FIRST_PURCHASE | ASCENDED_TO | STEPS_IN_CHAIN | ASCENSION_DATE |
|---|---|---|---|---|
| 1001 | PlayStation Network Gift Card 10 USD | Nintendo Switch Lite (Purple) | 2 | 2024-02-10 |
| 1002 | Used – Red Dead Redemption 2 (PS4) | Sony WH-CH520 Wireless Headphones (Blue) | 3 | 2024-03-01 |
| 1007 | GameSir T3 Lite Wired Controller | DXRacer Gaming Chair Formula Series | 2 | 2024-03-15 |
| 1023 | Apple iTunes Gift Card 10 GBP (UK) | META QUEST 3 512GB VR Headset | 3 | 2024-03-20 |
| 1045 | T-Dagger ARENA Keyboard (Red) | ASUS ROG Swift OLED PG27AQDM | 2 | 2024-02-28 |

---

### Quest 3 — The Many Lives of Used Items: Used Lifecycle Tracking

**The Problem:**
Products in the "Used Corner" have a hidden story — each item may have been sold new, returned, traded in, and resold multiple times. Without tracing the full lifecycle, the store undervalues the true profitability of serialized items.

**The Magical Solution:**

Uses a **Recursive CTE** to walk the chain of related sales for each serialized item, building a complete lineage path from its first transaction to its most recent:

```sql
WITH RECURSIVE item_lineage AS (
    -- Anchor: find used items currently in the used corner
    SELECT s.sale_id, s.serial_number, s.product_id, s.customer_id,
           s.sale_date, s.sale_type, s.related_sale_id,
           1 AS life_cycle_number,
           CAST(s.serial_number AS VARCHAR2(100)) AS lineage_path
    FROM sales s
    JOIN inventory i ON s.serial_number = i.serial_number
    WHERE i.current_location = 'Used corner'

    UNION ALL

    -- Recursive step: trace back to earlier transactions
    SELECT prev.sale_id, prev.serial_number, prev.product_id, prev.customer_id,
           prev.sale_date, prev.sale_type, prev.related_sale_id,
           il.life_cycle_number + 1,
           il.lineage_path || ' -> ' || prev.sale_id
    FROM sales prev
    JOIN item_lineage il ON prev.sale_id = il.related_sale_id
)
CYCLE sale_id SET is_loop TO 1 DEFAULT 0

SELECT product_id, serial_number, life_cycle_number, sale_date, sale_type, lineage_path
FROM item_lineage
ORDER BY serial_number, life_cycle_number DESC;
```

**SQL Features:** `WITH RECURSIVE` (Recursive CTE) · `UNION ALL` · `CYCLE` detection · lineage path string building

**Sample Insight:**

| SERIAL_NUMBER | PRODUCT_NAME | LIFE_CYCLE | SALE_DATE | SALE_TYPE | LINEAGE_PATH |
|---|---|---|---|---|---|
| SNY-WHCH520-BLUE-001 | Sony WH-CH520 Headphones | 3 | 2024-02-10 | USED | 30245 |
| SNY-WHCH520-BLUE-001 | Sony WH-CH520 Headphones | 2 | 2024-01-20 | TRADE_IN | 30245 → 30123 |
| SNY-WHCH520-BLUE-001 | Sony WH-CH520 Headphones | 1 | 2023-11-15 | NEW | 30245 → 30123 → 30001 |
| NIN-SWITCH-LITE-PUR-987 | Nintendo Switch Lite (Purple) | 3 | 2024-01-15 | USED | 31389 |
| NIN-SWITCH-LITE-PUR-987 | Nintendo Switch Lite (Purple) | 2 | 2023-12-10 | RETURN | 31389 → 31234 |
| NIN-SWITCH-LITE-PUR-987 | Nintendo Switch Lite (Purple) | 1 | 2023-09-05 | NEW | 31389 → 31234 → 31001 |

---

## 🧠 SQL Features Covered

| Feature | Used In |
|---------|---------|
| `WITH` (CTEs) | Quest 1 |
| `PIVOT` | Quest 1 |
| `LAG` (Window Function) | Quest 1 |
| `CASE WHEN` | Quest 1, Quest 2 |
| `MATCH_RECOGNIZE` | Quest 2 |
| `WITH RECURSIVE` (Recursive CTE) | Quest 3 |
| `CYCLE` Detection | Quest 3 |

---

## 📊 Business Impact Summary

| Quest | Business Value |
|-------|---------------|
| **Inventory Velocity** | Classify products as Hot, Cooling, Phantom, or Stable — align stock with real demand, reduce idle capital, and maximize sales |
| **Customer Journey Mapping** | Identify the most common purchase paths from low‑value to high‑value items, enabling targeted campaigns and smarter cross‑selling |
| **Used Item Lifecycle Tracking** | Trace every transaction of a single used item across its lifetime to calculate true profitability and guide trade‑in decisions |

---

> *"Let the Data Lead."*
