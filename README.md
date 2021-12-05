# Arquitectura

![Diagrama](./docs/arquitectura.png?style=centerme)

- Los **Proveedores** envian transacciones y eventos a Kafka, en donde se apilan para su posterior consumo.  Por motivos de simplicidad 
  se va a generar un tráfico de transacciones random.
- El **Kafka de Transacciones** recibe eventos y transacciones apilándolos para su posterior consumo (como especificado en el paso anterior).
- El **Detector de Anomalías** consume las transacciones y eventos almacenados en el Kafka azul (transacciones o eventos), y luego ejecuta 
  el modelo de predicción de anomalías (previamente entrenado). Si este módulo detecta anomalía, luego
  almacena esta transacción o evento en el Kafka rojo (anomalías).
- El **Kafka de Anomalías** recibe eventos y transacciones anómalos apilándolos para su posterior consumo.
- Los **Consumidores** no está incluido en esta prueba de concepto, pero potencialmente consumiría el Kakfa de anomalías
para realizar algún post procesamiento de acuerdo a las anomalías detectadas.

_Nota: Las Transacciones podrían representar cualquier tipo de información reelevante a analizar en tiempo real y 
predecir. Estas transacciones pueden darse en forma de flujos de paquetes de datos en la red, logs de una aplicación web,
metricas de consumo del sistema, hashes de archivos descargados, etc._

### Empezando...


