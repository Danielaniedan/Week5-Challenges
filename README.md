
-- Chinook Sales Analysis: SQL Queries (SQLite syntax)

-- Source: https://github.com/lerocha/chinook-database

-- Each query lists the business question and the techniques it uses.
-- =====================================================================
 
 
-- Q1. Overall sales baseline
-- Question: How much revenue did the store make, across how many invoices?
-- Techniques: AGGREGATE (SUM, COUNT, AVG, MIN, MAX)
SELECT
    COUNT(*)              AS total_invoices,
    ROUND(SUM(Total), 2)  AS total_revenue,
    ROUND(AVG(Total), 2)  AS avg_invoice_value,
    MIN(InvoiceDate)      AS first_sale,
    MAX(InvoiceDate)      AS last_sale
FROM Invoice;
 
 
-- Q2. Revenue by year
-- Question: How do sales change over time?
-- Techniques: AGGREGATE, GROUP BY, ORDER BY
SELECT
    strftime('%Y', InvoiceDate) AS sales_year,
    COUNT(*)                    AS invoices,
    ROUND(SUM(Total), 2)        AS revenue
FROM Invoice
GROUP BY sales_year
ORDER BY sales_year;
 
------------------------------------------------------------
-- Q3. TOP 3 PRODUCTS IN EACH CATEGORY USING ROW_NUMBER + PARTITION BY
------------------------------------------------------------
-- Business question:
-- What are the three highest-revenue tracks in each genre?
--
-- Approach:
-- The first CTE calculates revenue for each track within its genre.
-- The second CTE numbers tracks separately inside each genre, highest revenue first.
-- ROW_NUMBER() is used instead of RANK() because many Chinook tracks sold only once
-- and would tie; track_name is the tiebreaker so each genre returns at most 3 rows.

WITH track_genre_revenue AS (
    SELECT
        Genre.GenreId,
        Genre.Name AS genre,
        Track.TrackId,
        Track.Name AS track_name,
        SUM(InvoiceLine.UnitPrice * InvoiceLine.Quantity) AS track_revenue
    FROM Genre
    INNER JOIN Track
        ON Genre.GenreId = Track.GenreId
    INNER JOIN InvoiceLine
        ON Track.TrackId = InvoiceLine.TrackId
    GROUP BY Genre.GenreId, Genre.Name, Track.TrackId, Track.Name
),
ranked_tracks AS (
    SELECT
        genre,
        track_name,
        track_revenue,
        ROW_NUMBER() OVER (
            PARTITION BY GenreId
            ORDER BY track_revenue DESC, track_name
        ) AS category_rank
    FROM track_genre_revenue
)
SELECT
    genre,
    track_name,
    track_revenue,
    category_rank
FROM ranked_tracks
WHERE category_rank <= 3
ORDER BY genre, category_rank;
 
 
-- Q4. Top 10 customers by lifetime spend
-- Question: Who are the highest-value customers?
-- Techniques: INNER JOIN, AGGREGATE, GROUP BY, ORDER BY + LIMIT (top N)
SELECT
    c.CustomerId,
    c.FirstName || ' ' || c.LastName AS customer,
    c.Country,
    ROUND(SUM(i.Total), 2)           AS total_spent
FROM Customer c
INNER JOIN Invoice i
    ON i.CustomerId = c.CustomerId
GROUP BY c.CustomerId, customer, c.Country
ORDER BY total_spent DESC
LIMIT 10;
 
 
-- Q5. Repeat buyers
-- Question: Which customers buy most often, and do frequent buyers spend more?
-- Techniques: INNER JOIN, GROUP BY + HAVING, AGGREGATE
SELECT
    c.CustomerId,
    c.FirstName || ' ' || c.LastName AS customer,
    COUNT(i.InvoiceId)               AS purchases,
    ROUND(SUM(i.Total), 2)           AS total_spent,
    ROUND(AVG(i.Total), 2)           AS avg_order_value
FROM Customer c
INNER JOIN Invoice i
    ON i.CustomerId = c.CustomerId
GROUP BY c.CustomerId, customer
HAVING COUNT(i.InvoiceId) >= 7
ORDER BY purchases DESC, total_spent DESC;
 
 

-- 6. CASE WHEN TO CREATE CUSTOMER VALUE SEGMENTS

-- Business question:
-- How can customers be grouped into High, Medium, and Low value segments by lifetime spend?
--
-- Approach:
-- The first CTE calculates each customer's total invoice spend, retaining customers
-- with no invoices.
-- The second CTE uses NTILE(3) to split customers into three equal-sized groups by spend,
-- so the segments adapt to the data instead of relying on fixed dollar thresholds
-- (Chinook customers all spend similar amounts, so a fixed cutoff like 100 matches no one).
-- CASE then labels the groups: top third = High, middle third = Medium, bottom third = Low.

