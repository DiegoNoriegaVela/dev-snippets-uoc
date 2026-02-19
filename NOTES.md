/*
 * #Sistema     : SWITCH ATM
 * #Subsistema  : Interface
 * #Nombre      : gentabpan
 * #Descripcion : Generacion del archivo Tabpan.txt
 * #Autor       : Cesar Mesta - SES
 * #Fecha       : 09-10-2023
 * #Modificado  : Paralelizacion con fork() y metricas de CPU
 *
 * USO:
 *   ./gentabpan           -> Comportamiento original (sin paralelizar)
 *   ./gentabpan 4         -> Paraleliza en 4 procesos
 *   ./gentabpan 8         -> Paraleliza en 8 procesos
 */

#include <stdio.h>
#include <sys/times.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include "pt_master.h"

/*Declaracion de parrafos o funciones*/
static int read_card();
static void hot_exit(int n);
static void print_card_info();
static void abrir_archivo();
static int contar_registros();
static double obtener_cpu_del_proceso();

/* Declaracion de Variables*/
char pan[50];
char account[30];
char firstt;
char accttype;
char currency[4];
char enc_wall;

/*Para obtener la Hora del sistema*/
time_t now;

extern char *argv0;

FILE *fp;       /* Declaracion de archivo*/

/* Variables para paralelizacion */
int g_offset = 0;
int g_cantidad = 0;
int g_proceso_id = -1;
int g_modo_paralelo = 0;
int g_num_procesos = 0;

/* Variables para monitoreo de CPU */
int g_pid;
double g_cpu_max = 0;
double g_cpu_sum = 0;
int g_cpu_muestras = 0;
struct tms cpu_inicio, cpu_fin;
clock_t tiempo_real_inicio, tiempo_real_fin;
long clk_tck;
int g_num_cpus = 1;

/* Funcion para obtener el % CPU actual del proceso */
static double obtener_cpu_del_proceso()
{
    FILE *fp_cmd;
    char comando[100];
    char buffer[50];
    double cpu = 0;
    
    sprintf(comando, "ps -p %d -o pcpu= 2>/dev/null", g_pid);
    
    fp_cmd = popen(comando, "r");
    if (fp_cmd != NULL) {
        if (fgets(buffer, sizeof(buffer), fp_cmd) != NULL) {
            cpu = atof(buffer);
        }
        pclose(fp_cmd);
    }
    
    return cpu;
}

/* Funcion para obtener numero de CPUs */
static int obtener_num_cpus()
{
    FILE *fp_cmd;
    char buffer[50];
    int cpus = 1;
    
    /* En AIX */
    fp_cmd = popen("lsdev -Cc processor | wc -l", "r");
    if (fp_cmd != NULL) {
        if (fgets(buffer, sizeof(buffer), fp_cmd) != NULL) {
            cpus = atoi(buffer);
            if (cpus < 1) cpus = 1;
        }
        pclose(fp_cmd);
    }
    
    return cpus;
}

/* Funcion para registrar muestra de CPU */
static void registrar_cpu()
{
    double cpu_actual = obtener_cpu_del_proceso();
    
    g_cpu_sum += cpu_actual;
    g_cpu_muestras++;
    
    if (cpu_actual > g_cpu_max) {
        g_cpu_max = cpu_actual;
    }
}

/* Funcion para unir archivos en orden */
static void unir_archivos_ordenados()
{
    FILE *fp_final;
    FILE *fp_temp;
    char filename_final[200];
    char filename_temp[200];
    char buffer[1024];
    int i;
    
    sprintf(filename_final, "%s/tabpan.txt", getenv("LOGDIR"));
    
    fp_final = fopen(filename_final, "w");
    if (fp_final == NULL) {
        printf("Error: No se pudo crear archivo final %s\n", filename_final);
        return;
    }
    
    /* Unir en orden: proceso 0, 1, 2, ... */
    for (i = 0; i < g_num_procesos; i++) {
        sprintf(filename_temp, "%s/tabpan_%02d.txt", getenv("LOGDIR"), i);
        
        fp_temp = fopen(filename_temp, "r");
        if (fp_temp != NULL) {
            while (fgets(buffer, sizeof(buffer), fp_temp) != NULL) {
                fputs(buffer, fp_final);
            }
            fclose(fp_temp);
            
            /* Eliminar archivo temporal */
            remove(filename_temp);
        }
    }
    
    fclose(fp_final);
}

