#!/bin/bash
# =============================================================
# sim_pool_tcp.sh
# Simula una conexion TCP persistente tipo pool hacia Oracle.
# Objetivo: identificar en que umbral de tiempo un dispositivo
# de red (firewall/balanceador) corta conexiones idle.
#
# Uso: ./sim_pool_tcp.sh
# Ajustar INTERVALO para encontrar el umbral exacto del corte.
# =============================================================

HOST="10.34.58.101"
PUERTO="4584"
INTERVALO=300   # <- Bajar progresivamente: 240, 180, 120, 60...

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"; }

log "Iniciando simulacion de pool TCP hacia $HOST:$PUERTO"
log "Latido cada ${INTERVALO}s | Ctrl+C para salir"
echo "----------------------------------------------------"

INTENTO=1
while true; do
    log "Intento $INTENTO: conectando..."

    (
        CICLO=1
        while true; do
            log "Ciclo $CICLO - conexion idle por ${INTERVALO}s..."
            sleep $INTERVALO
            log "Ciclo $CICLO - enviando latido..."
            printf "\x00"   # byte nulo, solo para verificar si el socket sigue vivo
            CICLO=$((CICLO + 1))
        done
    ) | nc $HOST $PUERTO

    log ">>> CONEXION CAIDA en intento $INTENTO (idle=${INTERVALO}s) - reconectando en 5s..."
    INTENTO=$((INTENTO + 1))
    sleep 5
done