- Paso 1: [Descagar e Instalar Python](https://www.python.org/downloads/release/python-397/)
- Paso 2: [Descagar e Instalar Docker Hub](https://docs.docker.com/docker-hub/)
- Paso 3: Clonar el proyecto

```commandline  
git clone https://github.com/ignaciomgu/tp-comando-control.git

cd tp-comando-control/
```

- Paso 4: Instalar requerimientos

```bash 
pip install -r requirements.txt
```

- Paso 5: Zookeeper/Kafka con Prometheus/Grafana

Iniciar/Parar con:

```commandline 
$ docker-compose -f  zk-kafka-single-node-stack.yml up -d
$ docker-compose -f  zk-kafka-single-node-stack.yml down
```

Crear `transactions` y `anomalies` topics con 3 partitions and 1 répica.

Tópico de transacciones:

```commandline 
$ docker exec -it kafka101 \
kafka-topics \
--create \
--partitions 3 \
--replication-factor 1 \
--topic transactions \
--bootstrap-server kafka101:29092
```

Tópico de anomalías:

```commandline 
$ docker exec -it kafka101 \
kafka-topics \
--create \
--partitions 3 \
--replication-factor 1 \
--topic anomalies \
--bootstrap-server kafka101:29092
```

=== Accediendo Grafana Web UI

Grafana escucha en : http://localhost:3000

Login :

* user : `admin`
* password : `kafka`

=== Accediendo Prometheus Web UI

Prometheus escucha en : http://localhost:9090

### Entrenar Modelo

A modo ilustrativo, vamos a generar datos random con dos variables:

![Plot de Isolation Forest](./docs/isolation-forest-plot.png?style=centerme)

Como vemos, utilizaremos el modelo `Isolation Forest` para detectar outliers. Por lo que este modelo va a tratar de 
aislar los puntos de datos trazando líneas aleatorias sobre uno de los ejes de variables (muestreandolos) 
y luego de varias iteraciones, va a medir qué tan "difícil" fue aislar cada observación, 
por lo que en el archivo `standardizer/train.py` tenemos:

```python
from joblib import dump
import numpy as np
from sklearn.ensemble import IsolationForest

rng = np.random.RandomState(42)
X = 0.3 * rng.randn(500, 2)
X_train = np.r_[X + 2, X - 2]
X_train = np.round(X_train, 3)

clf = IsolationForest(n_estimators=50, max_samples=500, random_state=rng, contamination=0.01)
clf.fit(X_train)

dump(clf, './isolation_forest.joblib')
```

Luego de ejecutar este archivo, se creará otro en `standardizer/isolation_forest.joblib`. Y este será
el modelo entrenado (a utilizar más adelante).
A modo de ejemplo, aquí se utiliza `Isolation Forest` con un `dataset` "random", pero acá se podrían tener 
potencialmente varios modelos entrenados con distintos `datasets` y distintos `algoritmos`.

### Proveedor de Transacciones o Eventos

Vamos a ejecutar el proveedor de transacciones que va a enviar eventos al `Transactions Kafka Topic`. Por lo que 
en nuestro `realtime/transactions_provider.py` tenemos:

``` python
import json
import random
import time
from datetime import datetime
import numpy as np
from settings import TRANSACTIONS_TOPIC, DELAY, OUTLIERS_GENERATION_PROBABILITY
from realtime.helper import create_producer

_id = 0
producer = create_producer()

if producer is not None:
    while True:
        if random.random() <= OUTLIERS_GENERATION_PROBABILITY:
            X_test = np.random.uniform(low=-4, high=4, size=(1, 2)).tolist()
        else:
            X = 0.3 * np.random.randn(1, 2)
            X_test = (X + np.random.choice(a=[2, -2], size=1, p=[0.5, 0.5]))
            X_test = np.round(X_test, 3).tolist()

        current_time = datetime.utcnow().isoformat()

        record = {"id": _id, "data": X_test, "current_time": current_time}
        record = json.dumps(record).encode("utf-8")

        producer.produce(topic=TRANSACTIONS_TOPIC,
                         value=record,
                         key=str(_id))
        producer.flush()
        _id += 1
        time.sleep(DELAY)
```

En este bloque de código, el `Producer` es instanciado para enviar datos al respectivo Kafka topic, con una probabilidad de `OUTLIERS_GENERATION_PROBABILITY`,
con un `ID` incremental, la `Data` requerida por el módulo detector de anomalías y la `fecha` en UTC.

Entonces, una vez tenemos el `Kafka` corriendo con el `topic` `transactions`, ejecutamos la consola para ver los logs:

```commandline  
docker exec -it kafka101 kafka-console-consumer --bootstrap-server kafka101:29092 --topic transactions
```

Vamos a ejecutar el proveedor de transacciones:

```commandline 
realtime/transactions_provider.py
```

Luego de esto, veremos los logs de los eventos recibidos en consola (en tiempo real):

![Transactions logs](./docs/transactionsDemo.gif)

### Detección de Anomalías

Los eventos ya están cayendo, ahora tenemos que consumir los, pasarlos por nuestro modelo de predicción y filtrar los
outliers (anomalías).

Entonces, nuestro detector de anomalías (`realtime/anomalies_detector.py`) va a contener lo siguiente:

``` python
import json
import os
from joblib import load
import logging
from multiprocessing import Process
import numpy as np
from realtime.helper import create_producer, create_consumer
from settings import (TRANSACTIONS_TOPIC, TRANSACTIONS_CONSUMER_GROUP, 
                      ANOMALIES_TOPIC, NUM_PARTITIONS)

model_path = os.path.abspath('../standardizer/isolation_forest.joblib')
clf = load(model_path)


def detect():
    consumer = create_consumer(topic=TRANSACTIONS_TOPIC, 
                               group_id=TRANSACTIONS_CONSUMER_GROUP)

    producer = create_producer()

    while True:
        message = consumer.poll(timeout=50))
        if message is None:
            continue
        if message.error():
            logging.error("Consumer error: {}".format(message.error()))
            continue

        # Message that came from producer
        record = json.loads(message.value().decode('utf-8'))
        data = record["data"]

        prediction = clf.predict(data)

        if prediction[0] == -1:
            score = clf.score_samples(data)
            record["score"] = np.round(score, 3).tolist()

            _id = str(record["id"])
            record = json.dumps(record).encode("utf-8")

            producer.produce(topic=ANOMALIES_TOPIC,
                             value=record,
                             key=_id)
            producer.flush()

    consumer.close()

if __name__ == '__main__':
    for _ in range(NUM_PARTITIONS):
        p = Process(target=detect)
        p.start()
```

Como vemos en el código, un `Consumer` es instanciado para leer las transacciones del `Transactions Topic` de Kafka 
y enviar las transacciones o eventos detectados como anómalos (outliers) hacia el `Anomalies Topic` de Kafka.
Además de los datos que ya teníamos, enriquecerá el registro con el score dado por el modelo de predicción, una medida 
de "cuanto más" de datos se considerada un outlier. 
Hay que tener en cuenta que los únicos mensajes que son enviados al `Anomalies Topic` son aquellos cuya salida de predicción 
es igual a `-1`, asi es como este modelo de machine learning categoriza los datos que son outliers.
En el código también se observa que los `Transactions Topic` tiene 3 particione, lo que significa que en esta ocasión se está
utilizando multiprocesamiento para simular 3 consumidores de transacciones independientes y acelerar el proceso.
En entorno de producción, seguramente esos consumidores van a estar corriendo en distintos servidores.

Entonces, una vez tenemos el `Kafka` corriendo con el `topic` `anomalies`, ejecutamos la consola para ver los logs:

```commandline  
docker exec -it kafka101 kafka-console-consumer --bootstrap-server kafka101:29092 --topic anomalies
```

Luego ejecutamos el detector de anomalías que va a consumir `Transactions Topic`, ejecutar el modelo de predicción y enviar 
los registros anómalos a `Anomalies Topic`:

```commandline  
realtime/anomalies_detector.py
```

Finalmente por la consola del Kafka `Anomalies Topic` se deberían ver apareciendo las anomalías detectadas:

![Anomalies logs](./docs/anomaliesDemo.gif)

Acá dejo una visualización de como el `Proveedor de Transacciones` y el `Detector de Anomalias` corren al mismo tiempo.
El de arriba es la consola de logs del `Transactions Topic` (eventos o transacciones reportadas)
y el de abajo el de `Anomalies Topic` (anomalías detectadas por el modelo de machine learning):

![Full Demo logs](./docs/demo.gif)

### Una forma más abstracta de verlo

![Final Screen](./docs/finalScreen.png?style=centerme)

### Visualización de la actividad de Kafka en grafana

![Grafana](./docs/grafana.png?style=centerme)
