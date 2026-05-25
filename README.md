# Module 4 Homework: Analytics Engineering with dbt

In this homework, we'll use the dbt project in `04-analytics-engineering/taxi_rides_ny/` to transform NYC taxi data and answer questions by querying the models.

## Setup

1. Set up your dbt project following the [setup guide](../../../04-analytics-engineering/setup/)
2. Load the Green and Yellow taxi data for 2019-2020 and FHV trip data for 2019 into your warehouse (use static tables from [dtc github](https://github.com/DataTalksClub/nyc-tlc-data/), don't use offical tables from tlc because some values change from time to time)
3. Run `dbt build --target prod` to create all models and run tests

> **Note:** By default, dbt uses the `dev` target. You must use `--target prod` to build the models in the production dataset, which is required for the homework queries below.

After a successful build, you should have models like `fct_trips`, `dim_zones`, and `fct_monthly_zone_revenue` in your warehouse.

---


### My Setup:
- I needed to migrate my configurations which were based on local development using duckdb to cloud development using google cloud. I used Google Cloud Storage as my Data Lake and Google Cloud Bigquery as my Data Warehouse.

- I managed all my infrastructure as code using Terraform to create the bucket and datasets.

- I ingested all my files with Python into Google Cloud Storage.

- I read all the files, to create raw tables, using wildcard parameters. The query is as follows:
```sql
CREATE OR REPLACE EXTERNAL TABLE taxi_data_raw.fhv_taxi_data
OPTIONS (
  format = 'CSV',
  uris = ['gs://taxi-data-dbt-training/fhv_tripdata_*.csv.gz'],
  skip_leading_rows = 1
)
```
### Question 1. dbt Lineage and Execution

Given a dbt project with the following structure:

```
models/
├── staging/
│   ├── stg_green_tripdata.sql
│   └── stg_yellow_tripdata.sql
└── intermediate/
    └── int_trips_unioned.sql (depends on stg_green_tripdata & stg_yellow_tripdata)
```

If you run `dbt run --select int_trips_unioned`, what models will be built?

- [ ]`stg_green_tripdata`, `stg_yellow_tripdata`, and `int_trips_unioned` (upstream dependencies)
- [ ] Any model with upstream and downstream dependencies to `int_trips_unioned`
- [X] <span style = "color :green">**`int_trips_unioned`only**</span>
- [ ]`int_trips_unioned`, `int_trips`, and `fct_trips` (downstream dependencies)

**Solution:**
If i use the command select without a plus signal before or after the model, i just run the model wrote on the command.
Example:
```bash
dbt run --select int_trips_unioned #build the model int_trips_unioned

dbt run --select +int_trips_unioned #build the model int_trips_unioned and the upstream dependencies: stg_green_tripdata, stg_yellow_tripdata

dbt run --select int_trips_unioned+ #build the model int_trips_unioned and the downstream dependencies: int_trips, fct_trips

dbt run --select +int_trips_unioned+ #"Build all models, regardless of being upstream or downstream dependencies
```
---

### Question 2. dbt Tests

You've configured a generic test like this in your `schema.yml`:

```yaml
columns:
  - name: payment_type
    data_tests:
      - accepted_values:
          arguments:
            values: [1, 2, 3, 4, 5]
            quote: false
```

Your model `fct_trips` has been running successfully for months. A new value `6` now appears in the source data.

What happens when you run `dbt test --select fct_trips`?

- [ ] dbt will skip the test because the model didn't change
- [x] <span style = "color :green">**dbt will fail the test, returning a non-zero exit code**</span>
- [ ] dbt will pass the test with a warning about the new value
- [ ] dbt will update the configuration to include the new value
**Solution:**
dbt follows a simple rule: If the test query returns > 0 rows, the test fails.
---

### Question 3. Counting Records in `fct_monthly_zone_revenue`

After running your dbt project, query the `fct_monthly_zone_revenue` model.

What is the count of records in the `fct_monthly_zone_revenue` model?

- [ ] 12,998
- [ ] 14,120
- [x] <span style = "color :green"> **12,184** </span>
- [ ] 15,421

---
![images/image03.png](attachment:3a2d7144-0c5f-480d-b25d-b0c29af7368d:image.png)

### Question 4. Best Performing Zone for Green Taxis (2020)

Using the `fct_monthly_zone_revenue` table, find the pickup zone with the **highest total revenue** (`revenue_monthly_total_amount`) for **Green** taxi trips in 2020.

Which zone had the highest revenue?

- [x] <span style = "color :green">**East Harlem North**</span>
- [ ] Morningside Heights
- [ ] East Harlem South
- [ ] Washington Heights South
![images/image04.png](attachment:683025b3-ee3a-4127-9edf-b2d4f33ac4a7:image.png)
---

### Question 5. Green Taxi Trip Counts (October 2019)

Using the `fct_monthly_zone_revenue` table, what is the **total number of trips** (`total_monthly_trips`) for Green taxis in October 2019?

- [ ] 500,234
- [ ] 350,891
- [x] <span style = "color :green">**384,624**</span>
- [ ] 421,509

![images/image05.png]
---

### Question 6. Build a Staging Model for FHV Data

Create a staging model for the **For-Hire Vehicle (FHV)** trip data for 2019.

1. Load the [FHV trip data for 2019](https://github.com/DataTalksClub/nyc-tlc-data/releases/tag/fhv) into your data warehouse
2. Create a staging model `stg_fhv_tripdata` with these requirements:
   - Filter out records where `dispatching_base_num IS NULL`
   - Rename fields to match your project's naming conventions (e.g., `PUlocationID` → `pickup_location_id`)

What is the count of records in `stg_fhv_tripdata`?

- [ ] 42,084,899
- [ ] 43,244,693
- [ ] 22,998,722
- [x] 44,112,187

---
```sql
with source as (
    select * from {{ source('raw', 'fhv_taxi_data') }}
)

select 
    --base identifiers
    cast(dispatching_base_num as string) as dispatching_base_num,
    cast(Affiliated_base_number as string) as Affiliated_base_number,
    cast(SR_Flag as string) as SR_Flag,

    --timestamps
    cast(pickup_datetime as timestamp) as pickup_datetime,
    cast(dropoff_datetime as timestamp) as dropoff_datetime,

    --trip info
    cast(PUlocationID as integer) as pickup_location_id,
    cast(DOlocationID as integer) as dropoff_location_id,

from source 
-- Filter out records with null vendor_id (data quality requirement)
where dispatching_base_num is not null
```
![images/image06.png](attachment:16ed0f76-e814-4771-83a5-e1e9da3fdf70:image.png)
## Submitting the solutions

- Form for submitting: <https://courses.datatalks.club/de-zoomcamp-2026/homework/hw4>
with source as (
    select * from {{ source('raw', 'fhv_taxi_data') }}
)



## Learning in Public

We encourage everyone to share what they learned. This is called "learning in public".

Read more about the benefits [here](https://alexeyondata.substack.com/p/benefits-of-learning-in-public-and).

### Example post for LinkedIn

```
🚀 Week 4 of Data Engineering Zoomcamp by @DataTalksClub complete!

Just finished Module 4 - Analytics Engineering with dbt. Learned how to:

✅ Build transformation models with dbt
✅ Create staging, intermediate, and fact tables
✅ Write tests to ensure data quality
✅ Understand lineage and model dependencies
✅ Analyze revenue patterns across NYC zones

Transforming raw data into analytics-ready models - the T in ELT!

Here's my homework solution: <LINK>

Following along with this amazing free course - who else is learning data engineering?

You can sign up here: https://github.com/DataTalksClub/data-engineering-zoomcamp/
```

### Example post for Twitter/X

```
📈 Module 4 of Data Engineering Zoomcamp done!

- Analytics Engineering with dbt
- Transformation models & tests
- Data lineage & dependencies
- NYC taxi revenue analysis

My solution: <LINK>

Free course by @DataTalksClub: https://github.com/DataTalksClub/data-engineering-zoomcamp/
```
