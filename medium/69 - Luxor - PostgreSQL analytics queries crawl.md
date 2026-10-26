## Problem

Our analytics API serves customer sales counts from a PostgreSQL database. Requests to the customer lookup endpoint take several seconds and often time out, even though the database server looks healthy — no obvious CPU, memory, or disk exhaustion.  
  
The API runs as systemd service _sad-analytics-api_ on port 9090. Application code lives under /opt/sad/. Database credentials for debugging: connect as _saduser_ to database _analytics_db_ (password: _sadpassword_). The main table is _sales_data_.  
  
Find why customer lookups are slow and restore acceptable API response times.

## Solution

Test the customer sales count API and measure its response time:

```bash
curl -s -w "\nTotal time: %{time_total}s\n" \
  http://127.0.0.1:9090/api/customers/1234/sales/count
```

The API returns the expected result, but the total response time is approximately `12.79 seconds`, exceeding the challenge requirement of under `500 ms`.

Inspect the API service and application code:

```bash
sudo systemctl status sad-analytics-api --no-pager
cat /opt/sad/analytics_api.py
```

The service is running normally, and the API executes the following SQL query to count a customer's sales:

```
SELECT COUNT(*)
FROM sales_data
WHERE customer_id = 1234;
```

Investigate the PostgreSQL query performance. Connect to the database using the provided credentials:

```bash
PGPASSWORD=sadpassword psql -U saduser -d analytics_db
```

Inspect the table structure and existing indexes:

```
\d sales_data
```

The `sales_data` table contains the columns `id`, `product_id`, `customer_id`, `sale_date`, `amount`, and `region`. Its only index is the primary key on `id`. There is no index on `customer_id`.

Check the query execution plan:

```
EXPLAIN ANALYZE
SELECT COUNT(*)
FROM sales_data
WHERE customer_id = 1234;
```

PostgreSQL reports a `Parallel Seq Scan on sales_data`, removing approximately 1.67 million rows per worker while searching for the requested customer. The query takes approximately `1497 ms` to execute.
The missing index on **`customer_id`** forces PostgreSQL to scan the entire sales table to find the matching records. Although PostgreSQL uses parallel workers, the full-table scan is still expensive.

The database user can query the table but does not own it. Use the PostgreSQL administrative account to create the index on `customer_id` as the `postgres` user:

```bash
sudo -u postgres psql -d analytics_db -c \
"CREATE INDEX idx_sales_data_customer_id ON public.sales_data (customer_id);"
```

This creates a B-tree index on `customer_id`, adding a second index without changing the sales records. The index organizes customer IDs for efficient searching, allowing PostgreSQL to find matching rows without scanning the entire table.

Test the API endpoint:

```bash
curl -s -w "\nTotal time: %{time_total}s\n" \
  http://127.0.0.1:9090/api/customers/1234/sales/count
```

The total response time drops to approximately `0.158 seconds` (158 ms), satisfying the challenge requirement of under 500 ms.
