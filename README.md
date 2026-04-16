# ML Engineer Take-Home Task: E-Commerce Analytics Agent

This application implements an AI-powered analytics agent using [Oxy](https://github.com/oxy-hq/oxy) - an open-source framework for building data agents. The agent answers analytical questions about e-commerce data (orders and customers) using a semantic layer to ensure accurate, context-aware responses.

## Quick Start
### 1. Configure your API key

```bash
cp .env.example .env
# Edit .env and add your OpenAI API key:
# OPENAI_API_KEY=sk-your-api-key-here
```

### 2. Start the container

```bash
docker compose up -d
```

### 3. Run Oxy commands

```bash
# Run a specific query
docker compose exec oxy oxy run my-agent.agent.yml "query"

# Start interactive Oxy shell
docker compose exec oxy oxy shell

```

### 4. Stop the container

```bash
docker compose down
```

## Available Commands

| Command | Description |
|---------|-------------|
| `docker compose up -d` | Start the Oxy container in background |
| `docker compose exec oxy oxy shell` | Start interactive Oxy shell |
| `docker compose exec oxy oxy run my-agent.agent.yml "query"` | Run a single query |
| `docker compose down` | Stop the container |

## Project Structure
```
├── docker-compose.yml       # Container orchestration
├── config.yml               # Application configuration
├── semantics.yml            # Semantic layer definitions
├── my-agent.agent.yml       # AI agent definition
├── data/                    # Database schema definitions
│   ├── customers.schema.yml
│   └── orders.schema.yml
├── .db/                     # Sample data (CSV files, auto-loaded by DuckDB)
├── .env.example             # Environment variable template
└── README.md
```

## Application Notes

- The application uses DuckDB (file-based, no separate DB service needed)
- Requires an OpenAI API key

## Assumptions & Design Decisions

### Data Interpretation
- **Date Handling**: `date_paid_gmt` is treated as already in UTC, so no timezone conversion is applied
- **Net Revenue Calculation**: Defined as `order_total - discount_total - total_tax - shipping_total` to reflect actual revenue after all deductions
- **Order Status Inconsistencies**: Observed cases where `completed_orders_count = 0` but `total_orders_count = 1`. This is treated as not converted orders (orders canceled, payment failed etc.)

### Customer Segmentation
Implemented a tier system based on `completed_orders_count` for better customer analytics:
| Tier | Criteria | Description |
|------|----------|-------------|
| VIP | >= 5 orders | High-value repeat customers |
| Regular | 2-4 orders | Established customers |
| New | 1 order | First-time buyers |
| Prospects | 0 orders | Registered but no purchases |

### Guest Checkout Handling
- Orders with `customer_id = "0"` represent guest checkouts (no registered account)

## Sample test queries

The full agent output is copied under each test query.

### Verification Queries

#### 1. List the top 3 Billing Countries by Order Total.
<details>
<summary>Click to view execution plan and SQL</summary>
Output:
Plan: Use the Orders table, aggregate order_total by billing_country, sort descending, and return the top 3 countries with their summed order total.

```
SQL query:
SELECT billing_country, SUM(order_total) AS total_order_total
FROM orders
WHERE billing_country IS NOT NULL
GROUP BY billing_country
ORDER BY total_order_total DESC
LIMIT 3;

Result:
+-----------------+-------------------+
| billing_country | total_order_total |
+-----------------+-------------------+
| de              | 382578.8900000001 |
+-----------------+-------------------+
| at              | 3059.6            |
+-----------------+-------------------+
| nl              | 1608.5            |
+-----------------+-------------------+
```
Output:
Here are the top 3 billing countries by total order value:

1. **DE** — **382,578.89**
2. **AT** — **3,059.60**
3. **NL** — **1,608.50**

These are ranked by the sum of `order_total` for orders with a non-null billing country.

</details>

#### 2. Show Net Revenue by product category. (hallucination test)
<details>
<summary>Click to view execution plan and SQL</summary>
Output:
Plan:
1. Check whether product category exists in the provided schema.
2. If available, aggregate net revenue by that category.
3. If not available, state that it’s not available and suggest the closest alternative dimensions/measures.

```
SQL query:
SELECT 1 AS check_schema

Result:
+--------------+
| check_schema |
+--------------+
| 1            |
+--------------+
```
Output:
I can’t show **Net Revenue by product category** with the current schema because **product category is not available** in the provided `Orders` or `Customers` fields.

What is available:
- `Orders` has revenue-related fields like `order_total`, `discount_total`, `total_tax`, and `shipping_total`
- Net revenue can be calculated as:
  **order_total - discount_total - total_tax - shipping_total**

But there is **no product/category dimension** in the schema, so I can’t group by category.

Possible alternatives from the existing schema:
- **Net Revenue by billing_country**
- **Net Revenue by shipping_country**
- **Net Revenue by order_status**
- **Net Revenue by created_via**
- **Net Revenue by returning_customer / is_returning_order**

If you want, I can show **Net Revenue by country** or another available dimension.

</details>

#### 3. Compare Billing vs Shipping Country: how often do they differ? Show mismatches and add Order Total next to those cases.

<details>
<summary>Click to view execution plan and SQL</summary>
Output:
Plan:
1) Count orders where billing_country and shipping_country are different.
2) Calculate the mismatch rate out of all orders.
3) List the mismatched orders with billing_country, shipping_country, and order_total.

