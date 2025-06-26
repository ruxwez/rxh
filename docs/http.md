# Documentación del Módulo HTTP (`src/http/`)

## Visión General

El módulo `http` en RXH proporciona abstracciones y utilidades personalizadas sobre la biblioteca `hyper` para manejar operaciones HTTP específicas de un proxy. Se centra en enriquecer las solicitudes y respuestas con información relevante para el proxying y en proporcionar helpers para construir cuerpos de respuesta comunes.

## Estructura del Módulo

-   **`src/http/mod.rs`**:
    -   Declara los submódulos `body`, `request`, y `response`.
    -   Línea de Referencia: `src/http/mod.rs:3`

-   **`src/http/body.rs`**:
    -   Utilidades para crear cuerpos de solicitud y respuesta HTTP comunes.
    -   Línea de Referencia: `src/http/body.rs:1`

-   **`src/http/request.rs`**:
    -   Define `ProxyRequest`, una estructura que envuelve `hyper::Request` y añade información contextual necesaria para el proxying, como la dirección IP del cliente y la lógica para el header `Forwarded`.
    -   Línea de Referencia: `src/http/request.rs:1`

-   **`src/http/response.rs`**:
    -   Define `ProxyResponse` para envolver respuestas de servidores upstream y `LocalResponse` para respuestas generadas por el propio RXH. También maneja la adición del header `Server`.
    -   Línea de Referencia: `src/http/response.rs:1`

## Componentes Detallados

### 1. `src/http/body.rs`

Este archivo proporciona funciones convenientes para crear tipos de cuerpos de `hyper` comunes.

-   **`full<T: Into<Bytes>>(chunk: T) -> BoxBody<Bytes, hyper::Error>`**:
    -   **Propósito**: Crea un cuerpo HTTP que consiste en un único trozo (chunk) de datos. `BoxBody` es un tipo de cuerpo dinámico de `http_body_util`.
    -   **Uso**: Útil para respuestas pequeñas cuyo contenido completo se conoce de antemano.
    -   **Ejemplo**: `body::full("Hola Mundo")`
    -   **Línea de Referencia**: `src/http/body.rs:5`

-   **`empty() -> BoxBody<Bytes, hyper::Error>`**:
    -   **Propósito**: Crea un cuerpo HTTP vacío.
    -   **Uso**: Para solicitudes o respuestas que no tienen cuerpo (ej. GET, o respuestas 204 No Content).
    -   **Línea de Referencia**: `src/http/body.rs:12`

### 2. `src/http/request.rs` - `ProxyRequest<T>`

Esta estructura es fundamental para cómo RXH maneja las solicitudes entrantes que necesitan ser reenviadas.

-   **Definición (parcial)**:
    ```rust
    pub(crate) struct ProxyRequest<T> {
        request: Request<T>, // La solicitud original de Hyper
        client_addr: SocketAddr, // Dirección del cliente que hizo la solicitud al proxy
        server_addr: SocketAddr, // Dirección local del proxy que recibió la solicitud
        proxy_id: Option<String>, // ID opcional del proxy para el header "Forwarded"
    }
    ```
    -   **Línea de Referencia**: `src/http/request.rs:12`

-   **`new(request: Request<T>, client_addr: SocketAddr, server_addr: SocketAddr, proxy_id: Option<String>) -> Self`**:
    -   **Propósito**: Constructor para `ProxyRequest`.
    -   **Parámetros**:
        -   `request`: La `hyper::Request` original.
        -   `client_addr`: La `SocketAddr` del cliente conectado al proxy.
        -   `server_addr`: La `SocketAddr` del listener del proxy que aceptó esta conexión.
        -   `proxy_id`: Un nombre opcional para el proxy (configurado por el usuario, ej. `name` en `config::Server`), usado en el parámetro `by` del header `Forwarded`. Si es `None`, se usa `server_addr`.
    -   **Línea de Referencia**: `src/http/request.rs:23`

-   **`headers(&self) -> &HeaderMap`**:
    -   **Propósito**: Devuelve una referencia a los headers de la solicitud original.
    -   **Línea de Referencia**: `src/http/request.rs:37`

-   **`extensions_mut(&mut self) -> &mut Extensions`**:
    -   **Propósito**: Devuelve una referencia mutable a las extensiones de la solicitud. Esto es crucial para manejar actualizaciones de conexión (como WebSockets), ya que `hyper::upgrade::OnUpgrade` se almacena en las extensiones.
    -   **Línea de Referencia**: `src/http/request.rs:41`

