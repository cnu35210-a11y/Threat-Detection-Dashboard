# Threat Detection Dashboard — Mini-SIEM (JavaFX)

Aplicacion de escritorio en **Java 17 + JavaFX** que simula un mini SIEM/SOC:
ingesta logs (JSON/CSV), los correlaciona contra un motor de reglas estilo
**Sigma**, calcula severidad, mapea cada alerta a **MITRE ATT&CK**, y permite
simular una respuesta automatizada (**Mini-SOAR**) de contencion de red.

Proyecto pensado como pieza de portafolio/CV: arquitectura en capas, codigo
limpio, sin dependencias exoticas, y 100% funcional fuera de la caja.

## Arquitectura

Se aplica una arquitectura **MVC**, con la capa Controller a su vez dividida
en varios controllers especializados (uno por region de la pantalla) que se
componen mediante `fx:include`, coordinados por un controller "mediador":

```
com.detection
├── App.java                       # Bootstrap de JavaFX (extiende Application)
├── Launcher.java                  # Punto de entrada real para IDEs / java -jar (ver Troubleshooting)
├── model/                         # Modelo: entidades y contratos de dominio
│   ├── LogEntry.java                # Entidad principal (propiedades JavaFX observables)
│   ├── LogRecordDTO.java            # DTO de deserializacion JSON/CSV (Jackson)
│   ├── Severity.java                # Enum de severidad (INFO/LOW/MEDIUM/HIGH)
│   └── DetectionRule.java           # Contrato Strategy para reglas de deteccion
├── service/                       # Logica de negocio (sin dependencias de JavaFX UI)
│   ├── LogIngestionService.java     # Parsing de archivos JSON/CSV
│   ├── MockLogGenerator.java        # Generador de datos sinteticos realistas
│   ├── DetectionEngine.java         # Orquestador de reglas (correlacion + severidad)
│   ├── SoarService.java             # Simulacion de playbook de contencion (Mini-SOAR)
│   └── rule/                        # Reglas concretas (patron Strategy), estilo Sigma
│       ├── BruteForceSshRule.java
│       ├── MimikatzExecutionRule.java
│       ├── PrivilegeEscalationRule.java
│       ├── PortScanRule.java
│       └── EncodedPowerShellRule.java
├── controller/                    # Un controller por region de pantalla (responsabilidad unica)
│   ├── MainController.java          # Mediador: conecta sub-controllers + services, sin dibujar nada
│   ├── ToolbarController.java       # Busqueda, filtro de severidad, botones de carga/generacion
│   ├── MetricsPanelController.java  # KPIs de severidad + Top IPs origen
│   ├── LogsTableController.java     # TableView: columnas, filtrado, ordenamiento, seleccion
│   ├── DetailPanelController.java   # Log crudo + ficha MITRE + boton de playbook
│   └── ConsolePanelController.java  # Consola SOAR + barra de estado
└── util/
    └── JsonUtil.java                 # ObjectMapper de Jackson centralizado

resources/com/detection/
├── main.fxml                      # Layout raiz: compone los 5 fragmentos via <fx:include>
├── toolbar.fxml                   # Fragmento -> ToolbarController
├── metrics-panel.fxml             # Fragmento -> MetricsPanelController
├── logs-table.fxml                # Fragmento -> LogsTableController
├── detail-panel.fxml              # Fragmento -> DetailPanelController
├── console-panel.fxml             # Fragmento -> ConsolePanelController
└── dark-theme.css                 # Tema oscuro estilo SOC
```

### Por que varios controllers y no uno solo

Cada region de la pantalla (toolbar, metricas, tabla, detalle, consola) tiene
su **propio FXML + su propio controller**, con una responsabilidad unica y
sin conocer a los demas directamente:

- `main.fxml` incluye cada fragmento con `<fx:include source="..." fx:id="xxx"/>`.
  JavaFX inyecta automaticamente cada sub-controller en un campo
  `xxxController` de `MainController` (convencion estandar de FXML).
