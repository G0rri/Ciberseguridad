# Introducción

En esta práctica se ha desplegado un entorno virtualizado mediante
Docker que simula una infraestructura empresarial básica. El entorno
consta de un SIEM completo basado en la pila **ELK** (Elasticsearch,
Logstash y Kibana), un servidor víctima vulnerable y un nodo atacante.

## Explicación de la Arquitectura

El entorno de simulación está orquestado mediante **Docker Compose** y
se compone de cinco servicios aislados en una red puente interna:

- **Elasticsearch:** Actúa como el motor principal de indexación y base
  de datos donde se almacenan permanentemente los eventos de seguridad.

- **Logstash:** Es el procesador intermedio. Su función es recibir los
  eventos en bruto, formatearlos si fuera necesario, y enrutarlos hacia
  la base de datos creando el índice correspondiente.

- **Kibana:** Interfaz gráfica que consulta los datos de Elasticsearch
  para permitir la búsqueda, visualización y análisis del incidente.

- **Víctima:** Un contenedor basado en Ubuntu que expone el puerto 2222
  simulando un servidor vulnerable. Tiene instalado un servidor SSH y un
  agente ligero de recolección de datos (Filebeat).

- **Atacante:** Un contenedor ligero diseñado exclusivamente para
  generar tráfico malicioso y ejecutar ataques de fuerza bruta contra el
  contenedor víctima.

**Flujo de Logs:** El mecanismo de ingesta y detección funciona a través
de las siguientes etapas:

1.  **Generación del Evento:** Cuando el contenedor atacante realiza
    intentos fallidos de conexión contra la víctima, el servicio sshd de
    la víctima está configurado (mediante el parámetro -E) para escribir
    directamente estos registros de error en el archivo físico
    /var/log/auth.log.

2.  **Recolección:** El agente Filebeat, que se ejecuta en segundo plano
    dentro del propio contenedor víctima, monitorea constantemente este
    archivo de logs. Al detectar nuevas líneas, las empaqueta y las
    envía a través del puerto interno 5044.

3.  **Procesamiento:** Logstash recibe los paquetes de Filebeat en el
    puerto 5044, los procesa y los inyecta en Elasticsearch bajo el
    patrón de índice ssh-logs-%{+YYYY.MM.dd}.

4.  **Visualización:** Kibana mapea este índice, permitiendo identificar
    de forma clara en su panel "Discover" el origen del ataque (la IP
    interna de Docker del atacante), la naturaleza del fallo y la marca
    temporal exacta de cada intento.

# Procedimiento

Antes de levantar nada, **Elasticsearch** requiere que el sistema
anfitrión tenga un límite mayor de memoria mapeada. Así que ejecutamos
esto:

<img width="548" height="92" alt="image" src="https://github.com/user-attachments/assets/efc58d9a-f0a8-4718-9d0c-d41d414c571d" />

Creamos una carpeta para el proyecto y dentro creamos dos archivos:
**docker-compose.yml** y **logstash.conf.**

<img width="567" height="87" alt="image" src="https://github.com/user-attachments/assets/f229e762-886e-4695-bb99-5984569ff18d" />

Dentro del directorio creado, definimos el núcleo del SIEM con un
archivo **Docker** **Compose**.

<img width="425" height="387" alt="image" src="https://github.com/user-attachments/assets/5f4b2905-1e13-4425-bc04-4f3cb5f78174" />

Logstash necesita saber qué hacer con los datos. Así que creamos un
archivo de configuración para **logstash** que escuche en el puerto
**5044** (donde se enviarán los logs posteriormente) y los mande
directamente a **Elasticsearch**.

<img width="565" height="205" alt="image" src="https://github.com/user-attachments/assets/0c88a304-c851-4a36-a89f-c8e07229186f" />

Ahora con las configuraciones anteriormente establecidas, pasamos a
construir el Docker Compose con este comando:

<img width="567" height="173" alt="image" src="https://github.com/user-attachments/assets/5da090c4-064f-46f1-a255-41907049b602" />

Dentro de la carpeta de antes creamos una subcarpeta para la víctima y
dentro creamos un Dockerfile.

<img width="567" height="77" alt="image" src="https://github.com/user-attachments/assets/236b2bab-7911-4dbb-87fb-f8aa4daaace3" />

Como los contenedores base de Ubuntu no traen sistema de logs por
defecto, este Dockerfile instala SSH, para que se genere en el archivo
**auth.log**, y **Filebeat**.

