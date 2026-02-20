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

#define MAX_CICLOS      99
#define TIMEOUT_HB      30    /* segundos esperando respuesta del heartbeat */

time_t now;
extern char *argv0;

/* -------------------------------------------------- */
/* Cierre limpio al recibir Ctrl+C                    */
/* -------------------------------------------------- */
static void hot_exit(int n)
{
    time_t t;
    char ts[30];

    time(&t);
    strftime(ts, sizeof(ts), "%Y-%m-%d %H:%M:%S", localtime(&t));

    if (n == SIGINT)
        printf("\n[%s] [INFO  ] Ctrl+C recibido. Cerrando sesion Oracle...\n", ts);
    else
        printf("\n[%s] [INFO  ] Senal %d recibida. Cerrando...\n", ts, n);

    fflush(stdout);
    alarm(0);
    esp_abort();
    printf("[%s] [INFO  ] Sesion cerrada. Saliendo.\n", ts);
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
/* Idle usando sleep en chunks de 1 segundo           */
/* Mas confiable que alarm() con frameworks ESP       */
/* Muestra cuenta regresiva cada 60 segundos          */
/* -------------------------------------------------- */
static void idle_con_progreso(int intervalo, int ciclo)
{
    int restante = intervalo;
    char logbuf[200];

    while (restante > 0) {
        /* Mostrar progreso cada 60s o al inicio */
        if (restante == intervalo || restante % 60 == 0) {
            sprintf(logbuf, "Ciclo %d/%d - faltan %ds...",
                ciclo, MAX_CICLOS, restante);
            log_msg("WAIT  ", logbuf);
        }
        sleep(1);
        restante--;
    }
}

/* -------------------------------------------------- */
/* Valida que la reconexion fue exitosa               */
/* Retorna 0 si OK, -1 si fallo                       */
/* -------------------------------------------------- */
static int validar_reconexion()
{
    char      command[100];
    char      resultado[20];
    ESP_SLOT  slot;
    ESP_SLOTP slotp = &slot;

    memset(command,   '\0', sizeof(command));
    memset(resultado, '\0', sizeof(resultado));

    sprintf(command, "SELECT 'OK' AS resultado FROM DUAL");

    if (esp_read(slotp, "validacion", command, ESP_LOOK)) {
        esp_free(slotp);
        return -1;
    }

    while (!esp_fetch(slotp))
        esp_get(slotp, "resultado", resultado);

    esp_free(slotp);

    if (strcmp(resultado, "OK") == 0)
        return 0;

    return -1;
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
/* Reconexion con validacion                          */
/* Retorna 0 si reconecto OK, -1 si fallo             */
/* -------------------------------------------------- */
static int reconectar(int intento)
{
    char logbuf[200];
    int  max_reintentos = 3;
    int  i;

    for (i = 1; i <= max_reintentos; i++) {
        sprintf(logbuf, "Reconexion intento %d/%d...", i, max_reintentos);
        log_msg("INFO", logbuf);

        sw_logon_database(1);

        /* Validar que la sesion realmente quedo activa */
        if (validar_reconexion() == 0) {
            log_msg("INFO", "Reconexion validada exitosamente.");
            return 0;
        }

        log_msg("ERROR", "Reconexion fallo en validacion. Reintentando en 5s...");
        sleep(5);
    }

    log_msg("ERROR", "No se pudo reconectar tras 3 intentos.");
    return -1;
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

    /* Cabecera */
    system("clear");
    printf("%s\n", " simpool - Simulador de Pool Oracle ");
    printf("==============================================\n\n");
    printf("Intervalo       : %d segundos\n", intervalo);
    printf("Max ciclos      : %d\n",          MAX_CICLOS);
    printf("PID             : %d\n\n",         getpid());

    time(&t_inicio);
    strftime(str_inicio, sizeof(str_inicio), "%Y-%m-%d %H:%M:%S", localtime(&t_inicio));
    printf("Inicio          : %s\n\n", str_inicio);

    /* Senales - ANTES de fep_catch_sig para que no las pise */
    signal(SIGINT,  hot_exit);
    signal(SIGTERM, hot_exit);
    fep_catch_sig(0, hot_exit);

    sw_logon_database(1);

    /* Validar conexion inicial */
    if (validar_reconexion() != 0) {
        log_msg("ERROR", "No se pudo establecer sesion Oracle inicial.");
        exit(1);
    }

    log_msg("INFO", "Sesion Oracle establecida y validada. Iniciando ciclos.");
    printf("----------------------------------------------\n");

    /* Loop principal */
    while (ciclo <= MAX_CICLOS)
    {
        sprintf(logbuf, "Ciclo %d/%d - iniciando idle de %ds...",
            ciclo, MAX_CICLOS, intervalo);
        log_msg("INFO", logbuf);

        /* Idle con progreso cada 60s */
        idle_con_progreso(intervalo, ciclo);

        resultado = enviar_heartbeat(ciclo);

        if (resultado != 0)
        {
            sprintf(logbuf,
                ">>> CONEXION CAIDA ciclo %d/%d | idle=%ds | reconexion #%d",
                ciclo, MAX_CICLOS, intervalo, ++reconexiones);
            log_msg("ALERTA", logbuf);

            printf("----------------------------------------------\n");

            if (reconectar(reconexiones) != 0) {
                log_msg("ERROR", "Reconexion fallida. Abortando.");
                esp_abort();
                exit(1);
            }

            printf("----------------------------------------------\n");
        }

        ciclo++;
    }

    alarm(0);

    /* Resumen final */
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
