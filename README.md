# Northwind-SQL-Analysis.sql
Advanced SQL business intelligence analysis on the classic Northwind database using PostgreSQL and DBeaver. Showcases advanced window functions, CTEs, and time-series growth tracking
-- ====================================================================
-- NORTHWIND EXECUTIVE ANALYTICS & PROBLEM-SOLVING SUITE
-- Tooling: PostgreSQL / DBeaver
-- Description: 10 advanced business intelligence scripts designed to 
--              demonstrate deep statistical and macro-analytical thinking.
-- ====================================================================

-- --------------------------------------------------------------------
-- QUERY 1: Month-over-Month (MoM) Revenue Growth & Trailing Trajectory
-- Objective: Track financial momentum by measuring the sequential percentage 
--            change in net monthly revenue.
-- --------------------------------------------------------------------
```
WITH MonthlySales AS (
    SELECT 
        DATE_TRUNC('month', o.order_date)::date AS fiscal_month,
        ROUND(SUM(od.unit_price * od.quantity * (1 - od.discount))::numeric, 2) AS current_month_revenue
    FROM orders o
    JOIN order_details od ON o.order_id = od.order_id
    GROUP BY DATE_TRUNC('month', o.order_date)
),
RevenueTrends AS (
    SELECT 
        fiscal_month,
        current_month_revenue,
        LAG(current_month_revenue, 1) OVER (ORDER BY fiscal_month) AS previous_month_revenue
    FROM MonthlySales
)
SELECT 
    fiscal_month,
    current_month_revenue,
    previous_month_revenue,
    ROUND((current_month_revenue - previous_month_revenue), 2) AS net_revenue_change,
    ROUND(((current_month_revenue - previous_month_revenue) / previous_month_revenue * 100)::numeric, 2) AS mom_growth_percentage
FROM RevenueTrends
WHERE previous_month_revenue IS NOT NULL
ORDER BY fiscal_month ASC;
```



-- --------------------------------------------------------------------
-- QUERY 2: ABC Inventory Classification (The Pareto 80/20 Rule)
-- Objective: Classify inventory stock into strategic tiers based on cumulative 
--            revenue contribution to optimize warehouse capital.
-- --------------------------------------------------------------------
```
WITH ProductRevenue AS (
    SELECT 
        p.product_id,
        p.product_name,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS total_product_revenue
    FROM order_details od
    JOIN products p ON od.product_id = p.product_id
    GROUP BY p.product_id, p.product_name
),
RunningRevenue AS (
    SELECT 
        product_id,
        product_name,
        total_product_revenue,
        SUM(total_product_revenue) OVER () AS global_revenue,
        SUM(total_product_revenue) OVER (ORDER BY total_product_revenue DESC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_revenue
    FROM ProductRevenue
)
SELECT 
    product_name,
    ROUND(total_product_revenue::numeric, 2) AS revenue,
    ROUND((cumulative_revenue / global_revenue * 100)::numeric, 2) AS cumulative_revenue_percentage,
    CASE 
        WHEN (cumulative_revenue / global_revenue) <= 0.80 THEN 'Class A (Top 80% Revenue - Critical)'
        WHEN (cumulative_revenue / global_revenue) <= 0.95 THEN 'Class B (Next 15% Revenue - Moderate)'
        ELSE 'Class C (Bottom 5% Revenue - Low Priority)'
    END AS abc_inventory_class
FROM RunningRevenue
ORDER BY total_product_revenue DESC;
```


-- --------------------------------------------------------------------
-- QUERY 3: Customer Churn Risk & Dormancy Audit
-- Objective: Identify high-value corporate accounts showing critical indicators 
--            of churn based on date lapsed since their last transaction.
-- --------------------------------------------------------------------
```
WITH CustomerRecency AS (
    SELECT 
        o.customer_id,
        c.company_name,
        MAX(o.order_date) AS last_order_date,
        COUNT(DISTINCT o.order_id) AS lifetime_order_count,
        ROUND(SUM(od.unit_price * od.quantity * (1 - od.discount))::numeric, 2) AS lifetime_spend,
        (SELECT MAX(order_date) FROM orders) AS max_database_date
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    JOIN order_details od ON o.order_id = od.order_id
    GROUP BY o.customer_id, c.company_name
)
SELECT 
    company_name,
    lifetime_order_count,
    lifetime_spend,
    last_order_date,
    (max_database_date - last_order_date) AS days_since_last_purchase,
    CASE 
        WHEN (max_database_date - last_order_date) >= 180 AND lifetime_spend >= 10000 THEN 'CRITICAL RISK: High-Value Dormant Account'
        WHEN (max_database_date - last_order_date) >= 180 THEN 'Dormant Account'
        WHEN (max_database_date - last_order_date) >= 90 THEN 'At-Risk/Inactive'
        ELSE 'Active Account'
    END AS account_health_status
FROM CustomerRecency
ORDER BY days_since_last_purchase DESC, lifetime_spend DESC;
```