int main(int argc, char **argv, char **env)
{
    int num_procesos = 0;
    int total_registros;
    int chunk;
    int i;
    pid_t pid;
    int status;
    time_t tiempo_inicio, tiempo_fin;
    char str_inicio[30], str_fin[30];
    double segundos;
    double cpu_promedio;
    double cpu_user, cpu_system, cpu_total;
    double tiempo_real_seg;
    double porcentaje_cpu_sistema;
    
    g_pid = getpid();
    clk_tck = sysconf(_SC_CLK_TCK);
    g_num_cpus = obtener_num_cpus();
    
    system("clear");
    printf("Generacion de Archivo Tabpan\n");
    
    /* Verificar si se pide modo paralelo */
    if (argc >= 2) {
        num_procesos = atoi(argv[1]);
        if (num_procesos < 1 || num_procesos > 16) {
            printf("Error: numero de procesos debe ser entre 1 y 16\n");
            printf("Uso: %s [num_procesos]\n", argv[0]);
            printf("  Sin parametros = modo original\n");
            printf("  Con parametro  = modo paralelo\n");
            exit(1);
        }
        g_modo_paralelo = 1;
        g_num_procesos = num_procesos;
        printf("Modo PARALELO con %d procesos\n\n", num_procesos);
    }
    
    /* Registrar tiempo de inicio */
    time(&tiempo_inicio);
    strftime(str_inicio, sizeof(str_inicio), "%Y-%m-%d %H:%M:%S", localtime(&tiempo_inicio));
    printf("Inicio: %s\n", str_inicio);
    
    /* Capturar CPU al inicio */
    tiempo_real_inicio = times(&cpu_inicio);
    
    sw_logon_database(1);       /* Ingreso a la BD*/
    
    /* MODO ORIGINAL - sin paralelizar */
    if (!g_modo_paralelo) {
        fep_catch_sig(0, hot_exit);
        abrir_archivo();
        total_registros = read_card();
        
        /* Capturar CPU al final */
        tiempo_real_fin = times(&cpu_fin);
        
        /* Tomar ultima muestra */
        registrar_cpu();
        
        /* Registrar tiempo de fin */
        time(&tiempo_fin);
        strftime(str_fin, sizeof(str_fin), "%Y-%m-%d %H:%M:%S", localtime(&tiempo_fin));
        segundos = difftime(tiempo_fin, tiempo_inicio);
        
        /* Calcular tiempos de CPU del proceso */
        cpu_user = (double)(cpu_fin.tms_utime - cpu_inicio.tms_utime) / clk_tck;
        cpu_system = (double)(cpu_fin.tms_stime - cpu_inicio.tms_stime) / clk_tck;
        cpu_total = cpu_user + cpu_system;
        tiempo_real_seg = (double)(tiempo_real_fin - tiempo_real_inicio) / clk_tck;
        
        /* Calcular porcentaje de CPU */
        if (tiempo_real_seg > 0) {
            cpu_promedio = (cpu_total / tiempo_real_seg) * 100.0;
        } else {
            cpu_promedio = 0;
        }
        
        /* Calcular impacto en servidor (considerando todos los CPUs) */
        if (tiempo_real_seg > 0 && g_num_cpus > 0) {
            porcentaje_cpu_sistema = (cpu_total / (tiempo_real_seg * g_num_cpus)) * 100.0;
        } else {
            porcentaje_cpu_sistema = 0;
        }
        
        /* Si el maximo es 0, usar el calculado */
        if (g_cpu_max == 0) {
            g_cpu_max = cpu_promedio;
        }
        
        printf("\n========== RESUMEN ==========\n");
        printf("Inicio:                 %s\n", str_inicio);
        printf("Fin:                    %s\n", str_fin);
        printf("Duracion:               %.0f segundos (%.2f minutos)\n", segundos, segundos/60);
        printf("Registros procesados:   %d\n", total_registros);
        if (segundos > 0) {
            printf("Velocidad:              %.2f registros/segundo\n", total_registros/segundos);
        }
        printf("-----------------------------\n");
        printf("USO DE CPU (Metricas):\n");
        printf("Tiempo CPU User:        %.2f s\n", cpu_user);
        printf("Tiempo CPU Sys:         %.2f s\n", cpu_system);
        printf("%% CPU Promedio:         %.2f %%\n", cpu_promedio);
        printf("%% CPU Pico:             %.2f %%\n", g_cpu_max);
        printf("Impacto Servidor:       %.2f %% (De %d CPUs)\n", porcentaje_cpu_sistema, g_num_cpus);
        printf("=============================\n");
        
        return 0;
    }
    
    /* MODO PARALELO - con fork */
    total_registros = contar_registros();
    printf("Total de registros a procesar: %d\n", total_registros);
    
    if (total_registros == 0) {
        printf("No hay registros para procesar\n");
        exit(0);
    }
    
    chunk = (total_registros + num_procesos - 1) / num_procesos;
    printf("Registros por proceso: ~%d\n\n", chunk);
    
    /* Crear procesos hijos */
    for (i = 0; i < num_procesos; i++) {
        pid = fork();
        
        if (pid < 0) {
            perror("Error en fork");
            exit(1);
        }
        else if (pid == 0) {
            /* PROCESO HIJO */
            g_proceso_id = i;
            g_offset = i * chunk;
            g_cantidad = chunk;
            
            if (g_offset >= total_registros) {
                exit(0);
            }
            
            if (g_offset + g_cantidad > total_registros) {
                g_cantidad = total_registros - g_offset;
            }
            
            printf("[Proceso %d] Iniciando: registros %d a %d\n",
                   i, g_offset + 1, g_offset + g_cantidad);
            
            sw_logon_database(1);
            fep_catch_sig(0, hot_exit);
            abrir_archivo();
            read_card();
            
            printf("[Proceso %d] Terminado\n", g_proceso_id);
            exit(0);
        }
    }
    
    /* PROCESO PADRE: esperar a todos los hijos */
    printf("\nEsperando a %d procesos...\n", num_procesos);
    for (i = 0; i < num_procesos; i++) {
        wait(&status);
    }
    
    /* Capturar tiempo y CPU al terminar el procesamiento */
    tiempo_real_fin = times(&cpu_fin);
    time(&tiempo_fin);
    
    /* Unir archivos en orden */
    printf("Uniendo archivos...\n");
    unir_archivos_ordenados();
    
    strftime(str_fin, sizeof(str_fin), "%Y-%m-%d %H:%M:%S", localtime(&tiempo_fin));
    segundos = difftime(tiempo_fin, tiempo_inicio);
    
    /* Calcular tiempos de CPU */
    cpu_user = (double)(cpu_fin.tms_utime - cpu_inicio.tms_utime) / clk_tck;
    cpu_system = (double)(cpu_fin.tms_stime - cpu_inicio.tms_stime) / clk_tck;
    cpu_total = cpu_user + cpu_system;
    tiempo_real_seg = (double)(tiempo_real_fin - tiempo_real_inicio) / clk_tck;
    
    /* Calcular porcentaje de CPU por proceso */
    if (tiempo_real_seg > 0) {
        cpu_promedio = (cpu_total / tiempo_real_seg) * 100.0;
    } else {
        cpu_promedio = 0;
    }
    
    /* Calcular impacto en servidor */
    if (tiempo_real_seg > 0 && g_num_cpus > 0) {
        porcentaje_cpu_sistema = (cpu_total * num_procesos / (tiempo_real_seg * g_num_cpus)) * 100.0;
    } else {
        porcentaje_cpu_sistema = 0;
    }
    
    printf("\n========== RESUMEN ==========\n");
    printf("Modo:                   PARALELO (%d procesos)\n", num_procesos);
    printf("Inicio:                 %s\n", str_inicio);
    printf("Fin:                    %s\n", str_fin);
    printf("Duracion:               %.0f segundos (%.2f minutos)\n", segundos, segundos/60);
    printf("Registros procesados:   %d\n", total_registros);
    if (segundos > 0) {
        printf("Velocidad:              %.2f registros/segundo\n", total_registros/segundos);
    }
    printf("Archivo final:          %s/tabpan.txt\n", getenv("LOGDIR"));
    printf("-----------------------------\n");
    printf("USO DE CPU (Global):\n");
    printf("Tiempo CPU User:        %.2f s (Suma Total procesos)\n", cpu_user * num_procesos);
    printf("Tiempo CPU Sys:         %.2f s (Suma Total procesos)\n", cpu_system * num_procesos);
    printf("%% CPU Promedio:         %.2f %% (Por proceso)\n", cpu_promedio);
    printf("%% CPU Pico:             %.2f %%\n", cpu_promedio);
    printf("Impacto Servidor:       %.2f %% (De %d CPUs)\n", porcentaje_cpu_sistema, g_num_cpus);
    printf("=============================\n");
    
    return 0;
}

