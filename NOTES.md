while true; do 
    clear
    echo "=== MONITOR DE CONEXIÓN A BASE DE DATOS ==="
    echo "Hora actual: $(date '+%H:%M:%S')"
    echo "-------------------------------------------"
    # Filtramos el puerto de tu BD (cambia el 1521 si usan otro)
    netstat -an | grep 1521 | grep ESTABLISHED || echo "Sin conexiones activas."
    sleep 2
done

-------------

#!/bin/bash
# Script para simular el comportamiento del driver cada 5 minutos

USER="tu_usuario"
PASS="tu_password"
TNS="tu_tns"

echo "Iniciando simulador de Pool/Driver..."
(
echo "CONNECT $USER/$PASS@$TNS"
echo "SET PAGESIZE 0 FEEDBACK OFF VERIFY OFF HEADING OFF;"
echo "prompt [$(date '+%H:%M:%S')] -> Abriendo conexion y enviando primer SELECT..."
echo "SELECT 'ESTADO: CONECTADO OK' FROM DUAL;"

echo "prompt [$(date '+%H:%M:%S')] -> Durmiendo por 5 minutos (Simulando inactividad)..."
sleep 300

echo "prompt [$(date '+%H:%M:%S')] -> Despertando. Enviando el segundo SELECT..."
echo "SELECT 'ESTADO: SEGUNDA CONSULTA OK' FROM DUAL;"
echo "EXIT;"
) | sqlplus -s /nolog
