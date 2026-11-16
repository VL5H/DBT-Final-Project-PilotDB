# Summary of Our Fixes in PilotDB

Our goal with these fixes was to address the following limitations that the authors acknowledged in their original paper:
1. Oversampling - PilotDB's result often come back roughly twice as accurate as what the user asked for. The imply that the sample used to evaluate the query was much larger than it needed to be which intern decreases response times.
2. On skewed distributions and on queries that group results into many buckets, impacts the speed from 100x to 1.4x, rare groups force the sample to grow. 

We also discovered a method to address an additional issue mentioned by the authors in which queries that contain sub-queries within the WHERE key-word cause the program to crash.

To implement our fixes we modified the following files:
1. \pilotdb\execute.py
2. \pilotdb\pilot_engine\error_bounds.py
2. \pilotdb\utils\utils.py


## Fix 1 - Oversampling:
In error_bound.py we modified the code the handles the sample size for each query.

In the original implementation the code jusdges sample sizes based on worst case measurements:

- inflated upper bounds on the sample standard deviation
- reduced lower bounds on the sample mean
- lower bound on smaple size

These measurements are highly conservative and often call for far more data than is neccessary to get an accurate result. This can lead to data bloat and drag down query runtimes as PilotDB is constantly analyzing the worst possible case in terms of data distribution. For example in TPC-H query 1 the sample measurement based on the worst-case calls for nearly the entire table making sampling completely pointless. 

Original Implementation:
```python
def get_mean_sample_size(
    error,
    fp: float,
    fp1: float,
    fp2: float,
    pilot_sample_mean,
    pilot_sample_std,
    pilot_sample_size,
):
    std_ub = get_std_ub(pilot_sample_size, pilot_sample_std, failure_probability=fp1)
    mean_lb = get_mean_lb(
        pilot_sample_size, pilot_sample_mean, pilot_sample_std, failure_probability=fp2
    )
    z_val = norm.ppf(1 - fp / 2)
    return (z_val / error * std_ub / mean_lb) ** 2

def get_sample_rate(
    fp: float, sample_size: int, pilot_sample_rate: float, pilot_sample_size: int
):
    if pilot_sample_rate > 0.9:
        bernoulli_N_lb = pilot_sample_size
    else:
        bernoulli_N_lb = get_bernoulli_N_lb(pilot_sample_size, pilot_sample_rate, fp)
    assert sample_size <= bernoulli_N_lb, f"{sample_size} is too big"
    z_val = norm.ppf(1 - fp)
    p = _solve_quadratic(
        a=bernoulli_N_lb**2 + z_val**2 * bernoulli_N_lb,
        b=-(2 * bernoulli_N_lb * sample_size + z_val**2 * bernoulli_N_lb),
        c=sample_size**2,
    )[1]
    return p

def get_bernoulli_N_sample_rate(
    error, fp: float, fp1: float, pilot_sample_rate: float, pilot_sample_size: int
):
    bernoulli_N_lb = get_bernoulli_N_lb(pilot_sample_size, pilot_sample_rate, fp1)
    z_val = norm.ppf(1 - fp / 2)
    return 1 / (1 + error**2 * bernoulli_N_lb / z_val**2)
```

In our implementation we introduced a new plugin we call "bounds" mode that sizes the final sample directly from the pilot's measured mean and standard deviation as well as the estimated table size that is defined as pilot pages / pilot rate. These are then used to replace the worst-case measurements mentioned above. This allows queries like TPC-H query 1 to use a much smaller sample size and achieve a similar level of accuracy at a much faster execution time. 

**Note:** The bounds plugin can be enaged by first running the command "$env:PILOTDB_BOUNDS = "plugin";" in your terminal. This sets the PilotDB env var to "plugin" and tells the code to refer to the plugin measurments rather than the default worst-case.

Our Fixes:
```python
BOUNDS_MODE = os.environ.get("PILOTDB_BOUNDS", "worstcase").lower()

def get_mean_sample_size(
    error,
    fp: float,
    fp1: float,
    fp2: float,
    pilot_sample_mean,
    pilot_sample_std,
    pilot_sample_size,
):
    if BOUNDS_MODE == "plugin":
        # Our Fix 1: CLT sizing with the pilot's measured mean and std, no inflation.
        z_val = norm.ppf(1 - fp / 2)
        return (z_val / error * pilot_sample_std / abs(pilot_sample_mean)) ** 2
    std_ub = get_std_ub(pilot_sample_size, pilot_sample_std, failure_probability=fp1)
    mean_lb = get_mean_lb(
        pilot_sample_size, pilot_sample_mean, pilot_sample_std, failure_probability=fp2
    )
    z_val = norm.ppf(1 - fp / 2)
    return (z_val / error * std_ub / mean_lb) ** 2

def get_sample_rate(
    fp: float, sample_size: int, pilot_sample_rate: float, pilot_sample_size: int
):
    if BOUNDS_MODE == "plugin":
        # Handler for our fix: plug-in population estimate N_hat = pilot pages / pilot rate, instead of the Bernoulli worst-case lower bound (whose assertion is the published code's "<n> is too big" fallback).
        n_hat = pilot_sample_size / pilot_sample_rate
        return min(1.0, sample_size / n_hat)
    if pilot_sample_rate > 0.9:
        bernoulli_N_lb = pilot_sample_size
    else:
        bernoulli_N_lb = get_bernoulli_N_lb(pilot_sample_size, pilot_sample_rate, fp)
    assert sample_size <= bernoulli_N_lb, f"{sample_size} is too big"
    z_val = norm.ppf(1 - fp)
    p = _solve_quadratic(
        a=bernoulli_N_lb**2 + z_val**2 * bernoulli_N_lb,
        b=-(2 * bernoulli_N_lb * sample_size + z_val**2 * bernoulli_N_lb),
        c=sample_size**2,
    )[1]
    return p

def get_bernoulli_N_sample_rate(
    error, fp: float, fp1: float, pilot_sample_rate: float, pilot_sample_size: int
):
    z_val = norm.ppf(1 - fp / 2)
    if BOUNDS_MODE == "plugin":
        # Our Fix 2: plug-in population estimate instead of the worst-case lower bound.
        n_hat = pilot_sample_size / pilot_sample_rate
        return 1 / (1 + error**2 * n_hat / z_val**2)
    bernoulli_N_lb = get_bernoulli_N_lb(pilot_sample_size, pilot_sample_rate, fp1)
    return 1 / (1 + error**2 * bernoulli_N_lb / z_val**2)

```

