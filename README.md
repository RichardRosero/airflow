# 🚀 Laboratorio ETL: Inventario NAPS con Apache Airflow

Este proyecto implementa un pipeline de datos (DAG) modular en Apache Airflow para extraer datos de una base de datos PostgreSQL, transformar campos específicos y generar reportes en formato CSV.

---

## 📋 1. Guía de Instalación y Configuración

Sigue estos pasos en tu terminal de Ubuntu para replicar el entorno del laboratorio.
### instalación el instalador de paquetes global de python "pip" y el entorno con que virtualizaremos "venv"
```sudo apt install -y python3-pip python3-venv```
```
python --version
pip --version
```
### Preparación del entorno

```mkdir ~/airflow_lab
cd ~/airflow_lab
python3 -m venv lab_env
source lab_env/bin/activate
```

### Instalación del software
```
pip install "apache-airflow[postgres]==2.10.3" pandas --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.10.3/constraints-3.12.txt"
```

### Configuración inicial
```export AIRFLOW_HOME=~/airflow_lab
airflow db init
airflow db reset -y
airflow users create --username admin --firstname Richard --lastname Rosero --role Admin --email roseror@hitss.com --password admin
```

### Ejecución de Servicios (Abrir dos terminales)

Terminal 1 (Webserver):
```
source ~/airflow_lab/lab_env/bin/activate
export AIRFLOW_HOME=~/airflow_lab
airflow webserver --port 8080
```

Terminal 2 (Scheduler):
```
source ~/airflow_lab/lab_env/bin/activate
export AIRFLOW_HOME=~/airflow_lab
airflow scheduler
```

---

## 📂 2. Estructura de Carpetas y Conexión

1. Crear carpetas físicas:
```
mkdir ~/airflow_lab/dags
mkdir -p /home/roseror/reportes_csv
```

3. Configurar Conexión en UI (Admin > Connections):
   - Conn Id: postgres_db
   - Conn Type: Postgres
   - Host: 10.32.112.57

---

## 🛠️ 3. Código del DAG (~/airflow_lab/dags/dag_naps_final.py)
```
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from datetime import datetime, timedelta
import csv

default_args = {
    'owner': 'Richard_Rosero',
    'start_date': datetime(2025, 1, 1),
    'retries': 1,
}

def conectar_postgres():
    hook = PostgresHook(postgres_conn_id='postgres_db')
    hook.get_conn().close()
    print("Conexión exitosa.")

def extraer_datos(**kwargs):
    query = """
    SELECT t.hub, t.cluster, t.olt, t.frame, t.slot, t.puerto, 
           t.nap, t.puertos_nap, t.coordenadas, t.fecha_de_liberacion, 
           t.region, t.zona, t.latitud, t.longitud
    FROM public.inv_naps t
    WHERE cluster = 'G6C062'
    ORDER BY t.slot ASC, t.frame ASC, t.puerto ASC
    """
    hook = PostgresHook(postgres_conn_id='postgres_db')
    return hook.get_records(query)

def transformar_hub(**kwargs):
    ti = kwargs['ti']
    datos_crudos = ti.xcom_pull(task_ids='tarea_extraer')
    datos_transformados = []
    for fila in datos_crudos:
        fila_lista = list(fila)
        fila_lista[0] = str(fila_lista[0]).lower()
        datos_transformados.append(fila_lista)
    return datos_transformados

def guardar_a_csv(**kwargs):
    ti = kwargs['ti']
    datos_finales = ti.xcom_pull(task_ids='tarea_transformar')
    path = '/home/roseror/reportes_csv/resultado_naps.csv'
    with open(path, 'w', newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerows(datos_finales)
    print(f"Archivo guardado en: {path}")

with DAG(
    'laboratorio_naps_final',
    default_args=default_args,
    schedule_interval=None,
    catchup=False,
) as dag:

    t1 = PythonOperator(task_id='tarea_conectar', python_callable=conectar_postgres)
    t2 = PythonOperator(task_id='tarea_extraer', python_callable=extraer_datos)
    t3 = PythonOperator(task_id='tarea_transformar', python_callable=transformar_hub)
    t4 = PythonOperator(task_id='tarea_cargar', python_callable=guardar_a_csv)

    t1 >> t2 >> t3 >> t4
```
---

## 🔍 4. Verificación y Debugging
- Ver errores en CLI: ```airflow dags list-import-errors```
- Ver resultado físico: ```cat /home/roseror/reportes_csv/resultado_naps.csv```
