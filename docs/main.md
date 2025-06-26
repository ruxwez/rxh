# Documentación Principal del Proyecto RXH

## Visión General

RXH es un servidor proxy inverso, balanceador de carga y servidor de archivos estáticos de alto rendimiento construido en Rust. Utiliza `tokio` para operaciones asíncronas y `hyper` para el manejo de HTTP, lo que le permite manejar una gran cantidad de conexiones concurrentes de manera eficiente.

El proyecto está diseñado para ser configurable a través de un archivo TOML (por defecto `rxh.toml`), permitiendo a los usuarios definir múltiples servidores virtuales, cada uno con sus propias reglas de enrutamiento y acciones.

## Estructura del Proyecto

El código fuente se organiza en varios módulos, cada uno responsable de una parte específica de la funcionalidad:

- **`src/main.rs`**: Es el punto de entrada de la aplicación. Se encarga de:
    - Leer el archivo de configuración (p.ej., `rxh.toml`).
    - Inicializar el `Master` task.
    - Configurar la señal de apagado (Ctrl+C).
    - Ejecutar el `Master` task.

- **`src/lib.rs`**: Es la raíz de la biblioteca principal del proyecto. Define:
    - El módulo `Error` principal para el manejo de errores a alto nivel.
    - Constantes globales como `VERSION`.
    - Re-exporta los componentes principales como `Master` y `Server`.
    - Declara los módulos principales del proyecto: `config`, `http`, `sched`, `service`, `sync`, `task`.

- **`src/config/`**: Módulo responsable de cargar, parsear y validar la configuración del servidor desde un archivo TOML.
    - `mod.rs`: Define las estructuras principales que mapean la configuración (ej. `Config`, `Server`, `Pattern`, `Action`).
    - `deser.rs`: Implementa lógica de deserialización personalizada para `serde` para permitir formatos de configuración más flexibles y amigables.

- **`src/http/`**: Contiene abstracciones y utilidades relacionadas con HTTP.
    - `mod.rs`: Declara los submódulos.
    - `body.rs`: Funciones para crear cuerpos de respuesta HTTP comunes (ej. `full`, `empty`).
    - `request.rs`: Define `ProxyRequest`, que envuelve una `hyper::Request` y añade información del cliente y del servidor proxy. Implementa la lógica para generar el header `Forwarded` (RFC 7239).
    - `response.rs`: Define `ProxyResponse` y `LocalResponse` para manejar respuestas generadas por el proxy o por servidores upstream. Incluye la adición del header `Server: rxh/...`.

- **`src/sched/`**: Implementaciones de algoritmos de balanceo de carga.
    - `mod.rs`: Define el trait `Scheduler` y una factoría `make` para crear instancias de schedulers.
    - `wrr.rs`: Implementación del algoritmo Weighted Round Robin (WRR).

- **`src/service/`**: Lógica para manejar las solicitudes HTTP entrantes según la configuración.
    - `mod.rs`: Define la estructura `Rxh` que implementa `hyper::service::Service`. Esta es la pieza central que decide cómo manejar una solicitud basándose en los patrones de URI.
    - `files.rs`: Lógica para servir archivos estáticos desde el sistema de archivos.
    - `proxy.rs`: Lógica para reenviar (proxy) solicitudes a servidores upstream, incluyendo el manejo de actualizaciones de conexión (ej. WebSockets).

- **`src/sync/`**: Primitivas de sincronización personalizadas.
    - `mod.rs`: Declara los submódulos.
    - `notify.rs`: Implementa un sistema de `Notifier` y `Subscription` para el apagado ordenado (graceful shutdown) de tareas.
    - `ring.rs`: Implementa una estructura de datos `Ring` (array circular) para uso en schedulers como WRR, permitiendo un acceso atómico y cíclico a una lista de elementos.

- **`src/task/`**: Define la arquitectura de tareas del servidor (master-server).
    - `mod.rs`: Declara los submódulos.
    - `master.rs`: Define el `Master` task, responsable de inicializar, ejecutar y apagar ordenadamente todas las instancias de `Server` definidas en la configuración. Maneja la creación de "réplicas" de servidores si una configuración de servidor escucha en múltiples direcciones.
    - `server.rs`: Define la tarea `Server`, que representa una instancia de servidor individual escuchando en una dirección y puerto específicos. Es responsable de aceptar conexiones, gestionar un límite de conexiones concurrentes, y coordinar el apagado ordenado de las tareas que manejan las solicitudes.

- **`tests/`**: Contiene pruebas de integración.
    - `proxy.rs`: Pruebas específicas para la funcionalidad de proxy, balanceo de carga, servicio de archivos estáticos, y apagado ordenado.
    - `util/`: Módulos de utilidad para las pruebas, facilitando la creación de servidores backend simulados, clientes HTTP, y configuraciones de prueba.

## Flujo de Ejecución Básico

