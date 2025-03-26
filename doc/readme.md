## SQL Code for Data Analysis

This is the SQL code used to identify recipients of the 10 preventive services jeopardized by Kennedy v. Braidwood. The corresponding Redivis project can be viewed [here](https://redivis.com/workflows/h2b9-03vhykvre) once a Redivis account has been made.

### Step 1: Screening Date Selection
Select distinct beneficiary IDs and screening dates
```
SELECT DISTINCT
  BENE_ID,
  CAST(LINE_SRVC_BGN_DT AS date FORMAT 'DDMONYYYY') AS screen_date
FROM
  _source_
WHERE
  BENE_ID IS NOT NULL AND 
  (
    EXISTS (
      SELECT 1
      FROM UNNEST(@`bc_proc`) AS pattern
      WHERE LINE_PRCDR_CD LIKE pattern 
    )
  )
```
### Step 2: Latest Screening Date per Beneficiary per Year
Select the latest screening date for each beneficiary per year
```
SELECT *
FROM (
  SELECT *,
    ROW_NUMBER() OVER(PARTITION BY BENE_ID, EXTRACT(year FROM screen_date) ORDER BY screen_date DESC) rn
  FROM _source_
)
WHERE rn = 1
ORDER BY screen_date DESC
```

### Step 3: Exclusion Criteria
Select beneficiaries meeting procedure-based exclusion criteria
```
SELECT DISTINCT
  BENE_ID,
  CAST(LINE_SRVC_BGN_DT AS date FORMAT 'DDMONYYYY') AS exclusion_date
FROM
  _source_
WHERE
  BENE_ID IS NOT NULL AND 
  (
    EXISTS (
      SELECT 1
      FROM UNNEST(@`bc_proc_excl`) AS pattern
      WHERE LINE_PRCDR_CD LIKE pattern 
    )
  )
```
Select beneficiaries meeting diagnosis-based exclusion criteria
```
SELECT DISTINCT 
  BENE_ID,
  CAST(SRVC_BGN_DT AS date FORMAT 'DDMONYYYY') AS exclusion_date
FROM _source_
WHERE
  BENE_ID IS NOT NULL AND 
  (
    EXISTS (
      SELECT 1
      FROM UNNEST(@`bc_diag_excl`) AS pattern
      WHERE DGNS_CD_1 LIKE pattern OR DGNS_CD_2 LIKE pattern 
    )
  )
  ```

### Step 4: Join Screening Data
Join screening data with main source table
```
SELECT
  _source_.BENE_ID AS BENE_ID,
  _source_.AGE AS age,
  _source_.state AS state,
  _source_.year AS year,
  _source_.female AS female,
  _source_.yob AS yob,
  _source_.race AS race,
  _source_.month_year AS month_year,
  t1.screen_date AS screen_date
FROM
  _source_
  LEFT JOIN `screened` AS t1 
    ON _source_.BENE_ID = t1.BENE_ID
    AND _source_.year = EXTRACT(year from t1.screen_date)
```

### Step 5: Apply Exclusions and Filters
Apply exclusion criteria
```
SELECT *,
  CASE 
    WHEN EXISTS (
        SELECT 1 FROM `excluded` AS t1
        WHERE (_source_.BENE_ID = t1.BENE_ID
            AND _source_.month_year >= t1.exclusion_date)
        )
    THEN 1
    ELSE 0
    END AS excluded
FROM _source_
```

-- Filter data based on exclusion, gender, and age criteria
```
SELECT *
FROM _source_
WHERE excluded = 0 AND female = 1 AND age > 39 AND AGE < 66
```

### Step 6: Calculate Yearly Statistics
Calculate denominator, numerator, and screening rate by state
```
SELECT 
  state,
  COUNT(DISTINCT BENE_ID) AS denominator,
  SUM(screened) AS numerator,
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY BENE_ID, year ORDER BY screened DESC) beneyr
  FROM _source_
)
WHERE beneyr = 1 AND age > 17
GROUP BY state
ORDER BY state
```

### Citation
Michelle Bronsard. (2025). non_neema_marketscan_analysis [Workflow]. Redivis. https://redivis.com/workflows/h2b9-03vhykvre