-- --------------------------------------------------------------------
-- QUERY 4: Market Basket Analysis (Product Pair Affinity)
-- Objective: Uncover hidden cross-selling opportunities by identifying pairs 
--            of products frequently purchased together in the same order.
-- --------------------------------------------------------------------
```
SELECT 
    p1.product_name AS anchor_product,
    p2.product_name AS companion_product,
    COUNT(*) AS co_purchase_frequency
FROM order_details od1
JOIN order_details od2 ON od1.order_id = od2.order_id AND od1.product_id < od2.product_id
JOIN products p1 ON od1.product_id = p1.product_id
JOIN products p2 ON od2.product_id = p2.product_id
GROUP BY p1.product_name, p2.product_name
HAVING COUNT(*) >= 5
ORDER BY co_purchase_frequency DESC
LIMIT 10;


-- --------------------------------------------------------------------
-- QUERY 5: Employee Attainment vs Moving Average Performance Window
-- Objective: Evaluate sales performance against a dynamic 3-order moving average 
--            to separate short-term spikes from consistent performers.
-- --------------------------------------------------------------------
WITH EmployeeOrderValues AS (
    SELECT 
        o.employee_id,
        e.first_name || ' ' || e.last_name AS employee_name,
        o.order_id,
        o.order_date,
        ROUND(SUM(od.unit_price * od.quantity * (1 - od.discount))::numeric, 2) AS order_value
    FROM orders o
    JOIN order_details od ON o.order_id = od.order_id
    JOIN employees e ON o.employee_id = e.employee_id
    GROUP BY o.employee_id, e.first_name, e.last_name, o.order_id, o.order_date
)
SELECT 
    employee_name,
    order_date,
    order_value,
    ROUND(AVG(order_value) OVER (
        PARTITION BY employee_id 
        ORDER BY order_date, order_id
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    )::numeric, 2) AS rolling_3_order_average
FROM EmployeeOrderValues
ORDER BY employee_name, order_date ASC;
```


-- --------------------------------------------------------------------
-- QUERY 6: Supplier Shipping Velocity & SLA Breach Analysis
-- Objective: Isolate logistical bottlenecks by flagging suppliers whose products 
--            experience systemic delays past Promised Delivery SLA Dates.
-- --------------------------------------------------------------------
```
SELECT 
    s.company_name AS supplier_name,
    COUNT(DISTINCT o.order_id) AS total_shipped_orders,
    COUNT(CASE WHEN o.shipped_date > o.required_date THEN 1 END) AS sla_breach_count,
    ROUND((COUNT(CASE WHEN o.shipped_date > o.required_date THEN 1 END)::numeric / COUNT(DISTINCT o.order_id) * 100), 2) AS sla_breach_rate_percentage,
    ROUND(AVG(o.shipped_date - o.order_date), 1) AS average_dispatch_velocity_days
FROM orders o
JOIN order_details od ON o.order_id = od.order_id
JOIN products p ON od.product_id = p.product_id
JOIN suppliers s ON p.supplier_id = s.supplier_id
WHERE o.shipped_date IS NOT NULL
GROUP BY s.supplier_id, s.company_name
HAVING COUNT(DISTINCT o.order_id) >= 5
ORDER BY sla_breach_rate_percentage DESC;
```

-- --------------------------------------------------------------------
-- QUERY 7: Discount Elasticity Matrix
-- Objective: Measure elasticity trends by checking if applying price cuts 
--            actually drives statistically significant increases in volume.
-- --------------------------------------------------------------------
```
SELECT 
    p.product_name,
    ROUND(AVG(CASE WHEN od.discount = 0 THEN od.quantity END), 2) AS avg_qty_at_full_price,
    ROUND(AVG(CASE WHEN od.discount > 0 THEN od.quantity END), 2) AS avg_qty_discounted,
    ROUND(SUM(CASE WHEN od.discount > 0 THEN od.quantity END) - SUM(CASE WHEN od.discount = 0 THEN od.quantity END), 2) AS net_volume_shift,
    CASE 
        WHEN AVG(CASE WHEN od.discount > 0 THEN od.quantity END) > (AVG(CASE WHEN od.discount = 0 THEN od.quantity END) * 1.3) THEN 'Highly Elastic (Promotions Work)'
        ELSE 'Inelastic / Low Value Impact'
    END AS promotional_elasticity
FROM order_details od
JOIN products p ON od.product_id = p.product_id
GROUP BY p.product_id, p.product_name
HAVING COUNT(CASE WHEN od.discount > 0 THEN 1 END) > 3 AND COUNT(CASE WHEN od.discount = 0 THEN 1 END) > 3
ORDER BY net_volume_shift DESC;
```

