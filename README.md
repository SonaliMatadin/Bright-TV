# Bright-TV
BrightTV Viewership Analytics case study analyzing user profiles and session data to uncover usage trends, consumption drivers, low-viewership patterns, and growth opportunities. Includes UTC→SA time conversion, engagement insights, and strategic recommendations using Snowflake, Excel, PowerPoint, Canva, and Miro.

-- Total active users, sessions, and total watch time
SELECT 
    COUNT(DISTINCT USERID) AS total_active_users,
    COUNT(*) AS total_sessions,
    SUM(DATEDIFF('second', TIME '00:00:00', DURATION2))/3600 AS total_hours_watched
FROM BRIGHT.TV.BRIGHTTV_CSV;

----------------------------------------------------------------------------------------------------------------------------------------------------------

--Average viewing duration per session and per user
SELECT 
    USERID,
    ROUND(AVG(DATEDIFF('second', TIME '00:00:00', DURATION2))/60,2) AS avg_minutes_per_session
FROM BRIGHT.TV.BRIGHTTV_CSV
GROUP BY USERID
ORDER BY avg_minutes_per_session DESC;

-----------------------------------------------------------------------------------------------------------------------------------------------------------
--Usage trends by day of week (after converting UTC to SA time)

CREATE OR REPLACE VIEW BRIGHT.TV.BRIGHTTV_CLEAN AS
SELECT
    USERID,
    NAME,
    SURNAME,
    EMAIL,
    GENDER,
    RACE,
    AGE,
    PROVINCE,
    SOCIAL_MEDIA_HANDLE,
    CHANNEL2,

    /* --- Clean and standardize timestamps --- */
    CASE 
        WHEN TRY_TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'DD/MM/YYYY HH24:MI') IS NOT NULL 
            THEN TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'DD/MM/YYYY HH24:MI')
        WHEN TRY_TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'MM/DD/YYYY HH24:MI') IS NOT NULL 
            THEN TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'MM/DD/YYYY HH24:MI')
        ELSE NULL 
    END AS RECORDDATE_UTC,

    /* --- Convert to South Africa timezone --- */
    CONVERT_TIMEZONE('UTC', 'Africa/Johannesburg',
        CASE 
            WHEN TRY_TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'DD/MM/YYYY HH24:MI') IS NOT NULL 
                THEN TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'DD/MM/YYYY HH24:MI')
            WHEN TRY_TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'MM/DD/YYYY HH24:MI') IS NOT NULL 
                THEN TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'MM/DD/YYYY HH24:MI')
            ELSE NULL 
        END
    ) AS RECORDDATE_SA,

    /* --- Calculate duration in seconds, minutes, and hours --- */
    DATEDIFF('second', TIME '00:00:00', DURATION2) AS duration_seconds,
    ROUND(DATEDIFF('second', TIME '00:00:00', DURATION2) / 60, 2) AS duration_minutes,
    ROUND(DATEDIFF('second', TIME '00:00:00', DURATION2) / 3600, 4) AS duration_hours

FROM BRIGHT.TV.BRIGHTTV_CSV
WHERE 
    (TRY_TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'DD/MM/YYYY HH24:MI') IS NOT NULL
     OR TRY_TO_TIMESTAMP_NTZ(TRIM(RECORDDATE2), 'MM/DD/YYYY HH24:MI') IS NOT NULL)
    AND DURATION2 IS NOT NULL;

-----------------------------------------------------------------------------------------------------------------------------------------------------------
--Peak viewing hours
SELECT
    EXTRACT(HOUR FROM RECORDDATE_SA) AS hour_of_day,
    COUNT(*) AS session_count
FROM BRIGHT.TV.BRIGHTTV_CLEAN
GROUP BY hour_of_day
ORDER BY session_count DESC;

--------------------------------------------------------------------------------------------------------------------------------------------------------
--Viewership by demographic group (gender, race, age)
SELECT 
    GENDER,
    RACE,
    CASE 
        WHEN AGE < 25 THEN 'Under 25'
        WHEN AGE BETWEEN 25 AND 40 THEN '25-40'
        WHEN AGE BETWEEN 41 AND 60 THEN '41-60'
        ELSE '60+' 
    END AS age_group,
    COUNT(*) AS total_sessions,
    ROUND(SUM(DATEDIFF('second', TIME '00:00:00', DURATION2))/3600,2) AS total_hours
