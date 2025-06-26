# Documentación de Pruebas (`tests/`)

## Visión General

El proyecto RXH incluye un conjunto de pruebas de integración ubicadas en el directorio `tests/`. Estas pruebas están diseñadas para verificar la funcionalidad de extremo a extremo de las características clave del servidor, como el proxy inverso, el servicio de archivos estáticos, el balanceo de carga, el manejo de límites de conexión y el apagado ordenado.

Las pruebas hacen un uso extensivo de las capacidades asíncronas de Tokio y de utilidades personalizadas para simular clientes, servidores backend y para configurar instancias de RXH en entornos de prueba.

## Estructura del Directorio de Pruebas

-   **`tests/proxy.rs`**:
    -   Contiene la mayoría de las pruebas de integración. Cubre escenarios como:
        -   Funcionamiento básico del proxy inverso.
        -   Respuestas correctas para prefijos de URI incorrectos (404).
        -   Respuestas de error cuando los servidores backend no están disponibles (502).
        -   Verificación del header `Forwarded`.
        -   Apagado ordenado (graceful shutdown) y manejo de conexiones pendientes.
        -   Límites de conexión.
        -   Conexiones actualizadas (ej. WebSockets).
        -   Balanceo de carga (WRR).
        -   Servicio de archivos estáticos y errores 404 para archivos no encontrados.
        -   Funcionamiento de múltiples servidores (`Master` task) con diferentes configuraciones.
    -   Línea de Referencia: `tests/proxy.rs:1`

-   **`tests/util/`**:
    -   Este subdirectorio contiene módulos de utilidad que facilitan la escritura de las pruebas de integración.
    -   **`mod.rs`**: Declara los submódulos de utilidades. (Línea: `tests/util/mod.rs:1`)
    -   **`config.rs`**: Factorías para crear estructuras `rxh::config::Server` comunes usadas en las pruebas (ej. proxy a un solo backend, servir archivos). (Línea: `tests/util/config.rs:1`)
    -   **`http.rs`**: Utilidades para crear clientes HTTP, simular servidores backend, y ejecutar instancias de RXH (tanto `Server` como `Master`) en tareas de Tokio separadas para las pruebas. (Línea: `tests/util/http.rs:1`)
    -   **`service.rs`**: Abstracciones sobre servicios `hyper`, como `RequestInterceptor` que permite a las pruebas inspeccionar las solicitudes recibidas por un backend simulado. (Línea: `tests/util/service.rs:1`)
    -   **`tcp.rs`**: Utilidades de bajo nivel para TCP, como obtener sockets y listeners en puertos disponibles, y funciones para "hacer ping" (verificar conectividad) a servidores TCP antes de interactuar con ellos. (Línea: `tests/util/tcp.rs:1`)

## Utilidades Clave en `tests/util/`

### `tests/util/tcp.rs`

-   **`usable_socket() -> (TcpSocket, SocketAddr)`**:
    -   Crea un `TcpSocket` IPv4 vinculado al puerto 0 (`127.0.0.1:0`), lo que permite al SO elegir un puerto disponible. Devuelve el socket y la `SocketAddr` real asignada. Esencial para evitar colisiones de puertos en pruebas paralelas.
    -   Configura `SO_REUSEADDR`.
    -   Línea de Referencia: `tests/util/tcp.rs:10`
-   **`usable_tcp_listener() -> (TcpListener, SocketAddr)`**:
    -   Similar a `usable_socket`, pero llama a `socket.listen()` y devuelve un `TcpListener` y su `SocketAddr`.
    -   Línea de Referencia: `tests/util/tcp.rs:25`
-   **`ping_tcp_server(addr: SocketAddr)` async**:
    -   Intenta conectar a `addr` varias veces con pequeños delays (usando `tokio::task::yield_now()`). Si tiene éxito, cierra la conexión y retorna. Si falla después de varios intentos, entra en pánico. Asegura que un servidor esté listo antes de que una prueba intente usarlo.
    -   Línea de Referencia: `tests/util/tcp.rs:33`
-   **`ping_all(addrs: &[SocketAddr])` async**:
    -   Llama a `ping_tcp_server` para una lista de direcciones.
    -   Línea de Referencia: `tests/util/tcp.rs:51`

### `tests/util/http.rs`

Este es uno de los archivos de utilidad más importantes.

-   **`spawn_backend_server<S, B>(service: S) -> (SocketAddr, JoinHandle<()>)`**:
    -   Inicia un servidor backend simulado en una nueva tarea Tokio.
    -   `service` es una función o closure que maneja las solicitudes (debe implementar `hyper::service::Service`).
    -   Usa `usable_tcp_listener` para obtener un puerto.
    -   Devuelve la `SocketAddr` del backend y el `JoinHandle` de su tarea.
    -   Línea de Referencia: `tests/util/http.rs:21`
