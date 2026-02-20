/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Diagnostico de Red
 * #Nombre      : simpool
 * #Descripcion : Simula una conexion TCP persistente tipo pool hacia Oracle.
 *                Mantiene el socket idle N segundos y luego envia un byte
 *                para verificar si la conexion sigue viva.
 *                Objetivo: detectar si un dispositivo de red (firewall/
 *                balanceador) corta conexiones idle antes del intervalo.
 * #Autor       : Luis Alberto Guerrero Romero
 * #Fecha       : 2026-02-20
 *
 * COMPILACION:
 *   make
 *
 * USO:
 *   ./simpool <IP> <PUERTO> [INTERVALO]
 *   ./simpool 10.34.58.101 4584 300
 *   ./simpool 10.34.58.101 4584 120
 *
 * NOTA:
 *   Maximo 99 ciclos por ejecucion.
 *   Si INTERVALO no se especifica, usa 300 segundos por defecto.
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>

#define INTERVALO_DEFAULT  300
#define TIMEOUT_RECV       5
#define MAX_CICLOS         99

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
/* Abre socket TCP hacia host:puerto                  */
/* Retorna el fd del socket o -1 si fallo             */
/* -------------------------------------------------- */
static int conectar(const char *host, int puerto)
{
    int sockfd;
    struct sockaddr_in serv_addr;
    struct timeval tv;
    char logbuf[200];
    int optval = 1;

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        sprintf(logbuf, "Error creando socket: %s", strerror(errno));
        log_msg("ERROR", logbuf);
        return -1;
    }

    tv.tv_sec  = TIMEOUT_RECV;
    tv.tv_usec = 0;
    setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, (char*)&tv, sizeof(tv));
    setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, (char*)&optval, sizeof(optval));

    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port   = htons(puerto);

    if (inet_aton(host, &serv_addr.sin_addr) == 0) {
        sprintf(logbuf, "IP invalida: %s", host);
        log_msg("ERROR", logbuf);
        close(sockfd);
        return -1;
    }

    if (connect(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) < 0) {
        sprintf(logbuf, "No se pudo conectar a %s:%d -> %s", host, puerto, strerror(errno));
        log_msg("ERROR", logbuf);
        close(sockfd);
        return -1;
    }

    /* Mostrar puerto local efimero asignado por el SO */
    {
        struct sockaddr_in local_addr;
        socklen_t local_len = sizeof(local_addr);
        if (getsockname(sockfd, (struct sockaddr*)&local_addr, &local_len) == 0) {
            sprintf(logbuf, "Socket local: %s:%d -> %s:%d",
                inet_ntoa(local_addr.sin_addr), ntohs(local_addr.sin_port),
                host, puerto);
            log_msg("INFO", logbuf);
        }
    }

    return sockfd;
}

/* -------------------------------------------------- */
/* Verifica si el socket sigue vivo                   */
/* Retorna 0 si OK, -1 si la conexion esta muerta     */
/* -------------------------------------------------- */
static int verificar_socket(int sockfd, int ciclo)
{
    char buf[16];
    int n;
    char logbuf[200];

    buf[0] = '\x00';
    n = write(sockfd, buf, 1);
    if (n < 0) {
        sprintf(logbuf, "Ciclo %d - write fallo: %s", ciclo, strerror(errno));
        log_msg("ALERTA", logbuf);
        return -1;
    }

    n = recv(sockfd, buf, sizeof(buf), 0);
    if (n < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            /* Timeout: Oracle no respondio al byte nulo, socket vivo */
            sprintf(logbuf, "Ciclo %d - socket activo", ciclo);
            log_msg("OK", logbuf);
            return 0;
        }
        sprintf(logbuf, "Ciclo %d - recv fallo: %s (conexion cortada)", ciclo, strerror(errno));
        log_msg("ALERTA", logbuf);
        return -1;
    }

    if (n == 0) {
        sprintf(logbuf, "Ciclo %d - conexion cerrada por el servidor (EOF)", ciclo);
        log_msg("ALERTA", logbuf);
        return -1;
    }

    sprintf(logbuf, "Ciclo %d - socket activo, recibidos %d bytes", ciclo, n);
    log_msg("OK", logbuf);
    return 0;
}

/* -------------------------------------------------- */
/* Main                                               */
/* -------------------------------------------------- */
int main(int argc, char **argv)
{
    char  *host;
    int    puerto;
    int    intervalo   = INTERVALO_DEFAULT;
    int    sockfd      = -1;
    int    ciclo       = 1;
    int    resultado;
    int    reconexiones = 0;
    char   logbuf[200];
    time_t t_inicio;

    if (argc < 3) {
        printf("Uso: %s <IP> <PUERTO> [INTERVALO_SEGS]\n", argv[0]);
        printf("  Ejemplo: %s 10.34.58.101 4584 300\n", argv[0]);
        printf("  INTERVALO por defecto: %d segundos\n", INTERVALO_DEFAULT);
        printf("  Maximo de ciclos     : %d\n", MAX_CICLOS);
        exit(1);
    }

    host   = argv[1];
    puerto = atoi(argv[2]);

    if (puerto <= 0 || puerto > 65535) {
        printf("Error: puerto invalido [%s]\n", argv[2]);
        exit(1);
    }

    if (argc >= 4) {
        intervalo = atoi(argv[3]);
        if (intervalo < 10 || intervalo > 3600) {
            printf("Error: intervalo debe ser entre 10 y 3600 segundos\n");
            exit(1);
        }
    }

    printf("==============================================\n");
    printf("  simpool - Simulador de Pool TCP\n");
    printf("  Destino  : %s:%d\n", host, puerto);
    printf("  Intervalo: %d segundos\n", intervalo);
    printf("  Max ciclos: %d\n", MAX_CICLOS);
    printf("  Ctrl+C para salir antes\n");
    printf("==============================================\n\n");

    time(&t_inicio);

    while (ciclo <= MAX_CICLOS) {

        if (sockfd < 0) {
            sprintf(logbuf, "Conectando a %s:%d... (reconexion #%d)", host, puerto, reconexiones);
            log_msg("INFO", logbuf);

            sockfd = conectar(host, puerto);

            if (sockfd < 0) {
                log_msg("ERROR", "Fallo al conectar. Reintentando en 5s...");
                sleep(5);
                continue;
            }

            sprintf(logbuf, "Conexion establecida (fd=%d)", sockfd);
            log_msg("INFO", logbuf);
        }

        sprintf(logbuf, "Ciclo %d/%d - idle por %ds...", ciclo, MAX_CICLOS, intervalo);
        log_msg("INFO", logbuf);
        sleep(intervalo);

        resultado = verificar_socket(sockfd, ciclo);

        if (resultado != 0) {
            sprintf(logbuf,
                ">>> CONEXION CAIDA en ciclo %d/%d tras %ds idle | Reconexiones: %d",
                ciclo, MAX_CICLOS, intervalo, ++reconexiones);
            log_msg("ALERTA", logbuf);

            close(sockfd);
            sockfd = -1;
        }

        ciclo++;
    }

    /* Resumen final */
    printf("\n==============================================\n");
    printf("  RESUMEN FINAL\n");
    printf("  Ciclos completados : %d\n", MAX_CICLOS);
    printf("  Reconexiones       : %d\n", reconexiones);
    printf("  Intervalo probado  : %d segundos\n", intervalo);
    if (reconexiones == 0)
        printf("  Resultado          : SIN CAIDAS - conexion estable\n");
    else
        printf("  Resultado          : INESTABLE - revisar dispositivo de red\n");
    printf("==============================================\n");

    if (sockfd >= 0) close(sockfd);
    return 0;
}