-   **`into_forwarded(mut self) -> Request<T>`**:
    -   **Propósito**: Consume `ProxyRequest` y devuelve la `hyper::Request` interna, pero modificada para incluir (o actualizar) el header HTTP `Forwarded` según RFC 7239.
    -   **Lógica del Header `Forwarded` (RFC 7239)**:
        -   Este header permite a los proxies revelar información perdida en el proceso de proxying, como la IP del cliente original.
        -   Formato: `Forwarded: for={};by={};host={};proto={}`
            -   `for`: Identifica al cliente que inició la solicitud (aquí, `self.client_addr`).
            -   `by`: Identifica la interfaz del proxy que recibió la solicitud (aquí, `self.proxy_id` o `self.server_addr`).
            -   `host`: El valor original del header `Host` recibido por el proxy (aquí, tomado del `self.request.headers().get(header::HOST)` o, como fallback, `self.server_addr`).
            -   `proto`: Protocolo (no implementado explícitamente en este código, se podría añadir).
        -   Si ya existe un header `Forwarded` (de un proxy anterior), la nueva información se añade como una nueva entrada separada por coma.
        -   **Ejemplo**: `Forwarded: for=192.0.2.43, for=198.51.100.17;by=203.0.113.60;host=example.com`
    -   **Implementación**:
        1.  Determina el valor de `host` a partir del header `Host` de la solicitud entrante o usa `self.server_addr` si no está presente. (Línea `src/http/request.rs:88`)
        2.  Determina el valor de `by` usando `self.proxy_id` o `self.server_addr`. (Línea `src/http/request.rs:96`)
        3.  Construye la cadena `forwarded_value` con `for`, `by`, y `host`. (Línea `src/http/request.rs:99`)
        4.  Si ya existe un header `Forwarded` en la solicitud, concatena el valor existente con el nuevo. (Línea `src/http/request.rs:101`)
        5.  Inserta/reemplaza el header `Forwarded` en la solicitud. (Línea `src/http/request.rs:107`)
    -   **Línea de Referencia Método**: `src/http/request.rs:85`

### 3. `src/http/response.rs`

Este archivo maneja la creación y modificación de respuestas HTTP.

-   **`BoxBodyResponse = Response<BoxBody<Bytes, hyper::Error>>`**:
    -   **Propósito**: Un alias de tipo para un `hyper::Response` que usa un cuerpo dinámico (`BoxBody`). Simplifica las firmas de función.
    -   **Línea de Referencia**: `src/http/response.rs:10`

-   **`ProxyResponse<T>`**:
    -   **Definición (parcial)**:
        ```rust
        pub(crate) struct ProxyResponse<T> {
            response: Response<T>, // Respuesta original del servidor upstream
        }
        ```
        -   **Línea de Referencia**: `src/http/response.rs:15`
    -   **`new(response: Response<T>) -> Self`**:
        -   **Propósito**: Constructor, envuelve una respuesta de un servidor upstream.
        -   **Línea de Referencia**: `src/http/response.rs:21`
    -   **`into_forwarded(mut self) -> Response<T>`**:
        -   **Propósito**: Consume `ProxyResponse` y devuelve la `hyper::Response` interna, modificada para incluir el header `Server: rxh/<version>`.
        -   **Implementación**: Añade o reemplaza el header `Server` con el valor de `rxh_server_header()`.
        -   **Línea de Referencia**: `src/http/response.rs:27`

-   **`LocalResponse`**:
    -   **Propósito**: Namespace para construir respuestas que se originan directamente en RXH (ej. 404 Not Found, 502 Bad Gateway).
    -   **Línea de Referencia**: `src/http/response.rs:39`
    -   **`builder() -> http::response::Builder`**:
        -   **Propósito**: Devuelve un `http::response::Builder` pre-configurado con el header `Server: rxh/<version>`.
        -   **Línea de Referencia**: `src/http/response.rs:43`
    -   **`not_found() -> BoxBodyResponse`**:
        -   **Propósito**: Crea una respuesta HTTP 404 Not Found estándar.
        -   **Implementación**: Usa `LocalResponse::builder()`, establece el status 404, `Content-Type: text/plain`, y un cuerpo `body::full("HTTP 404 NOT FOUND")`.
        -   **Línea de Referencia**: `src/http/response.rs:49`
    -   **`bad_gateway() -> BoxBodyResponse`**:
        -   **Propósito**: Crea una respuesta HTTP 502 Bad Gateway estándar.
        -   **Implementación**: Similar a `not_found()`, pero con status 502 y cuerpo "HTTP 502 BAD GATEWAY".
        -   **Línea de Referencia**: `src/http/response.rs:58`

