## Dask

- Scales familiar analytics tools like Numpy, Pandas, and Scikit-Learn
- We love dask

  ```python
  import pandas as pd

  df = pandas.read_csv('path/to/file.csv')
  median_distance = df.groupby(df.account_id).distance.mean()


  import dask.dataframe as dd
  df = pandas.read_csv('path/to/file.csv')
  median_distance = df.groupby(df.account_id).distance.mean().compute()
  ```

- `dask.delayed` helps you parallelize your custom functions/algorithms
- Works on single machines 
  - Allows to use multiple cores
  - Reduce memory footprint
  - Reduce overhead to start working with it
- Works on machine cluster (i.e. AWS ECS)
  - Expand as you grow
  - No need for specialized machines or special tooling
  
### Compute duplicates and merge them, our use case

- Flow: 
  1. read data from source to "raw" files
  2. Translate new raw data to "interim" files
  3. Read existing data and merge with "interim" blindly
  4. Group by potential duplicate indicator fields, i.e. email
  5. Apply function to merge groups
  6. Save all to "processed" data
  
- We usually work with customer account data, this is more or less our flow
  
- AsyncIO helps on the first step writing to file each chunk we receive
- Dask takes over for the rest
  - Map customer account data into our internal data structure
    - Calculate gender, salary, lat/long, interests (from third party data)
  - Merge with existing accounts

```python
import dask
from dask.distributed import Client

client = Client()
client
```

```python
account_files = extract_accounts() # Extract accounts from source an save it to files

account_interim_files = (
    db.read_text(account_files)
    .map(json.loads)
    .flatten()
    .map(lambda record: translate_account(record))
    .map(json.dumps)
    .to_textfiles(f'data/interim/accounts-{now}-*.json')
)

new_accounts = (
    db.read_text(account_interim_files)
    .map(json.loads)
)
account_process_files = 'data/processed/accounts-*.json'
existing_accounts = (
    db.read_text(account_process_files)
    .map(json.loads)
)
all_accounts = db.concat([new_accounts, existing_accounts])

matcher = (
    all_accounts
    .map(extract_matching_info) # This function extracts account information used for matching, i.e. email
    .flatten()
    .to_dataframe(
        meta={'match_info': 'str', 'account': 'object'}
    )
)
all_accounts = matcher.groupby('match_info').apply(merge_customer)
all_acounts.map(json.dumps)
    .to_textfiles('data/processed/accounts-*.json')
```

```python
client.close()
```

### Final result compared with previous pipelines

- (comparisons made with 1.9 million customer data row)
- Less memory needed, 32Gb VM to 8Gb
- Less time 1 day -> 3 hours -> 30 minutes
- Smaller AWS bill ;-)