- `MainController` actua como **mediador** (patron *Mediator*): es el unico
  que conoce los `Service` de negocio y el unico que conecta las señales de
  un sub-controller con las acciones de otro (ej: cuando `ToolbarController`
  reporta un cambio en el texto de busqueda, `MainController` le pide a
  `LogsTableController` que filtre; cuando `LogsTableController` reporta una
  nueva seleccion, `MainController` le pide a `DetailPanelController` que
  muestre el detalle).
- Ningun sub-controller llama a otro sub-controller directamente: solo
  exponen propiedades/callbacks (`setOnPlaybookRequested`,
  `searchTextProperty()`, `selectedItemProperty()`, etc.) y es
  `MainController` quien los cablea en `initialize()`.

Esto permite testear, reemplazar o extender cada panel de forma aislada
(por ejemplo, cambiar el panel de metricas por un grafico) sin tocar el
resto del codigo.

**Flujo de datos:** `LogIngestionService` / `MockLogGenerator` producen
`LogEntry` "en bruto" → `DetectionEngine` los enriquece evaluando cada
`DetectionRule` registrada (fuerza bruta, Mimikatz, escalamiento de
privilegios, escaneo de puertos, PowerShell codificado) → el resultado se
vuelca en una `ObservableList` que `MainController` entrega a
`LogsTableController`, que la filtra en tiempo real con `FilteredList` +
`SortedList`.

> **Nota de diseño:** el proyecto se compila en modo *classpath* (sin
> `module-info.java`) para evitar los problemas habituales de JPMS con
> librerias que usan reflexion intensiva (Jackson). Es la configuracion mas
> robusta para un proyecto Maven + JavaFX de este tamaño.

## Solucion de problemas: "Error: JavaFX runtime components are missing"

Si al ejecutar el proyecto desde un IDE (boton "Run" de VS Code/IntelliJ) o
con `java -cp ...` te aparece:

```
Error: JavaFX runtime components are missing, and are required to run this application
```

**la causa NO es que falten las dependencias**: ocurre porque el IDE esta
lanzando directamente `com.detection.App`, y esa clase extiende
`javafx.application.Application`. JavaFX bloquea intencionalmente el arranque
directo de una subclase de `Application` cuando no se usa `--module-path`
(que es exactamente lo que `mvn javafx:run` arma automaticamente por
detras).

**Solucion:** ejecuta siempre `com.detection.Launcher` (no `App`) fuera de
Maven. Este proyecto ya trae:

- `Launcher.java`: una clase separada, que NO extiende `Application`, cuyo
  unico trabajo es llamar a `App.main(args)`. Al ser la clase que la JVM
  lanza directamente, el chequeo especial de JavaFX no se activa.
- `.vscode/launch.json`: configuracion lista para que el boton **Run** de la
  extension de Java para VS Code use `Launcher` en vez de `App`. Si tu IDE
  genera su propia configuracion automaticamente, edita el "Main class" /
  "mainClass" de esa configuracion a `com.detection.Launcher`.
- `pom.xml`: tanto `javafx-maven-plugin` como `maven-shade-plugin` ya apuntan
  a `Launcher`, asi que `mvn javafx:run` y `java -jar target/*.jar` funcionan
  sin tocar nada.

## Requisitos

- JDK 17 o superior
- Maven 3.8+
- Conexion a internet la primera vez (para descargar JavaFX 21 y Jackson desde Maven Central)

## Como ejecutar

### Opcion recomendada: Maven

```bash
cd detection
mvn clean javafx:run
```

Tambien puedes generar un JAR ejecutable "fat jar" (con todas las dependencias):

```bash
mvn clean package
java -jar target/threat-dashboard-1.0.0.jar
```

### Ejecutar desde un IDE (VS Code, IntelliJ, Eclipse) con el boton "Run"

Si le das "Run" directamente sobre `App.java` (o sobre `com.detection.App`
desde la paleta de comandos / CodeLens de Java en VS Code), veras este error:

