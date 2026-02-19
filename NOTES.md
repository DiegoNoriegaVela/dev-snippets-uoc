#!/bin/ksh
#====================================================================
# Script: gen_dummy_panacct.sh
# Compatibilidad: AIX (ksh)
# Descripcion: Genera N inserts dummy en ESP.PANACCT con PANs
#              consecutivos encriptados con 'encpan'.
#
# PANs generados: 7777000000000001 hasta 7777000000100000
#
# Uso: ./gen_dummy_panacct.sh [cantidad]
#   - cantidad: numero de registros (default: 100000)
#
# Para eliminar la data dummy despues:
#   DELETE FROM ESP.PANACCT WHERE DESCRIPTION = 'PRUEBA DUMMY';
#   COMMIT;
#====================================================================

CANTIDAD=${1:-100000}
ARCHIVO_SQL="dummy_panacct.sql"
BATCH_SIZE=500
DB_USER="esp"
DB_PASS="caja"

# Configuracion
BIN_PREFIX="7777"
INSTID="00000000000"
SUBINSTID="00000000000"
CURRENCY="604"

echo "=============================================="
echo " Generador de data dummy - ESP.PANACCT"
echo "=============================================="
echo " Registros  : ${CANTIDAD}"
echo " BIN        : ${BIN_PREFIX}"
echo " Rango PANs : ${BIN_PREFIX}000000000001 - ${BIN_PREFIX}$(printf '%012d' ${CANTIDAD})"
echo " Archivo    : ${ARCHIVO_SQL}"
echo "=============================================="
echo ""

# --- PASO 1: Generar PANs y encriptar ---
echo "[1/3] Generando y encriptando PANs..."

ENC_FILE="/tmp/encpans_$$.txt"
> "$ENC_FILE"

i=1
while [ $i -le $CANTIDAD ]; do
    # PAN consecutivo: 7777 + 12 digitos con padding
    PAN=$(printf '%s%012d' "$BIN_PREFIX" $i)

    # Encriptar con encpan
    ENC_RESULT=$(encpan "$PAN" 2>/dev/null | grep -i "encripted\|encrypted" | awk '{print $NF}')

    # Si no capturo con grep, intentar la ultima linea
    if [ -z "$ENC_RESULT" ]; then
        ENC_RESULT=$(encpan "$PAN" 2>/dev/null | tail -1 | awk '{print $NF}')
    fi

    # Si aun no hay resultado, capturar todo el output limpio
    if [ -z "$ENC_RESULT" ]; then
        ENC_RESULT=$(encpan "$PAN" 2>/dev/null | tr -d '[:space:]')
    fi

    echo "${PAN}|${ENC_RESULT}" >> "$ENC_FILE"

    if [ $(( i % 5000 )) -eq 0 ]; then
        echo "  ... $i / $CANTIDAD encriptados"
    fi

    i=$(( i + 1 ))
done
echo "  OK: $CANTIDAD PANs encriptados"

# --- PASO 2: Generar SQL ---
echo ""
echo "[2/3] Generando ${ARCHIVO_SQL}..."

cat > "$ARCHIVO_SQL" << 'HEADER'
-- ====================================================================
-- Data dummy ESP.PANACCT - Generado automaticamente
-- PANs con BIN 7777 consecutivos, encriptados con encpan
--
-- ELIMINAR DATA:
--   DELETE FROM ESP.PANACCT WHERE DESCRIPTION = 'PRUEBA DUMMY';
--   COMMIT;
-- ====================================================================
SET DEFINE OFF
HEADER

COUNT=0
while IFS='|' read PAN ENC_PAN; do
    COUNT=$(( COUNT + 1 ))

    # Cuenta consecutiva
    ACCOUNT=$(printf '%029d' $COUNT)

    # Alternar tipo de cuenta
    MODULO=$(( COUNT % 3 ))
    case $MODULO in
        0) ACCTTYPE="1"; LINE2="Ahorros"   ;;
        1) ACCTTYPE="2"; LINE2="Corriente"  ;;
        2) ACCTTYPE="3"; LINE2="CTS"        ;;
    esac

    echo "Insert into \"ESP\".\"PANACCT\" (PAN,INSTID,SUBINSTID,ACCOUNT,PSEUDOACCT,FIRSTT,ACCTTYPE,DESCRIPTION,LINE2DESCRIPTION,BAL,WDR,DEP,TSFIN,TSFOUT,CURRENCY,ENC_WALL) values ('${ENC_PAN}','${INSTID}','${SUBINSTID}','${ACCOUNT}',null,'F','${ACCTTYPE}','PRUEBA DUMMY','${LINE2}','Y','Y','Y','Y','Y','${CURRENCY}','Y');" >> "$ARCHIVO_SQL"

    if [ $(( COUNT % BATCH_SIZE )) -eq 0 ]; then
        echo "COMMIT;" >> "$ARCHIVO_SQL"
    fi

    if [ $(( COUNT % 10000 )) -eq 0 ]; then
        echo "  ... $COUNT inserts generados"
    fi

done < "$ENC_FILE"

# Commit final
echo "COMMIT;" >> "$ARCHIVO_SQL"
echo "" >> "$ARCHIVO_SQL"
echo "-- FIN: ${COUNT} registros insertados" >> "$ARCHIVO_SQL"
echo "SELECT COUNT(*) AS TOTAL_DUMMY FROM ESP.PANACCT WHERE DESCRIPTION = 'PRUEBA DUMMY';" >> "$ARCHIVO_SQL"

FILE_SIZE=$(ls -lh "$ARCHIVO_SQL" | awk '{print $5}')
echo "  OK: ${COUNT} inserts en ${ARCHIVO_SQL} (${FILE_SIZE})"

# --- PASO 3: Ejecutar en Oracle ---
echo ""
echo "[3/3] Ejecutar en Oracle"
echo ""
printf "Deseas ejecutar el SQL ahora? (s/n): "
read EJECUTAR

if [ "$EJECUTAR" = "s" ] || [ "$EJECUTAR" = "S" ]; then
    echo ""
    echo "Ejecutando con sqlplus ${DB_USER}..."
    echo ""

    sqlplus -S "${DB_USER}/${DB_PASS}" << EOSQL
WHENEVER SQLERROR EXIT SQL.SQLCODE;
SET TIMING ON;
SET FEEDBACK ON;
@${ARCHIVO_SQL}
EXIT;
EOSQL

    if [ $? -eq 0 ]; then
        echo ""
        echo "  OK: SQL ejecutado exitosamente"
    else
        echo ""
        echo "  ERROR al ejecutar. Revisa ${ARCHIVO_SQL}"
    fi
else
    echo ""
    echo "  Para ejecutar manualmente:"
    echo "  sqlplus ${DB_USER}/${DB_PASS} @${ARCHIVO_SQL}"
fi

# Limpieza
rm -f "$ENC_FILE"

echo ""
echo "=============================================="
echo " RESUMEN"
echo "=============================================="
echo " Archivo     : ${ARCHIVO_SQL}"
echo " Registros   : ${COUNT}"
echo " PAN inicio  : ${BIN_PREFIX}000000000001"
echo " PAN fin     : ${BIN_PREFIX}$(printf '%012d' ${COUNT})"
echo " Description : PRUEBA DUMMY"
echo ""
echo " ELIMINAR DATA DUMMY:"
echo "   DELETE FROM ESP.PANACCT WHERE DESCRIPTION = 'PRUEBA DUMMY';"
echo "   COMMIT;"
echo "=============================================="