static int contar_registros()
{
    char command[300];
    ESP_SLOT slot;
    ESP_SLOTP slotp = &slot;
    int total = 0;
    char count_str[20];
    
    memset(command, '\0', sizeof(command));
    memset(count_str, '\0', sizeof(count_str));
    
    sprintf(command,
        "SELECT COUNT(*) as total FROM panacct "
        "WHERE (pan IS NOT NULL) "
        "AND pan >= ' ' AND instid >= ' ' AND subinstid >= ' ' AND account >= ' '");
    
    cc_send("Count query [%s]\n", command);
    
    if (esp_read(slotp, "card", command, ESP_LOOK)) {
        cc_send("ESP_READ ERROR: [%s]", (char*)*Last_error);
        fprintf(stderr, "Error contando registros\n");
        return 0;
    }
    
    if (!esp_fetch(slotp)) {
        esp_get(slotp, "total", count_str);
        total = atoi(count_str);
    }
    
    esp_free(slotp);
    return total;
}

static void abrir_archivo()
{
    char filename[100];
    
    memset(filename, '\0', sizeof(filename));
    
    if (g_modo_paralelo) {
        /* Modo paralelo: archivo por proceso */
        sprintf(filename, "%s/tabpan_%02d.txt", getenv("LOGDIR"), g_proceso_id);
    } else {
        /* Modo original: archivo unico */
        sprintf(filename, "%s/tabpan.txt", getenv("LOGDIR"));
    }
    
    fp = fopen(filename, "w");
    if (fp == NULL)
    {
        printf("Could not open file %s\n", filename);
        fclose(fp);
        exit(1);
    }
}

