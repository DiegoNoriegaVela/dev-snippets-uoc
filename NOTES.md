Estimado equipo DBA,

Les escribo para solicitar su apoyo en la investigación de caídas recurrentes de sesión Oracle que estamos experimentando desde el servidor BWSAIX18. Hemos realizado pruebas exhaustivas que descartan problemas de red ya que la conexión es capa 2 directa, por lo que necesitamos verificar si la causa está del lado de Oracle.

Datos de la conexión:
  Servidor Oracle  : 10.237.32.17
  Puerto           : 4597
  Usuario Oracle   : ESP
  Servidor origen  : BWSAIX18 (10.237.32.13)

El comportamiento observado es el siguiente: las sesiones caen consistentemente después de un período de tiempo aunque haya tráfico activo cada 60 segundos, recibiendo errores ORA-03135, ORA-03113 y ORA-03114. Esto sugiere que Oracle está cerrando las sesiones activamente por alguna configuración de límite de tiempo.

Solicito que ejecuten las siguientes consultas:

1. Identificar el perfil del usuario ESP

SELECT username, profile, account_status
FROM dba_users
WHERE username = 'ESP';

2. Ver los límites configurados en el perfil del usuario ESP

SELECT profile, resource_name, resource_type, limit
FROM dba_profiles
WHERE profile = (
    SELECT profile FROM dba_users WHERE username = 'ESP'
)
AND resource_name IN (
    'IDLE_TIME',
    'CONNECT_TIME',
    'SESSIONS_PER_USER',
    'LOGICAL_READS_PER_SESSION',
    'CPU_PER_SESSION'
)
ORDER BY resource_name;

3. Verificar si el límite de recursos está habilitado a nivel de base de datos

SELECT name, value
FROM v$parameter
WHERE name IN (
    'resource_limit',
    'resource_manager_plan'
);

4. Ver sesiones activas actuales del usuario ESP

SELECT s.sid, s.serial#, s.username, s.status,
       s.machine, s.program,
       s.logon_time,
       ROUND((SYSDATE - s.logon_time) * 24 * 60, 2) AS minutos_conectado,
       s.last_call_et AS segundos_desde_ultima_actividad
FROM v$session s
WHERE s.username = 'ESP'
ORDER BY s.logon_time;

5. Ver historial de sesiones terminadas recientemente del usuario ESP

SELECT username, machine, program,
       logon_time,
       TO_CHAR(logon_time, 'HH24:MI:SS') AS hora_inicio,
       end_time,
       TO_CHAR(end_time, 'HH24:MI:SS') AS hora_fin,
       ROUND((end_time - logon_time) * 24 * 60, 2) AS duracion_minutos
FROM dba_audit_trail
WHERE username = 'ESP'
AND logon_time >= SYSDATE - 1
ORDER BY logon_time DESC;

6. Verificar si hay algún Resource Manager activo limitando sesiones

SELECT plan, status, comments
FROM dba_rsrc_plans
WHERE status = 'ACTIVE';

7. Verificar el archivo sqlnet.ora del servidor Oracle

Esto no es una consulta SQL sino un archivo de configuración. Solicito que revisen el contenido de $ORACLE_HOME/network/admin/sqlnet.ora buscando el parámetro SQLNET.EXPIRE_TIME y cualquier otro parámetro de timeout configurado.

Lo que esperamos determinar con estas consultas:
  - Si existe CONNECT_TIME o IDLE_TIME en el perfil del usuario ESP que esté cerrando sesiones activamente
  - Si resource_limit está habilitado a nivel de base de datos
  - Si SQLNET.EXPIRE_TIME del servidor tiene un valor bajo que provoque el cierre

Timestamps exactos de caídas registradas para referencia:
  2026-02-24 17:22:23  ORA-03135: connection lost contact
  2026-02-24 13:10:38  ORA-03114: not connected to Oracle
  2026-02-20 16:55:12  ORA-03113: end-of-file on communication channel

Quedo atento a los resultados y disponible para coordinar las pruebas que sean necesarias.

