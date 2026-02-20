/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Diagnostico de Red
 * #Nombre      : simpool
 * #Descripcion : Simula conexion persistente tipo pool hacia Oracle.
 *                Envia un heartbeat cada N segundos para detectar si
 *                un dispositivo de red corta conexiones idle.
 *                Muestra IP/puerto local y remoto de la conexion.
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
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <net/if.h>
#include "pt_master.h"

#define MAX_CICLOS 99

/* Variables globales igual que gentabpan */
time_t now;
extern char *argv0;

/* -------------------------------------------------- */
/* Obtiene IP local del servidor actual               */
/* Busca la primera interfaz activa distinta de lo    */
/* -------------------------------------------------- */
static void obtener_ip_local(char *ip_out, int maxlen)
{
    FILE *fp;
    char  buf[256];
    char  iface[32];
    char  ip[32];

    strncpy(ip_out, "desconocida", maxlen);

    /*
     * En AIX: ifconfig -a muestra todas las interfaces.
     * Buscamos la primera inet que no sea loopback.
     */
    fp = popen("ifconfig -a 2>/dev/null | grep 'inet ' | grep -v '127.0.0.1' | head -1", "r");
    if (fp != NULL) {
        if (fgets(buf, sizeof(buf), fp) != NULL) {
            /* formato AIX: "        inet 10.34.58.69 netmask ..." */
            if (sscanf(buf, " inet %31s", ip) == 1) {
                strncpy(ip_out, ip, maxlen);
            }
        }
        pclose(fp);
    }
}

/* -------------------------------------------------- */
/* Obtiene puerto efimero local de la conexion Oracle */
/* Parsea netstat buscando la conexion al puerto      */
/* destino y extrae el puerto local                   */
/* -------------------------------------------------- */
static int obtener_puerto_local(const char *ip_destino, const char *puerto_destino)
{
    FILE *fp;
    char  cmd[200];
    char  buf[512];
    int   puerto = 0;
    char  local[64], remoto[64];
    char  estado[32];

    /*
     * netstat -an en AIX muestra:
     * tcp  0  0  10.34.58.69.45422  10.34.58.101.4584  ESTABLISHED
     * El formato usa puntos como separador entre IP y puerto.
     */
    sprintf(cmd,
        "netstat -an 2>/dev/null | grep '%s.%s' | grep ESTABLISHED | head -1",
        ip_destino, puerto_destino);

    fp = popen(cmd, "r");
    if (fp != NULL) {
        if (fgets(buf, sizeof(buf), fp) != NULL) {
            /* Parsear: ignorar proto bytes bytes local remoto estado */
            if (sscanf(buf, "%*s %*s %*s %63s %63s %31s", local, remoto, estado) >= 2) {
                /*
                 * local tiene formato: 10.34.58.69.45422
                 * El puerto es lo que viene despues del ultimo punto
                 */
                char *ultimo_punto = strrchr(local, '.');
                if (ultimo_punto != NULL) {
                    puerto = atoi(ultimo_punto + 1);
                }
            }
        }
        pclose(fp);
    }

    return puerto;
}

/* -------------------------------------------------- */
/* Manejo de senal de salida - igual que gentabpan    */
/* -------------------------------------------------- */
static void hot_exit(int n)
{
    printf("Cancelled with [%d]\n", n);
    esp_abort();
    exit(0);
}

