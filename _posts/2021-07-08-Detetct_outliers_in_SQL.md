---
title: Detetcting outliers in SQL Server
last_modified_at: 2021-07-08 10:14:10 +0200
---

The following is a short note on how to detect outliers in SQL Server.

```sql
WITH orderedList AS (
    SELECT sj.Name, sjh.job_id, CONVERT(date, CAST(run_date AS CHAR(12))) AS run_date, run_status, run_duration as duration,
    ROW_NUMBER() OVER (PARTITION BY sjh.job_id ORDER BY run_duration) AS row_n
    FROM msdb.dbo.sysjobhistory sjh
    JOIN msdb.dbo.sysjobs sj ON sj.job_id = sjh.job_id
    WHERE sjh.step_id = 0
),
iqr AS (
    SELECT *
    ,(SELECT AVG(duration) FROM orderedList ol 
    WHERE orderedList.job_id = ol.job_id) AS mean_duration 
    ,(SELECT duration FROM orderedList ol 
    WHERE orderedList.job_id = ol.job_id 
    AND row_n = FLOOR((SELECT MAX(row_n) FROM orderedList ol WHERE orderedList.job_id = ol.job_id)*.25) 
    ) AS q_one
    ,(SELECT duration FROM orderedList ol 
    WHERE orderedList.job_id = ol.job_id 
    AND row_n = FLOOR((SELECT MAX(row_n) FROM orderedList ol WHERE orderedList.job_id = ol.job_id)*.75) 
    ) AS q_three
    , 1.5 * ((SELECT duration FROM orderedList ol 
    WHERE orderedList.job_id = ol.job_id 
    AND row_n = FLOOR((SELECT MAX(row_n) FROM orderedList ol WHERE orderedList.job_id = ol.job_id)*.75) 
    ) - (SELECT duration FROM orderedList ol 
    WHERE orderedList.job_id = ol.job_id 
    AND row_n = FLOOR((SELECT MAX(row_n) FROM orderedList ol WHERE orderedList.job_id = ol.job_id)*.25) 
    )) AS outlier_range
 FROM orderedList
)

SELECT *
    , q_one - outlier_range AS lower_outlier
    , q_three + outlier_range AS upper_outlier
FROM iqr
WHERE (duration >= q_three + outlier_range OR duration <= q_one - outlier_range)
/*
WHERE (duration >= (
(SELECT MAX(q_three) FROM iqr i WHERE i.job_id = iqr.job_id) +
(SELECT MAX(outlier_range) FROM iqr i WHERE i.job_id = iqr.job_id)
) 
OR duration <= (
(SELECT MAX(q_one) FROM iqr i WHERE i.job_id = iqr.job_id) -
(SELECT MAX(outlier_range) FROM iqr i WHERE i.job_id = iqr.job_id)
))
*/
AND duration > 0 AND outlier_range > 0
AND run_date >= CONVERT(varchar(10), dateadd(day,-7,GETDATE()), 112)

```