WITH customer_spend AS (
    SELECT
        Customer.CustomerId,
        Customer.FirstName,
        Customer.LastName,
        COALESCE(SUM(Invoice.Total), 0) AS lifetime_spend
    FROM Customer
    LEFT JOIN Invoice
        ON Customer.CustomerId = Invoice.CustomerId
    GROUP BY Customer.CustomerId, Customer.FirstName, Customer.LastName
),
spend_tiers AS (
    SELECT
        CustomerId,
        FirstName,
        LastName,
        lifetime_spend,
        NTILE(3) OVER (ORDER BY lifetime_spend DESC) AS spend_tier
    FROM customer_spend
)
SELECT
    CustomerId,
    FirstName,
    LastName,
    lifetime_spend,
    CASE
        WHEN spend_tier = 1 THEN 'High'
        WHEN spend_tier = 2 THEN 'Medium'
        ELSE 'Low'
    END AS customer_segment
FROM spend_tiers
ORDER BY lifetime_spend DESC, CustomerId;
 
 
-- Q7. Top 10 artists by revenue
-- Question: Which artists contribute the most sales?
-- Techniques: INNER JOIN (4 tables), AGGREGATE, ORDER BY + LIMIT (top N)
SELECT
    ar.Name                                    AS artist,
    SUM(il.Quantity)                           AS units_sold,
    ROUND(SUM(il.UnitPrice * il.Quantity), 2)  AS revenue
FROM InvoiceLine il
INNER JOIN Track  t  ON t.TrackId  = il.TrackId
INNER JOIN Album  al ON al.AlbumId = t.AlbumId
INNER JOIN Artist ar ON ar.ArtistId = al.ArtistId
GROUP BY ar.ArtistId, ar.Name
ORDER BY revenue DESC
LIMIT 10;
 
 
-- Q8. Tracks that have never sold
-- Question: Which catalog items have zero purchases?
-- Techniques: LEFT JOIN (anti-join with IS NULL), ORDER BY
SELECT
    t.TrackId,
    t.Name AS track,
    g.Name AS genre
FROM Track t
LEFT JOIN InvoiceLine il ON il.TrackId = t.TrackId
LEFT JOIN Genre g        ON g.GenreId  = t.GenreId
WHERE il.InvoiceLineId IS NULL
ORDER BY genre, track;
 
 
-- Q9. Sales rep performance
-- Question: How do support reps compare on customers and revenue?
-- Techniques: LEFT JOIN (keeps reps with no customers), AGGREGATE, GROUP BY
SELECT
    e.EmployeeId,
    e.FirstName || ' ' || e.LastName AS employee,
    e.Title,
    COUNT(DISTINCT c.CustomerId)     AS customers,
    ROUND(COALESCE(SUM(i.Total), 0), 2) AS revenue_managed
FROM Employee e
LEFT JOIN Customer c ON c.SupportRepId = e.EmployeeId
LEFT JOIN Invoice  i ON i.CustomerId   = c.CustomerId
GROUP BY e.EmployeeId, employee, e.Title
ORDER BY revenue_managed DESC;
 
 
-- Q10. Where do our customers live?
-- Question: Which countries and cities do we actually serve?
-- Techniques: DISTINCT, ORDER BY
SELECT DISTINCT
    Country,
    City
FROM Customer
ORDER BY Country, City;
 
 
-- Q11. Reporting structure
-- Question: Who reports to whom in the team?
-- Techniques: SELF JOIN (join of my choice, INNER)
SELECT
    e.FirstName || ' ' || e.LastName AS employee,
    e.Title                          AS employee_title,
    m.FirstName || ' ' || m.LastName AS manager,
    m.Title                          AS manager_title
FROM Employee e
INNER JOIN Employee m
    ON m.EmployeeId = e.ReportsTo
ORDER BY manager, employee;
 
 
-- Q12. Markets with multiple customers, ranked by average spend
-- Question: Which countries with at least 2 customers spend the most per customer?
-- Techniques: CTE, INNER JOIN, GROUP BY + HAVING, AGGREGATE (AVG, COUNT)
WITH customer_spend AS (
    SELECT
        c.CustomerId,
        c.Country,
        SUM(i.Total) AS spend
    FROM Customer c
    INNER JOIN Invoice i
        ON i.CustomerId = c.CustomerId
    GROUP BY c.CustomerId, c.Country
)
SELECT
    Country,
    COUNT(*)              AS customers,
    ROUND(AVG(spend), 2)  AS avg_spend_per_customer
FROM customer_spend
GROUP BY Country
HAVING COUNT(*) >= 2
ORDER BY avg_spend_per_customer DESC;
 
