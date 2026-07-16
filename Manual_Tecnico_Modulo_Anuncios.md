# Manual Técnico: Módulo de Anuncios y Sistema de Notificaciones

## 1. Introducción
Este documento detalla la arquitectura, flujos de trabajo y componentes involucrados en el ecosistema de notificaciones del Sistema de Gestión de Activos Institucionales (Web). Este ecosistema abarca tres áreas principales:
1. **Sistema de Toasts (Alertas efímeras de UI)**.
2. **Campana de Notificaciones (Feedback persistente en Topbar)**.
3. **Módulo de Anuncios (Envío manual de notificaciones)**.

---

## 2. Sistema de Toasts (Cola FIFO)

### 2.1. Arquitectura en AppContext
Anteriormente, el sistema de Toasts permitía únicamente una alerta activa a la vez. Si dos eventos ocurrían simultáneamente (ej. autoguardado + error de red), el segundo sobreescribía al primero. Se implementó una **Cola FIFO (First-In, First-Out)** en `AppContext.jsx`.

- **Estado**: `toastQueue` (Array de objetos).
- **Lógica de encolado**: `showToast(message, type, title)` agrega un nuevo objeto a la cola con un ID único (`Date.now()`).
- **Autodestrucción**: Cada toast inicializa su propio `setTimeout` al ser creado. La duración es variable según el tipo:
  - `success`: 3000ms
  - `info`: 4000ms
  - `warning`: 5000ms
  - `error`: 7000ms
- **Exposición de función Dismiss**: Cada entrada en la cola expone su función `onDismiss` para permitir que el componente visual fuerce el cierre antes de que expire el temporizador.

### 2.2. Componente Toast (`Toast.jsx`)
- Renderiza un *stack* visual mapeando sobre `toastQueue`. Los elementos se muestran en una columna flexible con `flex-col-reverse` (para que los más recientes aparezcan abajo o arriba dependiendo del flujo).
- **Accesibilidad**: Se añadieron atributos `role="alert"` y `aria-live="assertive"`.
- **CSS**: Se crearon las variables `--toast-duration` inyectadas dinámicamente y los keyframes `toast-enter` y `toast-exit` en `index.css`. Adicionalmente, incluye una barra de progreso que se agota (keyframes `shrinkWidth`).

---

## 3. Campana de Notificaciones (Topbar.jsx)

### 3.1. Optimizaciones de Rendimiento
- **Fetch en Paralelo**: El polling (cada 15s) solicitaba la lista de notificaciones y luego el conteo de no leídas de forma secuencial. Se refactorizó usando `Promise.all` para lanzar las consultas GraphQL (`OBTENER_MIS_NOTIFICACIONES` y `NOTIFICACIONES_NO_LEIDAS_QUERY`) de forma concurrente, reduciendo el tiempo de carga a la mitad.

### 3.2. Mejoras de UI/UX (Optimistic Updates)
- Cuando el usuario marca una notificación como leída, marca todas como leídas, o la oculta, la interfaz se actualiza **instantáneamente** modificando el estado local (`setNotificaciones` y `setNoLeidas`), sin esperar la respuesta del servidor. La llamada a la BD se hace en segundo plano (Fire-and-forget).
- **Badge de conteo inteligente**: Si existen más de 99 notificaciones sin leer, el badge de la campana muestra `99+` en lugar de desbordar el contenedor circular.
- Se implementó la clase `custom-scrollbar` para estilizar la barra de desplazamiento del menú desplegable.

---

## 4. Módulo de Anuncios (`Anuncios.jsx`)

### 4.1. Descripción
El módulo de "Anuncios y Notificaciones" es un sistema de comunicación interna que permite al usuario con rol de **Maestro** emitir mensajes o comunicados a diferentes audiencias (todos, por rol, por unidad, o un usuario específico).

El módulo se encuentra categorizado dentro del menú **SISTEMA** en la barra lateral de navegación principal.

### 4.2. Tipos de Audiencia Soportados
La mutación `createNotificacion` en el backend soporta 4 niveles de especificidad, ahora integrados en el frontend:
1. **GLOBAL**: Envía a todos los usuarios activos.
2. **ROL**: Requiere un ID de rol (Ej. "2" para Administrador). Envía a todos los que tengan ese rol.
3. **UNIDAD**: Requiere la clave de la unidad médica (Ej. "UMF 15"). Envía a todos los usuarios asignados a ella.
4. **PERSONAL**: Envía a un único usuario específico.

### 4.3. Buscadores Integrados (Autocomplete)
Para mejorar la usabilidad, se implementaron búsquedas asíncronas para resolver los IDs internos:
- **Audiencia UNIDAD**: Consume `GET_UNIDADES_FISICAS_QUERY` (`$search`). Busca automáticamente por coincidencia de **Nombre**, **Descripción Corta** o **Clave de Unidad**.
- **Audiencia PERSONAL**: Consume `GET_USUARIOS` (`$search`). Busca automáticamente por coincidencia de **Matrícula** o **Nombre**.
Ambos utilizan un *Debounce* de 400ms (tras escribir al menos 2 caracteres) para no saturar el servidor. 

### 4.4. Estructura Backend (`NotificacionMensaje`)
Es el modelo de datos para el comunicado global (padre).
- **id_notificacion**: ID único autoincremental.
- **titulo**: Título del comunicado (varchar 100).
- **mensaje**: Cuerpo del mensaje (varchar 1000).
- **tipo_audiencia**: `TODOS`, `ROL`, `UNIDAD`, `PERSONAL`.
- **id_audiencia**: Valor referencial convertido siempre a `VARCHAR(50)` para evitar colisiones de tipos. Para el caso de `PERSONAL`, se almacena e indexa directamente la **matrícula** del usuario en lugar del ID interno numérico.
- **id_remitente**: Quién envió el anuncio.
- **fecha_creacion**: Timestamps automáticos.

### 4.5. Diseño y Layout (Interfaz)
La pantalla (`Anuncios.jsx`) utiliza un diseño de cuadrícula (Grid) dividido en dos columnas con altura sincronizada (ambas forzadas a `650px` con disposición `flex flex-col` y alineación `items-stretch`).

#### Columna 1: Redactar anuncio
Un formulario interactivo con soporte de búsqueda para encontrar Unidades Médicas o Usuarios dependiendo del tipo de audiencia. Posee validaciones de tamaño y un scroll interno en el área de redacción para asegurar que el botón de "Enviar anuncio" permanezca fijo en la base inferior.

#### Columna 2: Historial de Anuncios Enviados
La página consulta `todasNotificaciones` aplicando parámetros de **paginación real** (`limit` y `offset`). El frontend dispone de botones de navegación (Siguiente / Anterior) que iteran de 50 en 50 anuncios de forma nativa desde la base de datos, garantizando una carga veloz del historial.
- **Visualización**: Muestra el título, mensaje, fecha y una etiqueta (Tag) que ilustra la audiencia con información complementaria (por ej. el nombre del rol o la matrícula enviada).
- **Eliminación**: Los usuarios con rol Maestro pueden borrar un anuncio enviado mediante el botón de papelera. Se consume `DELETE_NOTIFICACION_MUTATION`, lo cual ejecuta un borrado en cascada (elimina las lecturas y desaparece de la campana de todos los destinatarios).
