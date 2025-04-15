## Weighting the MarketScan data at the national level

To extrapolate from the study cohort to the population of enrollees with employer sponsored health insurance, we derived weights from the American Community Survey (ACS). Weights were computed as the ratio of counts of individuals in ACS in 2018, in strata defined by sex, age, and state, divided by corresponding strata counts within the 2018 cohort of beneficiaries in the MarketScan data with non-grandfathered in plans.

### Step 1: group the ACS data by year, age, state, and sex and get ACS weights
Ensure that individuals are covered through employer sponsored health insurance. Once the grouping is done, sum up person weights.
```
SELECT year,
  age,
  acs_sex,
  state,
  SUM(PERWT) AS PERWT,
  FROM _source_
where HINSEMP = "Has insurance through employer/union"
group by year, age, state, acs_sex
order by year, age, state, acs_sex
```

### Step 2: create sample counts by year, age, state, and sex in the MarketScan data
We are only interested in individuals with non-grandfathered-in plans, i.e., those whose insurance plan began after January 2011.
```
SELECT
	YEAR,
	AGE,
	SEX,
	state,
	COUNT(DISTINCT ENROLID) AS denom
FROM
	_source_
WHERE age >= 15 and age <= 64 and first_date >= '2011-01-01' 
GROUP BY
	YEAR,
	AGE,
	SEX,
	state
```

### Step 3: Join the original MarketScan data table with the ACS weights table and the sample counts table
The original MarketScan data table contains all individuals aged 15-64 whose insurance provider began after January 2011.
```
SELECT
	_source_.ENROLID AS ENROLID,
	_source_.YEAR AS YEAR,
	_source_.SEX AS SEX,
	_source_.dep AS dep,
	_source_.AGE AS AGE,
	_source_.REGION AS REGION,
	_source_.first_date AS first_date,
	_source_.state AS state,
	t1.year AS t1_year,
	t1.age AS t1_age,
	t1.acs_sex AS t1_acs_sex,
	t1.state AS t1_state,
	t1.PERWT AS t1_PERWT,
	t1.strata AS t1_strata
FROM
	_source_
	-- Step 1 (join):
	INNER JOIN `acs_weights_by_state_sex_year_age_output` AS t1 ON _source_.YEAR = t1.year
	AND _source_.SEX = t1.acs_sex
	AND _source_.AGE = t1.age
	AND _source_.state = t1.state
```
```
SELECT
	_source_.ENROLID AS ENROLID,
	_source_.YEAR AS YEAR,
	_source_.SEX AS SEX,
	_source_.dep AS dep,
	_source_.AGE AS AGE,
	_source_.REGION AS REGION,
	_source_.first_date AS first_date,
	_source_.state AS state,
	t1.year AS t1_year,
	t1.age AS t1_age,
	t1.acs_sex AS t1_acs_sex,
	t1.state AS t1_state,
	t1.PERWT AS t1_PERWT,
	t1.strata AS t1_strata,
	t2.YEAR AS t2_YEAR,
	t2.AGE AS t2_AGE,
	t2.SEX AS t2_SEX,
	t2.state AS t2_state,
	t2.denom AS t2_denom
FROM
	_source_
	-- Step 2 (join):
	INNER JOIN `sample_counts_by_year_age_sex_state_output` AS t2 ON _source_.YEAR = t2.YEAR
	AND _source_.SEX = t2.SEX
	AND _source_.AGE = t2.AGE
	AND t1.state = t2.state
```

### Step 4: create the MarketScan Person Weight variable
For each enrollee, divide the ACS strata count by the MarketScan strata count, and use this weight for all future operations (e.g., counting service recipients, calculating screening rates...).
```
SELECT
    	ENROLID,
   	denom,
	sex,
	age,
	PERWT AS weights,
	year,
    state,
	SAFE_DIVIDE(PERWT, denom) AS MSCPSWT
FROM
	_source_
```