-   **`spawn_backend_server_with_request_counter(weight: usize) -> (Backend, Arc<AtomicUsize>)`**:
    -   Especialización de `spawn_backend_server`. El backend simulado simplemente cuenta las solicitudes que recibe usando un `Arc<AtomicUsize>`.
    -   Devuelve una estructura `rxh::config::Backend` (con la dirección y peso) y el contador. Útil para pruebas de balanceo de carga.
    -   Línea de Referencia: `tests/util/http.rs:43`
-   **`spawn_backends_with_request_counters(weights: &[usize]) -> (Vec<Backend>, Vec<Arc<AtomicUsize>>)`**:
    -   Crea múltiples backends contadores de solicitudes.
    -   Línea de Referencia: `tests/util/http.rs:66`
-   **`spawn_reverse_proxy(config: rxh::config::Server) -> (SocketAddr, JoinHandle<()>)`**:
    -   Inicia una instancia de `rxh::Server` (un proxy RXH) en una nueva tarea Tokio con la configuración proporcionada.
    -   Llama a `rxh::Server::init` y luego a `server.run()`.
    -   Devuelve la `SocketAddr` en la que escucha el proxy y el `JoinHandle` de su tarea.
    -   Línea de Referencia: `tests/util/http.rs:82`
-   **`spawn_reverse_proxy_with_controllers(...) -> (SocketAddr, JoinHandle<()>, impl FnOnce(), watch::Receiver<rxh::State>)`**:
    -   Similar a `spawn_reverse_proxy`, pero también devuelve:
        -   Una closure `FnOnce()` que, al ser llamada, dispara el apagado del servidor RXH (a través de un `oneshot::channel`).
        -   Un `watch::Receiver<rxh::State>` para observar los cambios de estado del servidor RXH.
    -   Esto permite a las pruebas controlar el ciclo de vida del servidor y verificar sus estados.
    -   Línea de Referencia: `tests/util/http.rs:95`
-   **`spawn_master(config: rxh::config::Config) -> (Vec<SocketAddr>, JoinHandle<()>)`**:
    -   Inicia una instancia de `rxh::Master` con una configuración RXH completa (potencialmente múltiples servidores).
    -   Devuelve un vector de `SocketAddr`s en las que escuchan los servidores del master y el `JoinHandle`.
    -   Línea de Referencia: `tests/util/http.rs:120`
-   **`http_client<B: AsyncBody>(stream: TcpStream) -> SendRequest<B>`**:
    -   Establece una conexión de cliente HTTP/1 sobre un `TcpStream` dado.
    -   Devuelve un `SendRequest` de Hyper, que puede usarse para enviar múltiples solicitudes sobre esa conexión.
    -   La tarea de conexión de Hyper se ejecuta en segundo plano.
    -   Línea de Referencia: `tests/util/http.rs:130`
-   **`send_http_request_from<B>(from: TcpSocket, to: SocketAddr, req: Request<B>) -> (Parts, Bytes)`**:
    -   Envía una única solicitud HTTP desde `from` (un `TcpSocket` no conectado) a `to`.
    -   Conecta el socket, crea un cliente, envía la `req`, obtiene la respuesta, lee el cuerpo completo y devuelve las partes de la respuesta y el cuerpo como `Bytes`.
    -   Línea de Referencia: `tests/util/http.rs:140`
-   **`send_http_request<B>(to: SocketAddr, req: Request<B>) -> (Parts, Bytes)`**:
    -   Wrapper sobre `send_http_request_from` que usa un `usable_socket()` nuevo para cada llamada.
    -   Línea de Referencia: `tests/util/http.rs:153`
-   **`spawn_client<B>(target: SocketAddr, req: Request<B>) -> (SocketAddr, JoinHandle<()>)`**:
    -   Ejecuta `send_http_request` en una nueva tarea Tokio. Devuelve la `SocketAddr` del cliente y el `JoinHandle`.
    -   Línea de Referencia: `tests/util/http.rs:161`
-   **`request` (submódulo)**:
    -   Factorías simples para crear `hyper::Request`s comunes (ej. `empty()`, `empty_with_uri(uri)`).
    -   Línea de Referencia: `tests/util/http.rs:174`

### `tests/util/config.rs`

Proporciona funciones para generar configuraciones `rxh::config::Server` para escenarios de prueba comunes.

-   **`proxy::single_backend(address: SocketAddr) -> Server`**: Configuración para un proxy simple a un único backend.
    -   Línea de Referencia: `tests/util/config.rs:9`
