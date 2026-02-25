Para: Equipo DBA / Administración Oracle
Asunto: Solicitud de verificación de configuración Oracle – Caídas de sesión en BWSAIX18

Estimado equipo,
Les escribo para solicitar su apoyo en la investigación de caídas recurrentes de sesión Oracle que estamos experimentando desde el servidor BWSAIX18. Hemos realizado pruebas exhaustivas que descartan problemas de red ya que la conexión es capa 2 directa, por lo que necesitamos verificar si la causa está del lado de Oracle.
El comportamiento observado es el siguiente: las sesiones caen consistentemente después de un período de tiempo aunque haya tráfico activo cada 60 segundos, recibiendo errores ORA-03135, ORA-03113 y ORA-03114. Esto sugiere que Oracle está cerrando las sesiones activamente por alguna configuración de límite de tiempo.
Solicito que ejecuten las siguientes consultas:
1. Identificar el perfil del usuario de conexión
sqlSELECT username, profile, account_status
FROM dba_users
WHERE username IN ('ESP', 'ESPHWS')  -- ajustar al usuario real
ORDER BY username;
2. Ver los límites configurados en el perfil
sqlSELECT profile, resource_name, resource_type, limit
FROM dba_profiles
WHERE profile IN (
    SELECT profile FROM dba_users
    WHERE username IN ('ESP', 'ESPHWS')
)
AND resource_name IN (
    'IDLE_TIME',
    'CONNECT_TIME',
    'SESSIONS_PER_USER',
    'LOGICAL_READS_PER_SESSION',
    'CPU_PER_SESSION'
)
ORDER BY profile, resource_name;
3. Ver si el límite de recursos está habilitado a nivel de base de datos
sqlSELECT name, value
FROM v$parameter
WHERE name IN (
    'resource_limit',
    'resource_manager_plan'
);
4. Verificar el SQLNET.EXPIRE_TIME configurado en el servidor Oracle
sql-- Esto es un archivo de configuración, no una consulta SQL
-- Solicito que revisen el contenido de:
-- $ORACLE_HOME/network/admin/sqlnet.ora
-- Específicamente el parámetro SQLNET.EXPIRE_TIME
5. Ver sesiones activas actuales del usuario para comparar
sqlSELECT s.sid, s.serial#, s.username, s.status,
       s.machine, s.program,
       s.logon_time,
       ROUND((SYSDATE - s.logon_time) * 24 * 60, 2) AS minutos_conectado,
       s.last_call_et AS segundos_desde_ultima_actividad
FROM v$session s
WHERE s.username IN ('ESP', 'ESPHWS')
ORDER BY s.logon_time;
6. Ver el historial de sesiones terminadas recientemente
sqlSELECT username, machine, program,
       logon_time,
       TO_CHAR(logon_time, 'HH24:MI:SS') AS hora_inicio,
       end_time,
       TO_CHAR(end_time, 'HH24:MI:SS') AS hora_fin,
       ROUND((end_time - logon_time) * 24 * 60, 2) AS duracion_minutos
FROM dba_audit_trail
WHERE username IN ('ESP', 'ESPHWS')
AND logon_time >= SYSDATE - 1
ORDER BY logon_time DESC;
7. Verificar si hay algún Resource Manager activo limitando sesiones
sqlSELECT plan, status, comments
FROM dba_rsrc_plans
WHERE status = 'ACTIVE';

Lo que esperamos determinar con estas consultas es si existe algún CONNECT_TIME o IDLE_TIME configurado en el perfil del usuario que esté cerrando las sesiones activamente, si el resource_limit está habilitado a nivel de base de datos, y si el SQLNET.EXPIRE_TIME del servidor está configurado con un valor bajo que provoque el cierre de sesiones.
Quedo atento a los resultados y disponible para coordinar las pruebas que sean necesarias.
