# Documentación del Módulo de Tareas (`src/task/`)

## Visión General

El módulo `task` define la arquitectura de ejecución de alto nivel de RXH. Introduce un modelo "master-server" (o "master-worker" donde los "workers" son servidores), donde una tarea principal (`Master`) gestiona el ciclo de vida de múltiples tareas de servidor (`Server`). Cada tarea `Server` es responsable de escuchar en una dirección de red específica y manejar las conexiones entrantes para esa dirección según su configuración.

Este módulo es crucial para la capacidad de RXH de manejar múltiples configuraciones de servidor simultáneamente y para gestionar su apagado ordenado de manera centralizada.

## Estructura del Módulo

-   **`src/task/mod.rs`**:
    -   Proporciona una breve descripción del módulo.
    -   Declara los submódulos `master` y `server`.
    -   Línea de Referencia: `src/task/mod.rs:1`

-   **`src/task/master.rs`**:
    -   Define la estructura `Master` y su lógica.
    -   El `Master` inicializa, ejecuta y supervisa el apagado de todas las instancias de `Server`.
    -   Línea de Referencia: `src/task/master.rs:1`

-   **`src/task/server.rs`**:
    -   Define la estructura `Server` y su lógica.
    -   Un `Server` representa una única entidad de escucha (en un `SocketAddr` específico) que acepta conexiones y las delega a manejadores de servicio.
    -   Gestiona el límite de conexiones y su propio proceso de apagado ordenado para las conexiones que maneja.
    -   Línea de Referencia: `src/task/server.rs:1`

## Componentes Detallados

### 1. `src/task/master.rs` - La Tarea `Master`

-   **`Master` struct**:
    -   **Definición (parcial)**:
        ```rust
        pub struct Master {
            servers: Vec<Server>, // Instancias de Server gestionadas
            states: Vec<(SocketAddr, watch::Receiver<State>)>, // Para observar el estado de cada Server
            shutdown: Pin<Box<dyn Future<Output = ()> + Send>>, // Futuro que dispara el apagado del Master
            shutdown_notify: broadcast::Sender<()>, // Canal para notificar a los Servers que se apaguen
        }
        ```
    -   **Propósito**: Orquesta el ciclo de vida completo de la aplicación RXH. Es responsable de:
        1.  Inicializar todas las instancias de `Server` basadas en la `config::Config` global.
        2.  Ejecutar estas instancias de `Server` como tareas de Tokio separadas.
        3.  Esperar una señal de apagado global (ej. Ctrl+C).
        4.  Propagar la señal de apagado a todas las instancias de `Server`.
        5.  Esperar a que todas las instancias de `Server` terminen su apagado ordenado.
    -   **Línea de Referencia**: `src/task/master.rs:63`
    -   **Campos Clave**:
        -   `servers: Vec<Server>`: Almacena las instancias de `Server` que el `Master` ha creado y gestionará.
        -   `states: Vec<(SocketAddr, watch::Receiver<State>)>`: Mantiene un receptor del canal `watch` para el estado de cada `Server`, permitiendo al `Master` (o a un observador externo) monitorear el estado de cada servidor. `State` es un enum definido en `src/task/server.rs`.
        -   `shutdown: Pin<Box<dyn Future<Output = ()> + Send>>`: Un futuro que, cuando se completa, inicia el proceso de apagado del `Master` y, por extensión, de todos sus `Server`s. Por defecto, es un futuro pendiente (`future::pending()`).
        -   `shutdown_notify: broadcast::Sender<()>`: Un emisor de un canal de difusión de Tokio. Cuando el `Master` decide apagarse, envía una señal `()` a través de este canal. Cada `Server` se suscribe a este canal para recibir esta notificación.

-   **`Master::init(config: Config) -> Result<Self, crate::Error>`**:
    -   **Línea de Referencia**: `src/task/master.rs:83`
    -   **Propósito**: Constructor. Inicializa el `Master` a partir de la `config::Config` global.
    -   **Lógica**:
        1.  Crea un canal `broadcast::channel(1)` para `shutdown_notify`.
        2.  Itera sobre cada `server_config` en `config.servers`.
        3.  **Manejo de Réplicas**: Si una `server_config` especifica múltiples direcciones en su campo `listen` (ej. `listen = ["127.0.0.1:8080", "127.0.0.1:8081"]`), el `Master` crea una instancia separada de `Server` para *cada una* de estas direcciones. A esto se refiere como "réplica" en los comentarios del código. Cada réplica comparte la misma configuración base (`server_config.clone()`) pero se inicializa con un índice de `listen` diferente.
            -   Esto simplifica el diseño del `Server`, ya que cada `Server` solo necesita manejar un único `TcpListener`. La complejidad de múltiples listeners por configuración de servidor se maneja en el `Master`.
            -   Para cada réplica/dirección, llama a `Server::init(server_config.clone(), replica_index)?`.
            -   Almacena el `Server` resultante y una suscripción a su estado (`server.subscribe()`) en `self.servers` y `self.states` respectivamente.
        4.  Inicializa `self.shutdown` a `Box::pin(future::pending())`.