<img width="567" height="266" alt="image" src="https://github.com/user-attachments/assets/f1068277-226b-4144-829d-0b913fbc2da5" />

Ahora creamos otro archivo en el mismo directorio en el que estábamos
para que le diga a Filebeat qué leer y a dónde enviarlo.

<img width="452" height="197" alt="image" src="https://github.com/user-attachments/assets/a6d07480-e520-41fb-b6b0-d579d9c0bc1e" />

Ahora volvemos a la raíz de tu proyecto y modificamos el
docker-compose.yml para añadir el nuevo servicio de la víctima. Justo
debajo del servicio logstash:

<img width="467" height="374" alt="image" src="https://github.com/user-attachments/assets/f7346c11-2abd-45e5-8673-4b9abc25228f" />

Ahora volvemos a reconstruir el proyecto para incluir la víctima:

<img width="416" height="402" alt="image" src="https://github.com/user-attachments/assets/4c0cfaa5-5ceb-42e3-aeb6-68f3c7a516f2" />

En la carpeta principal del proyecto, crea una nueva carpeta llamada
atacante. Dentro, creamos un archivo llamado Dockerfile.

<img width="567" height="75" alt="image" src="https://github.com/user-attachments/assets/3d4f861d-5714-4659-aede-2096ab278d5b" />

Usaremos una imagen base muy ligera (Alpine Linux) a la que simplemente
le instalaremos el cliente SSH para poder lanzar conexiones contra la
víctima.

<img width="567" height="112" alt="image" src="https://github.com/user-attachments/assets/52bb34d0-60a3-43bd-8134-9b487badc674" />

Ahora volvemos al Docker Compose de la carpeta raíz para añadir la nueva
subcarpeta que contiene al atacante, tal y como hicimos antes con la
víctima.

<img width="567" height="440" alt="image" src="https://github.com/user-attachments/assets/f7d13215-88cd-468a-b13a-e67ecc366bee" />

Ahora vamos a aplicar los cambios borrando los contenedores anteriores
con -v por si acaso.

<img width="567" height="239" alt="image" src="https://github.com/user-attachments/assets/27e6e263-7693-4ef4-8b52-a4de268e45b5" />

Ejecutamos el ataque de fuerza bruta mediante un bucle automatizado en
bash. Este script realiza 10 intentos de conexión consecutivos por SSH
usando un usuario inexistente para forzar la generación de eventos de
error (Failed password) en la máquina víctima.

<img width="567" height="146" alt="image" src="https://github.com/user-attachments/assets/7b1ffe30-7e0d-4b6a-a049-6e34a79a5965" />

Comprobamos con curl que Elastisearch está cogiendo correctamente los
índices del ssh con el formateo correcto, para que kibana lo detecte.

<img width="567" height="43" alt="image" src="https://github.com/user-attachments/assets/5d22a2c3-3da2-4969-afa4-ac6d44159b85" />

Una vez hemos hecho todos los pasos, nos dirigimos a Kibana en el puerto
5601, y vamos a stack managment/Data views, y aquí dentro hay una opción
para crear un data view.

<img width="567" height="344" alt="image" src="https://github.com/user-attachments/assets/daefb70e-e4b2-4c86-a57a-c7af14402768" />

Aquí le ponemos un nombre representativo de lo que queremos ver, y en este
caso queremos ver los logs del ssh. Al lado de este cuadrado, vemos que
ha detectado un index con este formato, por lo que para sacar este y los
futuros logs ponemos esto en el index pattern.

<img width="643" height="335" alt="image" src="https://github.com/user-attachments/assets/81197789-0bd9-4870-b94d-c2c2e6bdc1fb" />

Una vez creado, nos dirigimos a la pestaña de Discover y vemos que nos
salen metricas y podemos ver con exactitud los logs de una manera más
eficiente y agradable para monitorear nuestras instalaciones.

<img width="570" height="235" alt="image" src="https://github.com/user-attachments/assets/c5dcae1e-7a5e-48db-9b40-eced1792826f" />

# Conclusión

Con esta práctica hemos logrado montar un **SIEM** completamente
funcional desde cero. Superamos el reto técnico de capturar la actividad
del servidor atacado dentro de Docker para enviarla con éxito al sistema
central. Al lanzar el ataque de prueba, pudimos comprobar en **Kibana**
lo fácil y visual que resulta identificar al atacante (su IP), el
momento exacto y el tipo de fallo ("Failed password"). En definitiva, el
entorno cumple su objetivo a la perfección y demuestra el valor real de
estas herramientas para vigilar y proteger una red.