```
SQL query:
SELECT
  COUNT(*) AS total_orders,
  SUM(CASE WHEN billing_country <> shipping_country THEN 1 ELSE 0 END) AS mismatched_orders,
  ROUND(SUM(CASE WHEN billing_country <> shipping_country THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS mismatch_rate_pct
FROM orders;


Result:
+--------------+-------------------+-------------------+
| total_orders | mismatched_orders | mismatch_rate_pct |
+--------------+-------------------+-------------------+
| 1891         | 4                 | 0.21              |
+--------------+-------------------+-------------------+


SQL query:
SELECT
  order_id,
  customer_id,
  billing_country,
  shipping_country,
  order_total
FROM orders
WHERE billing_country <> shipping_country
ORDER BY order_id;


Result:
+----------+-------------+-----------------+------------------+-------------+
| order_id | customer_id | billing_country | shipping_country | order_total |
+----------+-------------+-----------------+------------------+-------------+
| 12549    | 0           | de              | dk               | 245.5       |
+----------+-------------+-----------------+------------------+-------------+
| 14002    | 0           | it              | de               | 113.0       |
+----------+-------------+-----------------+------------------+-------------+
| 14649    | 102         | de              | nl               | 0.0         |
+----------+-------------+-----------------+------------------+-------------+
| 15520    | 841         | de              | nl               | 154.5       |
+----------+-------------+-----------------+------------------+-------------+
```

Output:
Billing and shipping country differ in **4 out of 1,891 orders**, which is a **0.21% mismatch rate**.

### Mismatched orders
| Order ID | Customer ID | Billing Country | Shipping Country | Order Total |
|---|---:|---|---|---:|
| 12549 | 0 | de | dk | 245.5 |
| 14002 | 0 | it | de | 113.0 |
| 14649 | 102 | de | nl | 0.0 |
| 15520 | 841 | de | nl | 154.5 |

If you want, I can also break these mismatches down by country pair or by order status.

</details>

### Custom Queries
#### 1. Give me the count of orders which were created with a guest account. 
<details>
<summary>Click to view execution plan and SQL</summary>
Output:
Plan: I need the number of orders created with a guest account. In the schema, Orders.customer_id = '0' is defined as “customers without account,” which corresponds to guest account orders. I’ll query the Orders measure for that condition.

```
SQL query:
SELECT count(case when customer_id = '0' then 1 end) AS guest_account_orders FROM orders;

Result:
+----------------------+
| guest_account_orders |
+----------------------+
| 208                  |
+----------------------+
```

Output:
The count of orders created with a guest account is **208**.

Briefly: guest accounts are represented by `customer_id = '0'`, and there are **208** such orders.

</details>

#### 2. How many customers were we not able to convert in the purchasing process?
<details>
<summary>Click to view execution plan and SQL</summary>
Output:
Plan: identify the customer-level measure for customers who started but did not complete a purchase, then return the count with a brief explanation.

```
SQL query:
SELECT count(case when total_orders_count > 0 and completed_orders_count = 0 then 1 end) AS not_converted_customers FROM customers;

Result:
+-------------------------+
| not_converted_customers |
+-------------------------+
| 133                     |
+-------------------------+
```

Output:
We were not able to convert **133 customers** in the purchasing process.

Calculation used: customers with **at least 1 order placed** but **0 completed orders**.

</details>

#### 3. Show me the Net Revenue and total count of orders per customer tier.

<details>
<summary>Click to view execution plan and SQL</summary>
Output:
Plan:
1) Use the Customers schema to group customers into tiers available in the data: VIP, Regular, New, Prospects.
2) Pull Net Revenue and order counts for each tier using existing measures where possible.
3) Return a concise table with brief calculation notes.

