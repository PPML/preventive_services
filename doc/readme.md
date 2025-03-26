## SQL code for data analysis

This is the SQL code used to identify recipients of breast cancer screenings. The Redivis project containing the code for all 10 preventive services studied can be viewed [here](https://redivis.com/workflows/h2b9-03vhykvre) once a Redivis account has been made.

### Step 1: Breast cancer screening and date of screen
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
```
SELECT *,
  COINS + COPAY + DEDUCT AS OOP
FROM _source_
order by ENROLID, screen_date
```

### Step 2: Breast cancer diagnosis and date of exclusionary diagnosis
Select enrollees meeting diagnosis-based exclusion criteria (codes listed [here](https://github.com/PPML/preventive_services/blob/main/doc/breastcancer_codes.xlsx)) and keep their earliest instance of an exclusionary diagnosis.
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
    FROM UNNEST(@`bc_excl`) AS pattern
    WHERE DX1 LIKE pattern OR 
          DX2 LIKE pattern OR
          DX3 LIKE pattern OR 
          DX4 LIKE pattern 
    )
order by ENROLID, SVCDATE
```
```
SELECT *
FROM (
  SELECT *,
  ROW_NUMBER() OVER(PARTITION BY ENROLID) rn
  FROM _source_
)
WHERE rn = 1
```

### Step 3: Apply exlcusion and USPSTF criteria (specifically the population newly recommended for the service since 2010)
Apply exclusion criteria and filter data based on gender and age guidelines
```
SELECT * 
  FROM _source_ 
  WHERE first_date >= '2011-01-01' 
  AND (exclusion_date > screen_date OR exclusion_date IS NULL) 
  AND AGE >= 40 AND AGE <= 49 AND SEX = 2
```

```
SELECT ENROLID, 
  YEAR,
  MIN(AGE) AS AGE,
  SUM(OOP) AS OOP,
  MAX(EGEOLOC) AS state,
  MIN(SEX) AS SEX,
  "bc" as screen
  FROM _source_
GROUP BY ENROLID, YEAR
ORDER BY ENROLID, YEAR
```

### Step 4: Calculate unweighted and weighted rates of screening use
Join the demographic table containing each enrollee's weight (`MSCPSWT`) and calculate denominator, numerator, and screening rates for no-cost breast cancer screenings
```
SELECT DISTINCT
  ENROLID
  FROM _source_
WHERE year > 2017 and OOP = 0
```
```
SELECT
    SUM(MSCPSWT) AS weighted,
    COUNT(DISTINCT ENROLID) AS unweighted,
    (COUNT(DISTINCT ENROLID) / t1_denom) * 100 AS unweighted_rate,
    (SUM(MSCPSWT) / t1_weights) * 100 AS weighted_rate,
    t1_weights as ACS_denom,
    t1_denom as MS_counts,
FROM
	_source_
GROUP BY
  t1_weights,
  t1_denom
```

### Citation
Michelle Bronsard. (2025). non_neema_marketscan_analysis [Workflow]. Redivis. https://redivis.com/workflows/h2b9-03vhykvre