FROM BRIGHT.TV.BRIGHTTV_CSV
GROUP BY 1,2,3
ORDER BY total_hours DESC;

-----------------------------------------------------------------------------------------------------------------------------------------------------------
-- Top provinces by engagement
SELECT 
    PROVINCE,
    COUNT(DISTINCT USERID) AS unique_users,
    ROUND(SUM(DATEDIFF('second', TIME '00:00:00', DURATION2))/3600,2) AS total_hours
FROM BRIGHT.TV.BRIGHTTV_CSV
GROUP BY PROVINCE
ORDER BY total_hours DESC;

-----------------------------------------------------------------------------------------------------------------------------------------------------------
-- Top channels by total hours watched
SELECT 
    CHANNEL2,
    COUNT(*) AS total_sessions,
    ROUND(SUM(DATEDIFF('second', TIME '00:00:00', DURATION2))/3600,2) AS total_hours
FROM BRIGHT.TV.BRIGHTTV_CSV
GROUP BY CHANNEL2
ORDER BY total_hours DESC
LIMIT 10;

-----------------------------------------------------------------------------------------------------------------------------------------------------------
--Identify lowest consumption days
SELECT
    TO_CHAR(
        CONVERT_TIMEZONE(
            'UTC', 
            'Africa/Johannesburg',
            CASE
                WHEN TRY_TO_TIMESTAMP_NTZ(RECORDDATE2, 'DD/MM/YYYY HH24:MI') IS NOT NULL 
                    THEN TO_TIMESTAMP_NTZ(RECORDDATE2, 'DD/MM/YYYY HH24:MI')
                WHEN TRY_TO_TIMESTAMP_NTZ(RECORDDATE2, 'MM/DD/YYYY HH24:MI') IS NOT NULL 
                    THEN TO_TIMESTAMP_NTZ(RECORDDATE2, 'MM/DD/YYYY HH24:MI')
                ELSE NULL
            END
        ),
        'YYYY-MM-DD'
    ) AS view_date,
    ROUND(SUM(DATEDIFF('second', TIME '00:00:00', DURATION2)) / 3600, 2) AS total_hours
FROM BRIGHT.TV.BRIGHTTV_CSV
WHERE
    TRY_TO_TIMESTAMP_NTZ(RECORDDATE2, 'DD/MM/YYYY HH24:MI') IS NOT NULL 
    OR TRY_TO_TIMESTAMP_NTZ(RECORDDATE2, 'MM/DD/YYYY HH24:MI') IS NOT NULL
GROUP BY view_date
ORDER BY total_hours ASC
LIMIT 5;

-----------------------------------------------------------------------------------------------------------------------------------------------------------
--Top-performing channels on similar weekdays:
SELECT
    CHANNEL2,
    COUNT(*) AS total_sessions,
    ROUND(SUM(DATEDIFF('second', TIME '00:00:00',DURATION2)) / 3600, 2) as total_hours
FROM BRIGHT.TV.BRIGHTTV_CLEAN
WHERE TO_CHAR(CONVERT_TIMEZONE('UTC', 'Africa/Johannesburg', RECORDDATE_CLEAN), 'Day')
      IN ('Tuesday', 'Wednesday')
GROUP BY CHANNEL2
ORDER BY total_hours DESC
LIMIT 5;

-----------------------------------------------------------------------------------------------------------------------------------------------------------
--Identify inactive or low-engagement users
SELECT 
    USERID,
    NAME,
    EMAIL,
    COUNT(*) AS total_sessions,
    ROUND(SUM(DATEDIFF('second', TIME '00:00:00', DURATION2))/3600,2) AS total_hours
FROM BRIGHT.TV.BRIGHTTV_CSV
GROUP BY 1,2,3
HAVING total_hours < 1
ORDER BY total_hours ASC;

-----------------------------------------------------------------------------------------------------------------------------------------------------------
-- Segment users by content preference
SELECT
    USERID,
    CHANNEL2 AS favorite_channel,
    COUNT(*) AS view_count
FROM BRIGHT.TV.BRIGHTTV_CSV
GROUP BY USERID, CHANNEL2
QUALIFY ROW_NUMBER() OVER (PARTITION BY USERID ORDER BY COUNT(*) DESC) = 1;

