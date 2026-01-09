# 🚀 Laboratorio ETL: Inventario NAPS con Apache Airflow  ubuntu: roseror contraseña: 123456

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

```
mkdir ~/airflow_lab
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
   - Host: 100.132.112.157

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


## 5 configuracion servidor smtp airflow
```
   sudo bash -c "$(curl -sL https://raw.githubusercontent.com/axllent/mailpit/develop/install.sh)"
   mailpit
   mailpit --smtp-auth-allow-insecure --listen 0.0.0.0:8025 --smtp 0.0.0.0:1025
   
```
   
# El servidor emisor que creaste (Mailpit)
   ```
   nano ~/airflow_lab/airflow.cfg
   ```
   ```
   smtp_host = 192.168.31.128
   smtp_starttls = False
   smtp_ssl = False
   smtp_user = 
   smtp_password = 
   smtp_port = 1025
   smtp_mail_from = airflow@example.com
```
### nota

Organización por Subcarpetas
No estás obligado a tener 50 archivos sueltos en una misma carpeta. Airflow escanea de forma recursiva. Puedes organizar así:
```
~/airflow_lab/dags/proyecto_naps/dag_final.py

~/airflow_lab/dags/proyecto_ventas/dag_ventas.py

~/airflow_lab/dags/utilitarios/limpieza.py
```

### si deseas agregar un crontab en el dag:
```
with DAG(
    'laboratorio_naps_final_fallar_mail',
    default_args=default_args,
    # Aquí agregas el Crontab entre comillas
    schedule_interval='0 5 * * *', 
    catchup=False,
) as dag:



Frecuencia,Expresión Cron,Descripción
Cada hora,'0 * * * *',Se ejecuta al minuto 0 de cada hora.
Diario (Medianoche),'0 0 * * *',Se ejecuta una vez al día a las 00:00.
Cada Lunes,'0 0 * * 1',Se ejecuta el primer día de la semana.
Días laborables,'0 9 * * 1-5',De lunes a viernes a las 9:00 AM.
```

### ver en powerautomate las notificaciones
https://make.powerautomate.com/ aqui vemos los logs en caso mandemos mal la estructura del mensaje
se debe crear un canal

def notify_teams(context, success=False):
    dag_id = context.get('task_instance').dag_id
    task_id = context.get('task_instance').task_id
    log_url = context.get('task_instance').log_url
    
    title = "✅ DAG EXITOSO" if success else "🚨 ALERTA DE FALLA"
    color = "good" if success else "attention"
    msg = f"El DAG **{dag_id}** finalizó." if success else f"La tarea **{task_id}** falló en el DAG **{dag_id}**."

    # Estructura de Adaptive Card que espera tu plantilla de Power Automate
    teams_payload = {
        "type": "message",
        "attachments": [{
            "contentType": "application/vnd.microsoft.card.adaptive",
            "content": {
                "type": "AdaptiveCard",
                "body": [
                    {"type": "TextBlock", "text": title, "weight": "Bolder", "size": "Large", "color": color},
                    {"type": "TextBlock", "text": msg, "wrap": True}
                ],
                "actions": [
                    {"type": "Action.OpenUrl", "title": "Ver Logs", "url": log_url}
                ],
                "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
                "version": "1.4"
            }
        }]
    }
    
    hook = HttpHook(http_conn_id='msteams_webhook', method='POST')
    hook.run(endpoint='', data=json.dumps(teams_payload), headers={"Content-Type": "application/json"})

# Wrappers para los callbacks
def on_failure(context):
    notify_teams(context, success=False)

def on_success(context):
    notify_teams(context, success=True)

# --- CONFIGURACIÓN DEL DAG ---
default_args = {
    'owner': 'Richard_Rosero',
    'start_date': datetime(2025, 1, 1),
    'retries': 0, # Configurado en 0 para aviso inmediato según tu solicitud
    'on_failure_callback': on_failure, # Se activa al primer fallo
}