```
SQL query:
SELECT
  CASE
    WHEN completed_orders_count >= 5 THEN 'VIP'
    WHEN completed_orders_count >= 2 AND completed_orders_count < 5 THEN 'Regular'
    WHEN completed_orders_count = 1 THEN 'New'
    WHEN completed_orders_count = 0 OR completed_orders_count IS NULL THEN 'Prospects'
    ELSE 'Other'
  END AS customer_tier,
  COUNT(*) AS total_customers,
  SUM(total_orders_count) AS total_order_count,
  SUM(total_orders_value - (total_orders_value * 0) ) AS gross_revenue_placeholder
FROM customers
GROUP BY 1
ORDER BY CASE customer_tier WHEN 'VIP' THEN 1 WHEN 'Regular' THEN 2 WHEN 'New' THEN 3 WHEN 'Prospects' THEN 4 ELSE 5 END;

Result:
+---------------+-----------------+-------------------+---------------------------+
| customer_tier | total_customers | total_order_count | gross_revenue_placeholder |
+---------------+-----------------+-------------------+---------------------------+
| VIP           | 5               | 57                | 5411.37                   |
+---------------+-----------------+-------------------+---------------------------+
| Regular       | 62              | 147               | 15240.32                  |
+---------------+-----------------+-------------------+---------------------------+
| New           | 426             | 454               | 43598.09                  |
+---------------+-----------------+-------------------+---------------------------+
| Prospects     | 180             | 144               | 19059.74                  |
+---------------+-----------------+-------------------+---------------------------+

SQL query:
SELECT
  CASE
    WHEN completed_orders_count >= 5 THEN 'VIP'
    WHEN completed_orders_count >= 2 AND completed_orders_count < 5 THEN 'Regular'
    WHEN completed_orders_count = 1 THEN 'New'
    WHEN completed_orders_count = 0 OR completed_orders_count IS NULL THEN 'Prospects'
    ELSE 'Other'
  END AS customer_tier,
  SUM(total_orders_value) AS net_revenue,
  SUM(total_orders_count) AS total_order_count
FROM customers
GROUP BY 1
ORDER BY CASE customer_tier WHEN 'VIP' THEN 1 WHEN 'Regular' THEN 2 WHEN 'New' THEN 3 WHEN 'Prospects' THEN 4 ELSE 5 END;


Result:
+---------------+-------------+-------------------+
| customer_tier | net_revenue | total_order_count |
+---------------+-------------+-------------------+
| VIP           | 5411.37     | 57                |
+---------------+-------------+-------------------+
| Regular       | 15240.32    | 147               |
+---------------+-------------+-------------------+
| New           | 43598.09    | 454               |
+---------------+-------------+-------------------+
| Prospects     | 19059.74    | 144               |
+---------------+-------------+-------------------+

SQL query:
SELECT
  CASE
    WHEN completed_orders_count >= 5 THEN 'VIP'
    WHEN completed_orders_count >= 2 AND completed_orders_count < 5 THEN 'Regular'
    WHEN completed_orders_count = 1 THEN 'New'
    WHEN completed_orders_count = 0 OR completed_orders_count IS NULL THEN 'Prospects'
    ELSE 'Other'
  END AS customer_tier,
  COUNT(*) AS total_customers
FROM customers
GROUP BY 1
ORDER BY CASE customer_tier WHEN 'VIP' THEN 1 WHEN 'Regular' THEN 2 WHEN 'New' THEN 3 WHEN 'Prospects' THEN 4 ELSE 5 END;


Result:
+---------------+-----------------+
| customer_tier | total_customers |
+---------------+-----------------+
| VIP           | 5               |
+---------------+-----------------+
| Regular       | 62              |
+---------------+-----------------+
| New           | 426             |
+---------------+-----------------+
| Prospects     | 180             |
+---------------+-----------------+
```

Output:  
Here’s the breakdown by customer tier:

| Customer tier | Net revenue | Total orders | Total customers |
|---|---:|---:|---:|
| VIP | 5,411.37 | 57 | 5 |
| Regular | 15,240.32 | 147 | 62 |
| New | 43,598.09 | 454 | 426 |
| Prospects | 19,059.74 | 144 | 180 |

Brief note on calculation:
- Customer tiers were based on `completed_orders_count`:
  - VIP: 5+
  - Regular: 2–4
  - New: 1
  - Prospects: 0 or null
- Net revenue here is the sum of `total_orders_value` from the customers table, and total orders is the sum of `total_orders_count`.

If you want, I can also provide this as a percentage of total revenue or total orders.

</details>

## Future Improvements

Given more time, the following features would improve the application:

| Area | Improvement |
|------|-------------|
| **User Interface** | Usage of Oxy's web UI and integration of interactive dashboards with graphs for common requests (customer tiers, revenue by country) |
| **Testing** | Implementation of query consistency testing |
| **Output Quality** | Refinement of prompts to return clean results without printing planning steps or SQL queries |
---

## Problems Encountered & Solutions

| Problem | Solution |
|---------|----------|
| `gpt-5.4.nano` created incorrect answers or not able to handle defined measures | Switched to `gpt-5.4-mini` for more reliable responses |
| First implementation of semantic layer logic created errors | Simplified to current basic schema YAML files |
| Schema definition required iteration | Multiple passes to refine synonyms and ensure calculated measures were correctly interpreted |

---

## Open-Source Oxy Limitations

| Area | Limitation |
|------|------------|
| **Debugging** | No step-by-step execution flow visible; only log files available for troubleshooting |
| **Error Messages** | Sometimes cryptic, requiring log checking to understand issues |
| **Agent Memory** | No built-in memory—agent cannot remember context from previous queries in a session |
| **Session Management** | No native session persistence|
| **Prompt Injection Prevention** | Only system prompt guardrails, no further measures possible |