```
Error: JavaFX runtime components are missing, and are required to run this application
```

Esto **no significa que falten las dependencias** (Maven ya las descargo a tu
`.m2`): ocurre porque Java bloquea el arranque directo de cualquier clase que
extienda `javafx.application.Application` cuando no se lanza a traves de un
module-path armado correctamente (que es justo lo que hace `mvn javafx:run`
por debajo).

**Solucion:** el proyecto incluye `com.detection.Launcher`, una clase que NO
extiende `Application` y solo delega a `App.main(...)`. Ejecuta esa clase en
su lugar:

- En VS Code: abre `Launcher.java` y dale "Run" (o clic derecho → *Run Java*)
  en vez de hacerlo sobre `App.java`.
- Con `java` directo, usa `com.detection.Launcher` como clase principal en
  vez de `com.detection.App` (el pom.xml y el jar ya estan configurados asi).

Si prefieres seguir lanzando `App.java` directamente desde el IDE, agrega
estos argumentos de VM en la configuracion de "Run" (`launch.json` en VS
Code o el "Run Configuration" del IDE), apuntando al SDK de JavaFX que Maven
ya descargo en tu repositorio local `~/.m2/repository/org/openjfx/...`:

```
--module-path "<ruta-a-tus-jars-javafx>" --add-modules javafx.controls,javafx.fxml
```

Pero es mas simple usar `Launcher.java`, que funciona sin tener que armar
esa ruta manualmente.

## Como usar la aplicacion

1. Pulsa **"⚡ Generar Logs de Prueba"** para poblar el dashboard al instante
   con un lote sintetico que incluye trafico benigno + 5 escenarios de
   ataque reconocidos por el motor de deteccion.
2. O bien pulsa **"Cargar Logs (JSON)"** y selecciona el archivo
   `sample-logs.json` incluido en la raiz del proyecto (o
   `src/main/resources/logs/sample-logs.json`) — o **"Cargar Logs (CSV)"**
   con `sample-logs.csv`.
3. Usa el combo de **Severidad** y el campo de **Busqueda** (arriba) para
   filtrar la tabla en tiempo real.
4. Selecciona cualquier fila para ver, a la derecha, el **log crudo (JSON)**
   y la ficha **MITRE ATT&CK** de la regla que se disparo.
5. Sobre una alerta de severidad **ALTA**, pulsa **"🚨 Ejecutar Playbook de
   Contencion"**: la consola inferior (Mini-SOAR) mostrara paso a paso la
   simulacion de bloqueo de IP (regla de `iptables`/`netsh` equivalente) y
   la alerta pasara a estado `CONTAINED`.

## Reglas de deteccion incluidas

| ID              | Nombre                              | Severidad | MITRE ATT&CK                                       |
|-----------------|--------------------------------------|-----------|-----------------------------------------------------|
| `BF-SSH-001`    | SSH Brute Force                      | HIGH      | T1110 – Brute Force                                  |
| `CRED-DUMP-001` | Mimikatz Execution                   | HIGH      | T1003 – OS Credential Dumping                        |
| `PRIV-ESC-001`  | Privilege Escalation                 | HIGH      | T1068 – Exploitation for Privilege Escalation        |
| `RECON-001`     | Port Scan / Network Reconnaissance   | MEDIUM    | T1046 – Network Service Discovery                    |
| `EXEC-PS-001`   | Suspicious Encoded PowerShell        | HIGH      | T1059.001 – Command and Scripting Interpreter: PS    |

Añadir una regla nueva solo requiere crear una clase que implemente
`DetectionRule` y registrarla en `DetectionEngine` — no se toca ninguna otra
parte del sistema (Open/Closed Principle).

## Posibles extensiones (ideas para el CV / demo en vivo)

- Persistir alertas en una base de datos embebida (H2/SQLite).
- Exportar el dashboard a PDF/CSV.
- Reglas configurables desde un archivo YAML externo (más fiel a Sigma real).
- Grafico de linea de tiempo de eventos (JavaFX Charts).
