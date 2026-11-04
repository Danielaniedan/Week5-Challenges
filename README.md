-- Week 5: Intermediate SQL Challenges
-- Database: Chinook (SQLite)
-- File: week5_challenges.sql

Notes:
-- * Track is treated as the product table; Genre is the product category.
-- * Track sales are measured as InvoiceLine.UnitPrice * InvoiceLine.Quantity.
-- * Customer value thresholds in Query 6 are illustrative; adjust them to the business.

------------------------------------------------------------
-- 1. SUBQUERY IN THE WHERE CLAUSE
------------------------------------------------------------
-- Business question:
-- Which tracks have a unit price higher than the average unit price of all tracks?
--
-- Approach:
-- The subquery calculates the average track price first.
-- The outer query returns only tracks priced above that average.

SELECT
    TrackId,
    Name AS track_name,
    UnitPrice
FROM Track
WHERE UnitPrice > (
    SELECT AVG(UnitPrice)
    FROM Track
)
ORDER BY UnitPrice DESC, track_name;


------------------------------------------------------------
-- 2. TWO CHAINED CTEs
------------------------------------------------------------
-- Business question:
-- Which artists generate the highest revenue, and what is each artist's revenue rank?
--
-- Approach:
-- The first CTE calculates revenue by artist from invoice line items.
-- The second CTE ranks those totals; the final query returns the top 10 artists.

WITH artist_revenue AS (
    SELECT
        Artist.ArtistId,
        Artist.Name AS artist_name,
        SUM(InvoiceLine.UnitPrice * InvoiceLine.Quantity) AS total_revenue
    FROM Artist
    INNER JOIN Album
        ON Artist.ArtistId = Album.ArtistId
    INNER JOIN Track
        ON Album.AlbumId = Track.AlbumId
    INNER JOIN InvoiceLine
        ON Track.TrackId = InvoiceLine.TrackId
    GROUP BY Artist.ArtistId, Artist.Name
),
ranked_artists AS (
    SELECT
        ArtistId,
        artist_name,
        total_revenue,
        RANK() OVER (ORDER BY total_revenue DESC) AS revenue_rank
    FROM artist_revenue
)
SELECT
    artist_name,
    total_revenue,
    revenue_rank
FROM ranked_artists
WHERE revenue_rank <= 10
ORDER BY revenue_rank, artist_name;


------------------------------------------------------------
-- 3. TOP 3 PRODUCTS IN EACH CATEGORY USING RANK + PARTITION BY
------------------------------------------------------------
-- Business question:
-- What are the three highest-revenue tracks in each genre?
--
-- Approach:
-- The first CTE calculates revenue for each track within its genre.
-- The second CTE ranks tracks separately inside each genre; ties share a rank.

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
        RANK() OVER (
            PARTITION BY GenreId
            ORDER BY track_revenue DESC
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
ORDER BY genre, category_rank, track_name;


------------------------------------------------------------
-- 4. ROW_NUMBER() TO DEDUPLICATE CUSTOMER RECORDS
------------------------------------------------------------
-- Business question:
-- If customer records contain duplicate email addresses, which record should be kept?
--
-- Approach:
-- ROW_NUMBER() numbers records sharing the same normalized email address.
-- Keep row 1 (the lowest CustomerId); blank or missing emails remain separate records.

WITH numbered_customers AS (
    SELECT
        CustomerId,
        FirstName,
        LastName,
        Email,
        Country,
        ROW_NUMBER() OVER (
            PARTITION BY
                CASE
                    WHEN Email IS NULL OR TRIM(Email) = ''
                        THEN 'missing-email-' || CustomerId
                    ELSE LOWER(TRIM(Email))
                END
            ORDER BY CustomerId
        ) AS row_num
    FROM Customer
)
SELECT
    CustomerId,
    FirstName,
    LastName,
    Email,
    Country
FROM numbered_customers
WHERE row_num = 1
ORDER BY CustomerId;


------------------------------------------------------------
-- 5. LAG() TO COMPARE MONTHLY REVENUE WITH THE PREVIOUS MONTH
------------------------------------------------------------
-- Business question:
-- How did each month's revenue change compared with the previous month?
--
-- Approach:
-- The first CTE aggregates invoice totals by calendar month.
-- LAG() brings the previous month's revenue onto the current month's row.

WITH monthly_revenue AS (
    SELECT
        STRFTIME('%Y-%m', InvoiceDate) AS sales_month,
        SUM(Total) AS monthly_revenue
    FROM Invoice
    GROUP BY STRFTIME('%Y-%m', InvoiceDate)
),
monthly_comparison AS (
    SELECT
        sales_month,
        monthly_revenue,
        LAG(monthly_revenue) OVER (ORDER BY sales_month) AS previous_month_revenue
    FROM monthly_revenue
)
SELECT
    sales_month,
    monthly_revenue,
    previous_month_revenue,
    monthly_revenue - previous_month_revenue AS revenue_change
FROM monthly_comparison
ORDER BY sales_month;


------------------------------------------------------------
-- 6. CASE WHEN TO CREATE CUSTOMER VALUE SEGMENTS
------------------------------------------------------------
-- Business question:
-- How can customers be grouped into High, Medium, and Low value segments by lifetime spend?
--
-- Approach:
-- The CTE calculates each customer's total invoice spend, retaining customers with no invoices.
-- CASE assigns segments using illustrative thresholds: High >= 100, Medium >= 40, otherwise Low.

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
)
SELECT
    CustomerId,
    FirstName,
    LastName,
    lifetime_spend,
    CASE
        WHEN lifetime_spend >= 100 THEN 'High'
        WHEN lifetime_spend >= 40 THEN 'Medium'
        ELSE 'Low'
    END AS customer_segment
