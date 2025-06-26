# Documentación del Módulo de Planificación (Scheduler) (`src/sched/`)

## Visión General

El módulo `sched` (scheduler o planificador) es responsable de la lógica de balanceo de carga en RXH. Cuando una solicitud se configura para ser reenviada (`Forward`) a múltiples servidores backend, un planificador decide a cuál de esos backends se debe enviar la solicitud actual.

El módulo define un trait genérico `Scheduler` y proporciona una factoría para crear instancias concretas de planificadores basadas en la configuración.

## Estructura del Módulo

-   **`src/sched/mod.rs`**:
    -   Define el trait `Scheduler`.
    -   Proporciona la función `make` que actúa como una factoría para diferentes algoritmos de planificación.
    -   Declara submódulos (actualmente solo `wrr`).
    -   Línea de Referencia: `src/sched/mod.rs:1`

-   **`src/sched/wrr.rs`**:
    -   Implementación del algoritmo de balanceo de carga Weighted Round Robin (WRR).
    -   Línea de Referencia: `src/sched/wrr.rs:1`

## Componentes Detallados

### 1. Trait `Scheduler` (`src/sched/mod.rs`)

-   **Definición**:
    ```rust
    pub trait Scheduler {
        fn next_server(&self) -> SocketAddr;
        // fn request_processed(server: SocketAddr); // Comentado, para futuros algoritmos
    }
    ```
    -   **Línea de Referencia**: `src/sched/mod.rs:11`

-   **Propósito**: Define la interfaz común para todos los algoritmos de balanceo de carga.
-   **Métodos**:
    -   **`next_server(&self) -> SocketAddr`**:
        -   Este es el método principal. Cuando se llama, el planificador debe devolver la `SocketAddr` del servidor backend que debería manejar la próxima solicitud.
        -   La implementación de este método contendrá la lógica específica del algoritmo de balanceo de carga.
    -   **`request_processed(server: SocketAddr)`** (actualmente comentado):
        -   Este método (comentado en el código fuente actual, línea `src/sched/mod.rs:17`) podría ser utilizado por algoritmos más avanzados (como "Least Connections" o algoritmos adaptativos) para notificar al planificador que una solicitud a un servidor particular ha sido completada. Esto permitiría al planificador ajustar su estado interno o sus decisiones futuras.

-   **Requisitos de Trait**: El trait `Scheduler` debe ser implementado por tipos que también sean `Send + Sync` porque el planificador puede ser compartido entre múltiples hilos/tareas que manejan solicitudes concurrentes. Esto se ve en la firma de `sched::make` y en el campo `scheduler` de `config::Forward`.

### 2. Factoría `make` (`src/sched/mod.rs`)

-   **Definición**:
    ```rust
    pub fn make(algorithm: Algorithm, backends: &Vec<Backend>) -> Box<dyn Scheduler + Send + Sync> {
        Box::new(match algorithm {
            Algorithm::Wrr => WeightedRoundRobin::new(backends),
        })
    }
    ```
    -   **Línea de Referencia**: `src/sched/mod.rs:21`

-   **Propósito**: Construye y devuelve una instancia concreta de un `Scheduler` (empaquetada en un `Box<dyn ...>`) basada en el `Algorithm` especificado y la lista de `Backend`s.
-   **Parámetros**:
    -   `algorithm: Algorithm`: Un enum (definido en `src/config/mod.rs:138`) que especifica qué algoritmo de balanceo de carga usar. Actualmente, solo soporta `Algorithm::Wrr`.
    -   `backends: &Vec<Backend>`: Una referencia a un vector de estructuras `Backend` (definidas en `src/config/mod.rs:105`), cada una conteniendo la dirección (`SocketAddr`) y el peso (`weight`) de un servidor upstream.
-   **Retorno**: Devuelve un `Box<dyn Scheduler + Send + Sync>`, que es un puntero en el heap a un objeto que implementa el trait `Scheduler` y es seguro para enviar entre hilos.
-   **Uso**: Esta función es llamada típicamente durante la deserialización de la configuración, específicamente cuando se convierte `ForwardOption` a `Forward` en `src/config/deser.rs:146`. El `Box<dyn Scheduler>` resultante se almacena en el campo `scheduler` de la estructura `config::Forward`.

### 3. `WeightedRoundRobin` (`src/sched/wrr.rs`)

-   **Definición (parcial)**:
    ```rust
    #[derive(Debug)]
    pub struct WeightedRoundRobin {
        cycle: Ring<SocketAddr>,
    }
    ```
    -   **Línea de Referencia**: `src/sched/wrr.rs:18`

-   **Propósito**: Implementa el algoritmo de balanceo de carga Weighted Round Robin (WRR). WRR distribuye las solicitudes a los servidores backend de forma secuencial, pero la cantidad de solicitudes que cada servidor recibe en un ciclo es proporcional a su peso. Un servidor con un peso mayor recibirá más solicitudes que uno con un peso menor.

-   **Campos**:
    -   `cycle: Ring<SocketAddr>`:
        -   Almacena una secuencia precalculada de direcciones de servidores backend. Esta secuencia representa un ciclo completo de WRR.
        -   `Ring` es una estructura de datos personalizada (definida en `src/sync/ring.rs`) que proporciona acceso circular y atómico a los elementos de un vector. Esto es eficiente porque el ciclo WRR se calcula una vez en la inicialización, y luego `next_server` solo necesita obtener el siguiente elemento del `Ring`.
        -   Línea de Referencia: `src/sched/wrr.rs:22`