-------------------------------------------------------------------------------------------------------------------------------------------------------------New subscriber growth opportunities
SELECT 
    PROVINCE,
    COUNT(DISTINCT USERID) AS active_users,
    ROUND(AVG(AGE)) AS avg_age
FROM BRIGHT.TV.BRIGHTTV_CSV
GROUP BY PROVINCE
ORDER BY active_users ASC;
-----------------------------------------------------------------------------------------------------------------------------------------------------------





WITH base AS (
    SELECT * FROM BRIGHT.TV.BRIGHTTV_CLEAN
),

-- 1. Total users, sessions, hours
kpi_totals AS (
    SELECT
        COUNT(DISTINCT USERID) AS total_active_users,
        COUNT(*) AS total_sessions,
        ROUND(SUM(duration_seconds)/3600,2) AS total_hours_watched
    FROM base
),

-- 2. Avg minutes per session (overall)
kpi_avg_session AS (
    SELECT 
        ROUND(AVG(duration_minutes),2) AS avg_minutes_per_session
    FROM base
),

-- 3. Peak viewing hour
kpi_peak_hour AS (
    SELECT 
        EXTRACT(HOUR FROM RECORDDATE_SA) AS peak_hour
    FROM base
    GROUP BY 1
    ORDER BY COUNT(*) DESC
    LIMIT 1
),

-- 4. Lowest consumption day
kpi_lowest_day AS (
    SELECT
        TO_CHAR(RECORDDATE_SA, 'YYYY-MM-DD') AS lowest_day
    FROM base
    GROUP BY 1
    ORDER BY SUM(duration_seconds) ASC
    LIMIT 1
),

-- 5. Top channel
kpi_top_channel AS (
    SELECT 
        CHANNEL2 AS top_channel
    FROM base
    GROUP BY CHANNEL2
    ORDER BY SUM(duration_seconds) DESC
    LIMIT 1
),

-- 6. Top province by hours
kpi_top_province AS (
    SELECT 
        PROVINCE AS top_province
    FROM base
    GROUP BY PROVINCE
    ORDER BY SUM(duration_seconds) DESC
    LIMIT 1
),

-- 7. Most active age group
kpi_age_group AS (
    SELECT 
        CASE 
            WHEN AGE < 25 THEN 'Under 25'
            WHEN AGE BETWEEN 25 AND 40 THEN '25-40'
            WHEN AGE BETWEEN 41 AND 60 THEN '41-60'
            ELSE '60+' 
        END AS age_group
    FROM base
    GROUP BY age_group
    ORDER BY SUM(duration_seconds) DESC
    LIMIT 1
),

-- 8. Count of low engagement users (<1 hr)
kpi_low_users AS (
    SELECT 
        COUNT(*) AS low_engagement_users
    FROM (
        SELECT USERID
        FROM base
        GROUP BY USERID
        HAVING SUM(duration_seconds)/3600 < 1
    )
),

-- 9. Province with fewest users
kpi_lowest_province AS (
    SELECT 
        PROVINCE AS smallest_province,
        COUNT(DISTINCT USERID) AS users_in_province
    FROM base
    GROUP BY PROVINCE
    ORDER BY users_in_province ASC
    LIMIT 1
),

-- 10. Avg age of lowest province
kpi_low_prov_age AS (
    SELECT 
        PROVINCE AS low_prov,
        ROUND(AVG(AGE)) AS avg_age_low_province
    FROM base
    GROUP BY PROVINCE
    ORDER BY COUNT(DISTINCT USERID) ASC
    LIMIT 1
)

SELECT
    t.total_active_users,
    t.total_sessions,
    t.total_hours_watched,
    a.avg_minutes_per_session,
    h.peak_hour,
    d.lowest_day,
    c.top_channel,
    p.top_province,
    g.age_group AS top_age_group,
    u.low_engagement_users,
    lp.smallest_province,
    lpa.avg_age_low_province
FROM kpi_totals t
CROSS JOIN kpi_avg_session a
CROSS JOIN kpi_peak_hour h
CROSS JOIN kpi_lowest_day d
CROSS JOIN kpi_top_channel c
CROSS JOIN kpi_top_province p
CROSS JOIN kpi_age_group g
CROSS JOIN kpi_low_users u
CROSS JOIN kpi_lowest_province lp
CROSS JOIN kpi_low_prov_age lpa;