-   **`Master::shutdown_on(mut self, future: impl Future + Send + 'static) -> Self`**:
    -   **Línea de Referencia**: `src/task/master.rs:106`
    -   **Propósito**: Permite configurar el futuro que activará el apagado del `Master`.
    -   **Lógica**:
        1.  Reemplaza `self.shutdown` con el `future` proporcionado.
        2.  Itera sobre `self.servers` y actualiza cada `Server` para que su propio apagado sea activado por una notificación del `self.shutdown_notify` del `Master`. Lo hace llamando a `server.shutdown_on(async move { shutdown_notification.recv().await })`, donde `shutdown_notification` es un `broadcast::Receiver` del `Master`.

-   **`Master::run(self) -> Result<(), crate::Error>` async**:
    -   **Línea de Referencia**: `src/task/master.rs:123`
    -   **Propósito**: El bucle principal de ejecución del `Master`.
    -   **Lógica**:
        1.  Crea un `tokio::task::JoinSet` para gestionar las tareas de los `Server`.
        2.  Para cada `server` en `self.servers`, lo ejecuta en una nueva tarea de Tokio usando `set.spawn(server.run())`.
        3.  Usa `tokio::select!` para esperar concurrentemente a dos eventos:
            -   La finalización del futuro `self.shutdown` (ej. Ctrl+C).
            -   Que alguna de las tareas `Server` en el `JoinSet` termine prematuramente con un error (`set.join_next()`).
        4.  **Si `self.shutdown` se completa o un servidor falla**:
            -   Imprime un mensaje indicando el inicio del apagado.
            -   Envía una señal `()` a través de `self.shutdown_notify.send(()).unwrap()`. Esto notificará a todos los `Server`s (que fueron configurados en `shutdown_on`) para que comiencen su propio apagado.
        5.  Espera a que todas las tareas `Server` en el `JoinSet` terminen (`while let Some(result) = set.join_next().await`).
        6.  Recopila cualquier error que haya ocurrido. Si hubo un error, lo devuelve. Si no, devuelve `Ok(())`.

-   **`Master::sockets(&self) -> Vec<SocketAddr>`**:
    -   **Línea de Referencia**: `src/task/master.rs:151`
    -   **Propósito**: Devuelve una lista de todas las `SocketAddr` en las que los `Server`s gestionados están escuchando. Útil para pruebas o información.

### 2. `src/task/server.rs` - La Tarea `Server`

-   **`Server` struct**:
    -   **Definición (parcial)**:
        ```rust
        pub struct Server {
            state: watch::Sender<State>, // Para publicar el estado actual del Server
            listener: TcpListener,       // El listener TCP para esta instancia de Server
            config: config::Server,      // La configuración específica para este Server
            address: SocketAddr,         // La SocketAddr real en la que está escuchando
            notifier: Notifier,          // Notifier para apagar ordenadamente las tareas de conexión
            shutdown: Pin<Box<dyn Future<Output = ()> + Send>>, // Futuro que dispara el apagado de este Server
            connections: Arc<Semaphore>, // Semáforo para limitar conexiones concurrentes
        }
        ```
    -   **Propósito**: Representa una única instancia de escucha de RXH. Es responsable de:
        1.  Escuchar en una `SocketAddr` específica.
        2.  Aceptar nuevas conexiones TCP.
        3.  Gestionar un límite de conexiones concurrentes usando un `Semaphore`.
        4.  Para cada conexión aceptada, generar una nueva tarea de Tokio que la maneje usando el servicio `Rxh` (`src/service/mod.rs`).
        5.  Manejar su propio proceso de apagado ordenado: notificar a todas las tareas de conexión activas y esperar a que terminen.
    -   **Línea de Referencia**: `src/task/server.rs:40`

-   **`State` enum**:
    -   **Definición**:
        ```rust
        pub enum State {
            Starting,
            Listening,
            MaxConnectionsReached(usize),
            ShuttingDown(ShutdownState),
        }
        ```
    -   **`ShutdownState` enum**:
        ```rust
        pub enum ShutdownState {
            PendingConnections(usize), // Número de conexiones activas restantes
            Done,
        }
        ```
    -   **Propósito**: Representa los diferentes estados por los que puede pasar un `Server`. Se publica a través de un canal `watch::Sender<State>`.
    -   **Líneas de Referencia**: `State` en `src/task/server.rs:60`, `ShutdownState` en `src/task/server.rs:71`.

