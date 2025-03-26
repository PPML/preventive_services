## SQL code for data analysis

This is the SQL code used to identify recipients of breast cancer screenings. The Redivis project containing the code for all 10 preventive services studied can be viewed [here](https://redivis.com/workflows/h2b9-03vhykvre) once a Redivis account has been made.

### Step 1: Breast cancer screenings and date of screen
Select distinct enrollees and their screening dates from the inpatient and outpatient claims data files, using the CPT and HCPCS codes denoting preventive breast cancer screening (codes listed [here](https://github.com/PPML/preventive_services/blob/main/doc/breastcancer_codes.xlsx)).
```
SELECT 
  ENROLID,
  SVCDATE AS screen_date,
  1 as bc,
  AGE,
  YEAR,
  EGEOLOC,
  COPAY,
  COINS,
  DEDUCT,
  SEX
FROM
	_source_
WHERE
  ENROLID IS NOT NULL AND 
  (
    EXISTS (
    SELECT 1
    FROM UNNEST(@`bc_proc`) AS pattern
    WHERE PROC1 LIKE pattern 
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
Select beneficiaries meeting diagnosis-based exclusion criteria
```
SELECT DISTINCT 
  ENROLID,
  SVCDATE AS exclusion_date,
  1 as bc,
  AGE,
FROM _source_
WHERE
  ENROLID IS NOT NULL AND 
    EXISTS (
    SELECT 1
    FROM UNNEST(@`bc_diag_excl:mgm3`) AS pattern
    WHERE DX1 LIKE pattern OR 
          DX2 LIKE pattern OR
          DX3 LIKE pattern OR 
          DX4 LIKE pattern 
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
