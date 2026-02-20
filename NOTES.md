/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Interface
 * #Nombre      : sim_pool
 * #Descripcion : Simula conexion persistente tipo pool hacia Oracle.
 *                Envia heartbeat cada N segundos para detectar
 *                si un dispositivo de red corta conexiones idle.
 * #Autor       : Luis Alberto Guerrero Romero
 * #Fecha       : 2026-02-20
 *
 * USO:
 *   ./sim_pool          -> Intervalo por defecto (300s)
 *   ./sim_pool 120      -> Heartbeat cada 120 segundos
 *   ./sim_pool 60       -> Heartbeat cada 60 segundos
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include "pt_master.h"

#define INTERVALO_DEFAULT 300

/* -------------------------------------------------- */
/* Log solo a pantalla                                */
/* -------------------------------------------------- */
static void log_msg(const char *nivel, const char *msg)
{
    time_t now;
    char ts[30];

    time(&now);
    strftime(ts, sizeof(ts), "%Y-%m-%d %H:%M:%S", localtime(&now));

    printf("[%s] [%s] %s\n", ts, nivel, msg);
    fflush(stdout);
}

/* -------------------------------------------------- */
/* Heartbeat: ejecuta SELECT FROM DUAL               */
/* Retorna 0 si OK, -1 si fallo                      */
/* -------------------------------------------------- */
static int enviar_heartbeat(int ciclo)
{
    ESP_SLOT slot;
    ESP_SLOTP slotp = &slot;
    char command[200];
    char resultado[50];
    char logbuf[200];

    memset(command,   '\0', sizeof(command));
    memset(resultado, '\0', sizeof(resultado));

    sprintf(command,
        "SELECT 'POOL_OK_' || TO_CHAR(SYSDATE,'HH24:MI:SS') AS resultado "
        "FROM DUAL");

    if (esp_read(slotp, "heartbeat", command, ESP_LOOK)) {
        sprintf(logbuf, "Ciclo %d - ESP_READ ERROR: %s", ciclo, (char*)*Last_error);
        log_msg("ERROR", logbuf);
        esp_free(slotp);
        return -1;
    }

    if (!esp_fetch(slotp)) {
        esp_get(slotp, "resultado", resultado);
        sprintf(logbuf, "Ciclo %d - BD responde: %s", ciclo, resultado);
        log_msg("OK", logbuf);
    }

    esp_free(slotp);
    return 0;
}

/* -------------------------------------------------- */
/* Main                                               */
/* -------------------------------------------------- */
int main(int argc, char **argv)
{
    int intervalo = INTERVALO_DEFAULT;
    int ciclo     = 1;
    int resultado;
    int reconexiones = 0;
    char logbuf[200];
    time_t t_inicio, t_caida;
    char ts_caida[30];

    if (argc >= 2) {
        intervalo = atoi(argv[1]);
        if (intervalo < 10 || intervalo > 3600) {
            printf("Error: intervalo debe ser entre 10 y 3600 segundos\n");
            printf("Uso: %s [segundos]\n", argv[0]);
            exit(1);
        }
    }

    printf("==============================================\n");
    printf("  sim_pool - Simulador de Pool TCP Oracle\n");
    printf("  Intervalo de heartbeat: %d segundos\n", intervalo);
    printf("  Ctrl+C para salir\n");
    printf("==============================================\n\n");

    time(&t_inicio);

    log_msg("INFO", "Conectando a Oracle...");
    sw_logon_database(1);
    log_msg("INFO", "Conexion establecida. Iniciando ciclos.");

    while (1) {
        sprintf(logbuf, "Ciclo %d - idle por %d segundos...", ciclo, intervalo);
        log_msg("INFO", logbuf);

        sleep(intervalo);

        resultado = enviar_heartbeat(ciclo);

        if (resultado != 0) {
            time(&t_caida);
            strftime(ts_caida, sizeof(ts_caida), "%Y-%m-%d %H:%M:%S", localtime(&t_caida));

            sprintf(logbuf,
                "CONEXION CAIDA en ciclo %d tras %ds idle - reconectando... (reconexion #%d)",
                ciclo, intervalo, ++reconexiones);
            log_msg("ALERTA", logbuf);

            sw_logon_database(1);
            log_msg("INFO", "Reconexion exitosa. Continuando...");
        }

        ciclo++;
    }

    return 0;
}
