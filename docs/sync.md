# Documentación del Módulo de Sincronización (`src/sync/`)

## Visión General

El módulo `sync` de RXH proporciona primitivas de sincronización personalizadas que se utilizan en otras partes del proyecto, especialmente para la coordinación entre tareas asíncronas de Tokio. Actualmente, contiene dos componentes principales: un sistema de notificación para el apagado ordenado (`notify`) y una estructura de datos de anillo (array circular) optimizada para acceso concurrente (`ring`).

## Estructura del Módulo

-   **`src/sync/mod.rs`**:
    -   Declara los submódulos `notify` y `ring`.
    -   Línea de Referencia: `src/sync/mod.rs:3`

-   **`src/sync/notify.rs`**:
    -   Implementa un mecanismo de notificación entre tareas, utilizado principalmente para el apagado ordenado (graceful shutdown).
    -   Define `Notifier`, `Subscription`, y `Notification`.
    -   Línea de Referencia: `src/sync/notify.rs:1`

-   **`src/sync/ring.rs`**:
    -   Implementa `Ring<T>`, una estructura de datos de array circular de solo lectura que permite a múltiples tareas obtener elementos de forma secuencial y atómica.
    -   Utilizado por el planificador `WeightedRoundRobin` (`src/sched/wrr.rs`).
    -   Línea de Referencia: `src/sync/ring.rs:1`

## Componentes Detallados

### 1. `src/sync/notify.rs` - Sistema de Notificación

Este sistema está diseñado para permitir que una tarea (el "notificador") envíe una señal a múltiples tareas suscriptoras y luego espere a que todas ellas acusen recibo de la notificación. Es fundamental para el apagado ordenado de RXH.

-   **`Notification` enum**:
    -   **Definición**:
        ```rust
        #[derive(Clone, Copy, Debug)]
        pub(crate) enum Notification {
            Shutdown,
        }
        ```
    -   **Propósito**: Representa el tipo de mensaje que se puede enviar. Actualmente, solo existe `Shutdown`, indicando a las tareas que deben comenzar su proceso de finalización.
    -   **Línea de Referencia**: `src/sync/notify.rs:14`

-   **`Notifier` struct**:
    -   **Definición (parcial)**:
        ```rust
        pub(crate) struct Notifier {
            notification_sender: broadcast::Sender<Notification>,
            acknowledge_receiver: mpsc::Receiver<()>,
            acknowledge_sender: mpsc::Sender<()>,
        }
        ```
    -   **Propósito**: La entidad que envía notificaciones y recopila acuses de recibo.
    -   **Campos**:
        -   `notification_sender: broadcast::Sender<Notification>`: El emisor de un canal de difusión (`tokio::sync::broadcast`). Todas las `Subscription`s activas recibirán mensajes enviados a través de este emisor.
        -   `acknowledge_receiver: mpsc::Receiver<()>`: El receptor de un canal MPSC (`tokio::sync::mpsc`). Cada `Subscription` envía un acuse de recibo (`()`) a este canal. El `Notifier` usa el receptor para contar o esperar los acuses.
        -   `acknowledge_sender: mpsc::Sender<()>`: Un emisor del canal MPSC de acuses. El `Notifier` clona este emisor para cada nueva `Subscription`.
    -   **Línea de Referencia**: `src/sync/notify.rs:20`
    -   **Métodos Clave**:
        -   **`new() -> Self`**: (Línea `src/sync/notify.rs:35`)
            -   Constructor. Inicializa el canal `broadcast` para notificaciones (capacidad 1, ya que solo se espera la última notificación de apagado) y el canal `mpsc` para acuses de recibo (capacidad 1, aunque la capacidad aquí es menos crítica ya que los `send().await` de los suscriptores esperarán si está lleno).
        -   **`subscribe(&self) -> Subscription`**: (Línea `src/sync/notify.rs:49`)
            -   Crea y devuelve una nueva `Subscription`.
            -   Llama a `self.notification_sender.subscribe()` para obtener un nuevo receptor para el canal de difusión.
            -   Clona `self.acknowledge_sender` para la nueva `Subscription`.
        -   **`send(&self, notification: Notification) -> Result<usize, broadcast::error::SendError<Notification>>`**: (Línea `src/sync/notify.rs:59`)
            -   Envía una `Notification` a todos los suscriptores activos a través del canal de difusión.
            -   Devuelve el número de suscriptores que recibieron el mensaje (o un error si no hay ninguno).
        -   **`collect_acknowledgements(self)` async**: (Línea `src/sync/notify.rs:67`)
            -   Espera a que todos los suscriptores envíen sus acuses de recibo.
            -   **Importante**: Primero hace `drop(acknowledge_sender)` (el que posee el `Notifier`). Esto es crucial porque un canal `mpsc` solo se cierra (y `recv()` devuelve `None`) cuando todos los emisores (`mpsc::Sender`) han sido eliminados. El `Notifier` espera que todas las `Subscription`s eliminen sus `acknowledge_sender` al terminar o al enviar su acuse y luego ser eliminadas.
            -   Luego, entra en un bucle `while let Some(_ack) = acknowledge_receiver.recv().await {}` que consume los acuses hasta que el canal se cierra (todos los emisores de las `Subscription`s han sido eliminados).
            -   Finalmente, hace `drop(notification_sender)` para limpiar el canal de difusión.