1.  **Inicio (`main.rs`)**:
    *   Se lee el archivo `rxh.toml`.
    *   La configuración es parseada en las estructuras definidas en `src/config/`.
    *   Se inicializa una instancia de `Master` (`src/task/master.rs`) con la configuración parseada.
    *   El `Master` configura una señal de apagado (generalmente Ctrl+C).
    *   El `Master` se ejecuta.

2.  **Inicialización del `Master`**:
    *   Por cada `[[server]]` en la configuración:
        *   Si un servidor tiene múltiples direcciones en `listen`, el `Master` crea una "réplica" (una instancia de `Server` task) para cada dirección.
        *   Cada instancia de `Server` (`src/task/server.rs`) es inicializada. Esto incluye:
            *   Vincular un socket TCP a la dirección y puerto especificados.
            *   Configurar un `Notifier` para el apagado ordenado.
            *   Establecer un semáforo para limitar las conexiones concurrentes.

3.  **Ejecución de `Server` tasks**:
    *   Cada `Server` comienza a escuchar conexiones en su socket.
    *   Cuando se acepta una nueva conexión:
        *   Se verifica si hay permisos disponibles en el semáforo de conexiones. Si no, la conexión espera o el servidor indica `MaxConnectionsReached`.
        *   Se obtiene un permiso.
        *   Se crea una instancia del servicio `Rxh` (`src/service/mod.rs`) con la configuración específica de ese servidor y la dirección del cliente.
        *   Se genera una nueva tarea `tokio` para manejar la conexión usando `hyper::server::conn::http1::Builder::serve_connection`. Esta tarea utiliza el servicio `Rxh`.

4.  **Manejo de Solicitudes (`Rxh` service)**:
    *   El servicio `Rxh` recibe la `hyper::Request`.
    *   Busca en los `patterns` configurados para ese servidor cuál coincide con el URI de la solicitud.
    *   Si se encuentra un patrón:
        *   **Acción `Forward`**:
            *   Se utiliza el `scheduler` (ej. `WRR` de `src/sched/wrr.rs`) para seleccionar un servidor backend.
            *   Se crea una `ProxyRequest` (`src/http/request.rs`), se añade el header `Forwarded`.
            *   La solicitud se reenvía al backend seleccionado usando `src/service/proxy.rs`.
            *   Si la conexión es una actualización (ej. WebSocket), se establece un túnel TCP bidireccional.
            *   La respuesta del backend se envuelve en una `ProxyResponse` (`src/http/response.rs`), se añade el header `Server`, y se devuelve al cliente.
        *   **Acción `Serve`**:
            *   Se utiliza `src/service/files.rs` para intentar servir un archivo estático del directorio raíz especificado.
            *   Se determina el `Content-Type` apropiado.
            *   Se devuelve el contenido del archivo o una respuesta 404.
    *   Si no se encuentra un patrón, se devuelve una respuesta 404 (`LocalResponse::not_found()`).

5.  **Apagado Ordenado**:
    *   Cuando el `Master` recibe la señal de apagado (ej. Ctrl+C):
        *   Envía una notificación de apagado a todos los `Server` tasks que gestiona.
    *   Cada `Server` task:
        *   Deja de aceptar nuevas conexiones.
        *   Envía una notificación de apagado (usando su `Notifier` de `src/sync/notify.rs`) a todas las tareas activas que manejan solicitudes.
        *   Espera a que todas estas tareas reconozcan la notificación (indicando que han terminado su trabajo actual).
        *   Una vez que todas las tareas de manejo de solicitudes han terminado, el `Server` task finaliza.
    *   El `Master` task espera a que todos los `Server` tasks finalicen.
    *   La aplicación termina.

## Puntos Clave para Recrear el Proyecto

-   **Configuración Flexible**: La deserialización personalizada en `src/config/deser.rs` es crucial para la usabilidad del archivo TOML. Permite múltiples formas de definir escuchas, backends y acciones.
-   **Manejo Asíncrono**: El uso extensivo de `tokio` y `async/await` es fundamental. Comprender cómo se gestionan las tareas y la concurrencia es clave.
-   **Servicio Hyper**: La implementación de `hyper::service::Service` en `src/service/mod.rs` es el núcleo del enrutamiento y manejo de solicitudes.
-   **Apagado Ordenado**: El sistema `Notifier`/`Subscription` en `src/sync/notify.rs` y su integración en `Server` y `Master` son vitales para no perder solicitudes en curso durante el apagado.
-   **Abstracciones HTTP**: Las envolturas `ProxyRequest` y `ProxyResponse` simplifican el manejo de metadatos específicos del proxy.
-   **Balanceo de Carga**: El trait `Scheduler` y sus implementaciones permiten extender fácilmente los algoritmos de balanceo. `Ring` es una optimización para schedulers basados en ciclos.
-   **Modelo Master-Server**: La separación de responsabilidades entre `Master` (gestión global) y `Server` (manejo de un listener específico) organiza la complejidad.

Este documento proporciona una visión general. Los documentos específicos de cada módulo (`config.md`, `http.md`, etc.) entrarán en más detalles sobre cada componente.
---
Recuerda que este es un resumen. Para una comprensión completa, es necesario revisar el código fuente referenciado.
