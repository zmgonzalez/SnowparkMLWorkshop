#NOTE: This demo references features which have not yet been announced and should be treated as internal only.  Contact michael.gregory@astronomer.io for details and updates.


SnowML is a new python framework for Machine Learning workloads with [Snowpark](https://docs.snowflake.com/en/developer-guide/snowpark/python/index.html).  Currently SnowML provides a model registry (storing ML tracking data and models in Snowflake tables and stages) as well as feature engineering primitives similar to scikit-learn (ie. LabelEncoder, OneHotEncoder, etc.)

This guide demonstrates how to use Apache Airflow to orchestrate a machine learning pipeline leveraging SnowML for feature engineering and model tracking.

Additionally, this demo uses the Docker frontend buildkit from Astronomer for building virtual environments for the Snowpark and SnowML.

## Prerequisites  
  
- Astro [CLI](https://docs.astronomer.io/astro/cli/get-started)
- Docker Desktop
- Git
- Snowflake account (or a [trial account](https://signup.snowflake.com/))
  
## ExternalPythonOperator for Snowpark  
  
As of this demo's creation Snowpark Python is supported only on Python 3.8.  While Apache Airflow is supported on Python 3.8 it will often be the case that managed Airflow services, such as Astronomer, will use the most up-to-date versions of Apache Airflow and Python.  As explained in the [ExternalPythonOperator tutorial](https://github.com/astronomer/docs/blob/pythonvirtualenv-tutorial/learn/external-python-operator.md) it is possible to run Snowpark Python tasks using the ExternalPythonOperator.  However, this leaves plenty of heavy lifting to manage multiple versions of python on an Airflow Docker image.  The Dockerfile in this repository shows examples of using 1) [miniconda](https://docs.conda.io/en/latest/miniconda.html), 2) [buildx](https://github.com/docker/buildx) image linking and 3) [pyenv](https://github.com/pyenv/pyenv-installer) to create the external python instance for the ExternalPythonOperator.  All of these options leave much to be desired in terms of: 
  
- Complexity: Dockerfiles quickly become lengthy, messy and difficult to maintain.
- Time: Developers spend lots of unproductive time watching Docker image building.
- Updates: During development it is often necessary to restart docker services or change settings. The options above do not support `cacheing` which means each restart requires the same lengthy build delay unless the developer manually caches images.
  
## Docker buildkit... with ❤️ from Astronomer
  
Astronomer has created a custom Docker frontend buildkit to drastically simplify this build process to only a couple of lines in the Dockerfile.  

```Dockerfile
# syntax=quay.io/astronomer/airflow-extensions:v1

FROM quay.io/astronomer/astro-runtime:7.4.1-base

PYENV 3.8 snowpark_env snowpark-requirements.txt
```

## Setup  
  
1. Clone this repository:
```bash
git clone https://github.com/astronomer/airflow-snowml-demo
cd airflow-snowml-demo
```
2. Edit airflow_settings.yaml and create a new connection called **'snowflake_default'** with your Snowflake account details.  Example:  
    ```text
    connections:
        - conn_id: snowflake_default
        conn_type: snowflake
        conn_schema: <schema_name>
        conn_login: <user_login_name>
        conn_password: <password>
        conn_port:
        conn_extra: {"extra__snowflake__account": "<account_id>", 
                    "extra__snowflake__warehouse": "<warehouse_name>", 
                    "extra__snowflake__database": "<database_name>", 
                    "extra__snowflake__region": "<region_name>", 
                    "extra__snowflake__role": "<user_role_name>", 
                    "extra__snowflake__insecure_mode": false}
    ```  
    The airflow_settings.yaml file listed in .gitignore so this should not be copied to git.  Alternatively you can enter this information in the Airflow UI after starting the Astronomer dev runtime. 
3.  Start Apache Airflow:
    ```sh
    astro dev start
    ```  

4. Create model_registry schema.  SnowML requires a database and schema to store model metadata.  The `check_registry` task of the demo DAG assumes that it has been setup already.  For example, the following can be run in python with Snowflake user credentials which have `sysadmin` role to create the Snowflake objects:
```python
from airflow.providers.snowflake.hooks.snowflake import SnowflakeHook
from snowflake.ml.registry import model_registry
from snowflake.snowpark import Session

hook = SnowflakeHook(
    account='account', 
    password='password',
    region='region',
    login='login',
    warehouse='warehouse',
    role='sysadmin',
)
snowpark_session = Session.builder.configs(hook._get_conn_params()).create()
_ = model_registry.create_model_registry(session=snowpark_session, database_name='MODEL_REGISTRY')

user_role_name = <your_user_role_name>

hook.run(f'''
    GRANT USAGE ON DATABASE MODEL_REGISTRY TO ROLE {user_role_name};
    GRANT USAGE, CREATE TABLE ON SCHEMA MODEL_REGISTRY.PUBLIC TO ROLE {user_role_name};
    GRANT SELECT, INSERT ON TABLE MODEL_REGISTRY.PUBLIC.METADATA TO ROLE {user_role_name};
    GRANT SELECT, INSERT ON TABLE MODEL_REGISTRY.PUBLIC.MODELS TO ROLE {user_role_name};
    GRANT SELECT ON VIEW MODEL_REGISTRY.PUBLIC.METADATA_LAST_DESCRIPTION TO ROLE {user_role_name};
    GRANT SELECT ON VIEW MODEL_REGISTRY.PUBLIC.METADATA_LAST_METRICS TO ROLE {user_role_name};
    GRANT SELECT ON VIEW MODEL_REGISTRY.PUBLIC.METADATA_LAST_NAME TO ROLE {user_role_name};
    GRANT SELECT ON VIEW MODEL_REGISTRY.PUBLIC.METADATA_LAST_REGISTRATION TO ROLE {user_role_name};
    GRANT SELECT ON VIEW MODEL_REGISTRY.PUBLIC.METADATA_LAST_TAGS TO ROLE {user_role_name};
    GRANT SELECT ON VIEW MODEL_REGISTRY.PUBLIC.MODELS_VIEW TO ROLE {user_role_name};
    GRANT CREATE STAGE ON SCHEMA MODEL_REGISTRY.PUBLIC TO ROLE {user_role_name};
''')

snowpark_session.close()
  
```

5. Connect to the Local Airflow instance UI at (http://localhost:8080) and login with **Admin/Admin**  
    If you did not add the Snowflake connection to airflow_settings.yaml add it now in the Admin -> Connections menu.  