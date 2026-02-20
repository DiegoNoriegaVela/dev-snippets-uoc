/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Diagnostico de Red
 * #Nombre      : simpool
 * #Descripcion : Simula conexion persistente tipo pool hacia Oracle.
 *                Envia heartbeat cada N segundos para detectar si un
 *                dispositivo de red corta conexiones idle.
 * #Autor       : Luis Alberto Guerrero Romero
 * #Fecha       : 2026-02-20
 *
 * USO:
 *   ./simpool <INTERVALO>
 *   ./simpool 300
 *   ./simpool 120
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
#include <signal.h>
#include "pt_master.h"

#define MAX_CICLOS 99

time_t now;
extern char *argv0;

/* Flag global para salida limpia */
volatile int g_salir = 0;

/* -------------------------------------------------- */
/* Cierre limpio al recibir Ctrl+C o senales          */
/* esp_abort() cierra la sesion Oracle correctamente  */
/* evitando que el socket quede en FIN_WAIT           */
/* -------------------------------------------------- */
static void hot_exit(int n)
{
    time_t t;
    char ts[30];

    time(&t);
    strftime(ts, sizeof(ts), "%Y-%m-%d %H:%M:%S", localtime(&t));

    if (n == SIGINT) {
        printf("\n[%s] [INFO  ] Ctrl+C recibido. Cerrando sesion Oracle...\n", ts);
    } else {
        printf("\n[%s] [INFO  ] Senal %d recibida. Cerrando sesion Oracle...\n", ts, n);
    }
    fflush(stdout);

    /*
     * esp_abort() cierra la sesion Oracle de forma ordenada.
     * Esto permite que el kernel complete correctamente el
     * handshake de cierre TCP (FIN/ACK) en lugar de dejar
     * el socket colgado en FIN_WAIT.
     */
    esp_abort();

    printf("[%s] [INFO  ] Sesion cerrada correctamente. Saliendo.\n", ts);
    fflush(stdout);
    exit(0);
}

/* -------------------------------------------------- */
/* Log con timestamp                                  */
/* -------------------------------------------------- */
static void log_msg(const char *nivel, const char *msg)
{
    time_t n;
    char ts[30];
    time(&n);
    strftime(ts, sizeof(ts), "%Y-%m-%d %H:%M:%S", localtime(&n));
    printf("[%s] [%-6s] %s\n", ts, nivel, msg);
    fflush(stdout);
}

/* -------------------------------------------------- */
/* Heartbeat: SELECT FROM DUAL                        */
/* Patron identico a gentabpan: read/fetch/get/free   */
/* Retorna 0 si OK, -1 si fallo                       */
/* -------------------------------------------------- */
static int enviar_heartbeat(int ciclo)
{
    char      command[300];
    char      resultado[20];
    char      ts_bd[20];
    char      logbuf[300];
    ESP_SLOT  slot;
    ESP_SLOTP slotp = &slot;

    memset(command,   '\0', sizeof(command));
    memset(resultado, '\0', sizeof(resultado));
    memset(ts_bd,     '\0', sizeof(ts_bd));

    sprintf(command,
        "SELECT 'OK' AS resultado, TO_CHAR(SYSDATE,'HH24:MI:SS') AS ts_bd "
        "FROM DUAL");

    cc_send("Heartbeat ciclo %d [%s]\n", ciclo, command);

    if (esp_read(slotp, "card", command, ESP_LOOK))
    {
        cc_send("ESP_READ ERROR: [%s]", (char*)*Last_error);
        fprintf(stderr, "Ciclo %d - ESP_READ ERROR: %s\n",
            ciclo, (char*)*Last_error);
        esp_free(slotp);
        return -1;
    }

    while (!esp_fetch(slotp))
    {
        esp_get(slotp, "resultado", resultado);
        esp_get(slotp, "ts_bd",     ts_bd);

        time(&now);
        strftime(logbuf, sizeof(logbuf), "%Y-%m-%d %H:%M:%S", localtime(&now));
        printf("[%s] [OK    ] Ciclo %d/%d - BD responde: %s @ %s\n",
            logbuf, ciclo, MAX_CICLOS, resultado, ts_bd);
        fflush(stdout);
    }

    esp_free(slotp);
    return 0;
}

