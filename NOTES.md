#!/bin/ksh
#====================================================================
# Script: gen_dummy_panacct.sh
# Compatibilidad: AIX (ksh)
# Descripcion: Genera N inserts dummy en ESP.PANACCT
#              replicando el formato exacto del insert original.
#
# PANs: 7777000000000001 al 7777000000100000 (consecutivos)
#
# Uso: ./gen_dummy_panacct.sh [cantidad]
#
# Eliminar data dummy:
#   DELETE FROM ESP.PANACCT WHERE DESCRIPTION = 'PRUEBA DUMMY   ';
#   COMMIT;
#====================================================================

CANTIDAD=${1:-100000}
ARCHIVO_SQL="dummy_panacct.sql"
BATCH_SIZE=500
DB_USER="esp"
DB_PASS="caja"

BIN_PREFIX="7777"

echo "=============================================="
echo " Generador de data dummy - ESP.PANACCT"
echo "=============================================="
echo " Registros  : ${CANTIDAD}"
echo " BIN        : ${BIN_PREFIX}"
echo " Archivo    : ${ARCHIVO_SQL}"
echo "=============================================="
echo ""

# --- PASO 1: Generar PANs y encriptar ---
echo "[1/3] Generando y encriptando PANs..."

ENC_FILE="/tmp/encpans_$$.txt"
> "$ENC_FILE"

i=1
while [ $i -le $CANTIDAD ]; do
    PAN=$(printf '%s%012d' "$BIN_PREFIX" $i)

    # Encriptar con encpan
    ENC_RESULT=$(encpan "$PAN" 2>/dev/null | grep -i "encripted\|encrypted" | awk '{print $NF}')

    if [ -z "$ENC_RESULT" ]; then
        ENC_RESULT=$(encpan "$PAN" 2>/dev/null | tail -1 | awk '{print $NF}')
    fi

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

# Valores fijos con padding de espacios (replicando el insert original)
# INSTID: CHAR(11) -> '890509     ' (6 digitos + 5 espacios)
INSTID="890509     "
# SUBINSTID: CHAR(11) -> '00         ' (2 digitos + 9 espacios)
SUBINSTID="00         "
# DESCRIPTION: CHAR(15) -> 'PRUEBA DUMMY   ' (12 chars + 3 espacios)
DESCRIPTION="PRUEBA DUMMY   "
# LINE2DESCRIPTION: CHAR(15) -> 'DUMMY          ' (5 chars + 10 espacios)
LINE2DESC="DUMMY          "
# TSFIN: CHAR(1) -> segun original parece ser 'Y'
# TSFOUT: CHAR(1) -> segun original parece ser 'Y'
# CURRENCY: CHAR(3) -> '604'

cat > "$ARCHIVO_SQL" << 'HEADER'
-- ====================================================================
-- Data dummy ESP.PANACCT - Generado automaticamente
-- PANs con BIN 7777 consecutivos, encriptados con encpan
--
-- ELIMINAR DATA:
--   DELETE FROM ESP.PANACCT WHERE DESCRIPTION LIKE 'PRUEBA DUMMY%';
--   COMMIT;
-- ====================================================================
SET DEFINE OFF
HEADER

COUNT=0
while IFS='|' read PAN ENC_PAN; do
    COUNT=$(( COUNT + 1 ))

    # ACCOUNT: CHAR(29) - numero consecutivo + padding espacios
    # Formato similar al original: 12 digitos + 17 espacios = 29
    ACCT_NUM=$(printf '%012d' $COUNT)
    ACCOUNT="${ACCT_NUM}                 "

    # PAN encriptado sin corchetes, solo el valor cifrado
    # encpan devuelve algo como "[1:m36nHLG9NLeW5xa1XVqfTw==]"
    # Quitamos corchetes si los trae
    PAN_VALUE=$(echo "$ENC_PAN" | tr -d '[]')

    # Padding del PAN a 49 chars (CHAR(49))
    PAN_LEN=${#PAN_VALUE}
    PAD_NEEDED=$(( 49 - PAN_LEN ))
    PAD_SPACES=""
    j=0
    while [ $j -lt $PAD_NEEDED ]; do
        PAD_SPACES="${PAD_SPACES} "
        j=$(( j + 1 ))
    done
    PAN_PADDED="${PAN_VALUE}${PAD_SPACES}"

    echo "Insert into \"ESP\".\"PANACCT\" (PAN,INSTID,SUBINSTID,ACCOUNT,PSEUDOACCT,FIRSTT,ACCTTYPE,DESCRIPTION,LINE2DESCRIPTION,BAL,WDR,DEP,TSFIN,TSFOUT,CURRENCY,ENC_WALL) values ('${PAN_PADDED}','${INSTID}','${SUBINSTID}','${ACCOUNT}',null,'F','1','${DESCRIPTION}','${LINE2DESC}','Y','Y','Y','Y','Y','604','Y');" >> "$ARCHIVO_SQL"

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
echo "SELECT COUNT(*) AS TOTAL_DUMMY FROM ESP.PANACCT WHERE DESCRIPTION LIKE 'PRUEBA DUMMY%';" >> "$ARCHIVO_SQL"

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
echo "   DELETE FROM ESP.PANACCT WHERE DESCRIPTION LIKE 'PRUEBA DUMMY%';"
echo "   COMMIT;"
echo "=============================================="