## Fix 2 - Variable Sample-Rate Ceiling:
In utils.py we modified the code that defines the maxmimum sample-rate or percentage of the largest table defined by the given query that PilotDB can actually use.

In the author's original implementation, this rate-ceiling is hard-coded with postgres having a default ceiling of 5%. This means that queries that require more samples than this 5% ceiling will automatically be rejected. This can once again be demonstrated with TPC-H query 1 that technically requires more that 5% of the total table to be sampled in order to produce accurate results. During testing, this query would still fail even with fixes 1 and 2 appplied. 

Original Implementation:
```python
def get_largest_sample_rate(dbms: str) -> float:
    if dbms == POSTGRES:
        return 5
    elif dbms == DUCKDB:
        return 75
    elif dbms == SQLSERVER:
        return 5
    else:
        return 10
```

In our implementaion we use another env variable called "PILOT_DB_MAX_RATE" to dynamically set the max sample rate as per the users needs (for our tests we set this variable to 20). By setting this variable the adjusted PilotDB implementation will be allowed to access larger percentages of the total query table, preventing queries that require larger samples from failing or falling back to just reading the entire table. This allows sampling to kick-in on more queries therby decreasing the total execution time without reducing result accuracy.

Our Fix:
```python
def get_largest_sample_rate(dbms: str) -> float:
    #  Our Fix: the default max sample rate (5% for Postgres) silently rejects feasible plans if they are over this limit (i.e. TPC-H Q1 at SF10 needs 6.6% under measurement-based
    # bounds). This can be overridden for any tests that need it via env var PILOTDB_MAX_RATE.
    override = os.environ.get("PILOTDB_MAX_RATE")
    if override is not None:
        return float(override)
    if dbms == POSTGRES:
        return 5
    elif dbms == DUCKDB:
        return 75
    elif dbms == SQLSERVER:
        return 5
    else:
        return 10
```

**Note:** The sample-rate ceiling can be adjusted by first running the command "$env:PILOTDB_MAX_RATE = "your number";" in your terminal. This sets the PilotDB env var and tells the code to refer to the new ceiling rather than the hard-coded defaults.

## Fix 3 - Sub-Query Errors: 
In execute.py we modified the section that handles non-rewritable queries.

In the original code the cost gate would recieve a pilot query before the placeholders for sub-queries has been filled in. This resulted in statements like "o_orderkey IN subquery_0" this in invalid syntax and would cause SQL to thwrow out any query that included a sub-query after the where key-word. This could specifically be seen in Query 18 and 22 of the TPC-H test set.

Original Implementation:
```python
if dbms == DUCKDB and not pq.is_rewritable:
        final_sample_rate = 1
        sampling_query = query.query
        subquery_results = {}
    elif directly_run_exact(conn, query.query, pilot_query, dbms, pq.largest_table):
        final_sample_rate = 1
        logging.info(f"retrieving query plan time: {timer.check('query_plan_time')}")
        subquery_results = {}
    else:
        # execute subqueries
        subquery_results = process_subqueries(dbms, conn, pq)
```

In our implementation we modified the if then clauses to first fill-in the place-holders for the sub-queries and then pass them to the cost-gate for execution. With this fix, Query 18 and 22 in TPC-H successfully run without throwing syntax errors.

Our Fix:
```python
if dbms == DUCKDB and not pq.is_rewritable:
        subquery_results = {}
    else:
        subquery_results = process_subqueries(dbms, conn, pq)
        for subquery_name, subquery_result in subquery_results.items():
            pilot_query = pilot_query.replace(subquery_name, subquery_result)
        logging.info(f"subquery execution time: {timer.check('subquery_execution')}")
```

## Conclusion
Using TPC-H SF10 at a 5% error target at a 95% confidence level, the original PilotDB implementation successfully ran sampling on 1 out of the total 15 queries with complete crashes on 2 queries. Our fixes as detailed above were able to improve this performance under the same circumstances with a result of sampling running succsessfully on 2 queries and 0 crashes. 
