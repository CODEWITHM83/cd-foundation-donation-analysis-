/*
===========================================================
Project: Cuddles Foundation Donation Analysis
Database: SQL Server
Author: Mithun Mondal
Description:
This script creates a donor analysis workflow using SQL Server.
It includes table creation, data quality checks, donor analysis,
repeat donor analysis, donation slab analysis, and summary queries
for Power BI / portfolio use.
===========================================================
*/

-----------------------------------------------------------
-- 1. CREATE TABLE
-----------------------------------------------------------
IF OBJECT_ID('dbo.donors', 'U') IS NOT NULL
    DROP TABLE dbo.donors;
GO

CREATE TABLE dbo.donors (
    identity_no VARCHAR(50),
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    phone VARCHAR(50),
    email VARCHAR(255),
    amount DECIMAL(18,2),
    remarks VARCHAR(500)
);
GO

-----------------------------------------------------------
-- 2. VIEW RAW DATA
-----------------------------------------------------------
SELECT *
FROM dbo.donors;
GO

-----------------------------------------------------------
-- 3. DATA QUALITY CHECKS
-----------------------------------------------------------

-- Total records
SELECT COUNT(*) AS total_records
FROM dbo.donors;
GO

-- Missing values count
SELECT
    SUM(CASE WHEN identity_no IS NULL OR LTRIM(RTRIM(identity_no)) = '' THEN 1 ELSE 0 END) AS missing_identity_no,
    SUM(CASE WHEN first_name IS NULL OR LTRIM(RTRIM(first_name)) = '' THEN 1 ELSE 0 END) AS missing_first_name,
    SUM(CASE WHEN last_name IS NULL OR LTRIM(RTRIM(last_name)) = '' THEN 1 ELSE 0 END) AS missing_last_name,
    SUM(CASE WHEN phone IS NULL OR LTRIM(RTRIM(phone)) = '' THEN 1 ELSE 0 END) AS missing_phone,
    SUM(CASE WHEN email IS NULL OR LTRIM(RTRIM(email)) = '' THEN 1 ELSE 0 END) AS missing_email,
    SUM(CASE WHEN amount IS NULL THEN 1 ELSE 0 END) AS missing_amount,
    SUM(CASE WHEN remarks IS NULL OR LTRIM(RTRIM(remarks)) = '' THEN 1 ELSE 0 END) AS missing_remarks
FROM dbo.donors;
GO

-- Duplicate donor IDs
SELECT
    identity_no,
    COUNT(*) AS duplicate_count
FROM dbo.donors
GROUP BY identity_no
HAVING COUNT(*) > 1
ORDER BY duplicate_count DESC;
GO

-- Invalid or blank emails
SELECT *
FROM dbo.donors
WHERE email IS NULL
   OR LTRIM(RTRIM(email)) = ''
   OR email NOT LIKE '%@%.%';
GO

-- Blank phone numbers
SELECT *
FROM dbo.donors
WHERE phone IS NULL
   OR LTRIM(RTRIM(phone)) = '';
GO

-----------------------------------------------------------
-- 4. CORE DONATION KPIs
-----------------------------------------------------------

-- Total donation
SELECT
    SUM(amount) AS total_donation
FROM dbo.donors;
GO

-- Average donation
SELECT
    AVG(amount) AS average_donation
FROM dbo.donors;
GO

-- Highest and lowest donation
SELECT
    MAX(amount) AS highest_donation,
    MIN(amount) AS lowest_donation
FROM dbo.donors;
GO

-- Unique donors
SELECT
    COUNT(DISTINCT identity_no) AS unique_donors
FROM dbo.donors;
GO

-----------------------------------------------------------
-- 5. DONOR-LEVEL ANALYSIS
-----------------------------------------------------------

-- Donor full name with donation amount
SELECT
    identity_no,
    CONCAT(ISNULL(first_name, ''), ' ', ISNULL(last_name, '')) AS donor_name,
    amount
FROM dbo.donors
ORDER BY amount DESC;
GO

-- Top 10 single donations
SELECT TOP 10
    identity_no,
    CONCAT(ISNULL(first_name, ''), ' ', ISNULL(last_name, '')) AS donor_name,
    amount
FROM dbo.donors
ORDER BY amount DESC;
GO

-- Top 10 donors by total contribution
SELECT TOP 10
    identity_no,
    CONCAT(MAX(ISNULL(first_name, '')), ' ', MAX(ISNULL(last_name, ''))) AS donor_name,
    COUNT(*) AS donation_count,
    SUM(amount) AS total_contribution
FROM dbo.donors
GROUP BY identity_no
ORDER BY total_contribution DESC;
GO

-----------------------------------------------------------
-- 6. REPEAT DONOR ANALYSIS
-----------------------------------------------------------

-- Repeat donors
SELECT
    identity_no,
    CONCAT(MAX(ISNULL(first_name, '')), ' ', MAX(ISNULL(last_name, ''))) AS donor_name,
    COUNT(*) AS donation_count,
    SUM(amount) AS total_contribution
FROM dbo.donors
GROUP BY identity_no
HAVING COUNT(*) > 1
ORDER BY total_contribution DESC;
GO

