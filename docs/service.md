# Documentación del Módulo de Servicio (`src/service/`)

## Visión General

El módulo `service` es el corazón del manejo de solicitudes HTTP en RXH. Define la lógica que se ejecuta para cada conexión entrante una vez que ha sido aceptada por una tarea `Server` (`src/task/server.rs`). La pieza central es la estructura `Rxh`, que implementa el trait `hyper::service::Service`. Esta implementación determina, basándose en la configuración del servidor y el URI de la solicitud, si se debe reenviar la solicitud a un backend (proxy) o servir un archivo estático.

## Estructura del Módulo

-   **`src/service/mod.rs`**:
    -   Define la estructura `Rxh` y su implementación de `hyper::service::Service`.
    -   Contiene la lógica principal de enrutamiento basada en patrones de URI.
    -   Declara los submódulos `files` y `proxy`.
    -   Línea de Referencia: `src/service/mod.rs:1`

-   **`src/service/files.rs`**:
    -   Contiene la lógica para servir archivos estáticos desde el sistema de archivos.
    -   Línea de Referencia: `src/service/files.rs:1`

-   **`src/service/proxy.rs`**:
    -   Contiene la lógica para reenviar solicitudes a servidores backend (upstream).
    -   Maneja las actualizaciones de conexión (ej., WebSockets) mediante la creación de túneles TCP.
    -   Línea de Referencia: `src/service/proxy.rs:1`

## Componentes Detallados

### 1. `Rxh` struct (`src/service/mod.rs`)

-   **Definición**:
    ```rust
    pub(crate) struct Rxh {
        config: &'static config::Server, // Configuración específica del servidor que maneja esta conexión
        client_addr: SocketAddr,        // Dirección del cliente conectado
        server_addr: SocketAddr,        // Dirección del listener del proxy que aceptó esta conexión
    }
    ```
    -   **Línea de Referencia**: `src/service/mod.rs:25`

-   **Propósito**: Implementa `hyper::service::Service` para manejar una solicitud HTTP individual. Cada nueva conexión aceptada por `task::Server` obtiene su propia instancia (o más bien, `hyper` llama a un constructor que la crea) de un servicio que se comporta como `Rxh`.
-   **Campos**:
    -   `config: &'static config::Server`: Una referencia estática a la configuración del `config::Server` (definido en `src/config/mod.rs`) que está manejando esta conexión. El lifetime `'static` se logra "leakeando" la configuración en `task::server.rs` (línea `src/task/server.rs:204`) para la duración de las tareas de manejo de solicitudes.
    -   `client_addr: SocketAddr`: La dirección IP y puerto del cliente que realizó la solicitud.
    -   `server_addr: SocketAddr`: La dirección IP y puerto del listener del servidor RXH que aceptó esta conexión.

-   **`new(config: &'static config::Server, client_addr: SocketAddr, server_addr: SocketAddr) -> Self`**:
    -   **Propósito**: Constructor para `Rxh`.
    -   **Llamado desde**: `src/task/server.rs` (específicamente en `Listener::listen`, línea `src/task/server.rs:301`), cuando se configura el `serve_connection` de Hyper.
    -   **Línea de Referencia**: `src/service/mod.rs:35`

