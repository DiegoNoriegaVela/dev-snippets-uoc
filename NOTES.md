import java.io.FileInputStream;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Properties;

public class Logger {

    public static final int DEBUG = 0;
    public static final int INFO  = 1;
    public static final int WARN  = 2;
    public static final int ERROR = 3;

    private static int currentLevel = INFO;
    private static boolean initialized = false;

    private static final SimpleDateFormat SDF = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static void init(String configPath) {
        Properties props = new Properties();
        try {
            FileInputStream fis = new FileInputStream(configPath);
            props.load(fis);
            fis.close();

            String level = props.getProperty("log.level", "INFO").trim().toUpperCase();
            switch (level) {
                case "DEBUG": currentLevel = DEBUG; break;
                case "INFO":  currentLevel = INFO;  break;
                case "WARN":  currentLevel = WARN;  break;
                case "ERROR": currentLevel = ERROR; break;
                default:      currentLevel = INFO;  break;
            }
            initialized = true;
            info("Logger inicializado en nivel: " + level);

        } catch (IOException e) {
            System.err.println("No se pudo leer config: " + configPath + ". Usando nivel INFO.");
            currentLevel = INFO;
            initialized = true;
        }
    }

    public static void debug(String message) { log(DEBUG, message); }
    public static void info(String message)  { log(INFO, message);  }
    public static void warn(String message)  { log(WARN, message);  }
    public static void error(String message) { log(ERROR, message); }

    private static void log(int level, String message) {
        if (!initialized) {
            currentLevel = INFO;
            initialized = true;
        }
        if (level < currentLevel) return;

        StackTraceElement caller = Thread.currentThread().getStackTrace()[3];

        String timestamp = SDF.format(new Date());
        String fileName  = caller.getFileName();
        int    line      = caller.getLineNumber();
        String tag       = getTag(level);

        System.out.println(String.format(
            "%s  [%s]  %-20s : %-4d | %s",
            timestamp, tag, fileName, line, message
        ));
    }

    private static String getTag(int level) {
        switch (level) {
            case DEBUG: return "DEBU";
            case INFO:  return "INFO";
            case WARN:  return "WARN";
            case ERROR: return "ERRO";
            default:    return "????";
        }
    }
}

