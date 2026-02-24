/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Auditoria de Tarjetas
 * #Nombre      : chk_dup_buttons
 * #Descripcion : Recorre la tabla BUTTONS buscando PANs en claro
 *                (que NO empiecen por "1:"). Por cada PAN en claro
 *                lo cifra con encpan via popen(), luego busca el PAN
 *                cifrado en BUTTONS. Si lo encuentra, significa que
 *                existe la misma tarjeta duplicada (una en claro y
 *                una cifrada). Lo reporta en pantalla y en
 *                duplicados.txt en el directorio actual.
 *
 * Compilar:
 *   cc -o chk_dup_buttons chk_dup_buttons.c -I<include_path> -L<lib_path> -lesp
 *
 * Uso:
 *   ./chk_dup_buttons
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include <signal.h>
#include "pt_master.h"

#define MAX_PAN         50
#define MAX_ENC_PAN     80
#define MAX_CMD         256
#define MAX_LINE        512
#define REPORTE_CADA    500     /* imprimir progreso cada N registros leidos */
#define ARCHIVO_DUP     "duplicados.txt"

/* -------------------------------------------------- */
/* Log con timestamp                                  */
/* -------------------------------------------------- */
static void log_msg(const char *nivel, const char *msg)
{
    time_t n;
    char ts[30];
    time(&n);
    strftime(ts, sizeof(ts), "%Y-%m-%d %H:%M:%S", localtime(&n));
    printf("[%s] [%-7s] %s\n", ts, nivel, msg);
    fflush(stdout);
}

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
        printf("\n[%s] [INFO   ] Ctrl+C recibido. Cerrando sesion Oracle...\n", ts);
    else
        printf("\n[%s] [INFO   ] Senal %d recibida. Cerrando...\n", ts, n);

    fflush(stdout);
    alarm(0);
    esp_abort();
    printf("[%s] [INFO   ] Sesion cerrada. Saliendo.\n", ts);
    fflush(stdout);
    exit(0);
}

/* -------------------------------------------------- */
/* Cifra un PAN usando encpan via popen               */
/* Replica la logica del script ksh:                  */
/*   encpan "$PAN" | grep -i "encripted|encrypted"    */
/*                 | awk '{print $NF}'                */
/* Si no encuentra esa linea, usa la ultima linea.    */
/* Limpia corchetes [] del resultado.                 */
/* Retorna 0 si OK, -1 si fallo.                      */
/* -------------------------------------------------- */
static int cifrar_pan(const char *pan, char *enc_pan_out, int out_size)
{
    FILE  *fp;
    char   cmd[MAX_CMD];
    char   linea[MAX_LINE];
    char   tmp_enc[MAX_ENC_PAN];
    char   last_line[MAX_LINE];
    char  *ptr;
    int    encontrado = 0;

    memset(enc_pan_out, '\0', out_size);
    memset(tmp_enc,     '\0', sizeof(tmp_enc));
    memset(last_line,   '\0', sizeof(last_line));

    snprintf(cmd, sizeof(cmd), "encpan '%s' 2>/dev/null", pan);

    fp = popen(cmd, "r");
    if (fp == NULL) {
        log_msg("ERROR", "No se pudo ejecutar encpan via popen");
        return -1;
    }

    while (fgets(linea, sizeof(linea), fp) != NULL)
    {
        /* Guardar siempre la ultima linea leida */
        strncpy(last_line, linea, sizeof(last_line) - 1);

        if (!encontrado)
        {
            /* Copiar en minusculas para busqueda case-insensitive */
            char lower[MAX_LINE];
            int  k;
            memset(lower, '\0', sizeof(lower));
            strncpy(lower, linea, sizeof(lower) - 1);
            for (k = 0; lower[k]; k++)
                if (lower[k] >= 'A' && lower[k] <= 'Z')
                    lower[k] = lower[k] + 32;

            /* grep -i "encripted\|encrypted" */
            if (strstr(lower, "encripted") || strstr(lower, "encrypted"))
            {
                /* awk '{print $NF}' - ultimo token separado por espacio */
                ptr = strrchr(linea, ' ');
                if (ptr && *(ptr+1) != '\0')
                    strncpy(tmp_enc, ptr + 1, sizeof(tmp_enc) - 1);
                else
                    strncpy(tmp_enc, linea, sizeof(tmp_enc) - 1);

                encontrado = 1;
            }
        }
    }
    pclose(fp);

    /* Si no encontro la linea con "encrypted", usar la ultima linea */
    if (!encontrado && strlen(last_line) > 0)
    {
        ptr = strrchr(last_line, ' ');
        if (ptr && *(ptr+1) != '\0')
            strncpy(tmp_enc, ptr + 1, sizeof(tmp_enc) - 1);
        else
            strncpy(tmp_enc, last_line, sizeof(tmp_enc) - 1);
    }

    /* Limpiar newlines, CR y espacios al final */
    {
        int len = strlen(tmp_enc);
        while (len > 0 && (tmp_enc[len-1] == '\n' ||
                            tmp_enc[len-1] == '\r' ||
                            tmp_enc[len-1] == ' '))
            tmp_enc[--len] = '\0';
    }

    /* Quitar corchetes [] si los trae: "[1:xxx]" -> "1:xxx" */
    {
        char clean[MAX_ENC_PAN];
        int  s = 0, d = 0;
        memset(clean, '\0', sizeof(clean));
        for (; tmp_enc[s] && d < (int)sizeof(clean)-1; s++)
            if (tmp_enc[s] != '[' && tmp_enc[s] != ']')
                clean[d++] = tmp_enc[s];
        strncpy(enc_pan_out, clean, out_size - 1);
    }

    if (strlen(enc_pan_out) == 0)
        return -1;

    return 0;
}

