CONTEXTO QUE PRESENTÁS AL INICIO
Problema      : Caídas recurrentes de conexión Oracle en servidor BWSAIX18
Origen        : BWSAIX18  →  10.237.32.13
Destino       : 10.237.32.17:4597 (Oracle TNS)
Protocolo     : TCP

Timestamps exactos de caídas registradas:
  2026-02-24 17:22:23  ORA-03135: connection lost contact
  2026-02-24 13:10:38  ORA-03114: not connected to Oracle
  2026-02-20 16:55:12  ORA-03113: end-of-file on communication channel

Comportamiento observado:
  - Sesión cae con tráfico activo (heartbeat cada 60s), no es idle
  - Después de la caída, reconexión queda bloqueada solo desde BWSAIX18
  - Desde otros servidores el mismo puerto funciona sin problema
  - Ping a 10.237.32.17 responde OK en todo momento
  - Telnet a 10.237.32.17:4597 queda en Trying... indefinido post-caída

BLOQUE 1: REVISIÓN DE LOGS
¿Qué buscar en los logs del firewall?
Pedirles que busquen en los logs para cada uno de los timestamps:

En los timestamps indicados buscar: cierre, expiración o teardown de sesión TCP entre 10.237.32.13 y 10.237.32.17:4597. Qué tipo de cierre fue: timeout, RST, FIN o política explícita.


Inmediatamente después de cada timestamp buscar: paquetes descartados o denegados desde 10.237.32.13 hacia 10.237.32.17:4597, especialmente paquetes SYN que son intentos de nueva conexión.


En el momento de la caída: ¿en qué estado quedó la sesión en la tabla del firewall? ESTABLISHED, CLOSE_WAIT, FIN_WAIT, TIME_WAIT o eliminada directamente.

Comandos según plataforma — pedirles que ejecuten según su firewall:
Cisco ASA:
  show conn address 10.237.32.13
  show asp drop
  show log | include 10.237.32.13
  show local-host 10.237.32.13

Palo Alto:
  show session all filter source 10.237.32.13 destination 10.237.32.17
  show session id <id>
  En GUI: Monitor → Logs → Traffic → filtrar src/dst con timestamps

Checkpoint:
  fw log -n | grep 10.237.32.13
  fw tab -t connections | grep 10.237.32.13

Fortinet:
  get system session list | grep 10.237.32.13
  diagnose sys session filter src 10.237.32.13
  diagnose sys session list

BLOQUE 2: CONFIGURACIÓN DE TIMEOUTS
Preguntas sobre timeouts:

¿Cuál es el TCP session lifetime configurado para conexiones entre la red de BWSAIX18 y la red Oracle? ¿Hay diferencia entre idle timeout y connection timeout absoluto?


¿El firewall manda RST a ambos extremos cuando expira una sesión TCP, o simplemente deja de rutear los paquetes silenciosamente? Esto es crítico porque si no manda RST ninguno de los dos extremos se entera del cierre.

Comandos para verificar timeouts:
Cisco ASA:
  show running-config timeout
  show running-config | include timeout

Palo Alto:
  show running security-policy
  En GUI: Objects → Security Profiles → verificar TCP timeouts

Checkpoint:
  fw ctl get int fwconn_timeout
  En SmartConsole: Global Properties → Stateful Inspection

Fortinet:
  show system session-ttl
  config system session-ttl
  show firewall policy

BLOQUE 3: REGLAS Y POLÍTICAS
Preguntas sobre reglas:

¿Existe alguna regla, ACL o política que aplique específicamente a 10.237.32.13 hacia 10.237.32.17:4597 que sea diferente al resto de servidores del entorno?


¿Hay alguna regla de rate limiting, connection limiting o threat prevention que pueda estar aplicando a BWSAIX18 específicamente?


¿Se realizó algún cambio de configuración en el firewall en las últimas semanas que afecte las redes 10.237.32.x?

Comandos para verificar reglas:
Cisco ASA:
  show access-list | include 10.237.32.13
  show access-list | include 10.237.32.17
  packet-tracer input <interface> tcp 10.237.32.13 1234 10.237.32.17 4597

Palo Alto:
  test security-policy-match source 10.237.32.13 destination 10.237.32.17
    protocol 6 destination-port 4597
  En GUI: Policies → Security → buscar reglas aplicando a esas IPs

Checkpoint:
  fw ctl zdebug + drop | grep 10.237.32.13
  En SmartConsole: buscar reglas con esas IPs

Fortinet:
  diagnose firewall iprope lookup 10.237.32.13 10.237.32.17 6 4597
  show firewall policy | grep 10.237.32

BLOQUE 4: VERIFICACIÓN EN TIEMPO REAL
Si pueden hacer la prueba durante la reunión:

Mientras ejecutamos el simulador, monitorear en tiempo real la sesión TCP entre 10.237.32.13 y 10.237.32.17:4597 y confirmar en qué momento y por qué causa el firewall la termina.

Cisco ASA:
  show conn address 10.237.32.13  (ejecutar cada 2 minutos)
  debug tcp packet <interface>    (con cuidado en producción)

Palo Alto:
  show session all filter source 10.237.32.13 (cada 2 minutos)
  En GUI: Monitor → App Scope → Network Monitor

Checkpoint:
  fw monitor -e "host(10.237.32.13)" -o /tmp/cap.pcap

Fortinet:
  diagnose sniffer packet any "host 10.237.32.13 and port 4597" 4

LO QUE ESPERÁS SALIR DE LA REUNIÓN
Al terminar la reunión deberías tener respuesta a estas tres preguntas concretas:
1. ¿El firewall está terminando la sesión TCP? Si sí, ¿por qué y a qué tiempo?
2. ¿El firewall manda RST o cierra silenciosamente?
3. ¿Hay alguna regla diferente aplicando a BWSAIX18 respecto a los otros servidores?
Con esas tres respuestas tenés la causa raíz confirmada y la solución específica para pedirles que implementen.
