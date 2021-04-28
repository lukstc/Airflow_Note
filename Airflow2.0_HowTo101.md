# Airflow2.0_HowTo_101

## Intro to Airflow

- **What is Airflow - 20min**
- **New feature in Airflow2.0 - 10min**
- **Airflow Web UI and platform: 10min**
- **ETL and ipdb demo - 20min**
- **Practice - 20min** 



## Platform

### UI

- DAG
- Security - grant user permission/access
- Browse
- Admin
  - Connection
  - Variable

### Tools

- Jupyter Notebook: dag/code update
- TTYD: for debug



## Preparation

- intall airflow & ipdb
- Change to Admin role (any current admin user could do that)

## Compile Your First DAG

- **desgin your workflow**
  
  - e.g: ETL
    - Step#1: fetch data
    - Step#2: do something
    - Step#3: load the result to db
- **compile dag file:**
  
  - set up connection / choose the right connection
- **test your dag:**
  
  - Unit test: do this part when you compiling your code
  - DAG Syntax check >> run: ```python [dag_file_name.py]```
- **deploy and run your dag:**
  
  - Upload your code via Jupyter Notebook
  - make sure your DAG compiled from the latest code, current scan cycle: **60s**
- **check result & debug**

  - **python syntax check >> DAG import errors >> task Log >> ipdb**

  - **python syntax check**: `python DAG_example_ipdb_demo.py`

  - **DAG import erros:** error found during DAG rendering

  - **task Log**: Suggestion: use Airflow task log to check each steps' middle output

  - **IPDB Airflow test** [ref link](https://airflow.apache.org/docs/apache-airflow/stable/tutorial.html#id2)

    - Ipdb: interactive debug

    - modified your task code: `ipdb.set_trace()`

    - **Note: do not set ipdb breakpoint at DAG level**

    - Test your dag with specific run code syntax: 

      ```python
      # command layout: command subcommand dag_id task_id date
      
      # testing print_date
      airflow tasks test tutorial print_date 2015-06-01
      
      # testing sleep
      airflow tasks test tutorial sleep 2015-06-01
      
      # testing templated
      airflow tasks test tutorial templated 2015-06-01
      ```

    - e.g.`airflow tasks test example_ipdb_demo push_data 2021-03-11T06:17:02.486315+00:00`

    - airflow tasks test example_ipdb_demo push_data 2021-03-12T06:19:32.121874+00:00

## Note + Q & A

- **import your own packages/module**
  - save your `python_file_name.py` at dir >> **ariflow/packages**
  - Sample import code: `import python_file_name `
- **import your data files:**
  - save your data file e.g. `xxx.json` or `xxx.csv` at dir >> **ariflow/data**
  - declear import path `/opt/airflow/dags/data/`
- **File/Project Management:**
  - create folder, naming accordingly, one project one folder
  - e.g. `Examples`
- **Naming Convention**
  - DAG py file name: **DAG\_[project/purpose]\_[function/usage]\_[Version]**
  - DAG name: **[project/purpose]\_[function/usage]_[Version]**
    - e.g. **ci_mysql_trialtrove_update_v1**
    - e.g. **example_dag_decorator_v2**
  - DAG tag: + project name/purpose

## Practice

- **official example dags: [link](https://github.com/apache/airflow/tree/master/airflow/example_dags)**
- **Tasks**
  - **Task#A1**: get current date and time and push result to **Task#A2**
  - **Task#A2**: get result from **Task#A1** 
  - **Task#B1**: get some data from a databse and push to **Task#B2**
  - **Task#B2**: pull data from  **Task#B2** print first 5 rows of the data  in log
  - **Task#C**: print all task done in log
- **Work Flow/Structure: **
  - **A1 >> A2 >> C**
  - **B1 >> B2 >> C**