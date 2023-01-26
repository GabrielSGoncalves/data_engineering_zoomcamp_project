## Setting up enviroment
### conda env zoomcamp_de
conda create -n zoomcamp_de python=3.9 jupyterlab pandas ipykernel pyarrow sqlalchemy psychopg2-binary
conda activate zoomcamp_de


## Postgres with Docker

## pgcli
```
pgcli -h localhost -p 5432 -u root -d ny_taxi
```

## Downloading the dataset
https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page
https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf

### Read data from NYC dataset for January of 2021
```python
df_nyc_jan21 = pd.read_parquet('datasets/yellow_tripdata_2021-01.parquet', engine='pyarrow')
```

### Get the DDL for table creation
```python
ddl_yellow_taxi_table  = pd.io.sql.get_schema(df_nyc_jan21, name='yellow_taxi_data')
print(ddl_yellow_taxi_table)
```

```
CREATE TABLE "yellow_taxi_data" (
"VendorID" INTEGER,
  "tpep_pickup_datetime" TIMESTAMP,
  "tpep_dropoff_datetime" TIMESTAMP,
  "passenger_count" REAL,
  "trip_distance" REAL,
  "RatecodeID" REAL,
  "store_and_fwd_flag" TEXT,
  "PULocationID" INTEGER,
  "DOLocationID" INTEGER,
  "payment_type" INTEGER,
  "fare_amount" REAL,
  "extra" REAL,
  "mta_tax" REAL,
  "tip_amount" REAL,
  "tolls_amount" REAL,
  "improvement_surcharge" REAL,
  "total_amount" REAL,
  "congestion_surcharge" REAL,
  "airport_fee" REAL
)
```

### Connect to database using Sqlalchemy
```python
from sqlalchemy import create_engine
engine = create_engine('postgresql://root:root@localhost:5432/ny_taxi')
engine.connect()
```
Create table with pandas.to_sql command
```python
df_nyc_jan21.head(0).to_sql(name='yellow_taxi_data', con=engine, if_exists='replace')
```
Check if table was created on pgcli terminal
```
\dt
```
```
+--------+------------------+-------+-------+
| Schema | Name             | Type  | Owner |
|--------+------------------+-------+-------|
| public | yellow_taxi_data | table | root  |
+--------+------------------+-------+-------+
SELECT 1
Time: 0.017s
```


Finally upload dataframe to postgres table using Jupyter and Sqlalchemy
```python
%%time
df_nyc_jan21.to_sql(name='yellow_taxi_data', con=engine, if_exists='append', chunksize=100_000)
```
Check the table size in pgcli
```sql
select count(1) from yellow_taxi_data;
```
```
+---------+
| count   |
|---------|
| 1369769 |
+---------+
SELECT 1
Time: 0.055s
```


## Using pgAdmin from Docker
To interact with Postgres Databases using a more suitable interface we are going to use [pgAdmin](https://www.pgadmin.org/).
One of the ways of using pgAdmin is through Docker, and that what we are going to do.
```
docker run -it \
  -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
  -e PGADMIN_DEFAULT_PASSWORD="root" \
  -p 8080:80 dpage/pgadmin4
``` 
Now if you open your web browswer on `localhost:8080` you will have access to pgAdmin.
One problem mentioned by the instructor is that as pgAdmin and Postgres database are running in separate containers, we need to specify a way for them to connect. To do it first we need to run the `network` command from docker.
```
docker network create pg-network
```
Next we need to specify the network to our Postgres Database Docker Image.
```
docker run -it \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root"\
  -e POSTGRES_DB="ny_taxi" \
  -v $(pwd)/ny_taxi_postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  --network=pg-network \
  --name pg-database \
  postgres:13
```
And also specify the network for the pgAdmin Docker Image.
```
docker run -it \
  -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
  -e PGADMIN_DEFAULT_PASSWORD="root" \
  -p 8080:80 \
  --network=pg-network \
  --name pgadmin dpage/pgadmin4
``` 
Now if you open your web browser on `localhost:8080` you open pgAdmin and you'll be able to access Postgres Database running on the other container.
[image]