-   **`proxy::single_backend_with_uri(address: SocketAddr, uri: &str) -> Server`**: Igual, pero solo para un prefijo de URI específico.
    -   Línea de Referencia: `tests/util/config.rs:14`
-   **`proxy::multiple_weighted_backends(backends: Vec<Backend>) -> Server`**: Configuración para proxy con múltiples backends ponderados (usa WRR).
    -   Línea de Referencia: `tests/util/config.rs:23`
-   **`files::serve(root: &str) -> Server`**: Configuración para servir archivos estáticos desde `root`.
    -   Línea de Referencia: `tests/util/config.rs:47`

### `tests/util/service.rs`

-   **`RequestInterceptor` struct**:
    -   Un servicio `hyper` que reenvía las partes de la solicitud y el cuerpo recibido a través de un canal `mpsc::Sender`.
    -   Permite a una prueba inspeccionar lo que un backend simulado realmente recibió.
    -   Devuelve una respuesta configurable (por defecto "Hello world").
    -   Línea de Referencia: `tests/util/service.rs:14`
-   **`AsyncBody` trait alias**: `Body<Data: Send, Error: Sync + Send + std::error::Error> + Send + 'static;`
    -   Simplifica las restricciones de tipo genérico para cuerpos de solicitud/respuesta en las funciones de utilidad.
    -   Línea de Referencia: `tests/util/service.rs:42`
-   **`serve_connection<S, B>(stream: TcpStream, service: S)`**:
    -   Helper para ejecutar un servicio `hyper` sobre un `TcpStream`.
    -   Línea de Referencia: `tests/util/service.rs:46`

## Flujo Típico de una Prueba de Integración (ej. `reverse_proxy_client` en `tests/proxy.rs`)

```rust
#[tokio::test]
async fn reverse_proxy_client() {
    // 1. Iniciar un servidor backend simulado.
    //    Usa `spawn_backend_server` de `tests/util/http.rs`.
    //    El servicio que se le pasa define la respuesta que dará este backend.
    let (server_addr, _) = spawn_backend_server(service_fn(|_| async {
        Ok(Response::new(Full::<Bytes>::from("Hello world")))
    }));

    // 2. Iniciar el servidor RXH (proxy) con una configuración que apunte al backend.
    //    Usa `spawn_reverse_proxy` de `tests/util/http.rs`.
    //    La configuración se crea con `config::proxy::single_backend` de `tests/util/config.rs`.
    let (proxy_addr, _) = spawn_reverse_proxy(config::proxy::single_backend(server_addr));

    // 3. Asegurarse de que ambos servidores estén escuchando.
    //    Usa `ping_all` de `tests/util/tcp.rs`.
    ping_all(&[server_addr, proxy_addr]).await;

    // 4. Enviar una solicitud HTTP al proxy.
    //    Usa `send_http_request` de `tests/util/http.rs`.
    //    La solicitud se crea con `request::empty` de `tests/util/http.rs`.
    let (_, body) = send_http_request(proxy_addr, request::empty()).await;

    // 5. Verificar la respuesta.
    //    El cuerpo de la respuesta debe ser el que envió el backend simulado.
    assert_eq!(body, String::from("Hello world"));
}
```
-   Línea de Referencia de la prueba: `tests/proxy.rs:28`

## Estrategia de Pruebas

-   **Pruebas de Integración de Caja Negra/Gris**: La mayoría de las pruebas tratan a RXH como una caja negra (interactuando solo a través de HTTP) o gris (inspeccionando estados o controlando el apagado a través de las utilidades).
-   **Aislamiento**: El uso de puertos efímeros (`usable_socket`) y la ejecución de cada servidor (RXH y backends simulados) en tareas Tokio separadas ayuda a aislar las pruebas.
-   **Control del Ciclo de Vida**: Funciones como `spawn_reverse_proxy_with_controllers` permiten un control fino sobre el servidor RXH para probar escenarios como el apagado ordenado y los límites de conexión.
-   **Inspección**: `RequestInterceptor` y la capacidad de definir servicios backend personalizados permiten inspeccionar cómo RXH modifica o reenvía las solicitudes.
-   **Cobertura de Escenarios**: Las pruebas intentan cubrir una variedad de configuraciones y casos de uso, incluyendo rutas felices y casos de error.

## Conclusión

El conjunto de pruebas de RXH es bastante completo y demuestra buenas prácticas para probar software de red asíncrono. Las utilidades en `tests/util/` son fundamentales para reducir el boilerplate y hacer que las pruebas sean más legibles y mantenibles. Estas pruebas proporcionan una red de seguridad importante para el desarrollo y refactorización del proyecto.
