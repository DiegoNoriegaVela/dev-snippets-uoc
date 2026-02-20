/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Diagnostico de Red
 * #Nombre      : simpool
 * #Descripcion : Simula conexion persistente tipo pool hacia Oracle.
 *                Usa la misma infraestructura de conexion del proyecto
 *                (sw_logon_database / esp_read) para establecer una
 *                sesion Oracle real y enviar un heartbeat cada N segundos.
 *                Objetivo: detectar si un dispositivo de red corta
 *                conexiones idle antes del intervalo configurado.
 * #Autor       : Luis Alberto Guerrero Romero
 * #Fecha       : 2026-02-20
 *
 * COMPILACION:
 *   make -f Makefile.simpool
 *
 * USO:
 *   ./simpool <IP> <PUERTO> <INTERVALO>
 *   ./simpool 10.34.58.101 4584 300
 *   ./simpool 10.34.58.101 4584 120
 *
 * NOTA:
 *   Maximo 99 ciclos por ejecucion.
 *   INTERVALO minimo 10s, maximo 3600s.
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include "pt_master.h"

#define MAX_CICLOS  99

/* -------------------------------------------------- */
/* Log a pantalla con timestamp                       */
/* -------------------------------------------------- */
static void log_msg(const char *nivel, const char *msg)
{
    time_t now;
    char ts[30];
    time(&now);
    strftime(ts, sizeof(ts), "%Y-%m-%d %H:%M:%S", localtime(&now));
    printf("[%s] [%-6s] %s\n", ts, nivel, msg);
    fflush(stdout);
}

/* -------------------------------------------------- */
/* Heartbeat: consulta a tabla real del sistema       */
/* Retorna 0 si OK, -1 si fallo                       */
/* -------------------------------------------------- */
static int enviar_heartbeat(int ciclo)
{
    ESP_SLOT slot;
    ESP_SLOTP slotp = &slot;
    char command[300];
    char sid[20];
    char status[20];
    char logbuf[300];

    memset(command, '\0', sizeof(command));
    memset(sid,     '\0', sizeof(sid));
    memset(status,  '\0', sizeof(status));

    /*
     * Consulta a V$SESSION usando la sesion actual.
     * Es la mas representativa porque confirma que la sesion
     * Oracle sigue activa, no solo el TCP.
     * Mismo patron que se usa en gentabpan con esp_read/esp_fetch/esp_get.
     */
    sprintf(command,
        "SELECT SID, STATUS FROM V$SESSION "
        "WHERE AUDSID = USERENV('SESSIONID') "
        "AND ROWNUM = 1");

    if (esp_read(slotp, "heartbeat", command, ESP_LOOK)) {
        sprintf(logbuf, "Ciclo %d - fallo: %s",
            ciclo, (char*)*Last_error);
        log_msg("ERROR", logbuf);
        esp_free(slotp);
        return -1;
    }

    if (!esp_fetch(slotp)) {
        esp_get(slotp, "sid",    sid);
        esp_get(slotp, "status", status);
        sprintf(logbuf, "Ciclo %d - sesion activa: SID=%s STATUS=%s",
            ciclo, sid, status);
        log_msg("OK", logbuf);
    }

    esp_free(slotp);
    return 0;
}

/* -------------------------------------------------- */
/* Imprime separador visual                           */
/* -------------------------------------------------- */
static void separador()
{
    printf("----------------------------------------------\n");
    fflush(stdout);
}

/* -------------------------------------------------- */
/* Main                                               */
/* -------------------------------------------------- */
int main(int argc, char **argv)
{
    char  *host;
    char  *puerto_str;
    int    intervalo;
    int    ciclo        = 1;
    int    resultado;
    int    reconexiones = 0;
    char   logbuf[300];
    time_t t_inicio, t_ahora;
    char   ts_inicio[30];

    if (argc != 4) {
        printf("Uso: %s <IP> <PUERTO> <INTERVALO_SEGS>\n", argv[0]);
        printf("  Ejemplo: %s 10.34.58.101 4584 300\n", argv[0]);
        printf("  INTERVALO : entre 10 y 3600 segundos\n");
        printf("  Max ciclos: %d\n", MAX_CICLOS);
        exit(1);
    }

    host       = argv[1];
    puerto_str = argv[2];
    intervalo  = atoi(argv[3]);

    if (intervalo < 10 || intervalo > 3600) {
        printf("Error: intervalo debe ser entre 10 y 3600 segundos\n");
        exit(1);
    }

    separador();
    printf("  simpool - Simulador de Pool Oracle\n");
    printf("  Destino   : %s:%s\n", host, puerto_str);
    printf("  Intervalo : %d segundos\n", intervalo);
    printf("  Max ciclos: %d\n", MAX_CICLOS);
    printf("  Ctrl+C para salir antes\n");
    separador();
    printf("\n");

    time(&t_inicio);
    strftime(ts_inicio, sizeof(ts_inicio), "%Y-%m-%d %H:%M:%S", localtime(&t_inicio));

    sprintf(logbuf, "Conectando a Oracle %s:%s ...", host, puerto_str);
    log_msg("INFO", logbuf);

    sw_logon_database(1);

    log_msg("INFO", "Sesion Oracle establecida. Iniciando ciclos de heartbeat.");
    separador();

    while (ciclo <= MAX_CICLOS) {

        sprintf(logbuf, "Ciclo %d/%d - conexion idle por %ds...",
            ciclo, MAX_CICLOS, intervalo);
        log_msg("INFO", logbuf);

        sleep(intervalo);

        resultado = enviar_heartbeat(ciclo);

        if (resultado != 0) {
            time(&t_ahora);
            sprintf(logbuf,
                ">>> CONEXION CAIDA ciclo %d/%d | idle=%ds | reconexion #%d",
                ciclo, MAX_CICLOS, intervalo, ++reconexiones);
            log_msg("ALERTA", logbuf);

            separador();
            log_msg("INFO", "Reconectando a Oracle...");
            sw_logon_database(1);
            log_msg("INFO", "Reconexion exitosa. Continuando...");
            separador();
        }

        ciclo++;
    }

    time(&t_ahora);
    printf("\n");
    separador();
    printf("  RESUMEN FINAL\n");
    printf("  Inicio             : %s\n", ts_inicio);
    {
        char ts_fin[30];
        strftime(ts_fin, sizeof(ts_fin), "%Y-%m-%d %H:%M:%S", localtime(&t_ahora));
        printf("  Fin                : %s\n", ts_fin);
    }
    printf("  Ciclos completados : %d\n", MAX_CICLOS);
    printf("  Intervalo probado  : %d segundos\n", intervalo);
    printf("  Reconexiones       : %d\n", reconexiones);
    if (reconexiones == 0)
        printf("  Resultado          : SIN CAIDAS - conexion estable\n");
    else
        printf("  Resultado          : INESTABLE - revisar red u Oracle\n");
    separador();

    return 0;
}
