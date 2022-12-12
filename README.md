# RSPN Testing

## Environment
Python 3.8 on Ubuntu 20.04

## Setting up
1. Download the codes.
```
# git clone https://github.com/kihyuk-nam/ai503-project.git
```
2. (Option) Create a virtual environment (using venv or conda).
  - For venv: (See:https://docs.python.org/3/library/venv.html)
    ```
    # python3 -m venv venv 
    # source venv/bin/activate
    (venv) #
    ```

3. Install dependencies. For example,
```
(venv) # pip install numpy pandas tables spflow, sqlparse, psycopg2

// If you want to use REST interface (Fast API), install the followings as well
(venv) # pip install fastapi uvicorn requests psutil

// cf. requirements.txt
```
## Running Option1: TrainDBCliModelRunner
### Training
```
(venv) # python3 TrainDBCliModelRunner.py train2 RSPN model/types/RSPN.py instacart data/files/instacart/csv/orders.csv data/files/ model/instances 0.3 10 10000000 

(some warnings...)
```
- **command**: command for RSPN testing (ex, **train2**)
- **modeltype_class**: name of the model (class) (ex, **RSPN**)
- **modeltype_uri**: path for the model class file (ex, **model/types/RSPN.py**)
- **data_name**: name(space) of the training dataset, (ex, **instacart**)
- **data_file**: path to the training dataset, (ex, **data/files/instacart/csv/orders.csv**)
- **metadata_root**: root dir of the metadata(.json or .hdf) (ex, **data/files/**)
- **model_path**: (root) path to the generated model (ex, **model/instances**)
- **rdc_threshold**: (float) RDC threshold (ex, **0.3**)
- **post_sampling_factor**: (int) post sampling factor (ex, **10**)
- **sample_size**: sample size (ex, **10000000**)

### Estimation
**TBD**

## Running Option2: Instantiate RSPN
### Training and Estimation
1. Write a script. For example, test.py
```
from model.types.RSPN import RSPN

app = RSPN()

# training
model_path = app.train('instacart',
                       '/home/nam/Projects/datasets/instacart/orders.csv',
                       'data/files/',
                       'model/instances',
                       0.3,
                       10,
                       10000000)
print(model_path)

# estimation
query = 'SELECT COUNT(*) FROM orders WHERE order_dow >= 2'
print(query)
result = app.estimate(query, 'instacart', app.table_csv_path, model_path, True)
print(result)
```
2. Check the result:
```
(venv) # python3 test.py

(some warnings...)

model/instances/ensemble1_single_instacart_10000000.pkl
SELECT COUNT(*) FROM orders WHERE order_dow >= 2
((1343995.131280017, 1347515.079651983), 1345755.105466)
```
## Running Option3: REST API (using Fast API)

1. Execute the **rest.py**. 
The default host address and port (http://0.0.0.0:8000) will be applied if no args specified.
- run the following command if not installed
```
(venv) # pip install fastapi uvicorn requests psutil
```
- launch the rest service
```
(venv) # python3 rest.py
```
  - For setting up your own address/port (e.g., http://127.0.0.1:8080):
    ```
    (venv) # python3 rest.py --rest_host 127.0.0.1 --rest_port 8080
    ```

2. Open a web browser and go to the address you specified.
(For default setting: **http://0.0.0.0:8000/docs**)

![rest-all-current](https://user-images.githubusercontent.com/24988105/187908169-eaf31492-84ef-4619-8acd-535ef2aab5c2.png)

### Training
1. Select the '/train/' and click 'Try it out'.
   - **/train-sync** runs synchronously (waits for termination)
   - **/train-async** runs asynchronously (runs the 'TrainDBCliModelRunner.py' and exits)
     - use 'check' to see its status
3. Input arguments and click 'execute'. 
   - **dataset**: name of the dataset, which will be used a prefix of the learned model name
   - **csv_path**: training data, which must be in the server directory
     (upload and remote URL are not yet supported)
   - **metadata_path**: .json or .hdf file containing metadata of the input(csv)
   - **model_path**: location of the model to be generated
   - **rdc_threshold**: (float) RDC threshold (ex, 0.3)
   - **post_sampling_factor**: (int) post sampling factor (ex, 10 or 30)
   - **sample_size**: sample size (ex, 10000000)
   
   For example:
#### train-sync
![rest-train-sync](https://user-images.githubusercontent.com/24988105/187906996-68cd018f-1e54-42d6-9768-bc7a513e2146.png)

#### train-async
![rest-train-async](https://user-images.githubusercontent.com/24988105/187907028-80c93992-332d-4e1a-ae3b-0a44e2ba9b22.png)

3. [Option] CLI
   - install **requests** package if not exists
   ```
   (venv) # pip install requests
   ```
   - write a script calling APIs. For example,
   ```
   import requests
   url = 'http://0.0.0.0:8000/train/'
   model_name = 'instacart'
   csv_path = '/datasets/instacart/orders.csv'
   metadata_path = 'data/files/'
   model_path = 'model/instances'
   rdc_threshold = 0.3
   post_sampling_factor = 10
   sample_size = 10000000
   payload = {'dataset':model_name, 'csv_path':dataset_path, 'metadata_path':metadata_path, 'model_path'=model_path,
              'rdc_threshold':rdc_threshold, 'post_sampling_factor':post_sampling_factor, 'sample_size':sample_size}
   response = requests.post(url, json=payload)
   result = response.json()
   print(response.status_code)
   print(result)
   ```
### Check Training Status
1. Select the '**/check**' and click 'Try it out'.
2. Input the pid you want to see and click 'execute'.
   - **pid**: process ID of the training process
![rest-check](https://user-images.githubusercontent.com/24988105/187907881-7986ffab-a4a4-4d67-bf57-e135f0ce0e75.png)



### Estimation (AQP)
1. Select the '**/estimate**' and click 'Try it out'.
2. Input arguments and click 'execute'.
   - **query**: an SQL statement to be approximated. 
     
     e.g., **SELECT COUNT(*) FROM orders WHERE order_dow >= 2**
     
     (currently COUNT is supported. SUM and AVG will be available soon)
     
     (The table name(orders) should match the csv name(orders.csv))
     
   - **dataset**: name(space) of the dataset you learned
   - **model_path**: location of the learned model, which must be uploaded in advance in the server filesystem.
   - **table_csv_path** (To be removed): location of the data file used for training
   - **show_confidence_intervals**: yes/no
   
   For example:
   
![rest-estimate](https://user-images.githubusercontent.com/24988105/187428163-f9f342a8-fe55-40df-91f9-82d4b7b7a1e8.png)


3. [Option] CLI
  ```
  import requests
  url = 'http://0.0.0.0:8000/estimate/'
  query = 'SELECT COUNT(*) FROM orders WHERE order_dow >= 2'
  dataset = 'instacart'
  table_csv_path = 'data/files/instacart/csv/orders.csv'
  model_path = 'model/instances/ensemble_single_instacart_10000000.pkl'
  show_confidence_intervals = 'true'
  payload = {'query':query,
             'dataset':dataset,
             'table_csv_path':table_csv_path,
             'model_path':model_path,
             'show_confidence_intervals':show_confidence_intervals}

  response = requests.get(url, params=payload)
  result = response.json()
  print(response.status_code)
  print(result['Estimated Value'])
  ```
## Using KubeFlow
- See [/interface](https://github.com/traindb-project/traindb-ml/tree/main/interface)

## License
This project is dual-licensed. Apache 2.0 is for traindb-ml, and MIT is for the RSPN codes from deepdb(https://github.com/DataManagementLab/deepdb-public)
