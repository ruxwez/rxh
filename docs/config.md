# Documentación del Módulo de Configuración (`src/config/`)

## Visión General

El módulo `config` es responsable de cargar, interpretar y validar la configuración del servidor RXH. La configuración se define típicamente en un archivo TOML (por defecto `rxh.toml`). Este módulo utiliza `serde` para la deserialización, pero incluye una lógica de deserialización personalizada significativa (`src/config/deser.rs`) para ofrecer una sintaxis de configuración más flexible y fácil de usar.

## Archivos Principales

-   **`src/config/mod.rs`**:
    -   Define las estructuras de datos Rust que representan la configuración.
    -   Estas estructuras están anotadas con atributos de `serde` para guiar la deserialización.
    -   Referencia al módulo `deser` para la lógica de deserialización personalizada.
-   **`src/config/deser.rs`**:
    -   Contiene la mayor parte de la lógica personalizada para deserializar el archivo TOML.
    -   Utiliza `serde::Deserialize` y el trait `Visitor` para manejar formatos de entrada complejos o variados.

## Estructuras de Datos Principales (`src/config/mod.rs`)

### 1. `Config`
   -   **Definición**:
        ```rust
        #[derive(Serialize, Deserialize, Debug, Clone)]
        pub struct Config {
            #[serde(rename = "server")]
            pub servers: Vec<Server>,
        }
        ```
   -   **Propósito**: Representa la raíz del archivo de configuración. Contiene una lista de definiciones de `Server`.
   -   **Archivo TOML Ejemplo**:
        ```toml
        [[server]]
        # ... config del primer servidor ...

        [[server]]
        # ... config del segundo servidor ...
        ```
   -   **Línea de Referencia**: `src/config/mod.rs:19`

### 2. `Server`
   -   **Definición (parcial)**:
        ```rust
        #[derive(Serialize, Debug, Clone)]
        pub struct Server {
            pub listen: Vec<SocketAddr>,
            #[serde(rename = "match")]
            pub patterns: Vec<Pattern>,
            #[serde(default = "default::max_connections")]
            pub max_connections: usize,
            pub name: Option<String>,
            #[serde(skip)]
            pub log_name: String,
        }
        ```
   -   **Propósito**: Define una instancia de servidor individual. Un servidor escucha en una o más direcciones (`listen`) y aplica una serie de `patterns` para decidir cómo manejar las solicitudes.
   -   **Deserialización Personalizada**: La deserialización de `Server` es manejada por `ServerVisitor` en `src/config/deser.rs` (referenciado indirectamente a través de `#[derive(Deserialize)]` y la implementación manual de `Deserialize` para `Server` en `src/config/deser.rs:158`). Esto permite que un servidor se defina con un patrón simple directamente bajo el servidor, o con una lista explícita de patrones bajo una clave `match`.
   -   **Campos Clave**:
        -   `listen`: Un `Vec<SocketAddr>` indicando en qué direcciones IP y puertos escuchará este servidor. Puede ser una sola cadena o una lista de cadenas. Manejado por `one_or_many` en `deser.rs`.
            -   Referencia: `src/config/mod.rs:48`, `src/config/deser.rs:35` (función `one_or_many`).
        -   `patterns`: Un `Vec<Pattern>` que define las reglas de enrutamiento. Si se usa un patrón simple (sin la clave `match`), `ServerVisitor` lo convierte en un vector de un solo elemento.
            -   Referencia: `src/config/mod.rs:52`.
        -   `max_connections`: Límite de conexiones concurrentes para este servidor. Por defecto a `1024` (definido en `default::max_connections`).
            -   Referencia: `src/config/mod.rs:56`, `src/config/mod.rs:219`.
        -   `name`: Nombre opcional para el servidor, usado en logs y en el header `Forwarded`.
            -   Referencia: `src/config/mod.rs:60`.
        -   `log_name`: Nombre usado para logging, generado internamente, incluye la IP y el nombre si está presente.
            -   Referencia: `src/config/mod.rs:65`.
   -   **Archivo TOML Ejemplo (simple y con `match`)**:
        ```toml
        # Simple (se convierte internamente a un patrón)
        [[server]]
        listen = "127.0.0.1:8000"
        forward = "127.0.0.1:9000"
        uri = "/api" # Opcional, por defecto "/"

        # Con `match` explícito
        [[server]]
        listen = "127.0.0.1:8001"
        match = [
            { uri = "/app", serve = "/var/www/app" },
            { uri = "/data", forward = "127.0.0.1:9002" }
        ]
        ```
   -   **Línea de Referencia Estructura**: `src/config/mod.rs:46`
   -   **Línea de Referencia Deserialización**: `src/config/deser.rs:158` (impl `Deserialize` for `Server`)