-- --------------------------------------------------------------------
-- QUERY 8: Cohort Purchase Behavior (First-Time vs Repeat Revenue Split)
-- Objective: Determine company financial dependence on customer acquisition 
--            versus client retention metrics.
-- --------------------------------------------------------------------
```
WITH CustomerOnboarding AS (
    SELECT 
        customer_id,
        MIN(order_date) AS acquisition_date
    FROM orders
    GROUP BY customer_id
),
OrderClassifications AS (
    SELECT 
        o.customer_id,
        o.order_id,
        o.order_date,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS order_value,
        CASE 
            WHEN o.order_date = co.acquisition_date THEN 'First-Time Acquisition Order'
            ELSE 'Returning Retention Order'
        END AS order_cohort_type
    FROM orders o
    JOIN order_details od ON o.order_id = od.order_id
    JOIN CustomerOnboarding co ON o.customer_id = co.customer_id
    GROUP BY o.customer_id, o.order_id, o.order_date, co.acquisition_date
)
SELECT 
    order_cohort_type,
    COUNT(DISTINCT order_id) AS total_orders_handled,
    ROUND(SUM(order_value)::numeric, 2) AS gross_revenue_contribution,
    ROUND((SUM(order_value) / SUM(SUM(order_value)) OVER () * 100)::numeric, 2) AS share_percentage
FROM OrderClassifications
GROUP BY order_cohort_type;
```

-- --------------------------------------------------------------------
-- QUERY 9: Warehouse Burn Rate & Immediate Stockout Risk Profiler
-- Objective: Protect inventory supply lines by finding rapid-moving products 
--            whose safety stock coverage is dangerously thin.
-- --------------------------------------------------------------------
```
WITH ProductVelocity AS (
    SELECT 
        product_id,
        SUM(quantity) AS total_units_sold,
        COUNT(DISTINCT order_id) AS total_orders_involved,
        -- Calculate average unit drain velocity per transaction
        ROUND(AVG(quantity), 2) AS average_units_per_order
    FROM order_details
    GROUP BY product_id
)
SELECT 
    p.product_name,
    p.units_in_stock,
    p.reorder_level,
    pv.average_units_per_order AS demand_velocity_per_order,
    ROUND((p.units_in_stock / pv.average_units_per_order), 1) AS orders_coverage_left,
    CASE 
        WHEN p.units_in_stock <= p.reorder_level AND p.units_in_stock < (pv.average_units_per_order * 2) THEN 'CRITICAL: Imminent Stockout Risk'
        WHEN p.units_in_stock <= p.reorder_level THEN 'Action Required: Replenish Inventory'
        ELSE 'Healthy Stock Levels'
    END AS operational_inventory_alert
FROM products p
JOIN ProductVelocity pv ON p.product_id = pv.product_id
WHERE p.discontinued = 0
ORDER BY orders_coverage_left ASC
LIMIT 15;
```

-- --------------------------------------------------------------------
-- QUERY 10: Spatial Density Market Analysis & Running Sales Share
-- Objective: Identify core global territories and calculate the running percentage 
--            share across different shipping destination nations.
-- --------------------------------------------------------------------
```
WITH CountrySales AS (
    SELECT 
        o.ship_country AS target_nation,
        COUNT(DISTINCT o.order_id) AS total_shipments,
        ROUND(SUM(od.unit_price * od.quantity * (1 - od.discount))::numeric, 2) AS localized_revenue
    FROM orders o
    JOIN order_details od ON o.order_id = od.order_id
    WHERE o.ship_country IS NOT NULL
    GROUP BY o.ship_country
),
SpatialShare AS (
    SELECT 
        target_nation,
        total_shipments,
        localized_revenue,
        SUM(localized_revenue) OVER () AS total_global_revenue,
        SUM(localized_revenue) OVER (ORDER BY localized_revenue DESC) AS running_spatial_total
    FROM CountrySales
)
SELECT 
    target_nation,
    total_shipments,
    localized_revenue,
    ROUND((localized_revenue / total_global_revenue * 100), 2) AS national_market_share_percentage,
    ROUND((running_spatial_total / total_global_revenue * 100), 2) AS cumulative_global_share_percentage
FROM SpatialShare
ORDER BY localized_revenue DESC;
```
