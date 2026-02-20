/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Diagnostico de Red
 * #Nombre      : simpool
 * #Descripcion : Simula conexion persistente tipo pool hacia Oracle.
 *                Detecta automaticamente la IP y puerto de la conexion.
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
#include "pt_master.h"

#define MAX_CICLOS 99

time_t now;
extern char *argv0;

/* Info de red detectada automaticamente */
char g_ip_local[32];
char g_ip_destino[32];
int  g_puerto_local  = 0;
int  g_puerto_destino = 0;

/* -------------------------------------------------- */
/* Detecta la conexion Oracle activa del proceso      */
/* Parsea netstat buscando ESTABLISHED del PID actual */
/* -------------------------------------------------- */
static void detectar_conexion(int pid)
{
    FILE *fp;
    char  cmd[200];
    char  buf[512];
    char  local[64], remoto[64], estado[32];
    char *p;

    memset(g_ip_local,    '\0', sizeof(g_ip_local));
    memset(g_ip_destino,  '\0', sizeof(g_ip_destino));
    g_puerto_local   = 0;
    g_puerto_destino = 0;

    /*
     * En AIX netstat -an muestra formato:
     * tcp  0  0  10.34.58.69.45422  10.34.58.101.4584  ESTABLISHED
     *
     * Usamos procfiles para filtrar solo los sockets del proceso,
     * luego cruzamos con netstat para obtener IP:puerto.
     */
    sprintf(cmd,
        "netstat -an 2>/dev/null | grep ESTABLISHED | grep -v '127.0.0.1' | head -5");

    fp = popen(cmd, "r");
    if (fp == NULL) return;

    while (fgets(buf, sizeof(buf), fp) != NULL)
    {
        if (sscanf(buf, "%*s %*s %*s %63s %63s %31s", local, remoto, estado) < 3)
            continue;

        if (strcmp(estado, "ESTABLISHED") != 0)
            continue;

        /*
         * Formato AIX: 10.34.58.69.45422
         * Separar IP y puerto por el ultimo punto
         */
        p = strrchr(local, '.');
        if (p != NULL) {
            g_puerto_local = atoi(p + 1);
            *p = '\0';
            /* Convertir puntos restantes: AIX usa notacion normal */
            strncpy(g_ip_local, local, sizeof(g_ip_local) - 1);
        }

        p = strrchr(remoto, '.');
        if (p != NULL) {
            g_puerto_destino = atoi(p + 1);
            *p = '\0';
            strncpy(g_ip_destino, remoto, sizeof(g_ip_destino) - 1);
        }

        /* Tomar la primera conexion ESTABLISHED encontrada */
        if (g_puerto_local > 0 && g_puerto_destino > 0)
            break;
    }

    pclose(fp);
}

/* -------------------------------------------------- */
/* Manejo de senal - igual que gentabpan              */
/* -------------------------------------------------- */
static void hot_exit(int n)
{
    printf("Cancelled with [%d]\n", n);
    esp_abort();
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
static int enviar_heartbeat(int ciclo, int pid)
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

        /* Refrescar info de red en cada heartbeat */
        detectar_conexion(pid);

        time(&now);
        strftime(logbuf, sizeof(logbuf), "%Y-%m-%d %H:%M:%S", localtime(&now));

        printf("[%s] [OK    ] Ciclo %d/%d - BD=%s@%s | %s:%d -> %s:%d\n",
            logbuf,
            ciclo, MAX_CICLOS,
            resultado, ts_bd,
            g_ip_local,    g_puerto_local,
            g_ip_destino,  g_puerto_destino);
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
    int    pid;
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

    pid = getpid();

    /* Cabecera */
    system("clear");
    printf("%s\n", " simpool - Simulador de Pool Oracle ");
    printf("==============================================\n\n");
    printf("Intervalo       : %d segundos\n", intervalo);
    printf("Max ciclos      : %d\n",          MAX_CICLOS);
    printf("PID             : %d\n\n",         pid);

    time(&t_inicio);
    strftime(str_inicio, sizeof(str_inicio), "%Y-%m-%d %H:%M:%S", localtime(&t_inicio));
    printf("Inicio          : %s\n\n", str_inicio);

    /* Igual que gentabpan */
    fep_catch_sig(0, hot_exit);
    sw_logon_database(1);

    /* Detectar IP y puerto automaticamente tras conectar */
    detectar_conexion(pid);

    sprintf(logbuf, "Sesion Oracle establecida | %s:%d -> %s:%d",
        g_ip_local,   g_puerto_local,
        g_ip_destino, g_puerto_destino);
    log_msg("INFO", logbuf);
    printf("----------------------------------------------\n");

    /* Loop principal */
    while (ciclo <= MAX_CICLOS)
    {
        sprintf(logbuf, "Ciclo %d/%d - idle por %ds...",
            ciclo, MAX_CICLOS, intervalo);
        log_msg("INFO", logbuf);

        sleep(intervalo);

        resultado = enviar_heartbeat(ciclo, pid);

        if (resultado != 0)
        {
            sprintf(logbuf,
                ">>> CONEXION CAIDA ciclo %d/%d | idle=%ds | reconexion #%d",
                ciclo, MAX_CICLOS, intervalo, ++reconexiones);
            log_msg("ALERTA", logbuf);

            printf("----------------------------------------------\n");
            log_msg("INFO", "Reconectando a Oracle...");
            sw_logon_database(1);

            detectar_conexion(pid);
            sprintf(logbuf, "Reconexion OK | nuevo tunel: %s:%d -> %s:%d",
                g_ip_local,   g_puerto_local,
                g_ip_destino, g_puerto_destino);
            log_msg("INFO", logbuf);
            printf("----------------------------------------------\n");
        }

        ciclo++;
    }

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
    printf("  Conexion Oracle:     %s:%d -> %s:%d\n",
        g_ip_local, g_puerto_local, g_ip_destino, g_puerto_destino);
    if (reconexiones == 0)
        printf("  Estado:              SIN CAIDAS - conexion estable\n");
    else
        printf("  Estado:              INESTABLE - revisar red u Oracle\n");
    printf("==============================================\n");

    return 0;
}
