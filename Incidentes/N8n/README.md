# Sistema de Detección y Respuesta Ante Incidentes (n8n)

Este repositorio contiene la implementación de un workflow funcional en n8n diseñado para detectar, registrar y notificar posibles incidentes de ciberseguridad de forma automatizada.

## 1. Descripción del incidente que se detecta

El sistema detecta intentos de inicio de sesión (logins) anómalos provenientes de direcciones IP con mala reputación, integrando fuentes de Inteligencia de Amenazas (Threat Intelligence) externas. Se considera incidente crítico cualquier intento de acceso originado desde una IP catalogada previamente como maliciosa (ataques de fuerza bruta, nodos de salida Tor maliciosos, botnets, etc.).

## 2. Explicación de la lógica de detección

El flujo de automatización (workflow) sigue un proceso lineal de evaluación y respuesta:

1. **Ingesta de datos:** Un nodo Webhook (Trigger) recibe un payload en formato JSON con los datos del intento de inicio de sesión (IP, usuario, estado).
2. **Auditoría centralizada:** Un nodo PostgreSQL inserta incondicionalmente el evento en la tabla `registro_logins` para mantener un log histórico auditable.
3. **Enriquecimiento de datos:** Un nodo HTTP Request consulta la API de AbuseIPDB enviando la IP del atacante.
4. **Toma de decisiones:** Un nodo lógico `IF` evalúa el campo `abuseConfidenceScore` devuelto por la API. 
   * **Si el score es < 50 (Rama False):** El flujo termina, considerándose un evento de bajo riesgo.
   * **Si el score es >= 50 (Rama True):** Se dispara la respuesta a incidentes.
5. **Respuesta Activa:** Se envía una notificación por correo electrónico al SOC (vía MailHog) con los detalles técnicos del atacante. Seguidamente, un segundo nodo de PostgreSQL ejecuta un UPSERT (Insert or Update) para registrar la IP en una tabla de contención (`ips_baneadas`), simulando un bloqueo.

## 3. Justificación de los criterios utilizados

* **Auditoría total:** Se decide registrar el 100% del tráfico antes de la evaluación para garantizar la trazabilidad forense ante posibles investigaciones futuras.
* **Umbral de detección (Score >= 50):** Se establece este límite en AbuseIPDB para evitar falsos positivos con IPs residenciales dinámicas, asegurando que las alertas y bloqueos solo se aplican a actores maliciosos confirmados.
* **Baneo mediante UPSERT:** Al utilizar la restricción `UNIQUE` en la base de datos y la operación "Insert or Update", se optimiza el rendimiento del sistema evitando errores de duplicidad de claves si el mismo atacante realiza ataques iterativos masivos.

## 4. Instrucciones para probar el workflow

Para probar el sistema de forma local:

Primero debemos de iniciar el docker compose con:

```
sudo docker compose -f dc-n8n.yml up -d
```

1. Acceder a la interfaz de n8n e importar el fichero `.json` adjunto.
2. Configurar las credenciales de PostgreSQL y la API Key de AbuseIPDB en los nodos correspondientes.
3. Hacer doble clic en el nodo **Webhook** y pulsar en **"Listen for test event"** para abrir el puerto de escucha temporal.
4. Abrir una terminal en el host y ejecutar el siguiente comando simulando un ataque desde una IP listada en listas negras:

```bash
curl -X POST http://localhost:5678/webhook-test/login-api \
     -H "Content-Type: application/json" \
     -d '{
           "ip": "185.220.101.46",
           "usuario": "administrador",
           "estado": "failed"
         }'
```

5. Verificar la recepción del correo de alerta en el cliente web de MailHog (http://localhost:8025).
6. Verificar la inserción de la IP en la base de datos PostgreSQL en la tabla ips_baneadas.
```
sudo docker exec -it postgres psql -U n8n_user -d n8n_db -c "SELECT * FROM ips_baneadas WHERE ip = '185.220.101.46';"
```