/* -------------------------------------------------- */
/* Busca el PAN cifrado en BUTTONS                    */
/* Retorna 1 si existe, 0 si no, -1 si error          */
/* -------------------------------------------------- */
static int buscar_pan_cifrado_en_buttons(const char *pan_cifrado)
{
    char      command[512];
    char      cnt_str[20];
    int       cnt;
    ESP_SLOT  slot;
    ESP_SLOTP slotp = &slot;

    memset(command, '\0', sizeof(command));
    memset(cnt_str, '\0', sizeof(cnt_str));

    /*
     * Buscamos con LIKE para tolerar posibles paddings de CHAR en la columna.
     * El PAN cifrado comienza con "1:..." segun el patron del sistema.
     */
    snprintf(command, sizeof(command),
        "SELECT COUNT(*) AS CNT FROM BUTTONS WHERE PAN LIKE '%s%%'",
        pan_cifrado);

    cc_send("buscar cifrado: %s\n", command);

    if (esp_read(slotp, "card", command, ESP_LOOK)) {
        cc_send("buscar_pan: ESP_READ ERROR: [%s]", (char*)*Last_error);
        esp_free(slotp);
        return -1;
    }

    cnt = 0;
    while (!esp_fetch(slotp)) {
        esp_get(slotp, "cnt", cnt_str);
        cnt = atoi(cnt_str);
    }
    esp_free(slotp);

    return (cnt > 0) ? 1 : 0;
}