-   **`Subscription` struct**:
    -   **Definición (parcial)**:
        ```rust
        pub(crate) struct Subscription {
            notification_receiver: broadcast::Receiver<Notification>,
            acknowledge_sender: mpsc::Sender<()>,
        }
        ```
    -   **Propósito**: Utilizado por las tareas "trabajadoras" para recibir notificaciones y enviar acuses de recibo.
    -   **Campos**:
        -   `notification_receiver: broadcast::Receiver<Notification>`: El receptor del canal de difusión para recibir notificaciones.
        -   `acknowledge_sender: mpsc::Sender<()>`: El emisor para enviar un acuse de recibo al `Notifier`.
    -   **Línea de Referencia**: `src/sync/notify.rs:89`
    -   **Métodos Clave**:
        -   **`new(notification_receiver: broadcast::Receiver<Notification>, acknowledge_sender: mpsc::Sender<()>) -> Self`**: (Línea `src/sync/notify.rs:99`)
            -   Constructor simple.
        -   **`receive_notification(&mut self) -> Option<Notification>`**: (Línea `src/sync/notify.rs:116`)
            -   Intenta recibir una notificación del canal de difusión de forma no bloqueante (`try_recv()`).
            -   Descarta errores `Closed` y `Lagged` (asumiendo que no deberían ocurrir con el uso correcto del `Notifier`). Devuelve `None` si el buffer está vacío.
        -   **`acknowledge_notification(&self)` async**: (Línea `src/sync/notify.rs:127`)
            -   Envía un acuse de recibo (`()`) al `Notifier` a través del canal `mpsc`.
            -   Hace `unwrap()` en el resultado del envío, asumiendo que el receptor del `Notifier` siempre estará presente mientras se necesiten acuses.

-   **Flujo de Apagado Ordenado usando `notify`**:
    1.  Una entidad central (ej. `task::Server`) crea un `Notifier`.
    2.  Cada tarea trabajadora (ej. una tarea que maneja una conexión HTTP) obtiene una `Subscription` del `Notifier` al inicio.
    3.  En su bucle principal o en puntos de chequeo, la tarea trabajadora llama a `subscription.receive_notification()`.
    4.  Cuando se inicia el apagado, la entidad central llama a `notifier.send(Notification::Shutdown)`.
    5.  Las tareas trabajadoras detectan la `Notification::Shutdown`. Terminan su trabajo actual (ej. completar la solicitud HTTP en curso).
    6.  Una vez finalizado, la tarea trabajadora llama a `subscription.acknowledge_notification().await`. Luego, la tarea puede terminar, lo que eliminará su `Subscription` y, por lo tanto, su `acknowledge_sender`.
    7.  La entidad central, después de enviar la notificación, llama a `notifier.collect_acknowledgements().await`. Este futuro se completará solo cuando todos los acuses hayan sido recibidos (es decir, todos los `acknowledge_sender` de las `Subscription`s hayan sido eliminados).

### 2. `src/sync/ring.rs` - Array Circular Atómico

