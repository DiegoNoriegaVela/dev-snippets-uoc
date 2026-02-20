/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Diagnostico de Red
 * #Nombre      : simpool
 * #Descripcion : Simula una conexion TCP persistente tipo pool hacia Oracle.
 *                Realiza un handshake TNS minimo para que Oracle acepte la
 *                conexion como valida, luego la deja idle N segundos y envia
 *                un TNS ping para verificar si sigue viva.
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
#define TIMEOUT_RECV       10
#define MAX_CICLOS         99

/*
 * TNS Connect packet minimo valido.
 * Header TNS (8 bytes):
 *   [0-1] longitud total (big-endian)
 *   [2-3] checksum paquete = 0
 *   [4]   tipo: 0x01 = CONNECT
 *   [5]   flags = 0
 *   [6-7] header checksum = 0
 */
static unsigned char TNS_CONNECT[] = {
    0x00, 0x57,              /* longitud total: 87 bytes         */
    0x00, 0x00,              /* checksum paquete: 0              */
    0x01,                    /* tipo: CONNECT                    */
    0x00,                    /* flags                            */
    0x00, 0x00,              /* header checksum: 0               */
    0x01, 0x38,              /* version TNS: 314 (Oracle 11g+)  */
    0x01, 0x2C,              /* version minima compatible        */
    0x00, 0x00,              /* opciones de servicio             */
    0x00, 0x00,              /* SDU size                         */
    0x00, 0x00,              /* TDU size                         */
    0x7F, 0xFF,              /* caracteristicas protocolo        */
    0x7F, 0x08,              /* caracteristicas protocolo 2      */
    0x00, 0x00,              /* bytes de datos connect           */
    0x00, 0x4F,              /* longitud connect data            */
    0x00, 0x00,              /* offset connect data              */
    0x00, 0x00, 0x00, 0x00, /* max recv data                    */
    0x00, 0x00, 0x00, 0x00, /* ANO data                         */
    0x00, 0x00,              /* flags 2                          */
    /* Connect data: descripcion TNS minima */
    0x28, 0x44, 0x45, 0x53, 0x43, 0x52, 0x49, 0x50,
    0x54, 0x49, 0x4F, 0x4E, 0x3D, 0x28, 0x43, 0x4F,
    0x4E, 0x4E, 0x45, 0x43, 0x54, 0x5F, 0x44, 0x41,
    0x54, 0x41, 0x3D, 0x28, 0x53, 0x45, 0x52, 0x56,
    0x49, 0x43, 0x45, 0x5F, 0x4E, 0x41, 0x4D, 0x45,
    0x3D, 0x58, 0x45, 0x29, 0x29, 0x29
};

/*
 * TNS ping: paquete DATA vacio (8 bytes solo header).
 * Lo usan los drivers para validar la conexion sin ejecutar SQL.
 */
static unsigned char TNS_PING[] = {
    0x00, 0x08,   /* longitud: 8 bytes */
    0x00, 0x00,   /* checksum: 0       */
    0x06,         /* tipo: DATA        */
    0x00,         /* flags             */
    0x00, 0x00    /* header checksum   */
};

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
/* Envia buffer al socket                             */
/* Retorna 0 si OK, -1 si fallo                       */
/* -------------------------------------------------- */
static int enviar(int sockfd, unsigned char *datos, int len, const char *desc)
{
    int n;
    char logbuf[200];
    n = write(sockfd, datos, len);
    if (n < 0) {
        sprintf(logbuf, "write(%s) fallo: %s", desc, strerror(errno));
        log_msg("ERROR", logbuf);
        return -1;
    }
    if (n != len) {
        sprintf(logbuf, "write(%s) incompleto: %d/%d bytes", desc, n, len);
        log_msg("ERROR", logbuf);
        return -1;
    }
    return 0;
}

/* -------------------------------------------------- */
/* Realiza el handshake TNS inicial con Oracle        */
/* Retorna 0 si OK, -1 si fallo                       */
/* -------------------------------------------------- */
static int tns_handshake(int sockfd)
{
    char buf[512];
    int  n;
    char logbuf[200];

    if (enviar(sockfd, TNS_CONNECT, sizeof(TNS_CONNECT), "TNS_CONNECT") < 0)
        return -1;

    log_msg("INFO", "TNS Connect enviado, esperando respuesta...");

    n = recv(sockfd, buf, sizeof(buf), 0);
    if (n <= 0) {
        sprintf(logbuf, "Sin respuesta al TNS Connect: %s",
            n == 0 ? "EOF" : strerror(errno));
        log_msg("ERROR", logbuf);
        return -1;
    }

    sprintf(logbuf, "Respuesta TNS: %d bytes, tipo=0x%02X", n, (unsigned char)buf[4]);
    log_msg("INFO", logbuf);

    switch ((unsigned char)buf[4]) {
        case 0x02:
            log_msg("INFO", "TNS ACCEPT: Oracle acepto la conexion");
            return 0;

        case 0x04:
            /*
             * REFUSE: Oracle no reconoce el servicio del connect data
             * pero la conexion TCP existe y llego al servidor.
             * Para efectos de la prueba de red esto es suficiente.
             */
            log_msg("ALERTA", "TNS REFUSE: Oracle rechazo el servicio");
            log_msg("INFO",   "TCP llego al servidor, la red no corto la conexion");
            log_msg("INFO",   "Continuando prueba de idle sobre este socket...");
            return 0;

        case 0x05:
            log_msg("INFO", "TNS REDIRECT recibido");
            return -1;

        case 0x0B:
            log_msg("INFO", "TNS RESEND: reenviando connect...");
            if (enviar(sockfd, TNS_CONNECT, sizeof(TNS_CONNECT), "TNS_CONNECT_RESEND") < 0)
                return -1;
            n = recv(sockfd, buf, sizeof(buf), 0);
            if (n > 0 && (unsigned char)buf[4] == 0x02) {
                log_msg("INFO", "TNS ACCEPT tras RESEND");
                return 0;
            }
            return -1;

        default:
            sprintf(logbuf, "TNS tipo desconocido: 0x%02X - continuando",
                (unsigned char)buf[4]);
            log_msg("INFO", logbuf);
            return 0;
    }
}

