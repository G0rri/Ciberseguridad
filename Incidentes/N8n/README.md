# Sistema de Detección de Incidentes con n8n

## 1. Descripción del incidente que se detecta
El sistema detecta posibles ataques de fuerza bruta dirigidos a los sistemas de autenticación. Concretamente, alerta sobre múltiples intentos fallidos de inicio de sesión (más de 3) provenientes de una misma dirección IP. Adicionalmente, el sistema audita y registra los inicios de sesión exitosos (legítimos) en una base de datos para mantener un control de accesos continuo.

## 2. Explicación de la lógica de detección
El flujo automatizado se estructura de la siguiente manera:
* **Entrada (Webhook):** Recibe eventos en formato JSON simulando logs de autenticación con los campos `ip_address`, `username`, `status` y `attempt_count`.
* **Enrutamiento (Switch):** Clasifica el tráfico evaluando el campo `status`. 
    * Si el estado es `success`, el flujo se desvía para registrar el evento de forma silenciosa en una base de datos PostgreSQL.
    * Si el estado es `failed`, el evento avanza por la ruta principal de análisis de incidentes.
* **Análisis (If):** Evalúa si el campo `attempt_count` (número de intentos fallidos) es mayor a 3. 
    * Si la condición se cumple (True), se deduce que se trata de un ataque sistemático y se procede a generar una alerta.
    * Si no se cumple (False), el flujo termina sin acción, asumiendo que es un error tipográfico humano puntual.
* **Respuesta (Send Email / Postgres):** Emisión de una alerta crítica por correo electrónico (vía SMTP local) al administrador en caso de ataque, o inserción de los datos en la tabla `accesos_correctos` de PostgreSQL en caso de acceso legítimo.

## 3. Justificación de los criterios utilizados
* **Umbral de intentos:** Se ha elegido el límite de "más de 3 intentos" porque es un estándar común y objetivo en ciberseguridad para diferenciar un error humano de un ataque de diccionario o fuerza bruta.
* **Enrutamiento (Switch vs único IF):** Se ha implementado un Switch inicial para separar drásticamente el tráfico de auditoría (accesos correctos) del tráfico de incidentes (alertas de seguridad), optimizando el rendimiento del workflow y evitando cruces de datos.
* **Herramientas de respuesta:** Se ha utilizado un servidor SMTP simulado (Mailhog) para la notificación de alertas, aislando el entorno de pruebas y evitando la exposición de credenciales reales. Para la auditoría, se ha aprovechado el contenedor PostgreSQL proporcionado para garantizar la persistencia de los registros en el mismo stack.

## 4. Instrucciones para probar el workflow
1. Desplegar el entorno de contenedores ejecutando `docker compose -f dc-n8n.yml up -d`.
2. Acceder al contenedor PostgreSQL y crear la tabla de auditoría mediante el siguiente comando en la terminal: 
   `docker exec -it postgres psql -U n8n_user -d n8n_db -c "CREATE TABLE accesos_correctos (id SERIAL PRIMARY KEY, ip VARCHAR(50), usuario VARCHAR(50), fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP);"`
3. Importar el archivo JSON del workflow en la interfaz de n8n (puerto 5678) y hacer clic en **Listen for test event** en el nodo Webhook.
4. **Simular un ataque (Genera alerta de Email):** Ejecutar en la terminal:
   `curl -X POST http://localhost:5678/webhook-test/alerta-login -H "Content-Type: application/json" -d '{"ip_address": "203.0.113.42", "username": "root", "status": "failed", "attempt_count": 5}'`
   *Comprobación:* Verificar la recepción del correo en la interfaz web de Mailhog (puerto 8025).
5. **Simular un acceso legítimo (Guarda en Base de Datos):** Hacer clic de nuevo en *Listen for test event* y ejecutar en la terminal:
   `curl -X POST http://localhost:5678/webhook-test/alerta-login -H "Content-Type: application/json" -d '{"ip_address": "192.168.1.50", "username": "admin", "status": "success", "attempt_count": 1}'`
   *Comprobación:* Ejecutar el nodo Postgres en n8n (o consultar la base de datos vía CLI) para confirmar la inserción del nuevo registro.

## 5. Reflexión sobre posibles mejoras
Una posible mejora técnica sería sustituir el envío de correo electrónico por una acción directa de mitigación, como utilizar el nodo HTTP Request para consumir la API de un Firewall (ej. pfSense o Cloudflare) y bloquear la IP atacante de forma automática. Además, se podría sustituir el Webhook pasivo por una conexión directa a una herramienta SIEM real (como Wazuh), automatizando la ingesta de alertas en tiempo real en lugar de depender de peticiones POST simuladas.
