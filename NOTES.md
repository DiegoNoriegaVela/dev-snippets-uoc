/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Diagnostico de Red
 * #Nombre      : simpool
 * #Descripcion : Simula conexion persistente tipo pool hacia Oracle.
 *                Envia un heartbeat cada N segundos para detectar si
 *                un dispositivo de red corta conexiones idle.
 * #Autor       : Luis Alberto Guerrero Romero
 * #Fecha       : 2026-02-20
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

#define MAX_CICLOS 99

/* Variables globales igual que gentabpan */
time_t now;
extern char *argv0;

/* Variables del heartbeat */
char hb_account[30];
char hb_pan[50];

/* -------------------------------------------------- */
/* Manejo de señal de salida - igual que gentabpan    */
/* -------------------------------------------------- */
static void hot_exit(int n)
{
    printf("Cancelled with [%d]\n", n);
    esp_abort();
    exit(0);
}

/* -------------------------------------------------- */
/* Heartbeat: un solo registro de panacct             */
/* Mismo patron exacto que read_card en gentabpan     */
/* Retorna 0 si OK, -1 si fallo                       */
/* -------------------------------------------------- */
static int enviar_heartbeat(int ciclo)
{
    char command[600];
    char logbuf[300];
    ESP_SLOT slot;
    ESP_SLOTP slotp = &slot;

    memset(command,     '\0', sizeof(command));
    memset(hb_account,  '\0', sizeof(hb_account));
    memset(hb_pan,      '\0', sizeof(hb_pan));

    /* SELECT minimo sobre panacct, mismo filtro que gentabpan */
    sprintf(command,
        "SELECT pan, account FROM panacct "
        "WHERE (pan IS NOT NULL) "
        "AND pan >= ' ' AND instid >= ' ' AND subinstid >= ' ' AND account >= ' ' "
        "AND ROWNUM = 1");

    cc_send("Heartbeat ciclo %d [%s]\n", ciclo, command);

    if (esp_read(slotp, "card", command, ESP_LOOK))
    {
        cc_send("ESP_READ ERROR: [%s]", (char*)*Last_error);
        fprintf(stderr, "Heartbeat ciclo %d - ESP_READ ERROR: %s\n",
            ciclo, (char*)*Last_error);
        esp_free(slotp);
        return -1;
    }

    while (!esp_fetch(slotp))
    {
        esp_get(slotp, "account", hb_account);
        esp_get(slotp, "pan",     hb_pan);

        time(&now);
        printf("[%s] [OK    ] Ciclo %d/%d - BD responde OK | account=%.8s...\n",
            ctime_r(&now, logbuf) ? strtok(logbuf, "\n") : "??:??:??",
            ciclo, MAX_CICLOS, hb_account);
        fflush(stdout);
    }

    esp_free(slotp);
    return 0;
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
/* Main - misma firma que gentabpan                   */
/* -------------------------------------------------- */
int main(int argc, char **argv, char **env)
{
    char  *host;
    char  *puerto_str;
    int    intervalo;
    int    ciclo        = 1;
    int    resultado;
    int    reconexiones = 0;
    char   logbuf[300];
    time_t t_inicio, t_fin;
    char   str_inicio[30], str_fin[30];
    double segundos;

    /* Validar argumentos */
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

    /* Cabecera igual a gentabpan */
    system("clear");
    printf("%s\n", " simpool - Simulador de Pool Oracle ");
    printf("==============================================\n\n");
    printf("Destino  : %s:%s\n",   host, puerto_str);
    printf("Intervalo: %d segundos\n", intervalo);
    printf("Max ciclos: %d\n",     MAX_CICLOS);
    printf("PID: %d\n\n",          getpid());

    /* Registro de tiempo de inicio */
    time(&t_inicio);
    strftime(str_inicio, sizeof(str_inicio), "%Y-%m-%d %H:%M:%S", localtime(&t_inicio));
    printf("Inicio: %s\n\n", str_inicio);

    /* Conectar a Oracle - igual que gentabpan */
    fep_catch_sig(0, hot_exit);
    sw_logon_database(1);

    log_msg("INFO", "Sesion Oracle establecida. Iniciando ciclos de heartbeat.");
    printf("----------------------------------------------\n");

    /* Loop principal */
    while (ciclo <= MAX_CICLOS)
    {
        sprintf(logbuf, "Ciclo %d/%d - conexion idle por %ds...",
            ciclo, MAX_CICLOS, intervalo);
        log_msg("INFO", logbuf);

        /* Conexion queda idle N segundos, igual que el pool real */
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

    /* Resumen final - igual a gentabpan */
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
    printf("  Ciclos completados:  %d\n", MAX_CICLOS);
    printf("  Intervalo probado:   %d segundos\n", intervalo);
    printf("  Reconexiones:        %d\n", reconexiones);
    if (reconexiones == 0)
        printf("  Estado:              SIN CAIDAS - conexion estable\n");
    else
        printf("  Estado:              INESTABLE - revisar red u Oracle\n");
    printf("==============================================\n");

    return 0;
}