/* -------------------------------------------------- */
/* Main                                               */
/* -------------------------------------------------- */
int main(int argc, char **argv, char **env)
{
    ESP_SLOT  slot_scan;
    ESP_SLOTP slotp_scan = &slot_scan;

    char pan_claro[MAX_PAN];
    char pan_cifrado[MAX_ENC_PAN];
    char logbuf[512];
    char cmd_scan[256];

    int  total_leidos     = 0;
    int  total_claro      = 0;
    int  total_duplicados = 0;
    int  resultado;

    time_t t_inicio, t_fin;
    char   str_inicio[30], str_fin[30];
    double segundos;
    FILE  *fdup = NULL;

    /* --- Cabecera --- */
    system("clear");
    printf("==============================================\n");
    printf("  chk_dup_buttons - Deteccion de PANs dup.  \n");
    printf("==============================================\n\n");
    printf("Archivo de reporte : %s\n", ARCHIVO_DUP);
    printf("PID                : %d\n\n", getpid());

    time(&t_inicio);
    strftime(str_inicio, sizeof(str_inicio), "%Y-%m-%d %H:%M:%S", localtime(&t_inicio));
    printf("Inicio             : %s\n\n", str_inicio);

    /* --- Senales - ANTES de fep_catch_sig para que no las pise --- */
    signal(SIGINT,  hot_exit);
    signal(SIGTERM, hot_exit);
    fep_catch_sig(0, hot_exit);

    /* --- Conexion Oracle --- */
    sw_logon_database(1);

    /* Validar sesion inicial */
    {
        char      cmd_val[100];
        char      val_res[10];
        ESP_SLOT  slot_val;
        ESP_SLOTP slotp_val = &slot_val;

        memset(cmd_val, '\0', sizeof(cmd_val));
        memset(val_res, '\0', sizeof(val_res));
        sprintf(cmd_val, "SELECT 'OK' AS resultado FROM DUAL");

        if (esp_read(slotp_val, "validacion", cmd_val, ESP_LOOK)) {
            log_msg("ERROR", "No se pudo establecer sesion Oracle.");
            exit(1);
        }
        while (!esp_fetch(slotp_val))
            esp_get(slotp_val, "resultado", val_res);
        esp_free(slotp_val);

        if (strcmp(val_res, "OK") != 0) {
            log_msg("ERROR", "Validacion de sesion Oracle fallida.");
            exit(1);
        }
    }
    log_msg("INFO", "Sesion Oracle establecida. Iniciando barrido BUTTONS...");
    printf("----------------------------------------------\n");

    /* --- Abrir archivo de reporte --- */
    fdup = fopen(ARCHIVO_DUP, "w");
    if (fdup == NULL) {
        log_msg("ERROR", "No se pudo crear duplicados.txt en el directorio actual.");
        esp_abort();
        exit(1);
    }
    fprintf(fdup, "=== Reporte de PANs duplicados en BUTTONS ===\n");
    fprintf(fdup, "Inicio : %s\n", str_inicio);
    fprintf(fdup, "==============================================\n\n");
    fflush(fdup);

    /* --- Abrir cursor sobre toda la tabla BUTTONS --- */
    /*
     * Traemos todos los PANs. Los que empiecen por "1:" se ignoran
     * (ya estan cifrados). Solo procesamos los que estan en claro.
     *
     * NOTA: Ajusta las columnas extra si necesitas mas datos en el reporte.
     */
    memset(cmd_scan, '\0', sizeof(cmd_scan));
    sprintf(cmd_scan, "SELECT PAN FROM BUTTONS ORDER BY PAN");

    if (esp_read(slotp_scan, "card", cmd_scan, ESP_LOOK)) {
        snprintf(logbuf, sizeof(logbuf),
            "ESP_READ ERROR al abrir cursor BUTTONS: %s",
            (char*)*Last_error);
        log_msg("ERROR", logbuf);
        fclose(fdup);
        esp_abort();
        exit(1);
    }

    /* --- Barrido registro por registro --- */
    while (!esp_fetch(slotp_scan))
    {
        memset(pan_claro,   '\0', sizeof(pan_claro));
        memset(pan_cifrado, '\0', sizeof(pan_cifrado));

        esp_get(slotp_scan, "pan", pan_claro);
        total_leidos++;

        /* Progreso periodico */
        if (total_leidos % REPORTE_CADA == 0) {
            snprintf(logbuf, sizeof(logbuf),
                "Leidos: %d | En claro: %d | Duplicados: %d",
                total_leidos, total_claro, total_duplicados);
            log_msg("PROG   ", logbuf);
        }

        /*
         * Si empieza por "1:" => ya esta cifrado => ignorar.
         */
        if (strncmp(pan_claro, "1:", 2) == 0)
            continue;

        /* Limpiar trailing spaces que genera un campo CHAR de Oracle */
        {
            int len = strlen(pan_claro);
            while (len > 0 && pan_claro[len-1] == ' ')
                pan_claro[--len] = '\0';
        }

        if (strlen(pan_claro) == 0)
            continue;

        total_claro++;

        /* --- Cifrar el PAN en claro --- */
        if (cifrar_pan(pan_claro, pan_cifrado, sizeof(pan_cifrado)) != 0) {
            snprintf(logbuf, sizeof(logbuf),
                "WARN: no se pudo cifrar PAN [%s] (registro %d)",
                pan_claro, total_leidos);
            log_msg("WARN   ", logbuf);
            fprintf(fdup, "WARN: no cifrado PAN=[%s] en registro %d\n\n",
                pan_claro, total_leidos);
            fflush(fdup);
            continue;
        }

        snprintf(logbuf, sizeof(logbuf),
            "Claro [%s] -> Cifrado [%s]", pan_claro, pan_cifrado);
        cc_send("%s\n", logbuf);

        /* --- Buscar PAN cifrado en BUTTONS --- */
        resultado = buscar_pan_cifrado_en_buttons(pan_cifrado);

        if (resultado == 1)
        {
            /* DUPLICADO: existe en claro Y cifrado en la misma tabla */
            total_duplicados++;

            snprintf(logbuf, sizeof(logbuf),
                "*** DUPLICADO #%d ***  PAN_CLARO=[%s]  PAN_CIFRADO=[%s]",
                total_duplicados, pan_claro, pan_cifrado);

            printf("\n");
            log_msg("ALERTA ", logbuf);
            printf("\n");

            fprintf(fdup, "DUPLICADO #%d\n", total_duplicados);
            fprintf(fdup, "  PAN en claro : %s\n", pan_claro);
            fprintf(fdup, "  PAN cifrado  : %s\n", pan_cifrado);
            fprintf(fdup, "\n");
            fflush(fdup);
        }
        else if (resultado == -1)
        {
            snprintf(logbuf, sizeof(logbuf),
                "ERROR al consultar PAN cifrado [%s]", pan_cifrado);
            log_msg("ERROR  ", logbuf);
        }
        /* resultado == 0: sin duplicado, continuar */
    }

    esp_free(slotp_scan);

    /* --- Cerrar reporte --- */
    time(&t_fin);
    strftime(str_fin, sizeof(str_fin), "%Y-%m-%d %H:%M:%S", localtime(&t_fin));
    segundos = difftime(t_fin, t_inicio);

    fprintf(fdup, "==============================================\n");
    fprintf(fdup, "Fin              : %s\n", str_fin);
    fprintf(fdup, "Duracion         : %.0f seg (%.2f min)\n", segundos, segundos/60);
    fprintf(fdup, "Total leidos     : %d\n", total_leidos);
    fprintf(fdup, "PANs en claro    : %d\n", total_claro);
    fprintf(fdup, "Duplicados       : %d\n", total_duplicados);
    fprintf(fdup, "==============================================\n");
    fclose(fdup);

    /* --- Resumen pantalla --- */
    printf("\n==============================================\n");
    printf("            RESUMEN DE EJECUCION              \n");
    printf("==============================================\n\n");
    printf("Inicio              : %s\n", str_inicio);
    printf("Fin                 : %s\n", str_fin);
    printf("Duracion            : %.0f seg (%.2f min)\n\n", segundos, segundos/60);
    printf("Registros leidos    : %d\n", total_leidos);
    printf("PANs en claro       : %d\n", total_claro);
    printf("DUPLICADOS hallados : %d\n", total_duplicados);

    if (total_duplicados > 0) {
        printf("\n  *** ATENCION: Se encontraron %d PAN(s) duplicados.\n",
            total_duplicados);
        printf("  *** Revisa: %s\n", ARCHIVO_DUP);
    } else {
        printf("\n  OK: No se encontraron duplicados.\n");
    }

    printf("==============================================\n");

    alarm(0);
    esp_abort();
    return 0;
}
