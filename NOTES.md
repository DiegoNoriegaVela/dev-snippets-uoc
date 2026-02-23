Subject: Evidencia sobre causa raíz en desconexiones Oracle – Servidor BWSAIX18

Estimado equipo de Infraestructura,

Como es de su conocimiento, el servidor BWSAIX18 viene presentando caídas periódicas en los drivers de conexión Oracle con errores ORA-03135, ORA-03113 y ORA-03114. Si bien el problema era conocido, no se contaba con evidencia concreta sobre su causa raíz. El pasado viernes, con el apoyo de Luis Guerrero del equipo de backend, realizamos una serie de pruebas que nos permiten documentar el comportamiento observado y presentar evidencia para su análisis.


METODOLOGÍA

Para las pruebas se desarrolló un binario de diagnóstico llamado simpool, construido con la misma infraestructura de conexión Oracle que utilizan los procesos del sistema, es decir, usa las mismas llamadas internas (sw_logon_database, esp_read, esp_fetch) con las que trabajan los drivers en producción. Esto garantiza que la prueba replica fielmente el comportamiento de una conexión real del pool y no una simulación genérica.

El programa funciona de la siguiente manera: establece una sesión Oracle, la deja inactiva (idle) durante un intervalo configurable de segundos, y luego ejecuta una consulta para verificar si la conexión sigue activa. Registra con timestamp exacto cada ciclo, el resultado obtenido y cualquier error Oracle que se presente. Si detecta una caída, reconecta automáticamente y continúa la prueba.


RESULTADOS OBSERVADOS

Al ejecutar la prueba con ciclos de 300 segundos (5 minutos), se observó el siguiente comportamiento:

Durante los primeros 11 ciclos, la conexión se mantuvo estable sin ninguna caída:

  15:46:10  Sesión Oracle establecida
  15:51:10  Ciclo 1  – BD responde OK
  15:56:10  Ciclo 2  – BD responde OK
  ...
  16:41:10  Ciclo 11 – BD responde OK

En el ciclo 12 el comportamiento cambió abruptamente:

  16:41:10  Ciclo 12 inicia período idle de 300 segundos
  16:46:10  Se envía la consulta de heartbeat
  16:55:12  Se recibe ORA-03113: end-of-file on communication channel

La consulta fue enviada correctamente a las 16:46:10 pero no recibió respuesta. El proceso permaneció bloqueado aproximadamente 9 minutos hasta que Oracle cerró la conexión desde el servidor emitiendo el EOF. Este comportamiento es consistente con un escenario donde los paquetes enviados por el cliente son descartados silenciosamente por un dispositivo intermedio, Oracle espera respuesta, no la recibe, y finalmente cierra la sesión.

Inmediatamente después, al intentar reconectar, se identificó un segundo problema: el tráfico nuevo desde BWSAIX18 hacia Oracle en 10.237.32.17:4597 no llega. Las pruebas realizadas fueron:

  - Ping a 10.237.32.17 responde correctamente con 0% de pérdida y 2ms de latencia.
  - Telnet desde BWSAIX18 al puerto 4597 queda en "Trying..." indefinidamente sin respuesta ni rechazo.
  - Desde otros servidores del entorno el telnet al mismo puerto y servidor conecta sin problema.
  - El listener Oracle está activo, confirmado porque otros servidores se conectan exitosamente.

Esto indica que el bloqueo es selectivo desde BWSAIX18 como origen, ya que la conectividad básica existe pero las conexiones TCP nuevas al puerto Oracle no llegan a establecerse, dejando al driver sin posibilidad de recuperarse tras la caída.

El resultado final observado es el siguiente:

  1. Driver envía una consulta → no recibe respuesta
  2. Oracle cierra la sesión con ORA-03113
  3. Driver intenta reconectar → conexión TCP bloqueada
  4. Driver queda sin conexión Oracle, sin posibilidad de recuperarse automáticamente


SOLICITUD

Con base en los comportamientos observados, solicitamos revisar dos puntos:

1. La configuración del dispositivo de red intermedio entre BWSAIX18 y Oracle 10.237.32.17:4597, en particular cualquier timeout de sesión TCP que pueda estar expirando conexiones activas y descartando paquetes silenciosamente.

2. Las reglas de acceso hacia el puerto 4597 de 10.237.32.17 para verificar si BWSAIX18 está incluido como origen permitido para establecer nuevas conexiones TCP.

Quedamos disponibles para proporcionar los logs completos de la prueba o realizar pruebas adicionales que consideren necesarias.

Saludos,
Diego Noriega
Switch ATM – Backend

(Pruebas realizadas con el apoyo de Luis Guerrero)