/* -------------------------------------------------- */
/* Heartbeat: SELECT FROM DUAL                        */
/* Mismo patron exacto que gentabpan: read/fetch/free */
/* Retorna 0 si OK, -1 si fallo                       */
/* -------------------------------------------------- */
static int enviar_heartbeat(int ciclo, const char *ip_dest, const char *pto_dest)
{
    char      command[600];
    char      resultado[50];
    char      ts_bd[30];
    char      logbuf[300];
    int       puerto_local;
    char      ip_local[32];
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
        fprintf(stderr, "Heartbeat ciclo %d - ESP_READ ERROR: %s\n",
            ciclo, (char*)*Last_error);
        esp_free(slotp);
        return -1;
    }

    while (!esp_fetch(slotp))
    {
        esp_get(slotp, "resultado", resultado);
        esp_get(slotp, "ts_bd",     ts_bd);

        /* Obtener info de red de la conexion activa */
        obtener_ip_local(ip_local, sizeof(ip_local));
        puerto_local = obtener_puerto_local(ip_dest, pto_dest);

        time(&now);
        strftime(logbuf, sizeof(logbuf), "%Y-%m-%d %H:%M:%S", localtime(&now));

        printf("[%s] [OK    ] Ciclo %d/%d - BD=%s@%s | Red: %s:%d -> %s:%s\n",
            logbuf,
            ciclo, MAX_CICLOS,
            resultado, ts_bd,
            ip_local, puerto_local,
            ip_dest, pto_dest);
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
    char   ip_local[32];
    int    puerto_local;
    time_t t_inicio, t_fin;
    char   str_inicio[30], str_fin[30];
    double segundos;

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

    /* Obtener info de red antes de conectar */
    obtener_ip_local(ip_local, sizeof(ip_local));

    /* Cabecera igual a gentabpan */
    system("clear");
    printf("%s\n", " simpool - Simulador de Pool Oracle ");
    printf("==============================================\n\n");
    printf("Destino         : %s:%s\n",   host, puerto_str);
    printf("IP Local        : %s\n",      ip_local);
    printf("Intervalo       : %d segundos\n", intervalo);
    printf("Max ciclos      : %d\n",      MAX_CICLOS);
    printf("PID             : %d\n\n",    getpid());

    time(&t_inicio);
    strftime(str_inicio, sizeof(str_inicio), "%Y-%m-%d %H:%M:%S", localtime(&t_inicio));
    printf("Inicio          : %s\n\n", str_inicio);

    /* Igual que gentabpan */
    fep_catch_sig(0, hot_exit);
    sw_logon_database(1);

    /* Puerto efimero asignado luego de conectar */
    puerto_local = obtener_puerto_local(host, puerto_str);
    sprintf(logbuf, "Sesion Oracle establecida | Tunel: %s:%d -> %s:%s",
        ip_local, puerto_local, host, puerto_str);
    log_msg("INFO", logbuf);
    printf("----------------------------------------------\n");

    /* Loop principal */
    while (ciclo <= MAX_CICLOS)
    {
        sprintf(logbuf, "Ciclo %d/%d - conexion idle por %ds...",
            ciclo, MAX_CICLOS, intervalo);
        log_msg("INFO", logbuf);

        sleep(intervalo);

        resultado = enviar_heartbeat(ciclo, host, puerto_str);

        if (resultado != 0)
        {
            sprintf(logbuf,
                ">>> CONEXION CAIDA ciclo %d/%d | idle=%ds | reconexion #%d",
                ciclo, MAX_CICLOS, intervalo, ++reconexiones);
            log_msg("ALERTA", logbuf);

            printf("----------------------------------------------\n");
            log_msg("INFO", "Reconectando a Oracle...");
            sw_logon_database(1);

            /* Nuevo puerto efimero tras reconectar */
            puerto_local = obtener_puerto_local(host, puerto_str);
            sprintf(logbuf, "Reconexion OK | Nuevo tunel: %s:%d -> %s:%s",
                ip_local, puerto_local, host, puerto_str);
            log_msg("INFO", logbuf);
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
    printf("  Destino:             %s:%s\n", host, puerto_str);
    printf("  IP Local:            %s\n",  ip_local);
    if (reconexiones == 0)
        printf("  Estado:              SIN CAIDAS - conexion estable\n");
    else
        printf("  Estado:              INESTABLE - revisar red u Oracle\n");
    printf("==============================================\n");

    return 0;
}
