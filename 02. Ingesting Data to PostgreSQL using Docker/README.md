# Ingesting Data to PostgreSQL

## Table of Contents
- [Setting Up PostgreSQL with Docker](#setting_up_postgreSQL_with_docker)
- [Accessing PostgreSQL via CLI](#accessing_postgresql_via_cli)
- [Load and manipulate the dataset via Jupiter Notebook](#load_and_manipulate_the_dataset_via_jupiter_notebook)


## Setting Up PostgreSQL with Docker

### We will see how to set up and run PostgreSQL locally using Docker.

![Diagram](diagram.png)


### We'll cover environment configuration, volume mapping, and database access.



Now, I need to formulate a command to start PostgreSQL. In Docker, we use the `docker run` command to execute a container. We have an image, `postgres:13`, which specifies both the software and its version. Additionally, we need to configure PostgreSQL.

```bash
docker run -it \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="db_data" \
  -v c:/Users/Farhad_Mustafayev/Documents/data-engineering/basics_setup/docker_s/data_postgres:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:13

```

### Explanation:
- `-e` flags define environment variables (user, password, database name).
- `-v` flag maps a host directory to the container's PostgreSQL data directory.
- `-p` maps port 5432 from the container to the host, allowing external connections.



Configuration in Docker is achieved using environment variables. These variables allow us to define specific settings for the container. In this case, we need to specify the database user, the corresponding password, and the database name.

To set environment variables when running a container with `docker run`, we use the `-e` flag, followed by the parameter and its assigned value. For this demonstration, I will set the user as `root`, the password as `root`, and name the database **db_data**, as this will be the dataset we use for practicing SQL.

**Volumes** in Docker enable mapping a folder from the host machine's file system to a folder inside the container. Since PostgreSQL is a database, it must store data persistently. By default, Docker containers do not retain state between runs, meaning that without volume mapping, all stored data would be lost upon restarting the container. To ensure data persistence, we need to mount a directory from our host machine to a directory within the container. This process is known as **mounting**. To achieve this, I will create a folder named **data_postgres** in Visual Studio Code and map it to `/var/lib/postgresql/data` within the container. The -v flag is used for volume mounting, followed by the path on the host machine and the corresponding path inside the container. Docker needs full windows-like path `c:/Users/Farhad_Mustafayev/Documents/dataengineering/basics_setup/docker_s/data_postgres`. For linux, you can do `-v $(pwd) /data_postgres:/var/lib/postgresql/data \`.

Additionally, we need to expose PostgreSQL’s default port to allow external access. This is necessary to send SQL queries to the database and receive responses. We achieve this by mapping a port on the host machine to the PostgreSQL port inside the container using the `-p` flag, specifically `-p 5432:5432`.


### Linux Alternative:

```bash
docker run -it \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="db_data" \
  -v $(pwd)/data_postgres:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:13
```
---
### Verifying Persistent Storage:
#### After running the container, check the `data_postgres` folder in Visual Studio Code.
Now, I can inspect the contents of the data_postgres folder in Visual Studio Code. Since it is mapped to the container’s data directory, we can observe that PostgreSQL has generated its required files there.

### PostgreSQL should have created data files within the mapped directory.
---
## Accessing PostgreSQL via CLI
#### To interact with the database, install and use `pgcli`, a Python-based PostgreSQL client:
Next, I will attempt to access the database. I will open a new terminal window in Git Bash and run `pgcli`, a command-line interface for interacting with PostgreSQL. If not already installed, it can be installed via `pip install pgcli`.



```bash
pgcli -h localhost -p 5432 -u root -d db_data
```
Using `pgcli`, I can connect to the database and execute queries. To establish a connection, I need to specify the following parameters:
•	-h (host) → localhost
•	-p (port) → 5432
•	-u (username) → root
•	-d (database) → db_data


### Expected output:
```bash
root@localhost:db_data>
```
### Checking Database Contents:
```bash
root@localhost:db_data> \dt  # List tables (empty initially)
root@localhost:db_data> SELECT 1;  # Test query
```
---
## Load and manipulate the dataset via Jupiter Notebook

Next, I will open another Git Bash terminal to use **Jupyter Notebook**, as I intend to load and manipulate the dataset using Python. Since the dataset is in **Parquet** format, I need the **PyArrow** library. Consequently, the **Dockerfile** must be modified to include the necessary dependencies.

  - Download the data file:
      - command: `curl -L -o data_to_postgres_d.parquet https:// /data_to_postgres_d.parquet`
      - description: The dataset is in Parquet format, which requires the `pyarrow` library.
  - Modify Dockerfile to include required dependencies:
  
      - dockerfile:
      ```bash
        - "FROM python:3.9"
        - "RUN pip install pandas pyarrow"
        - "WORKDIR /app"
        - "COPY pipeline.py pipeline.py"
        - "ENTRYPOINT [\"python\", \"pipeline.py\"]"
      ```
      - description: Ensures that necessary Python libraries are installed within the Docker container.

### process_data:
#### Read Parquet file and inspect metadata:
```python
import pandas as pd
import pyarrow.parquet as pq
from time import time
pq.read_metadata('data_to_postgres_d.parquet')

#description: Reads metadata to understand the structure of the dataset.

#Load Parquet file and inspect schema:

file = pq.ParquetFile('data_to_postgres_d.parquet')
table = file.read()
table.schema

#description: "Loads and inspects the dataset schema."

#Convert to Pandas DataFrame:

df = table.to_pandas()
df.tpep_pickup_datetime = pd.to_datetime(df.tpep_pickup_datetime)
df.tpep_dropoff_datetime = pd.to_datetime(df.tpep_dropoff_datetime)
   
#description: Converts the dataset into a Pandas DataFrame and ensures datetime columns are in the correct format.
```

Using Jupyter, I will convert the Parquet file into a Pandas **DataFrame** and access its contents. However, before inserting this data into PostgreSQL, I need to generate a **DDL (Data Definition Language) statement** that PostgreSQL can interpret. Pandas provides a convenient way to generate this **CREATE TABLE** statement automatically when writing data to a database.
To establish the connection to PostgreSQL, I need to specify:
- Database type → postgresql
-	Username → root
-	Password → root
-	Host → localhost
-	Port → 5432
-	Database name → db_data

#### Create a PostgreSQL connection:
```python
from sqlalchemy import create_engine
engine = create_engine('postgresql://root:root@localhost:5432/db_data')
engine.connect()
      
#### Generate SQL schema from DataFrame:
      
print(pd.io.sql.get_schema(df, name='data_in_postgres_d', con=engine))
```
    

Once the connection is established, we can proceed with loading the dataset into PostgreSQL. However, given the large size of the dataset, inserting all rows at once may not be efficient. Instead, we will **batch-process** the data by splitting it into smaller chunks using Pandas **iterators**. This approach allows for efficient insertion while preventing performance bottlenecks.


### insert data:
#### Test batch processing:
```python
batches_iter = file.iter_batches(batch_size=100000)
df = next(batches_iter).to_pandas()

#description: Uses Pandas iterators to create smaller batches of data before inserting into PostgreSQL.

df.head(0).to_sql(name='data_in_postgres_d', con=engine, if_exists='replace') # Create an empty table:
 
#description: Creates the table schema in PostgreSQL without inserting data.
```

### insert data in batches

```python
t_start = time()
count = 0
for batch in file.iter_batches(batch_size=100000):
    count += 1
    batch_df = batch.to_pandas()
    print(f'inserting batch {count}...')
    b_start = time()
    batch_df.to_sql(name='data_in_postgres_d', con=engine, if_exists='append')
    b_end = time()
    print(f'inserted! time taken {b_end-b_start:10.3f} seconds.\n')
t_end = time()
print(f'Completed! Total time taken was {t_end-t_start:10.3f} seconds for {count} batches.')

#description: Inserts data in smaller batches to avoid performance issues with large datasets.
```