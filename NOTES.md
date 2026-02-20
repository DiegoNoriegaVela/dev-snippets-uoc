#!/bin/bash
# sim_pool_tcp.sh
# Simula conexion TCP idle tipo pool usando /dev/tcp nativo de bash.
# Objetivo: encontrar el umbral de idle timeout del dispositivo de red.
# Ajustar INTERVALO progresivamente: 300 -> 240 -> 180 -> 120 -> 60

HOST="10.34.58.101"
PUERTO="4584"
INTERVALO=300

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"; }

log "Iniciando simulacion hacia $HOST:$PUERTO"
log "Intervalo actual: ${INTERVALO}s | Ctrl+C para salir"
echo "----------------------------------------------------"

INTENTO=1
while true; do
    log "Intento $INTENTO: conectando..."

    # Abre el socket TCP como file descriptor 3
    exec 3<>/dev/tcp/$HOST/$PUERTO 2>/dev/null

    if [ $? -ne 0 ]; then
        log "ERROR: No se pudo conectar. Reintentando en 5s..."
        sleep 5
        continue
    fi

    log "Conexion establecida (FD3 abierto)"

    CICLO=1
    while true; do
        log "Ciclo $CICLO - idle por ${INTERVALO}s..."
        sleep $INTERVALO

        # Intentamos escribir en el socket para ver si sigue vivo
        printf "\x00" >&3 2>/dev/null
        if [ $? -ne 0 ]; then
            log ">>> CONEXION CAIDA en ciclo $CICLO (idle=${INTERVALO}s)"
            break
        fi

        log "Ciclo $CICLO - socket sigue activo"
        CICLO=$((CICLO + 1))
    done

    # Cierra el file descriptor
    exec 3>&- 2>/dev/null
    exec 3<&- 2>/dev/null

    log "Reconectando en 5s..."
    INTENTO=$((INTENTO + 1))
    sleep 5
done

python --version 2>/dev/null || python3 --version 2>/dev/null
echo "test" > /dev/tcp/10.34.58.101/4584 2>/dev/null && echo "dev/tcp ok"