/* -------------------------------------------------- */
/* Main - misma firma que gentabpan                   */
/* -------------------------------------------------- */
int main(int argc, char **argv, char **env)
{
    int    intervalo;
    int    ciclo        = 1;
    int    resultado;
    int    reconexiones = 0;
    char   logbuf[300];
    time_t t_inicio, t_fin;
    char   str_inicio[30], str_fin[30];
    double segundos;

    if (argc != 2) {
        printf("Uso: %s <INTERVALO_SEGS>\n", argv[0]);
        printf("  Ejemplo: %s 300\n", argv[0]);
        printf("  INTERVALO : entre 10 y 3600 segundos\n");
        printf("  Max ciclos: %d\n", MAX_CICLOS);
        exit(1);
    }

    intervalo = atoi(argv[1]);

    if (intervalo < 10 || intervalo > 3600) {
        printf("Error: intervalo debe ser entre 10 y 3600 segundos\n");
        exit(1);
    }

    /* Cabecera igual a gentabpan */
    system("clear");
    printf("%s\n", " simpool - Simulador de Pool Oracle ");
    printf("==============================================\n\n");
    printf("Intervalo       : %d segundos\n", intervalo);
    printf("Max ciclos      : %d\n",          MAX_CICLOS);
    printf("PID             : %d\n\n",         getpid());

    time(&t_inicio);
    strftime(str_inicio, sizeof(str_inicio), "%Y-%m-%d %H:%M:%S", localtime(&t_inicio));
    printf("Inicio          : %s\n\n", str_inicio);

    /*
     * Registrar senales para cierre limpio.
     * fep_catch_sig registra las senales del framework ESP.
     * Adicionalmente registramos SIGINT (Ctrl+C) y SIGTERM
     * directamente con signal() para garantizar el cierre
     * correcto del socket Oracle.
     */
    fep_catch_sig(0, hot_exit);
    signal(SIGINT,  hot_exit);
    signal(SIGTERM, hot_exit);

    sw_logon_database(1);

    log_msg("INFO", "Sesion Oracle establecida. Iniciando ciclos de heartbeat.");
    printf("----------------------------------------------\n");

    /* Loop principal */
    while (ciclo <= MAX_CICLOS)
    {
        sprintf(logbuf, "Ciclo %d/%d - idle por %ds...",
            ciclo, MAX_CICLOS, intervalo);
        log_msg("INFO", logbuf);

        sleep(intervalo);

        resultado = enviar_heartbeat(ciclo);

        if (resultado != 0)
        {
            sprintf(logbuf,
                ">>> CONEXION CAIDA ciclo %d/%d | idle=%ds | reconexion #%d",
                ciclo, MAX_CICLOS, intervalo, ++reconexiones);
            log_msg("ALERTA", logbuf);

            printf("----------------------------------------------\n");
            log_msg("INFO", "Reconectando a Oracle...");
            sw_logon_database(1);
            log_msg("INFO", "Reconexion exitosa. Continuando...");
            printf("----------------------------------------------\n");
        }

        ciclo++;
    }

    /* Resumen final igual a gentabpan */
    time(&t_fin);
    strftime(str_fin, sizeof(str_fin), "%Y-%m-%d %H:%M:%S", localtime(&t_fin));
    segundos = difftime(t_fin, t_inicio);

    printf("\n==============================================\n");
    printf("            RESUMEN DE EJECUCION              \n");
    printf("==============================================\n\n");
    printf("TIEMPOS:\n");
    printf("  Inicio:              %s\n", str_inicio);
    printf("  Fin:                 %s\n", str_fin);
    printf("  Duracion total:      %.0f seg (%.2f min)\n\n", segundos, segundos/60);
    printf("RESULTADO:\n");
    printf("  Ciclos completados:  %d\n",  MAX_CICLOS);
    printf("  Intervalo probado:   %d segundos\n", intervalo);
    printf("  Reconexiones:        %d\n",  reconexiones);
    if (reconexiones == 0)
        printf("  Estado:              SIN CAIDAS - conexion estable\n");
    else
        printf("  Estado:              INESTABLE - revisar red u Oracle\n");
    printf("==============================================\n");

    return 0;
}
