---
title: Writing a Pandas Dataframe to SnowFlake
last_modified_at: 2022-10-17 08:18:20 +0200
---

The following describes how to write a Pandas DataFrame to SnowFlake. It is using both the SnowFlake function `write_pandas` and the SnowFlake Pandas helper `pd_writer`.


## Write data

<!-- {% raw %} -->
```python
import pandas as pd
from snowflake.connector.pandas_tools import pd_writer, write_pandas
from snowflake.connector import connect as sf_connect
from snowflake.sqlalchemy import URL
from sqlalchemy import create_engine


keyvault = ''
USERNAME = ''
PASSWORD = ''
ACCOUNT = ''
WAREHOUSE=''
ROLE=''

database_name = ''.upper()
schema_name = ''.upper()

# Create your DataFrame

table_name = 'cities'
df = pd.DataFrame(data=[['Stephen','Oslo'],['Jane','Stockholm']],columns=['Name','City'])

# Option 1 (creates table)
engine = create_engine(URL(
                        account = ACCOUNT,
                        user = USERNAME,
                        password = PASSWORD,
                        database = database_name,
                        schema = schema_name,
                        warehouse = WAREHOUSE,
                        role=ROLE,
                      ))
df.to_sql(name=table_name, con=engine, if_exists="replace", index=False, method=pd_writer)

# Option 2 (requires a created table)
cnx = sf_connect(
        user=USERNAME,
        password=PASSWORD,
        account=ACCOUNT,
        database = database_name,
        schema = schema_name,
        warehouse = WAREHOUSE,
        role=ROLE,
    )
success, nchunks, nrows, _ = write_pandas(cnx, df, table_name.upper(), database=database_name, schema=schema_name)
```
<!-- {% endraw %} -->

Note that if the DataFrame contains DateTime objects they must be `tz` aware, this can be accomplished by the following:

<!-- {% raw %} -->
```python
df.assign(execution_time = pd.Timestamp.now())
df['execution_time'] = pd.to_datetime(df['execution_time'], utc=True)
```
<!-- {% endraw %} -->