### 3. `Pattern`
   -   **Definición**:
        ```rust
        #[derive(Serialize, Deserialize, Debug, Clone)]
        pub struct Pattern {
            #[serde(default = "default::uri")]
            pub uri: String,
            #[serde(flatten)]
            pub action: Action,
        }
        ```
   -   **Propósito**: Define una regla de coincidencia de URI y la acción a tomar si la URI de una solicitud coincide.
   -   **Campos Clave**:
        -   `uri`: Un prefijo de URI. Si la URI de la solicitud comienza con este valor, el patrón coincide. Por defecto es `/` (definido en `default::uri`).
            -   Referencia: `src/config/mod.rs:80`, `src/config/mod.rs:215`.
        -   `action`: La `Action` a ejecutar. `#[serde(flatten)]` significa que los campos de `Action` (ej. `forward` o `serve`) aparecen directamente bajo el patrón en el TOML.
            -   Referencia: `src/config/mod.rs:83`.
   -   **Archivo TOML Ejemplo**:
        ```toml
        # Dentro de un bloque `match = [` o como patrón simple de un servidor
        { uri = "/images", serve = "/var/www/images" }
        { uri = "/api/v1", forward = "http://localhost:3000" }
        ```
   -   **Línea de Referencia**: `src/config/mod.rs:77`

### 4. `Action`
   -   **Definición**:
        ```rust
        #[derive(Serialize, Deserialize, Debug, Clone)]
        #[serde(rename_all = "lowercase")]
        pub enum Action {
            Forward(Forward),
            Serve(String),
        }
        ```
   -   **Propósito**: Enum que representa la acción a tomar cuando un `Pattern` coincide.
   -   **Variantes**:
        -   `Forward(Forward)`: Reenvía la solicitud a uno o más servidores backend. Contiene la configuración de `Forward`.
        -   `Serve(String)`: Sirve archivos estáticos desde el directorio especificado (la `String` es la ruta al directorio raíz).
   -   **Archivo TOML Ejemplo**:
        ```toml
        # Para Forward (la clave "forward" en el TOML)
        forward = "127.0.0.1:8080"
        # o
        forward = { algorithm = "WRR", backends = [{ address = "...", weight = ... }] }

        # Para Serve (la clave "serve" en el TOML)
        serve = "/path/to/static/files"
        ```
   -   **Línea de Referencia**: `src/config/mod.rs:190`

### 5. `Forward`
   -   **Definición (parcial)**:
        ```rust
        #[derive(Serialize, Deserialize)]
        #[serde(from = "ForwardOption")]
        pub struct Forward {
            pub backends: Vec<Backend>,
            pub algorithm: Algorithm,
            #[serde(skip)]
            pub scheduler: Box<dyn Scheduler + Sync + Send>,
        }
        ```
   -   **Propósito**: Contiene la configuración para la acción de reenvío (proxy).
   -   **Deserialización Personalizada**: Utiliza `#[serde(from = "ForwardOption")]`, lo que significa que primero se deserializa a `ForwardOption` (en `deser.rs`) y luego se convierte a `Forward`. Esto permite múltiples sintaxis para definir los backends y el algoritmo.
   -   **Campos Clave**:
        -   `backends`: Un `Vec<Backend>` listando los servidores upstream.
            -   Referencia: `src/config/mod.rs:161`.
        -   `algorithm`: El `Algorithm` de balanceo de carga a usar (ej. `WRR`).
            -   Referencia: `src/config/mod.rs:164`.
        -   `scheduler`: Una instancia concreta del trait `Scheduler` (ej. `WeightedRoundRobin`), creada durante la deserialización (específicamente en `ForwardOption::from` dentro de `src/config/deser.rs:146`). No se serializa/deserializa directamente (`#[serde(skip)]`).
            -   Referencia: `src/config/mod.rs:168`.
   -   **Línea de Referencia Estructura**: `src/config/mod.rs:158`
   -   **Línea de Referencia Deserialización**: `src/config/deser.rs:118` (`ForwardOption`)

### 6. `Backend`
   -   **Definición**:
        ```rust
        #[derive(Serialize, Deserialize, Debug, Clone)]
        #[serde(from = "BackendOption")]
        pub struct Backend {
            pub address: SocketAddr,
            pub weight: usize,
        }
        ```
   -   **Propósito**: Representa un único servidor backend (upstream).
   -   **Deserialización Personalizada**: Utiliza `#[serde(from = "BackendOption")]`. `BackendOption` en `deser.rs` permite que un backend se defina simplemente como una dirección IP:puerto, o como un objeto con `address` y `weight`.
   -   **Campos Clave**:
        -   `address`: La `SocketAddr` del servidor backend.
        -   `weight`: El peso del backend para algoritmos de balanceo de carga ponderados (como WRR). Por defecto es 1 si no se especifica (manejado en `BackendOption::from`).
            -   Referencia: `src/config/deser.rs:61`.
   -   **Archivo TOML Ejemplo**:
        ```toml
        # Formato simple (dentro de una lista de `forward`)
        "127.0.0.1:8080"

        # Formato con peso (dentro de una lista de `forward` o `backends`)
        { address = "127.0.0.1:8081", weight = 2 }
        ```
   -   **Línea de Referencia Estructura**: `src/config/mod.rs:105`
   -   **Línea de Referencia Deserialización**: `src/config/deser.rs:53` (`BackendOption`)

### 7. `Algorithm`
   -   **Definición**:
        ```rust
        #[derive(Serialize, Deserialize, Debug, Clone, Copy)]
        pub enum Algorithm {
            #[serde(rename = "WRR")]
            Wrr,
        }
        ```
   -   **Propósito**: Enum para los algoritmos de balanceo de carga soportados. Actualmente solo `WRR` (Weighted Round Robin).
   -   **Línea de Referencia**: `src/config/mod.rs:138`

## Lógica de Deserialización Personalizada (`src/config/deser.rs`)

Este archivo es crucial para la flexibilidad de la configuración.

### `one_or_many<'de, T, D>(...) -> Result<Vec<T>, D::Error>`
   -   **Propósito**: Permite que un campo en TOML que espera una lista (array) también acepte un solo valor. Si es un solo valor, se envuelve en un `Vec` de un elemento.
   -   **Ejemplo de Uso**: El campo `listen` de un `Server` puede ser `"127.0.0.1:8080"` o `["127.0.0.1:8080", "127.0.0.1:8081"]`.
   -   **Implementación**: Deserializa a un enum intermedio `OneOrMany<T>` que tiene variantes `One(T)` y `Many(Vec<T>)`, y luego convierte esto a `Vec<T>`.
   -   **Línea de Referencia**: `src/config/deser.rs:35`

### `BackendOption`
   -   **Definición**:
        ```rust
        #[derive(Serialize, Deserialize, Debug)]
        #[serde(untagged)]
        pub(super) enum BackendOption {
            Simple(SocketAddr),
            Weighted { address: SocketAddr, weight: usize },
        }
        ```
   -   **Propósito**: Permite que un backend se especifique como una `SocketAddr` simple (ej. `"127.0.0.1:8080"`) o como una tabla con `address` y `weight`.
   -   **Conversión**: `impl From<BackendOption> for Backend` convierte `BackendOption` a la estructura `Backend` final, asignando un peso de `1` si se usó la forma simple.
   -   **Línea de Referencia**: `src/config/deser.rs:53` (enum), `src/config/deser.rs:61` (impl From)

### `ForwardOption`
   -   **Definición**:
        ```rust
        #[derive(Serialize, Deserialize, Debug)]
        #[serde(untagged)]
        pub(super) enum ForwardOption {
            #[serde(deserialize_with = "one_or_many")]
            Simple(Vec<Backend>), // Permite una sola dirección o lista de direcciones/backends
            WithAlgorithm {
                algorithm: Algorithm,
                backends: Vec<Backend>,
            },
        }
        ```
   -   **Propósito**: Permite múltiples formas de definir la configuración de `forward`:
        1.  Una única dirección de backend (ej. `forward = "127.0.0.1:8080"`).
        2.  Una lista de direcciones/backends (ej. `forward = ["127.0.0.1:8080", { address = "127.0.0.1:8081", weight = 2 }]`).
        3.  Una tabla explícita con `algorithm` y `backends`.
   -   **Conversión**: `impl From<ForwardOption> for Forward` convierte `ForwardOption` a `Forward`.
        -   Si se usa la forma `Simple`, el algoritmo por defecto es `Algorithm::Wrr`.
        -   Importante: Aquí es donde se instancia el `Scheduler` (`sched::make(algorithm, &backends)`).
   -   **Línea de Referencia**: `src/config/deser.rs:118` (enum), `src/config/deser.rs:134` (impl From)

### `ServerVisitor` y `impl<'de> Deserialize<'de> for Server`
   -   **Propósito**: Maneja la deserialización de la estructura `Server`. La principal complejidad que aborda es permitir que un `Server` se defina con:
        1.  Un "patrón simple": campos `uri`, `forward` o `serve` directamente bajo `[[server]]`.
        2.  Una "cláusula `match`": un campo `match` que es una lista de `Pattern`s.
        No permite mezclar un patrón simple con una cláusula `match`, ni múltiples acciones (ej. `forward` y `serve`) en un patrón simple.
   -   **Implementación**:
        -   `impl<'de> Deserialize<'de> for Server` (línea `src/config/deser.rs:158`): Llama a `deserializer.deserialize_struct` con `ServerVisitor`.
        -   `ServerVisitor` (línea `src/config/deser.rs:167`): Implementa el trait `serde::de::Visitor`. Su método `visit_map` itera sobre los campos del TOML para un `[[server]]`.
        -   `Field` enum (línea `src/config/deser.rs:173`): Ayuda a identificar los campos mientras se parsea.
        -   Maneja la lógica para distinguir entre un patrón simple y una cláusula `match`, y para construir la lista `patterns` del `Server`.
        -   Realiza validaciones como:
            -   No mezclar patrón simple y `match` (`Error::MixedSimpleAndMatch`, línea `src/config/deser.rs:207`).
            -   No mezclar acciones `forward` y `serve` en un patrón simple (`Error::MixedActions`, línea `src/config/deser.rs:220`).
            -   Asegurar que haya al menos una configuración de acción (`Error::MissingConfig`, línea `src/config/deser.rs:228`).
            -   Asegurar que el campo `listen` esté presente.
   -   **Línea de Referencia**: `src/config/deser.rs:167` (ServerVisitor)

### Valores por Defecto (`src/config/mod.rs::default`)
   -   El módulo `default` (líneas `src/config/mod.rs:212-222`) proporciona funciones para valores por defecto usados por `serde`, como `default::uri()` (devuelve `/`) y `default::max_connections()` (devuelve `1024`).

## Flujo de Carga de Configuración

1.  En `main.rs` (línea `src/main.rs:3`): `tokio::fs::read_to_string("rxh.toml").await?` lee el archivo.
2.  `toml::from_str(&config_string)?` invoca a `serde_toml` para parsear la cadena.
3.  `serde` utiliza las implementaciones de `Deserialize` (incluyendo las personalizadas en `deser.rs`) para poblar la estructura `Config` y sus componentes.
    -   `ServerVisitor` maneja la lógica compleja para `Server`.
    -   `ForwardOption` y `BackendOption` permiten las sintaxis flexibles para `forward` y `backends`.
    -   `one_or_many` permite flexibilidad en campos que son listas.
4.  Durante la conversión de `ForwardOption` a `Forward`, se crea la instancia del `Scheduler` (`sched::make`).
5.  El resultado es una estructura `Config` completamente poblada, lista para ser usada por el `Master` task.

## Conclusión

El módulo `config` de RXH es un ejemplo robusto de cómo usar `serde` con deserialización personalizada para crear una experiencia de configuración potente y amigable para el usuario. La separación de las estructuras de datos (`mod.rs`) y la lógica de deserialización (`deser.rs`) mantiene el código organizado. La capacidad de definir servidores, patrones y acciones de múltiples maneras sin sacrificar la estructura interna es un logro clave de este módulo.