-   **`Server::init(config: config::Server, replica: usize) -> Result<Self, io::Error>`**:
    -   **Línea de Referencia**: `src/task/server.rs:86`
    -   **Propósito**: Constructor. Prepara un `Server` para ejecutarse.
    -   **Lógica**:
        1.  Crea un canal `watch::channel(State::Starting)` para `self.state`.
        2.  Crea un `TcpSocket` (v4 o v6 según la dirección en `config.listen[replica]`).
        3.  Configura `socket.set_reuseaddr(true)` (en no-Windows).
        4.  Vincula (`bind`) el socket a `config.listen[replica]`. `replica` es el índice para seleccionar la dirección si la configuración del servidor tiene múltiples `listen` entries.
        5.  Comienza a escuchar (`listen`) en el socket para crear el `TcpListener`.
        6.  Obtiene la `local_addr()` real del listener (importante si el puerto era 0).
        7.  Crea un `Notifier` (`src/sync/notify.rs`) para este `Server`.
        8.  Inicializa `self.shutdown` a `future::pending()`.
        9.  Crea un `Arc<Semaphore>` con `config.max_connections` permisos.

-   **`Server::shutdown_on(mut self, future: impl Future + Send + 'static) -> Self`**:
    -   **Línea de Referencia**: `src/task/server.rs:127`
    -   **Propósito**: Configura el futuro que activará el apagado de *este* `Server` específico. Es llamado por `Master::shutdown_on`.
    -   **Lógica**: Asigna el `future` proporcionado a `self.shutdown`.

-   **`Server::run(self) -> Result<(), crate::Error>` async**:
    -   **Línea de Referencia**: `src/task/server.rs:147`
    -   **Propósito**: El bucle principal de ejecución del `Server`.
    -   **Lógica**:
        1.  Genera `log_name` para este servidor.
        2.  Actualiza `self.state` a `State::Listening`.
        3.  **Leak de Configuración**: `Box::leak(Box::new(config))` convierte `self.config` a `&'static config::Server`. Esto es necesario porque el servicio `Rxh` y las tareas que genera necesitan una referencia estática a la configuración. Se revierte con `Box::from_raw` al final para evitar una fuga de memoria real. (Línea `src/task/server.rs:204`)
        4.  Crea una estructura `Listener` interna (ver más abajo) que encapsula la lógica de aceptación de bucle.
        5.  Usa `tokio::select!` para esperar concurrentemente a:
            -   `listener.listen()`: El resultado del bucle de aceptación de conexiones.
            -   `self.shutdown`: La finalización del futuro de apagado de este servidor.
        6.  **Inicio del Apagado**:
            -   Cuando `select!` termina (ya sea por un error en `listen` o porque `self.shutdown` se completó), el `Server` comienza su apagado.
            -   Hace `drop(listener)` para cerrar el `TcpListener` y dejar de aceptar nuevas conexiones.
            -   Llama a `self.notifier.send(Notification::Shutdown)`. Si devuelve `Ok(num_tasks)`, significa que hay `num_tasks` conexiones activas que necesitan cerrarse.
            -   Actualiza `self.state` a `State::ShuttingDown(ShutdownState::PendingConnections(num_tasks))`.
            -   Llama a `self.notifier.collect_acknowledgements().await` para esperar a que todas las tareas de conexión activas acusen recibo de la notificación de apagado.
        7.  **Limpieza**: Deshace el `Box::leak` de la configuración usando `unsafe { Box::from_raw(...) }`.
        8.  Actualiza `self.state` a `State::ShuttingDown(ShutdownState::Done)`.