-   **`impl Service<Request<Incoming>> for Rxh`**:
    -   **Línea de Referencia**: `src/service/mod.rs:46`
    -   **`type Response = BoxBodyResponse;`**: El tipo de respuesta que este servicio producirá (definido en `src/http/response.rs`).
    -   **`type Error = hyper::Error;`**: El tipo de error.
    -   **`type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;`**: La respuesta se devuelve de forma asíncrona.
    -   **`call(&mut self, request: Request<Incoming>) -> Self::Future`**:
        -   **Propósito**: Este es el método principal llamado por `hyper` para cada solicitud HTTP.
        -   **Lógica Principal**:
            1.  Captura los campos `client_addr`, `server_addr`, y `config` de `self` para moverlos al futuro asíncrono. (Línea `src/service/mod.rs:55`)
            2.  Registra el tiempo de inicio para logging. (Línea `src/service/mod.rs:59`)
            3.  Obtiene el URI y el método de la `request` para logging y enrutamiento. (Líneas `src/service/mod.rs:62-63`)
            4.  **Enrutamiento por Patrón**: Itera sobre `config.patterns` (la lista de `config::Pattern` para este servidor). (Línea `src/service/mod.rs:65`)
                -   Busca el primer patrón cuyo `pattern.uri` sea un prefijo del `request.uri().to_string()`.
            5.  **Si no se encuentra patrón**: Devuelve una respuesta 404 Not Found usando `LocalResponse::not_found()`. (Línea `src/service/mod.rs:70`)
            6.  **Si se encuentra un patrón**: Ejecuta la `action` asociada con el patrón (`pattern.action`):
                -   **`Action::Forward(Forward { scheduler, .. })`**: (Línea `src/service/mod.rs:74`)
                    -   Obtiene el `proxy_id` del `config.name` del servidor.
                    -   Crea una `ProxyRequest` (`src/http/request.rs`) con la solicitud original, `client_addr`, `server_addr`, y el `proxy_id`.
                    -   Llama a `scheduler.next_server()` para obtener la `SocketAddr` del backend.
                    -   Llama a `proxy::forward(request, backend_addr).await` (del submódulo `src/service/proxy.rs`) para realizar el reenvío.
                -   **`Action::Serve(directory)`**: (Línea `src/service/mod.rs:80`)
                    -   Extrae la parte de la ruta del URI de la solicitud. Si comienza con `/`, se elimina el primer carácter para obtener una ruta relativa.
                    -   Llama a `files::transfer(path, directory).await` (del submódulo `src/service/files.rs`) para servir el archivo.
            7.  **Logging**: Si la operación fue exitosa (se obtuvo una `Ok(response)`), registra la solicitud (cliente, servidor, método, URI, status HTTP, tiempo transcurrido). (Línea `src/service/mod.rs:88`)
            8.  Devuelve la `response`.
        -   **Línea de Referencia Método `call`**: `src/service/mod.rs:50`

### 2. `src/service/files.rs` - Servidor de Archivos Estáticos

-   **`transfer(path: &str, root: &str) -> Result<BoxBodyResponse, hyper::Error>`**:
    -   **Propósito**: Sirve un archivo estático.
    -   **Parámetros**:
        -   `path: &str`: La ruta relativa al archivo solicitada (ej. "index.html", "css/style.css"). Se asume que no comienza con "/".
        -   `root: &str`: La ruta al directorio raíz desde donde se deben servir los archivos (configurado en `Action::Serve`).
    -   **Lógica**:
        1.  **Canonicalización y Validación de Rutas**:
            -   Canonicaliza `root` para obtener una ruta absoluta y resolver symlinks. Si falla (ej. el directorio no existe), devuelve 404. (Línea `src/service/files.rs:10`)
            -   Une `root` canónico con `path` y canonicaliza el resultado para obtener la ruta completa al archivo solicitado. Si falla (ej. el archivo no existe), devuelve 404. (Línea `src/service/files.rs:14`)
            -   **Medida de Seguridad (Path Traversal)**: Verifica que la ruta canónica del archivo (`file`) realmente comience con la ruta canónica del directorio raíz (`directory`). Esto previene ataques de path traversal (ej. `../../../etc/passwd`). También verifica que `file` sea un archivo. Si no, devuelve 404. (Línea `src/service/files.rs:18`)
        2.  **Determinación del `Content-Type`**:
            -   Obtiene la extensión del archivo.
            -   Hace un `match` simple para determinar el `Content-Type` (html, css, js, png, jpeg, o text/plain por defecto). (Línea `src/service/files.rs:22`)
        3.  **Lectura y Envío del Archivo**:
            -   **TODO**: El comentario (línea `src/service/files.rs:31`) indica planes futuros para manejar archivos grandes (streaming, gzip, `Transfer-Encoding: chunked`).
            -   Actualmente, lee el archivo completo en memoria usando `tokio::fs::read(file).await`. (Línea `src/service/files.rs:33`)
            -   Si la lectura es exitosa, crea una respuesta 200 OK usando `LocalResponse::builder()`, establece el `Content-Type`, y usa `crate::http::body::full(content)` para el cuerpo.
            -   Si la lectura falla, devuelve 404.
    -   **Línea de Referencia Función**: `src/service/files.rs:8`