/* -------------------------------------------------- */
/* Abre socket TCP y realiza handshake TNS            */
/* Retorna fd del socket o -1 si fallo               */
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

    /* Mostrar tupla completa: puerto local efimero -> destino */
    {
        struct sockaddr_in local_addr;
        socklen_t local_len = sizeof(local_addr);
        if (getsockname(sockfd, (struct sockaddr*)&local_addr, &local_len) == 0) {
            sprintf(logbuf, "Socket: %s:%d -> %s:%d",
                inet_ntoa(local_addr.sin_addr), ntohs(local_addr.sin_port),
                host, puerto);
            log_msg("INFO", logbuf);
        }
    }

    /* Handshake TNS para que Oracle vea una conexion real */
    if (tns_handshake(sockfd) < 0) {
        log_msg("ERROR", "Handshake TNS fallo");
        close(sockfd);
        return -1;
    }

    return sockfd;
}

/* -------------------------------------------------- */
/* Envia TNS ping y verifica si el socket sigue vivo  */
/* Retorna 0 si OK, -1 si conexion muerta             */
/* -------------------------------------------------- */
static int verificar_socket(int sockfd, int ciclo)
{
    char buf[64];
    int  n;
    char logbuf[200];

    if (enviar(sockfd, TNS_PING, sizeof(TNS_PING), "TNS_PING") < 0) {
        sprintf(logbuf, "Ciclo %d - ping fallo al enviar (conexion cortada)", ciclo);
        log_msg("ALERTA", logbuf);
        return -1;
    }

    n = recv(sockfd, buf, sizeof(buf), 0);

    if (n < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            /* Timeout: Oracle no respondio al ping pero el socket sigue vivo */
            sprintf(logbuf, "Ciclo %d - socket activo (Oracle no respondio al ping, normal)", ciclo);
            log_msg("OK", logbuf);
            return 0;
        }
        sprintf(logbuf, "Ciclo %d - error: %s (conexion cortada por red)", ciclo, strerror(errno));
        log_msg("ALERTA", logbuf);
        return -1;
    }

    if (n == 0) {
        sprintf(logbuf, "Ciclo %d - EOF: conexion cerrada por servidor", ciclo);
        log_msg("ALERTA", logbuf);
        return -1;
    }

    sprintf(logbuf, "Ciclo %d - socket activo, Oracle respondio %d bytes (tipo=0x%02X)",
        ciclo, n, (unsigned char)buf[4]);
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
    int    intervalo    = INTERVALO_DEFAULT;
    int    sockfd       = -1;
    int    ciclo        = 1;
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
    printf("  simpool - Simulador de Pool TCP + TNS\n");
    printf("  Destino   : %s:%d\n", host, puerto);
    printf("  Intervalo : %d segundos\n", intervalo);
    printf("  Max ciclos: %d\n", MAX_CICLOS);
    printf("  Ctrl+C para salir antes\n");
    printf("==============================================\n\n");

    time(&t_inicio);

    while (ciclo <= MAX_CICLOS) {

        if (sockfd < 0) {
            sprintf(logbuf, "Conectando a %s:%d... (reconexion #%d)",
                host, puerto, reconexiones);
            log_msg("INFO", logbuf);

            sockfd = conectar(host, puerto);

            if (sockfd < 0) {
                log_msg("ERROR", "Fallo al conectar. Reintentando en 5s...");
                sleep(5);
                continue;
            }

            sprintf(logbuf, "Conexion lista: TCP + TNS handshake completado (fd=%d)", sockfd);
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

    printf("\n==============================================\n");
    printf("  RESUMEN FINAL\n");
    printf("  Ciclos completados : %d\n", MAX_CICLOS);
    printf("  Reconexiones       : %d\n", reconexiones);
    printf("  Intervalo probado  : %d segundos\n", intervalo);
    if (reconexiones == 0)
        printf("  Resultado          : SIN CAIDAS - conexion estable\n");
    else
        printf("  Resultado          : INESTABLE - revisar red u Oracle\n");
    printf("==============================================\n");

    if (sockfd >= 0) close(sockfd);
    return 0;
}