-- New vs Repeat donor summary
WITH donor_summary AS (
    SELECT
        identity_no,
        COUNT(*) AS donation_count,
        SUM(amount) AS total_amount
    FROM dbo.donors
    GROUP BY identity_no
)
SELECT
    CASE
        WHEN donation_count = 1 THEN 'New Donor'
        ELSE 'Repeat Donor'
    END AS donor_type,
    COUNT(*) AS donor_count,
    SUM(total_amount) AS total_donation
FROM donor_summary
GROUP BY
    CASE
        WHEN donation_count = 1 THEN 'New Donor'
        ELSE 'Repeat Donor'
    END
ORDER BY total_donation DESC;
GO

-----------------------------------------------------------
-- 7. DONATION SLAB ANALYSIS
-----------------------------------------------------------

SELECT
    CASE
        WHEN amount < 2000 THEN 'Below 2K'
        WHEN amount BETWEEN 2000 AND 4999 THEN '2K - 5K'
        WHEN amount BETWEEN 5000 AND 9999 THEN '5K - 10K'
        WHEN amount BETWEEN 10000 AND 24999 THEN '10K - 25K'
        ELSE '25K+'
    END AS donation_slab,
    COUNT(*) AS donor_count,
    SUM(amount) AS total_donation,
    AVG(amount) AS avg_donation
FROM dbo.donors
GROUP BY
    CASE
        WHEN amount < 2000 THEN 'Below 2K'
        WHEN amount BETWEEN 2000 AND 4999 THEN '2K - 5K'
        WHEN amount BETWEEN 5000 AND 9999 THEN '5K - 10K'
        WHEN amount BETWEEN 10000 AND 24999 THEN '10K - 25K'
        ELSE '25K+'
    END
ORDER BY total_donation DESC;
GO

-----------------------------------------------------------
-- 8. HIGH VALUE DONOR ANALYSIS
-----------------------------------------------------------

-- High value single donations (10K+)
SELECT
    identity_no,
    CONCAT(ISNULL(first_name, ''), ' ', ISNULL(last_name, '')) AS donor_name,
    amount
FROM dbo.donors
WHERE amount >= 10000
ORDER BY amount DESC;
GO

-- High value donors by total contribution (10K+)
SELECT
    identity_no,
    CONCAT(MAX(ISNULL(first_name, '')), ' ', MAX(ISNULL(last_name, ''))) AS donor_name,
    COUNT(*) AS donation_count,
    SUM(amount) AS total_contribution
FROM dbo.donors
GROUP BY identity_no
HAVING SUM(amount) >= 10000
ORDER BY total_contribution DESC;
GO

-----------------------------------------------------------
-- 9. EMAIL DOMAIN ANALYSIS
-----------------------------------------------------------

SELECT
    RIGHT(email, LEN(email) - CHARINDEX('@', email)) AS email_domain,
    COUNT(*) AS donor_count
FROM dbo.donors
WHERE email IS NOT NULL
  AND LTRIM(RTRIM(email)) <> ''
  AND email LIKE '%@%'
GROUP BY RIGHT(email, LEN(email) - CHARINDEX('@', email))
ORDER BY donor_count DESC;
GO

-----------------------------------------------------------
-- 10. DONORS WITH REMARKS
-----------------------------------------------------------

SELECT *
FROM dbo.donors
WHERE remarks IS NOT NULL
  AND LTRIM(RTRIM(remarks)) <> '';
GO

-----------------------------------------------------------
-- 11. CLEAN ANALYSIS VIEW
-----------------------------------------------------------
IF OBJECT_ID('dbo.vw_donor_analysis', 'V') IS NOT NULL
    DROP VIEW dbo.vw_donor_analysis;
GO

CREATE VIEW dbo.vw_donor_analysis AS
SELECT
    identity_no,
    CONCAT(ISNULL(first_name, ''), ' ', ISNULL(last_name, '')) AS donor_name,
    phone,
    email,
    amount,
    remarks,
    CASE
        WHEN amount < 2000 THEN 'Below 2K'
        WHEN amount BETWEEN 2000 AND 4999 THEN '2K - 5K'
        WHEN amount BETWEEN 5000 AND 9999 THEN '5K - 10K'
        WHEN amount BETWEEN 10000 AND 24999 THEN '10K - 25K'
        ELSE '25K+'
    END AS donation_slab,
    CASE
        WHEN email IS NULL OR LTRIM(RTRIM(email)) = '' THEN 'Missing Email'
        ELSE 'Email Available'
    END AS email_status,
    CASE
        WHEN phone IS NULL OR LTRIM(RTRIM(phone)) = '' THEN 'Missing Phone'
        ELSE 'Phone Available'
    END AS phone_status
FROM dbo.donors;
GO

-----------------------------------------------------------
-- 12. DASHBOARD SUMMARY QUERY
-----------------------------------------------------------

SELECT
    COUNT(*) AS total_records,
    COUNT(DISTINCT identity_no) AS unique_donors,
    SUM(amount) AS total_donation,
    AVG(amount) AS avg_donation,
    MAX(amount) AS max_donation,
    MIN(amount) AS min_donation,
    SUM(CASE WHEN amount >= 10000 THEN 1 ELSE 0 END) AS high_value_donation_count
FROM dbo.donors;
GO