### 3. `src/service/proxy.rs` - Lógica de Proxy

-   **`forward(mut request: ProxyRequest<Incoming>, to: SocketAddr) -> Result<BoxBodyResponse, hyper::Error>`**:
    -   **Propósito**: Reenvía la `request` al servidor backend en la dirección `to`.
    -   **Parámetros**:
        -   `request: ProxyRequest<Incoming>`: La solicitud entrante envuelta, que contiene metadatos del proxy. Es `mut` porque se le pueden quitar extensiones.
        -   `to: SocketAddr`: La dirección del servidor backend al que se reenviará la solicitud (determinada por el `Scheduler`).
    -   **Lógica**:
        1.  **Conexión al Backend**: Intenta conectar al backend usando `TcpStream::connect(to).await`. Si falla, devuelve `LocalResponse::bad_gateway()`. (Línea `src/service/proxy.rs:17`)
        2.  **Handshake HTTP con el Backend**:
            -   Establece una conexión HTTP/1 con el backend usando `hyper::client::conn::http1::Builder`.
                -   `preserve_header_case(true)` y `title_case_headers(true)`: Opciones para mantener la capitalización original de los headers.
            -   `handshake(stream).await?` devuelve una tupla `(sender, connection)`.
                -   `sender: SendRequest<B>`: Para enviar la solicitud al backend.
                -   `connection: Connection<TokioIo<Upgraded>, B>`: Un futuro que maneja la conexión; debe ser ejecutado en una tarea separada (`tokio::task::spawn`).
            -   (Líneas `src/service/proxy.rs:21-26`)
        3.  **Manejo de Upgrade (ej. WebSockets)**:
            -   Verifica si la `request` original del cliente contenía un header `Upgrade` (`request.headers().contains_key(header::UPGRADE)`). (Línea `src/service/proxy.rs:34`)
            -   Si es así, extrae el objeto `OnUpgrade` de las `request.extensions_mut()`. Este objeto es proporcionado por Hyper cuando detecta una solicitud de upgrade. Se guarda para usarlo más tarde si el backend también acepta el upgrade.
        4.  **Envío de la Solicitud al Backend**:
            -   Convierte `ProxyRequest` a `hyper::Request` con el header `Forwarded` añadido: `request.into_forwarded()`. (`src/service/proxy.rs:45`)
            -   Envía esta solicitud al backend: `sender.send_request(...).await?`.
        5.  **Manejo de la Respuesta del Backend**:
            -   La `response` del backend se recibe.
            -   **Si la respuesta es `101 SWITCHING_PROTOCOLS`**: (Línea `src/service/proxy.rs:47`)
                -   Indica que el backend aceptó la solicitud de upgrade.
                -   Se debe haber guardado un `client_upgrade` (del paso 3). Si no, es un error (el backend aceptó un upgrade que el cliente no pidió), y se devuelve `LocalResponse::bad_gateway()`.
                -   Se extrae el `server_upgrade` (objeto `OnUpgrade`) de las extensiones de la respuesta del backend.
                -   Se llama a `tokio::task::spawn(tunnel(client_upgrade, server_upgrade))` para establecer el túnel bidireccional. (Línea `src/service/proxy.rs:51`)
            -   **Para todas las respuestas (incluyendo 101 después de iniciar el túnel)**:
                -   La respuesta del backend se envuelve: `ProxyResponse::new(response.map(|body| body.boxed()))`.
                -   Se llama a `.into_forwarded()` para añadir el header `Server: rxh/...`.
                -   Se devuelve esta respuesta final al cliente. (Línea `src/service/proxy.rs:59`)
    -   **Línea de Referencia Función**: `src/service/proxy.rs:13`