-   **`new(backends: &Vec<Backend>) -> Self`**:
    -   **Propósito**: Constructor para `WeightedRoundRobin`.
    -   **Lógica de Inicialización**:
        1.  Crea un vector vacío `cycle_vec`.
        2.  Itera sobre cada `Backend` en el vector `backends` de entrada.
        3.  Para cada backend, añade su `address` al `cycle_vec` un número de veces igual a su `weight`.
            -   **Ejemplo**: Si un backend tiene `address: A` y `weight: 3`, la dirección `A` se añade tres veces al `cycle_vec`.
            -   **TODO**: El comentario `// TODO: Interleaved WRR` (línea `src/sched/wrr.rs:30`) sugiere que la implementación actual es un WRR simple (no intercalado). Un WRR intercalado intentaría distribuir las selecciones de servidores de mayor peso de manera más uniforme a lo largo del ciclo en lugar de agruparlas. Por ejemplo, con A(2), B(1), un WRR simple podría ser `[A, A, B]`, mientras que uno intercalado podría ser `[A, B, A]`. La implementación actual es la simple: `[A, A, ..., B, B, ..., C, C, ...]`.
        4.  Crea una instancia de `Ring::new(cycle_vec)` y la almacena en el campo `cycle`.
    -   **Línea de Referencia**: `src/sched/wrr.rs:26`

-   **`impl Scheduler for WeightedRoundRobin`**:
    -   **`next_server(&self) -> SocketAddr`**:
        -   **Implementación**: Simplemente llama a `self.cycle.next_as_owned()`. El `Ring` se encarga de seleccionar atómicamente el siguiente `SocketAddr` del ciclo precalculado, volviendo al principio cuando llega al final.
        -   **Línea de Referencia**: `src/sched/wrr.rs:44`

-   **Ejemplo de Funcionamiento de WRR**:
    -   Servidores: A (peso 1), B (peso 3), C (peso 2).
    -   El `cycle` precalculado (según la implementación actual) sería: `[A, B, B, B, C, C]`.
    -   Llamadas sucesivas a `next_server()` devolverían: `A`, `B`, `B`, `B`, `C`, `C`, `A`, `B`, ... y así sucesivamente.

## Integración con el Resto del Sistema

1.  **Configuración (`src/config/`)**:
    -   El usuario especifica los backends y, opcionalmente, un algoritmo en el archivo TOML.
    -   Durante la deserialización (en `src/config/deser.rs`, específicamente `ForwardOption::from`), se llama a `sched::make(algorithm, &backends)`.
    -   La instancia del `Scheduler` (ej. `WeightedRoundRobin`) se almacena en `config::Forward.scheduler`.

2.  **Manejo de Solicitudes (`src/service/mod.rs`)**:
    -   Cuando una solicitud HTTP coincide con un `Pattern` cuya `Action` es `Forward`:
        -   El servicio `Rxh` accede al campo `scheduler` de la estructura `config::Forward` asociada.
        -   Llama al método `scheduler.next_server()` para obtener la `SocketAddr` del próximo backend al que se debe reenviar la solicitud.
        -   Esta `SocketAddr` se pasa luego a `src/service/proxy.rs::forward` para realizar la conexión y el reenvío.

## Puntos Clave y Consideraciones

-   **Extensibilidad**: El diseño con el trait `Scheduler` y la factoría `make` facilita la adición de nuevos algoritmos de balanceo de carga en el futuro. Solo se necesitaría:
    1.  Añadir una nueva variante al enum `config::Algorithm`.
    2.  Implementar la nueva estructura del planificador (que implemente `trait Scheduler`).
    3.  Actualizar la función `sched::make` para construir el nuevo planificador.
-   **Eficiencia de WRR**: La implementación de `WeightedRoundRobin` es eficiente para la selección del próximo servidor porque precalcula el ciclo. La selección en tiempo de ejecución es una simple recuperación de un elemento de `Ring`, que es atómica y rápida.
-   **Atomicidad**: El uso de `sync::ring::Ring` (que internamente usa `AtomicUsize`) es crucial para que el planificador sea seguro para usarse concurrentemente por múltiples tareas que podrían estar solicitando el `next_server` al mismo tiempo.
-   **WRR No Intercalado**: Como se mencionó, el WRR actual no es intercalado. Para cargas muy pequeñas o patrones de tráfico específicos, un WRR intercalado podría ofrecer una distribución ligeramente mejor, pero la implementación actual es más simple.
-   **Estado del Scheduler**: El `Scheduler` es de solo lectura (`&self` en `next_server`). Los algoritmos que necesitan modificar su estado (como "Least Connections" o algunos tipos de WRR dinámicos) requerirían acceso mutable, probablemente a través de primitivas de sincronización interna como `Mutex` o `RwLock`, o utilizando `Atomic`s para sus contadores.

El módulo `sched` proporciona una base sólida y extensible para las capacidades de balanceo de carga de RXH, comenzando con una implementación eficiente y comúnmente utilizada de WRR.
