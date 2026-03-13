# Sistema de Detección y Respuesta Ante Incidentes (n8n)

Este repositorio contiene la implementación de un workflow en n8n diseñado para automatizar la detección, registro y notificación de posibles incidentes de ciberseguridad. 

## 1. ¿Qué incidente detectamos?

El objetivo principal es cazar intentos de inicio de sesión anómalos. Para ello, el sistema analiza las direcciones IP entrantes y las cruza con fuentes de Inteligencia de Amenazas. Si un intento de acceso proviene de una IP catalogada como maliciosa (por ejemplo, atacantes de fuerza bruta, nodos de salida Tor o botnets), el sistema lo clasifica inmediatamente como un incidente crítico.

## 2. Lógica de detección paso a paso

El flujo sigue un proceso lineal muy claro para evaluar la amenaza y actuar en consecuencia:

1. **Ingesta de datos:** Un nodo Webhook hace de "puerta de entrada" recibiendo un JSON con los datos del intento de login (IP, usuario y estado).
2. **Auditoría centralizada:** Antes de preguntar nada, un nodo de PostgreSQL guarda el evento en la tabla `registro_logins`. Así nos aseguramos de tener un histórico completo y auditable para posibles análisis forenses.
3. **Enriquecimiento:** Un nodo HTTP Request coge la IP del atacante y hace una consulta a la API de AbuseIPDB para comprobar su reputación.
4. **Toma de decisiones:** Un nodo lógico `IF` evalúa la puntuación (`abuseConfidenceScore`) que nos devuelve la API:
   * **Score < 50 (Rama False):** Falsa alarma o riesgo bajo. El flujo termina aquí.
   * **Score >= 50 (Rama True):** Amenaza confirmada. Se activa la respuesta a incidentes.
5. **Respuesta Activa:** Se envía una alerta técnica por correo al equipo de seguridad (SOC) usando MailHog. Inmediatamente después, otro nodo PostgreSQL coge la IP y le hace un UPSERT (Insert or Update) en la tabla `ips_baneadas` para simular un bloqueo rápido a nivel de red.

## 3. Justificación de las decisiones técnicas

* **¿Por qué guardarlo todo?** He decidido registrar el 100% del tráfico entrante, no solo los ataques. En ciberseguridad, lo que hoy parece tráfico normal, mañana puede ser la clave para entender el origen de una brecha.
* **El umbral de 50:** Se ha establecido este límite en AbuseIPDB para no generar alertas innecesarias (falsos positivos) provocadas por IPs residenciales dinámicas. Solo disparamos la respuesta contra atacantes confirmados.
* **El uso de UPSERT para banear:** Al usar la restricción `UNIQUE` en la base de datos junto con la operación "Insert or Update", evitamos que el contenedor de PostgreSQL colapse dando errores si un mismo atacante lanza miles de peticiones seguidas. Si la IP ya está en la lista negra, el sistema simplemente la actualiza sin fallar.

## 4. Instrucciones para probar el workflow

Para levantar y probar el entorno localmente, sigue estos pasos:

Primero, inicia los servicios de la infraestructura con Docker Compose:

```bash
sudo docker compose -f dc-n8n.yml up -d
```
1. Accede a la interfaz de n8n e importa el fichero .json adjunto en este repositorio.

2. Configura tus propias credenciales de PostgreSQL y tu API Key de AbuseIPDB en los nodos correspondientes.

3. Haz doble clic en el nodo Webhook y pulsa en "Listen for test event" para abrir el puerto de escucha temporal.

4. Abre una terminal en tu host y lanza este comando simulando un ataque desde una IP listada en listas negras:

```
curl -X POST http://localhost:5678/webhook-test/login-api \
     -H "Content-Type: application/json" \
     -d '{
           "ip": "185.220.101.46",
           "usuario": "administrador",
           "estado": "failed"
         }'
```

5. Comprueba que ha llegado el correo de alerta abriendo el cliente web de MailHog (http://localhost:8025).

6. Verifica que la IP ha sido "bloqueada" consultando la base de datos PostgreSQL:

```
sudo docker exec -it postgres psql -U n8n_user -d n8n_db -c "SELECT * FROM ips_baneadas WHERE ip = '185.220.101.46';"
```

Para comprobarlo con una IP que no sea maliciosa usamos el mismo comando pero cambiamos la IP a verificar, por ejemplo la 8.8.8.8. Y el flujo no se completará hasta el final.

## 5. Reflexión y posibles mejoras

Aunque el sistema cumple su propósito, en un entorno de producción real o en una infraestructura autoalojada compleja, se podrían implementar las siguientes mejoras:

* **Bloqueo real en el Firewall:** Ahora mismo el baneo es una simulación en una base de datos. La evolución lógica sería conectar n8n (mediante un nodo SSH o una petición HTTP) directamente a un firewall perimetral o a un proxy inverso para aplicar una regla de denegación real (Drop) bloqueando el tráfico de esa IP en milisegundos.

* **Notificaciones instantáneas:** El correo electrónico puede ser un canal lento para incidentes críticos. Integrar un nodo de mensajería (como Telegram, Slack o Mattermost) permitiría al administrador recibir la alerta en tiempo real en su teléfono móvil.

* **Monitorización de servicios reales:** Sustituir el Webhook de prueba por un sistema que lea activamente los logs de intentos de acceso fallidos de servicios expuestos reales (como un servidor VPN, gestores de contraseñas o un NAS) para dotar al flujo de una utilidad defensiva práctica.
