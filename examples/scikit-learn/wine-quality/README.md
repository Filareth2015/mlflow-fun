# mlflow-fun - scikit-learn - Wine Quality Example

## Overview
* Wine Quality Elastic Net Example
* This example demonstrates all features of MLflow training and prediction.
* Saves model in pickle format
* Saves plot artifacts
* Shows several ways to run training - _mlflow run_, run against Databricks cluster, call egg from notebook, etc.
* Shows several ways to run prediction  - web server,  mlflow.load_model(), UDF, etc.

## Setup

```
pip install mlflow
pip install cloudpickle
pip install matplotlib
pip install pyarrow # for Spark UDF example
```

## Training

Source: [main_train_wine_quality.py](main_train_wine_quality.py) and [train_wine_quality.py](wine_quality/train_wine_quality.py).

### Standard Python Main Run

To run with standard main function:
```
python main_train_wine_quality.py 0.5 0.5 wine-quality.csv
```

### Project Runs

These runs use the [MLproject](MLproject) file. For more details see [MLflow documentation - Running Projects](https://mlflow.org/docs/latest/projects.html#running-projects).

**mlflow local**
```
mlflow run . -P alpha=0.01 -P l1_ratio=0.75 -P run_origin=LocalRun
```

**mlflow github**
```
mlflow run https://github.com/amesar/mlflow-fun.git#examples/scikit-learn/wine-quality \
  -P alpha=0.01 -P l1_ratio=0.75 -P run_origin=GitRun
```

**mlflow Databricks remote** - Run against Databricks. 

See [Remote Execution on Databricks](https://mlflow.org/docs/latest/projects.html#remote-execution-on-databricks) and [mlflow_run_cluster.json](mlflow_run_cluster.json).

Setup.
```
export MLFLOW_TRACKING_URI=databricks
```
The token and tracking server URL will be picked up from your Databricks CLI ~/.databrickscfg default profile.

Now run.
```
mlflow run https://github.com/amesar/mlflow-fun.git#examples/scikit-learn/wine-quality \
  -P alpha=0.01 -P l1_ratio=0.75 -P run_origin=GitRun \
  -P data_path=/dbfs/tmp/data/wine-quality.csv \
  --experiment-id=2019 \
  --mode databricks --cluster-spec mlflow_run_cluster.json
```

### Databricks Cluster Runs

You can also package your code as an egg and run it with the standard Databricks REST API endpoints
[job/runs/submit](https://docs.databricks.com/api/latest/jobs.html#runs-submit) 
or [jobs/run-now](https://docs.databricks.com/api/latest/jobs.html#run-now) 
using the [spark_python_task](https://docs.databricks.com/api/latest/jobs.html#jobssparkpythontask). 

#### Setup

Build the egg.
```
python setup.py bdist_egg
```

Upload the data file, main file and egg to your Databricks cluster.
```
databricks fs cp main_train_wine_quality.py dbfs:/tmp/jobs/wine_quality/main.py
databricks fs cp wine-quality.csv dbfs:/tmp/jobs/wine_quality/wine-quality.csv
databricks fs cp \
  dist/mlflow_wine_quality-0.0.1-py3.6.egg \
  dbfs:/tmp/jobs/wine_quality/mlflow_wine_quality-0.0.1-py3.6.egg
```


#### Run Submit

##### Run with new cluster

Define your run in [run_submit_new_cluster.json](run_submit_new_cluster.json) and launch the run.

```
curl -X POST -H "Authorization: Bearer MY_TOKEN" \
  -d @run_submit_new_cluster.json  \
  https://myshard.cloud.databricks.com/api/2.0/jobs/runs/submit
```

##### Run with existing cluster

Every time you build a new egg, you need to upload (as described above) it to DBFS and restart the cluster.
```
databricks clusters restart --cluster-id 1222-015510-grams64
```

Define your run in [run_submit_existing_cluster.json](run_submit_existing_cluster.json) and launch the run.
```
curl -X POST -H "Authorization: Bearer MY_TOKEN" \
  -d @run_submit_existing_cluster.json  \
  https://myshard.cloud.databricks.com/api/2.0/jobs/runs/submit
```

#### Job Run Now

##### Run with new cluster

First create a job with the spec file [create_job_new_cluster.json](create_job_new_cluster.json). 
```
databricks jobs create --json-file create_job_new_cluster.json
```

Then run the job with desired parameters.
```
databricks jobs run-now --job-id $JOB_ID --python-params ' [ 0.3, 0.3, "/dbfs/tmp/jobs/wine_quality/wine-quality.csv" ] '
```

##### Run with existing cluster
First create a job with the spec file [create_job_existing_cluster.json](create_job_existing_cluster.json).
```
databricks jobs create --json-file create_job_existing_cluster.json
```

Then run the job with desired parameters.
```
databricks jobs run-now --job-id $JOB_ID --python-params ' [ 0.3, 0.3, "/dbfs/tmp/jobs/wine_quality/wine-quality.csv" ] '
```


#### Run egg from Databricks notebook

Create a notebook with the following cell. Attach it to the existing cluster described above.
```
from wine_quality import train_wine_quality
data_path = "/dbfs/tmp/jobs/wine_quality/wine-quality.csv"
train_wine_quality.train(data_path, 0.4, 0.4, "from_notebook_with_egg")
```

## Predictions

You can make predictions in the following ways:
1. Use MLflow's serving web server and submit predictions via HTTP calls
2. Call mlflow.sklearn.load_model() from your own serving code and then make predictions
4. Call mlflow.pyfunc.load_pyfunc() from your own serving code and then make predictions
5. Batch prediction with Spark UDF (user-defined function)


See MLflow documentation:
* [Tutorial - Serving the Model](https://www.mlflow.org/docs/latest/tutorial.html#serving-the-model)
* [Quickstart - Saving and Serving Models](https://www.mlflow.org/docs/latest/quickstart.html#saving-and-serving-models)
* [mlflow.pyfunc.spark_udf](https://www.mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#mlflow.pyfunc.spark_udf)


### Data for predictions
[wine-quality.json](wine-quality.json):
```
[
  {
    "fixed acidity": 7,
    "volatile acidity": 0.27,
    "citric acid": 0.36,
    "residual sugar": 20.7,
    "chlorides": 0.045,
    "free sulfur dioxide": 45,
    "total sulfur dioxide": 170,
    "density": 1.001,
    "pH": 3,
    "sulphates": 0.45,
    "alcohol": 8.8
  }, 
  . . . . .
]
```

### 1. Serving Models from MLflow Web Server

In one window run the server.
```
mlflow pyfunc serve -p 5001 -r 7e674524514846799310c41f10d6b99d -m model
```

In another window, submit a prediction.
```
curl -X POST -H "Content-Type:application/json" -d @wine-quality.json http://localhost:5001/invocations

[
    5.551096337521979,
    5.297727513113797,
    5.427572126267637,
    5.562886443251915,
    5.562886443251915
]
```

### 2. Predict with mlflow.sklearn.load_model()

```
python scikit_predict.py 7e674524514846799310c41f10d6b99d

predictions: [5.55109634 5.29772751 5.42757213 5.56288644 5.56288644]
```
From [scikit_predict.py](scikit_predict.py):
```
model = mlflow.sklearn.load_model("model",run_id="7e674524514846799310c41f10d6b99d")
df = pd.read_json("wine-quality.json")
predicted = model.predict(df)
print("predicted:",predicted)
```

### 3. Predict with mlflow.pyfunc.load_pyfunc()

```
python pyfunc_predict.py 7e674524514846799310c41f10d6b99d

predictions: [5.55109634 5.29772751 5.42757213 5.56288644 5.56288644]
```
From [pyfunc_predict.py](pyfunc_predict.py):
```
model_uri = mlflow.start_run("7e674524514846799310c41f10d6b99d").info.artifact_uri +  "/model"
model = mlflow.pyfunc.load_pyfunc(model_uri)
df = pd.read_json("wine-quality.json")
predicted = model.predict(df)
print("predicted:",predicted)
```

### 4. Batch prediction with Spark UDF (user-defined function)

Scroll right to see prediction column.

```
pip install pyarrow

spark-submit --master local[2] spark_udf_predict.py 7e674524514846799310c41f10d6b99d

+-------+---------+-----------+-------+-------------+-------------------+----+--------------+---------+--------------------+----------------+------------------+
|alcohol|chlorides|citric acid|density|fixed acidity|free sulfur dioxide|  pH|residual sugar|sulphates|total sulfur dioxide|volatile acidity|        prediction|
+-------+---------+-----------+-------+-------------+-------------------+----+--------------+---------+--------------------+----------------+------------------+
|    8.8|    0.045|       0.36|  1.001|          7.0|               45.0| 3.0|          20.7|     0.45|               170.0|            0.27| 5.551096337521979|
|    9.5|    0.049|       0.34|  0.994|          6.3|               14.0| 3.3|           1.6|     0.49|               132.0|             0.3| 5.297727513113797|
|   10.1|     0.05|        0.4| 0.9951|          8.1|               30.0|3.26|           6.9|     0.44|                97.0|            0.28| 5.427572126267637|
|    9.9|    0.058|       0.32| 0.9956|          7.2|               47.0|3.19|           8.5|      0.4|               186.0|            0.23| 5.562886443251915|
```
From [spark_udf_predict.py](spark_udf_predict.py):
```
spark = SparkSession.builder.appName("ServePredictions").getOrCreate()
df = spark.read.option("inferSchema",True).option("header", True).csv("wine-quality.csv")
df = df.drop("quality")

udf = mlflow.pyfunc.spark_udf(spark, "model", run_id="7e674524514846799310c41f10d6b99d")
df2 = df.withColumn("prediction", udf(*df.columns))
df2.show(10)
```

### 5. Unpickle model artifact file without MLflow and predict
You can directly read the model pickle file and then make predictions.
From [pickle_predict.py](pickle_predict.py):
```
pickle_path = "/opt/mlflow/mlruns/3/11df004981b443908d9286d54d24dc27/artifacts/model/model.pkl"
with open(pickle_path, 'rb') as f:
    model = pickle.load(f)
df = pd.read_json("wine-quality.json")
predicted = model.predict(df)
print("predicted:",predicted)
```
