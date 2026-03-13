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

<img src="media/image1.png" style="width:5.70762in;height:0.96863in" />

Creamos una carpeta para el proyecto y dentro creamos dos archivos:
**docker-compose.yml** y **logstash.conf.**

<img src="media/image2.png" style="width:5.90556in;height:0.90139in" />

Dentro del directorio creado, definimos el núcleo del SIEM con un
archivo **Docker** **Compose**.

<img src="media/image3.png" style="width:4.4372in;height:4.04273in" />

Logstash necesita saber qué hacer con los datos. Así que creamos un
archivo de configuración para **logstash** que escuche en el puerto
**5044** (donde se enviarán los logs posteriormente) y los mande
directamente a **Elasticsearch**.

<img src="media/image4.png" style="width:5.90556in;height:2.1375in" />

Ahora con las configuraciones anteriormente establecidas, pasamos a
construir el Docker Compose con este comando:

<img src="media/image5.png" style="width:5.90556in;height:1.79722in" />

Dentro de la carpeta de antes creamos una subcarpeta para la víctima y
dentro creamos un Dockerfile.

<img src="media/image6.png" style="width:5.90556in;height:0.8in" />

Como los contenedores base de Ubuntu no traen sistema de logs por
defecto, este Dockerfile instala SSH, para que se genere en el archivo
**auth.log**, y **Filebeat**.

<img src="media/image7.png" style="width:5.90556in;height:2.77083in" />

Ahora creamos otro archivo en el mismo directorio en el que estábamos
para que le diga a Filebeat qué leer y a dónde enviarlo.

<img src="media/image8.png" style="width:4.71867in;height:2.05233in" />

Ahora volvemos a la raíz de tu proyecto y modificamos el
docker-compose.yml para añadir el nuevo servicio de la víctima. Justo
debajo del servicio logstash:

<img src="media/image9.png" style="width:4.87758in;height:3.90883in" />

Ahora volvemos a reconstruir el proyecto para incluir la víctima:

<img src="media/image10.png" style="width:4.34593in;height:4.21101in" />

En la carpeta principal del proyecto, crea una nueva carpeta llamada
atacante. Dentro, creamos un archivo llamado Dockerfile.

<img src="media/image11.png" style="width:5.90556in;height:0.78542in" />

Usaremos una imagen base muy ligera (Alpine Linux) a la que simplemente
le instalaremos el cliente SSH para poder lanzar conexiones contra la
víctima.

<img src="media/image12.png" style="width:5.90556in;height:1.16806in" />

Ahora volvemos al Docker Compose de la carpeta raíz para añadir la nueva
subcarpeta que contiene al atacante, tal y como hicimos antes con la
víctima.

<img src="media/image13.png" style="width:5.90556in;height:4.58681in" />

Ahora vamos a aplicar los cambios borrando los contenedores anteriores
con -v por si acaso.

<img src="media/image14.png" style="width:5.90556in;height:2.48611in" />

Ejecutamos el ataque de fuerza bruta mediante un bucle automatizado en
bash. Este script realiza 10 intentos de conexión consecutivos por SSH
usando un usuario inexistente para forzar la generación de eventos de
error (Failed password) en la máquina víctima.
<img src="media/image15.png" style="width:5.90556in;height:1.51597in" />

Comprobamos con curl que Elastisearch está cogiendo correctamente los
índices del ssh con el formateo correcto, para que kibana lo detecte.

<img src="media/image16.png" style="width:5.90556in;height:0.45in" />

Una vez hemos hecho todos los pasos, nos dirigimos a Kibana en el puerto
5601, y vamos a stack managment/Data views, y aquí dentro hay una opción
para crear un data view.

<img src="media/image17.png" style="width:5.90556in;height:3.58819in" />

<img src="media/image18.png" style="width:3.45417in;height:3.48889in" /><img src="media/image19.png" style="width:3.22222in;height:1.11458in" />Aquí
le ponemos un nombre representativo de lo que queremos ver, y en este
caso queremos ver los logs del ssh. Al lado de este cuadrado, vemos que
ha detectado un index con este formato, por lo que para sacar este y los
futuros logs ponemos esto en el index pattern.

Una vez creado, nos dirigimos a la pestaña de Discover y vemos que nos
salen metricas y podemos ver con exactitud los logs de una manera más
eficiente y agradable para monitorear nuestras instalaciones.

<img src="media/image20.png" style="width:5.90556in;height:2.4625in" />

# Conclusión

Con esta práctica hemos logrado montar un **SIEM** completamente
funcional desde cero. Superamos el reto técnico de capturar la actividad
del servidor atacado dentro de Docker para enviarla con éxito al sistema
central. Al lanzar el ataque de prueba, pudimos comprobar en **Kibana**
lo fácil y visual que resulta identificar al atacante (su IP), el
momento exacto y el tipo de fallo ("Failed password"). En definitiva, el
entorno cumple su objetivo a la perfección y demuestra el valor real de
estas herramientas para vigilar y proteger una red.
