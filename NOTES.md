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
 *   make -f Makefile.simpool
 *
 * USO:
 *   ./simpool <IP> <PUERTO> [INTERVALO]
 *   ./simpool 10.34.58.101 4584 300
 *   ./simpool 10.34.58.101 4584 120
 *
 * NOTA:
 *   Si INTERVALO no se especifica, usa 300 segundos por defecto.
 *   Bajar el intervalo progresivamente para encontrar el umbral
 *   exacto del timeout del dispositivo de red.
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
#define TIMEOUT_CONEXION   10    /* segundos para establecer conexion */
#define TIMEOUT_RECV       5     /* segundos para esperar respuesta   */

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

    /* Crear socket */
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        sprintf(logbuf, "Error creando socket: %s", strerror(errno));
        log_msg("ERROR", logbuf);
        return -1;
    }

    /* Timeout de recv para detectar socket muerto */
    tv.tv_sec  = TIMEOUT_RECV;
    tv.tv_usec = 0;
    setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, (char*)&tv, sizeof(tv));

    /* Activar TCP keepalive del sistema operativo */
    setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, (char*)&optval, sizeof(optval));

    /* Armar direccion destino */
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family      = AF_INET;
    serv_addr.sin_port        = htons(puerto);

    if (inet_aton(host, &serv_addr.sin_addr) == 0) {
        sprintf(logbuf, "IP invalida: %s", host);
        log_msg("ERROR", logbuf);
        close(sockfd);
        return -1;
    }

    /* Conectar */
    if (connect(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) < 0) {
        sprintf(logbuf, "No se pudo conectar a %s:%d -> %s", host, puerto, strerror(errno));
        log_msg("ERROR", logbuf);
        close(sockfd);
        return -1;
    }

    return sockfd;
}

/* -------------------------------------------------- */
/* Verifica si el socket sigue vivo                   */
/* Intenta escribir un byte y leer la respuesta       */
/* Retorna 0 si OK, -1 si la conexion esta muerta     */
/* -------------------------------------------------- */
static int verificar_socket(int sockfd, int ciclo)
{
    char buf[16];
    int n;
    char logbuf[200];

    /* Enviar byte de prueba */
    buf[0] = '\x00';
    n = write(sockfd, buf, 1);
    if (n < 0) {
        sprintf(logbuf, "Ciclo %d - write fallo: %s (conexion cortada por red)", ciclo, strerror(errno));
        log_msg("ALERTA", logbuf);
        return -1;
    }

    /*
     * Intentar leer respuesta con timeout.
     * Oracle no responde a bytes nulos pero si el socket fue
     * reseteado por el firewall, recv retorna error inmediatamente.
     * Si hace timeout, el socket sigue vivo (silencio esperado de Oracle).
     */
    n = recv(sockfd, buf, sizeof(buf), 0);
    if (n < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            /* Timeout de recv: Oracle no respondio pero socket vivo */
            sprintf(logbuf, "Ciclo %d - socket activo (Oracle no responde al byte nulo, esperado)", ciclo);
            log_msg("OK", logbuf);
            return 0;
        }
        /* Error real: conexion muerta */
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
    char      *host;
    int        puerto;
    int        intervalo = INTERVALO_DEFAULT;
    int        sockfd    = -1;
    int        ciclo     = 1;
    int        resultado;
    int        reconexiones = 0;
    char       logbuf[200];
    time_t     t_inicio;

    /* Validar argumentos */
    if (argc < 3) {
        printf("Uso: %s <IP> <PUERTO> [INTERVALO_SEGS]\n", argv[0]);
        printf("  Ejemplo: %s 10.34.58.101 4584 300\n", argv[0]);
        printf("  INTERVALO por defecto: %d segundos\n", INTERVALO_DEFAULT);
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
    printf("  Ctrl+C para salir\n");
    printf("==============================================\n\n");

    time(&t_inicio);

    /* Loop principal: conectar -> idle -> verificar -> repetir */
    while (1) {

        /* Conectar si no hay socket activo */
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

        /* Dejar la conexion idle N segundos (igual que un pool real) */
        sprintf(logbuf, "Ciclo %d - conexion idle por %ds...", ciclo, intervalo);
        log_msg("INFO", logbuf);
        sleep(intervalo);

        /* Verificar si el socket sobrevivio el idle */
        resultado = verificar_socket(sockfd, ciclo);

        if (resultado != 0) {
            /* Conexion muerta: cerrar y reconectar en siguiente iteracion */
            sprintf(logbuf,
                ">>> CONEXION CAIDA en ciclo %d tras %ds idle | Total reconexiones: %d",
                ciclo, intervalo, ++reconexiones);
            log_msg("ALERTA", logbuf);

            close(sockfd);
            sockfd = -1;
        }

        ciclo++;
    }

    return 0;
}

# ==============================================
# Makefile.simpool
# Compilacion de simpool para AIX
#
# Uso:
#   make -f Makefile.simpool        -> compila
#   make -f Makefile.simpool clean  -> limpia binario
# ==============================================

CC      = cc
CFLAGS  = -O2 -q64
LIBS    = -lsocket
TARGET  = simpool
SRC     = simpool.c

# Compilacion principal
$(TARGET): $(SRC)
	$(CC) $(CFLAGS) -o $(TARGET) $(SRC) $(LIBS)
	@echo ""
	@echo "Compilacion exitosa: ./$(TARGET)"
	@echo "Uso: ./$(TARGET) <IP> <PUERTO> [INTERVALO]"
	@echo "Ej:  ./$(TARGET) 10.34.58.101 4584 300"

# Limpiar binario
clean:
	rm -f $(TARGET)

.PHONY: clean