FROM customer_spend
ORDER BY lifetime_spend DESC, CustomerId;


------------------------------------------------------------
-- 7. RUNNING TOTAL USING SUM() OVER (ORDER BY ...)
------------------------------------------------------------
-- Business question:
-- What is the cumulative revenue generated month by month?
--
-- Approach:
-- The CTE first calculates revenue for each month.
-- SUM() OVER orders the months and adds each month's revenue to all preceding months.

WITH monthly_revenue AS (
    SELECT
        STRFTIME('%Y-%m', InvoiceDate) AS sales_month,
        SUM(Total) AS monthly_revenue
    FROM Invoice
    GROUP BY STRFTIME('%Y-%m', InvoiceDate)
)
SELECT
    sales_month,
    monthly_revenue,
    SUM(monthly_revenue) OVER (
        ORDER BY sales_month
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total_revenue
FROM monthly_revenue
ORDER BY sales_month;


------------------------------------------------------------
-- 8. REWRITE A COMPLEX SUBQUERY AS A CTE
------------------------------------------------------------
-- Business question:
-- Which customers have lifetime spending above the average lifetime spend per customer?
--
-- Approach:
-- The CTE calculates one lifetime-spend value per customer; the second CTE calculates
-- the average of those customer totals. The final query compares each customer to that average.
--
-- Alternative subquery approach (for comparison):
-- SELECT CustomerId, FirstName, LastName, total_spend
-- FROM (
--     SELECT Customer.CustomerId, Customer.FirstName, Customer.LastName,
--            COALESCE(SUM(Invoice.Total), 0) AS total_spend
--     FROM Customer
--     LEFT JOIN Invoice ON Customer.CustomerId = Invoice.CustomerId
--     GROUP BY Customer.CustomerId, Customer.FirstName, Customer.LastName
-- ) AS customer_totals
-- WHERE total_spend > (
--     SELECT AVG(total_spend)
--     FROM (
--         SELECT Customer.CustomerId,
--                COALESCE(SUM(Invoice.Total), 0) AS total_spend
--         FROM Customer
--         LEFT JOIN Invoice ON Customer.CustomerId = Invoice.CustomerId
--         GROUP BY Customer.CustomerId
--     ) AS average_customer_totals
-- );
--
-- Choice:
-- I would choose the CTE version because each calculation has a clear name and can be
-- inspected or reused. A subquery can be concise for a small one-off calculation, but
-- nested subqueries become harder to read as the logic grows.

WITH customer_spend AS (
    SELECT
        Customer.CustomerId,
        Customer.FirstName,
        Customer.LastName,
        COALESCE(SUM(Invoice.Total), 0) AS total_spend
    FROM Customer
    LEFT JOIN Invoice
        ON Customer.CustomerId = Invoice.CustomerId
    GROUP BY Customer.CustomerId, Customer.FirstName, Customer.LastName
),
average_customer_spend AS (
    SELECT AVG(total_spend) AS average_spend
    FROM customer_spend
)
SELECT
    customer_spend.CustomerId,
    customer_spend.FirstName,
    customer_spend.LastName,
    customer_spend.total_spend,
    average_customer_spend.average_spend
FROM customer_spend
CROSS JOIN average_customer_spend
WHERE customer_spend.total_spend > average_customer_spend.average_spend
ORDER BY customer_spend.total_spend DESC;

