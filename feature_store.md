# Feature store for small big data

## What is a feature store and why do I need it?

- It is like a data warehouse of features for machine learning
- 80% of data science is data wrangling -> precious work
- Many fields (sometimes expensive to calculate) are used in several models
- Better collaboration between team members
- Identify easier data drift (versioning)

- Examples:
  - Michaelangelo (Uber) on top of Hive and Cassandra
    <https://eng.uber.com/michelangelo-machine-learning-platform/>
  - Feast based on Google Could Service like Big Query and Big Table
    <https://github.com/feast-dev/feast>
  - Hopsworks uses Hive and MySQL Cluster
    <https://hopsworks.readthedocs.io/en/1.1/featurestore/featurestore.html>

- All the examples are either proprietary solutions or expensive to implement and maintain.

## Our take

- We build our own solution based on already existing/inexpensive tools:
  - Parquet files
  - Dask
  - AWS Athena
- Mostly for offline usage, although due to the nature and size of our data, it would be a problem to use online too.
  
- Not yet to be released to the public, we are still working on a few rough edges.

### Consumer high level API 

- This is how we update the features on our store
  ```python
  fs = FeatureStore(config={....})
  group = fs.get_group('customer')
  group.calculate()
  ```
  
- This is how we read them from the store
  ```python
  fs = FeatureStore(config={....})
  group = fs.get_group('customer')
  f_set = group.get_features(['distance', 'gender'], exclude=['*'], filters=['age >= 30'])
  # f_set is a dask dataframe
  ```
  
  
### Adding new features

- Function based
  - One function responsible for one or many features 
  - Dask helps with some of the calculations
```python
def customer_general_features():
    stm = select(
    [...]
    
    ).where(...) 
    df = get_data(stm)
    df = df.assign(
        distance=df[['lat', 'long']].apply(distance, meta=float)
        ...
    )
    ...
    return df
```

- Configuration to keep track of all the fields
```python
FEATURE_STORE=dict(
    customer=dict(
        fields={
            'distance': float
            ...
        },
        functions=[
            'import.path.to.customer_general_info',
            ...
        ]
    )
)
```

- Why the configuration?
  - Share field between clients and allows for per client customization
  - Reproducibility
  
- The magic sauce
  - Collect all the functions and save to parquet the result
  - To read, create a tmp table on Athena and run the query against it, courtesy of
    aws data wrangler <https://aws-data-wrangler.readthedocs.io>
    
```python
def build_store(
    orgid: str, funcs: Iterable, types: Dict, **kwargs: Any
) -> dd.DataFrame:
    """Build a feature store."""
    tasks = iter(
        compute([delayed(func)(orgid=orgid, **kwargs) for func in funcs])[0]
    )

    ddf = next(tasks)
    for features_df in tasks:
        ddf = ddf.merge(
            features_df, how='outer', left_index=True, right_index=True
        )

    # Assign correct types
    ddf = ddf.astype(types)
    
    return ddf
```
  
```python
def create_table(
    db: str, name: str, path: str, columns_types: Dict[str, str], **kwargs: Any
) -> None:
    """Create a new Athena/Glue table."""
    if name not in [
        table["Name"] for table in wr.catalog.get_tables(database=db)
    ]:
        if 'id' not in columns_types:
            columns_types['id'] = 'string'

        wr.catalog.create_parquet_table(
            database=db,
            table=name,
            path=path,
            columns_types=columns_types,
            **kwargs
        )
```

- Normal SQL queries can be run on Athena

```python
def run_query(
    sql: str, db: str, path: str, chunks: bool = False, **kwargs
) -> Union[pd.DataFrame, Iterator[pd.DataFrame]]:
    """Run query against Athena/Glue database."""
    return wr.athena.read_sql_query(
        sql, db, s3_output=path, chunksize=chunks or None, **kwargs
    )
```