-   **`rxh_server_header() -> String`**:
    -   **Propósito**: Función helper inline que genera la cadena para el header `Server`.
    -   **Implementación**: Formatea la cadena como `rxh/<VERSION>`, donde `VERSION` es la constante de la versión del crate (definida en `src/lib.rs`).
    -   **Línea de Referencia**: `src/http/response.rs:81`
    -   **TODO**: Menciona que este header podría ser configurable en el futuro.

## Flujo de Uso Típico

1.  **Solicitud Entrante al Proxy (`src/service/mod.rs`)**:
    -   Una `hyper::Request<Incoming>` llega.
    -   Se crea una `ProxyRequest` usando `ProxyRequest::new(...)`, pasando la solicitud original, la IP del cliente, la IP del listener del proxy, y el nombre configurado del proxy.

2.  **Decisión de Reenvío (`src/service/mod.rs` -> `src/service/proxy.rs`)**:
    -   Si la acción es `Forward`:
        -   Se llama a `proxy::forward(...)` con la `ProxyRequest`.
        -   Dentro de `proxy::forward`, antes de enviar la solicitud al backend:
            -   Se extrae `OnUpgrade` de las extensiones de `ProxyRequest` si está presente (para WebSockets, etc.). (`src/service/proxy.rs:34`)
            -   Se llama a `request.into_forwarded()` para obtener la `hyper::Request` con el header `Forwarded` correctamente establecido. (`src/service/proxy.rs:45`)
            -   Esta solicitud modificada se envía al servidor backend.

3.  **Respuesta del Backend (`src/service/proxy.rs`)**:
    -   Se recibe una `hyper::Response` del servidor backend.
    -   Si el status es `101 SWITCHING_PROTOCOLS` y había una solicitud de upgrade, se maneja el upgrade (se llama a `tunnel`). (`src/service/proxy.rs:47`)
    -   La respuesta del backend se envuelve: `ProxyResponse::new(response.map(|body| body.boxed()))`. (`src/service/proxy.rs:59`)
    -   Se llama a `.into_forwarded()` sobre esta `ProxyResponse` para añadir el header `Server: rxh/...`. (`src/service/proxy.rs:59`)
    -   Esta respuesta final se devuelve al cliente original.

4.  **Respuesta Local (`src/service/mod.rs` o `src/service/files.rs`)**:
    -   Si el proxy genera la respuesta directamente (ej. 404, 502, o un archivo estático):
        -   Se usa `LocalResponse::not_found()`, `LocalResponse::bad_gateway()`, o `LocalResponse::builder()` para construir la respuesta.
        -   Estas funciones ya incluyen el header `Server: rxh/...`.
        -   El cuerpo se crea usando `crate::http::body::full(...)`.

## Puntos Clave

-   **Contextualización de Solicitudes**: `ProxyRequest` es clave para añadir información vital (IP del cliente, ID del proxy) que no está directamente en `hyper::Request` y para implementar correctamente RFC 7239 (`Forwarded` header).
-   **Modificación de Respuestas**: `ProxyResponse` asegura que las respuestas de los backends sean marcadas por RXH (vía header `Server`). `LocalResponse` hace lo mismo para respuestas generadas localmente.
-   **Manejo de Upgrades**: La capacidad de acceder a `extensions_mut()` en `ProxyRequest` es esencial para que el módulo `service::proxy` pueda manejar upgrades de conexión.
-   **Cuerpos Simplificados**: `http::body` ofrece una API más sencilla para cuerpos comunes que usar directamente `Full` o `Empty` de `http_body_util` y `BoxBody` en todo lugar.

Este módulo abstrae detalles de `hyper` y añade la lógica específica que un proxy HTTP necesita para interactuar correctamente con clientes y servidores upstream, manteniendo la conformidad con RFCs relevantes y proveyendo información útil.