-   **`tunnel(client: OnUpgrade, server: OnUpgrade)` async fn**:
    -   **Propósito**: Establece un túnel TCP bidireccional entre el cliente y el servidor backend después de una negociación de upgrade exitosa (ej. para WebSockets).
    -   **Parámetros**:
        -   `client: OnUpgrade`: El futuro de upgrade para la conexión con el cliente.
        -   `server: OnUpgrade`: El futuro de upgrade para la conexión con el backend.
    -   **Lógica**:
        1.  Espera a que ambas conexiones (cliente y backend) se actualicen: `tokio::try_join!(client, server).unwrap()`. Esto devuelve las dos mitades de E/S actualizadas (`Upgraded`). (Línea `src/service/proxy.rs:70`)
        2.  Copia datos bidireccionalmente entre `upgraded_client` y `upgraded_server` usando `tokio::io::copy_bidirectional`. (Línea `src/service/proxy.rs:72`)
        3.  Registra el número de bytes transferidos o cualquier error.
    -   **Contexto de Ejecución**: Esta función se ejecuta en una nueva tarea `tokio` porque los futuros `OnUpgrade` no se resuelven hasta que la respuesta `101` se haya enviado completamente al cliente.
    -   **Línea de Referencia Función**: `src/service/proxy.rs:68`

## Flujo de Servicio

1.  Una tarea `Server` (`src/task/server.rs`) acepta una conexión TCP.
2.  Para esa conexión, `hyper` utiliza el servicio `Rxh` (configurado con la `config::Server` apropiada, `client_addr`, `server_addr`).
3.  El método `Rxh::call` se invoca para cada solicitud en esa conexión.
4.  `Rxh::call` determina el `config::Pattern` coincidente.
5.  Según la `config::Action` del patrón:
    -   **`Forward`**:
        -   El `Scheduler` selecciona un backend.
        -   `proxy::forward` maneja la conexión al backend, la modificación de la solicitud (`ProxyRequest::into_forwarded`), el envío, el manejo de upgrades (`tunnel`), la modificación de la respuesta (`ProxyResponse::into_forwarded`), y la devuelve.
    -   **`Serve`**:
        -   `files::transfer` maneja la validación de la ruta, la lectura del archivo, la determinación del `Content-Type`, y la construcción de la respuesta (usando `LocalResponse`).
6.  La `BoxBodyResponse` resultante se devuelve a `hyper`, que la envía al cliente.
7.  Se registra la transacción.

## Puntos Clave

-   **Implementación de `hyper::service::Service`**: `Rxh` es el punto de entrada para `hyper` en la lógica de aplicación de RXH.
-   **Enrutamiento Centralizado**: `Rxh::call` centraliza la decisión de enrutamiento basada en los patrones de URI configurados.
-   **Separación de Lógica de Acción**: Las acciones específicas (`Serve`, `Forward`) están delegadas a submódulos (`files.rs`, `proxy.rs`), manteniendo `Rxh::call` enfocado en el enrutamiento.
-   **Manejo de Upgrade Transparente**: La lógica de upgrade en `proxy::forward` y `tunnel` permite que protocolos como WebSockets funcionen a través del proxy.
-   **Seguridad en Servidor de Archivos**: La canonicalización y validación de rutas en `files::transfer` es crucial para prevenir vulnerabilidades de path traversal.
-   **Uso de Abstracciones HTTP**: Se apoya en `ProxyRequest`, `ProxyResponse`, `LocalResponse` y `http::body` del módulo `src/http/` para simplificar el código y encapsular lógica común.

El módulo `service` integra eficazmente la configuración, la planificación de carga, y las operaciones HTTP para cumplir con el rol principal de RXH como proxy inverso y servidor de archivos.