-   **`Ring<T>` struct**:
    -   **Definición (parcial)**:
        ```rust
        #[derive(Debug)]
        pub(crate) struct Ring<T> {
            values: Vec<T>,
            next: AtomicUsize,
        }
        ```
    -   **Propósito**: Proporciona acceso de solo lectura, circular y atómico a una secuencia de elementos `T`. Está diseñado para escenarios donde múltiples tareas necesitan obtener elementos de una lista de forma round-robin sin bloqueos explícitos.
    -   **Campos**:
        -   `values: Vec<T>`: El vector que almacena los elementos del anillo.
        -   `next: AtomicUsize`: Un contador atómico que almacena el índice del próximo elemento a devolver. El uso de `AtomicUsize` asegura que la obtención del próximo índice sea segura entre hilos.
    -   **Línea de Referencia**: `src/sync/ring.rs:8`
    -   **Métodos Clave**:
        -   **`new(values: Vec<T>) -> Self`**: (Línea `src/sync/ring.rs:18`)
            -   Constructor. Toma un `Vec<T>` y lo almacena.
            -   Inicializa `next` a `AtomicUsize::new(0)`.
            -   `assert!(values.len() > 0, ...)`: Asegura que el vector no esté vacío, ya que la lógica de módulo (`%`) podría fallar o el concepto de "siguiente" no tendría sentido.
        -   **`next_index(&self) -> usize`**: (Línea `src/sync/ring.rs:28`)
            -   Calcula el índice del próximo elemento.
            -   Si `values.len() == 1`, siempre devuelve `0`.
            -   Si no, usa `self.next.fetch_add(1, Ordering::Relaxed) % self.values.len()`.
                -   `fetch_add(1, Ordering::Relaxed)`: Incrementa atómicamente `self.next` en 1 y devuelve el valor *anterior* al incremento. `Ordering::Relaxed` es suficiente aquí porque solo nos interesa la atomicidad de la operación de incremento, no su ordenamiento con otras operaciones de memoria en diferentes hilos (el módulo se encarga del resto).
                -   `% self.values.len()`: Asegura que el índice se mantenga dentro de los límites del vector, creando el comportamiento circular.
        -   **`next_as_ref(&self) -> &T`**: (Línea `src/sync/ring.rs:38`)
            -   Devuelve una referencia al próximo elemento en el anillo, usando `next_index()`.
        -   **`next_as_owned(&self) -> T`** (requiere `T: Copy`): (Línea `src/sync/ring.rs:45`)
            -   Devuelve una copia del próximo elemento.
        -   **`next_as_cloned(&self) -> T`** (requiere `T: Clone`, actualmente `#[allow(dead_code)]`): (Línea `src/sync/ring.rs:53`)
            -   Devuelve un clon del próximo elemento.

-   **Uso en `WeightedRoundRobin`**:
    -   El planificador `WeightedRoundRobin` (`src/sched/wrr.rs`) usa `Ring<SocketAddr>` para almacenar su ciclo precalculado de direcciones de backend.
    -   Cuando `WeightedRoundRobin::next_server()` es llamado (potencialmente por múltiples tareas concurrentemente), llama a `ring.next_as_owned()` (ya que `SocketAddr` es `Copy`), lo cual es una operación atómica y eficiente para obtener el próximo backend en el ciclo.

## Puntos Clave

-   **Coordinación de Apagado**: El sistema `notify` es una solución elegante y robusta para el problema común del apagado ordenado en aplicaciones concurrentes. Se basa en primitivas de `tokio::sync` (`broadcast` y `mpsc`) para la comunicación.
-   **Seguridad en Concurrencia para Schedulers**: `Ring<T>` permite que los planificadores como `WRR` sean compartidos de forma segura y eficiente entre tareas sin necesidad de bloqueos más pesados (como `Mutex`) para la simple selección del próximo elemento. El uso de `AtomicUsize` es clave.
-   **Abstracciones Específicas del Dominio**: En lugar de usar directamente canales `broadcast` y `mpsc` en toda la lógica de apagado, o `AtomicUsize` y aritmética modular en los schedulers, `sync` proporciona abstracciones (`Notifier`, `Ring`) que son más expresivas y menos propensas a errores para sus casos de uso específicos.
-   **Rendimiento**: `Ordering::Relaxed` en `Ring::next_index` es una elección consciente para el rendimiento, adecuada porque la única garantía necesaria es la atomicidad del `fetch_add`, no un ordenamiento estricto con otras variables.

El módulo `sync` encapsula lógica de sincronización importante que contribuye a la robustez y eficiencia de RXH, especialmente en su manejo de la concurrencia y el ciclo de vida de las tareas.