-   **`Listener` struct interna (`src/task/server.rs`)**:
    -   **Definición (parcial)**:
        ```rust
        struct Listener<'a> {
            listener: TcpListener,
            config: &'static config::Server,
            notifier: &'a Notifier, // Referencia al Notifier del Server
            state: &'a watch::Sender<State>, // Referencia al state sender del Server
            connections: Arc<Semaphore>, // Referencia al Semaphore del Server
        }
        ```
    -   **Línea de Referencia**: `src/task/server.rs:237`
    -   **`Listener::listen(&self) -> Result<(), crate::Error>` async**:
        -   **Línea de Referencia**: `src/task/server.rs:258`
        -   **Lógica del Bucle de Aceptación**:
            1.  Bucle infinito `loop { ... }`.
            2.  **Límite de Conexiones**:
                -   Si `self.connections.available_permits() == 0`, actualiza el estado a `State::MaxConnectionsReached` y espera a que un permiso esté disponible usando `self.connections.clone().acquire_owned().await.unwrap()`.
                -   Una vez que se obtiene un permiso, si el estado era `MaxConnectionsReached`, lo revierte a `State::Listening`.
            3.  Acepta una nueva conexión: `self.listener.accept().await?` -> `(stream, client_addr)`.
            4.  Obtiene una `Subscription` del `self.notifier` para la nueva tarea.
            5.  **Genera Tarea de Conexión**: Llama a `tokio::task::spawn(async move { ... })`.
                -   Dentro de la tarea:
                    -   Se sirve la conexión usando `hyper::server::conn::http1::Builder::new().serve_connection(stream, Rxh::new(config, client_addr, server_addr)).with_upgrades().await`. `Rxh::new` crea el servicio de Hyper.
                    -   Si la tarea recibe una `Notification::Shutdown` a través de su `subscription`, llama a `subscription.acknowledge_notification().await`.
                    -   Al final de la tarea (cuando la conexión se cierra o después del acuse de recibo), el `permit` del semáforo se libera automáticamente (porque `acquire_owned` devuelve un `OwnedSemaphorePermit` que lo hace en su `Drop`).

## Flujo de Vida de las Tareas

1.  **`main.rs`**: Crea e inicia `Master`.
2.  **`Master::init`**: Crea instancias de `Server` para cada dirección de escucha configurada.
3.  **`Master::run`**:
    -   Configura el futuro de apagado global del `Master` (ej. Ctrl+C).
    -   Llama a `server.shutdown_on(...)` para cada `Server`, vinculando su apagado a una notificación del `Master`.
    -   Ejecuta cada `Server::run` en una tarea de Tokio separada.
4.  **`Server::run`**:
    -   Comienza a escuchar conexiones (`Listener::listen`).
    -   Para cada conexión aceptada (y si hay permisos de semáforo):
        -   Genera una nueva tarea de Tokio.
        -   Esta tarea utiliza `Rxh` (el servicio de `src/service/mod.rs`) para manejar las solicitudes HTTP en esa conexión.
        -   La tarea también obtiene una `Subscription` al `Notifier` del `Server`.
5.  **Apagado**:
    -   Se activa el futuro de apagado del `Master` (Ctrl+C).
    -   `Master::run` detecta esto y envía `()` por su `shutdown_notify`.
    -   Cada `Server::run` detecta la señal de su futuro `shutdown` (que estaba esperando la notificación del `Master`).
    -   `Server::run` deja de aceptar conexiones (al hacer `drop(listener)`).
    -   `Server::run` envía `Notification::Shutdown` a través de su propio `Notifier` a todas sus tareas de conexión activas.
    -   Las tareas de conexión activas terminan su trabajo actual, envían un acuse (`acknowledge_notification`), y finalizan.
    -   `Server::run` espera todos los acuses (`collect_acknowledgements`) y luego finaliza su propia tarea.
    -   `Master::run` espera a que todas las tareas `Server` finalicen.
    -   La aplicación termina.

## Puntos Clave

-   **Modelo Jerárquico**: `Master` gestiona `Server`s, y cada `Server` gestiona tareas de conexión. Esto estructura bien la complejidad.
-   **Gestión de Réplicas**: El `Master` simplifica la configuración al permitir que una definición de servidor en TOML escuche en múltiples puertos, creando internamente instancias de `Server` separadas.
-   **Apagado Ordenado en Cascada**: El apagado se propaga desde el `Master` a los `Server`s, y de cada `Server` a sus tareas de conexión, utilizando el sistema `Notifier`/`Subscription` de `src/sync/notify.rs`.
-   **Control de Conexiones**: Cada `Server` utiliza un `Semaphore` para limitar el número de conexiones activas que maneja, previniendo la sobrecarga.
-   **Publicación de Estado**: El uso de `watch::channel` por parte de `Server` permite que su estado sea observable, lo cual es útil para pruebas, monitoreo o interfaces de gestión futuras.
-   **Manejo de Lifetime de Configuración**: El truco de `Box::leak` y `Box::from_raw` en `Server::run` es una técnica para proporcionar una referencia `'static` de la configuración a las tareas de Tokio sin usar `Arc` en todas partes, bajo la premisa de que la configuración vivirá tanto como las tareas que la usan.

El módulo `task` es fundamental para la concurrencia, la gestión del ciclo de vida y la robustez general de la aplicación RXH.