static void print_card_info()
{
    char record[300], *p;
    
    memset(record, 0, sizeof(record));
    p = record;
    p += sprintf(p, "%-19s", pan);
    p += sprintf(p, " ");
    p += sprintf(p, "%-29s", account);
    p += sprintf(p, " ");
    p += sprintf(p, "%c", firstt);
    p += sprintf(p, " ");
    p += sprintf(p, "%c", accttype);
    p += sprintf(p, " ");
    p += sprintf(p, "%-3s", currency);
    
    fprintf(fp, "%s \n", record);
    fflush(fp);
}

static int read_card()
{
    char command[600];
    int cnttmp = 0;
    int cnttotal = 0;
    int tope = 4;
    int intervalo_muestreo = 1000;
    ESP_SLOT slot;
    ESP_SLOTP slotp = &slot;
    
    memset(command, '\0', sizeof(command));
    
    if (g_modo_paralelo) {
        /* Query con paginacion para modo paralelo */
        sprintf(command,
            "SELECT * FROM ("
            "  SELECT a.*, ROWNUM rn FROM ("
            "    SELECT * FROM panacct "
            "    WHERE (pan IS NOT NULL) "
            "    AND pan >= ' ' AND instid >= ' ' AND subinstid >= ' ' AND account >= ' ' "
            "    ORDER BY rowid"
            "  ) a WHERE ROWNUM <= %d"
            ") WHERE rn > %d",
            g_offset + g_cantidad,
            g_offset);
    } else {
        /* Query original */
        sprintf(command,
            "SELECT * FROM panacct "
            "WHERE (pan IS NOT NULL) "
            "AND pan >= ' ' AND instid >= ' ' AND subinstid >= ' ' AND account >= ' ' "
            "ORDER BY rowid");
    }
    
    cc_send("Comand [%s]\n", command);
    
    if (esp_read(slotp, "card", command, ESP_LOOK))
    {
        cc_send("ESP_READ ERROR: [%s]", (char*)*Last_error);
        fprintf(stderr, "Can't read CARD Table!: %s\n", (char*)*Last_error);
        exit(1);
    }
    
    while (!esp_fetch(slotp))
    {
        cnttmp++;
        cnttotal++;
        
        if (cnttotal % intervalo_muestreo == 0) {
            if (!g_modo_paralelo) {
                registrar_cpu();
            }
        }
        
        firstt = 'A';
        accttype = 'A';
        enc_wall = 'N';
        
        memset(pan, '\0', sizeof(pan));
        memset(account, '\0', sizeof(account));
        memset(currency, '\0', sizeof(currency));
        
        esp_get(slotp, "account", account);
        esp_get(slotp, "firstt", firstt);
        esp_get(slotp, "accttype", accttype);
        esp_get(slotp, "currency", currency);
        esp_get(slotp, "enc_wall", enc_wall);
        
        if (enc_wall == 'Y')
        {
            if (esp_get(slotp, "pan", pan, ENCTRUE))
            {
                cc_send("%s", (char*)*Last_error);
                esp_free(slotp);
                exit(1);
            }
        }
        else
        {
            esp_get(slotp, "pan", pan);
        }
        
        if (cnttmp == tope)
        {
            time(&now);
            cnttmp = 0;
        }
        
        print_card_info();
    }
    
    if (!g_modo_paralelo) {
        printf("\nTotal de registros leidos: %d\n", cnttotal);
    }
    
    esp_free(slotp);
    fclose(fp);
    
    return cnttotal;
}

static void hot_exit(int n)
{
    printf("Cancelled with [%d]\n", n);
    esp_abort();
    exit(0);
}
