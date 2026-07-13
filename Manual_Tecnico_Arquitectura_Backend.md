# MANUAL TÉCNICO OFICIAL: ARQUITECTURA DEL LADO DEL SERVIDOR (BACKEND)
**Ecosistema:** Sistema Integral de Trazabilidad y Gestión de Activos Institucionales  
**Tecnologías Básicas:** Node.js, Express, Apollo Server (GraphQL), TypeORM, Microsoft SQL Server (MSSQL), TypeScript  
**Versión del Documento:** 2.0.0-PROD  
**Fecha de Revisión Arquitectónica:** Julio 2026  

---

## 1. ESTRUCTURA COMPLETA DE CARPETAS Y ARCHIVOS (BACKEND)

El backend del Ecosistema de Gestión de Activos Institucionales está construido sobre **TypeScript** utilizando un enfoque modular, transaccional y fuertemente tipado. A continuación se presenta el árbol de directorios y archivos exacto y completo de todo el repositorio, sin omisiones, abreviaciones ni truncamientos.

```text
Sistema-Gestion-Activos-Institucionales-Back/
├── .env
├── .env.example
├── .gitignore
├── MIGRATION_clave_unidad_usuarios.sql
├── MIGRATION_rotacion.sql
├── README.md
├── S1_MODELO_BD_NUEVO.sql
├── add-nom-pc.ts
├── fill-qr.ts
├── find_user.js
├── jest.config.js
├── package-lock.json
├── package.json
├── revert-db.ts
├── tsconfig.json
├── update-origen.ts
└── src/
    ├── index.ts
    ├── __tests__/
    │   └── resolvers/
    │       ├── bienes.test.ts
    │       ├── garantias.test.ts
    │       └── incidencias.test.ts
    ├── config/
    │   ├── database.ts
    │   └── environment.ts
    ├── entities/
    │   ├── Archivo.ts
    │   ├── AtributoPorTipoDispositivo.ts
    │   ├── Bien.ts
    │   ├── BienAtributo.ts
    │   ├── BienMonitor.ts
    │   ├── Bitacora.ts
    │   ├── CatAtributoTecnico.ts
    │   ├── CatCategoriaActivo.ts
    │   ├── CatInmueble.ts
    │   ├── CatModelo.ts
    │   ├── CatUnidadMedida.ts
    │   ├── Contacto.ts
    │   ├── CuentaPC.ts
    │   ├── EspecificacionTI.ts
    │   ├── Garantia.ts
    │   ├── Incidencia.ts
    │   ├── Inmueble.ts
    │   ├── Marca.ts
    │   ├── MesaCorrespondencia.ts
    │   ├── MovimientoInventario.ts
    │   ├── Nota.ts
    │   ├── NotificacionLectura.ts
    │   ├── NotificacionMensaje.ts
    │   ├── PrestamoBien.ts
    │   ├── ProgramasPC.ts
    │   ├── Proveedor.ts
    │   ├── RegistroSalida.ts
    │   ├── RegistroSalidaBien.ts
    │   ├── ReporteGarantia.ts
    │   ├── Rol.ts
    │   ├── Segmento.ts
    │   ├── SolicitudCambio.ts
    │   ├── TipoDispositivo.ts
    │   ├── TipoIncidencia.ts
    │   ├── Ubicacion.ts
    │   ├── Unidad.ts
    │   ├── UnidadACargo.ts
    │   └── Usuario.ts
    ├── graphql/
    │   ├── dataloaders/
    │   │   └── index.ts
    │   ├── resolvers/
    │   │   ├── atributos.resolver.ts
    │   │   ├── auth.resolver.ts
    │   │   ├── bienes.resolver.ts
    │   │   ├── bitacora.resolver.ts
    │   │   ├── catalogos.resolver.ts
    │   │   ├── index.ts
    │   │   ├── mesaCorrespondencia.resolver.ts
    │   │   ├── movimientos.resolver.ts
    │   │   ├── notificaciones.resolver.ts
    │   │   ├── prestamos.resolver.ts
    │   │   ├── salidas.resolver.ts
    │   │   ├── solicitudesCambio.resolver.ts
    │   │   ├── transaccionales.resolver.ts
    │   │   ├── ubicaciones.resolver.ts
    │   │   └── usuarios.resolver.ts
    │   └── schema/
    │       └── index.ts
    ├── middleware/
    │   ├── auth.middleware.ts
    │   └── context.ts
    ├── scripts/
    │   ├── create-prestamos-table.ts
    │   ├── hash-passwords.ts
    │   └── test.ts
    ├── subscribers/
    │   └── BitacoraSubscriber.ts
    ├── tests/
    │   └── README.md
    └── utils/
        ├── apolloPlugin.ts
        ├── asyncLocalStorage.ts
        ├── errors.ts
        ├── logger.ts
        ├── pagination.ts
        └── __tests__/
            ├── errors.test.ts
            └── pagination.test.ts
```

### Patrón de Arquitectura y Responsabilidad por Directorio

El sistema adopta una **Arquitectura en Capas Limpias orientada a Dominio sobre GraphQL (Clean Layered Domain-Driven GraphQL Architecture)**. Esta separación garantiza un bajo acoplamiento entre la capa de transporte (Express/HTTP/GraphQL), la capa de lógica de negocio (Resolvers y Helpers) y la capa de persistencia (TypeORM/MSSQL).

1. **`src/index.ts` (Punto de Entrada & Bootstrap):**  
   Orquesta la inicialización del servidor. Se encarga de conectar al motor de base de datos SQL Server antes de levantar HTTP, configurar middlewares de seguridad (`helmet`, `cors`, `express-rate-limit`), instanciar el servidor `ApolloServer` y vincular el plugin de traza asíncrona (`asyncContextPlugin`).
2. **`src/config/` (Configuración e Infraestructura):**  
   Alberga centralizadamente la configuración del sistema. `environment.ts` parsea y valida estrictamente las variables de entorno (`dotenv`), estableciendo defaults seguros e instanciando las políticas del servidor (límites de tasa, timeout de BD, profundidad máxima de GraphQL). `database.ts` instancia y configura el `DataSource` de **TypeORM** adaptado al driver de **Microsoft SQL Server (mssql)**, inyectando el pool de conexiones y el registro de suscriptores de eventos.
3. **`src/entities/` (Capa de Modelado de Datos ORM):**  
   Contiene las 38 clases de entidad mapeadas 1:1 o relacionalmente con las tablas del motor SQL Server mediante decoradores de TypeORM (`@Entity`, `@PrimaryColumn`, `@Column`, `@ManyToOne`, `@OneToMany`). Define las restricciones de nulidad, tipos SQL nativos (`uniqueidentifier`, `varchar`, `decimal`, `bit`, `datetime`) y relaciones bidireccionales.
4. **`src/graphql/schema/` (Contrato API GraphQL - TypeDefs):**  
   En `index.ts` se definen exhaustivamente en sintaxis SDL (Schema Definition Language) todos los tipos, inputs, queries, mutations y enums expuestos a los clientes (Frontend Web React y Agente de Escritorio Windows C#).
5. **`src/graphql/resolvers/` (Capa de Controladores y Lógica de Negocio):**  
   Estructurados por dominio institucional (`bienes`, `usuarios`, `auth`, `incidencias/transaccionales`, `salidas`, `prestamos`, `solicitudesCambio`, etc.). Reciben las peticiones GraphQL, validan autorización y formato, gestionan transacciones SQL complejas con `EntityManager` y retornan las estructuras tipadas esperadas.
6. **`src/graphql/dataloaders/` (Capa de Optimización de Consultas N+1):**  
   Implementa el patrón de lote y caché por solicitud (`DataLoader`). Agrupa consultas individuales a tablas catálogos o relacionales (ej. marcas, modelos, ubicaciones, especificaciones TI) en una única consulta SQL con cláusula `IN (...)` por cada ciclo de eventos del event loop.
7. **`src/middleware/` (Autenticación y Contexto de Ejecución):**  
   `context.ts` intercepta la cabecera HTTP `Authorization`, verifica y decodifica el JSON Web Token (`JWT`), extrae la IP real (considerando proxies y balanceadores) y crea el objeto `GraphQLContext`. `auth.middleware.ts` proporciona guardas de seguridad (`requireAuth`, `requireRole`) para restringir el acceso a resolutores según roles institucionales (`SUPERADMIN`, `ADMIN`, `ESTANDAR`, etc.).
8. **`src/subscribers/` (Auditoría Continua e Intrusiva):**  
   `BitacoraSubscriber.ts` implementa `EntitySubscriberInterface` de TypeORM para interceptar de manera transparente los eventos `afterInsert`, `afterUpdate` y `afterRemove` en todo el motor. Recupera el ID de usuario activo desde memoria asíncrona (`AsyncLocalStorage`) y graba de forma transaccional el movimiento exacto en la tabla de bitácora sin requerir llamadas manuales en los resolutores.
9. **`src/utils/` (Servicios Transversales y Diagnóstico):**  
   Contiene herramientas de soporte crítico: `asyncLocalStorage.ts` y `apolloPlugin.ts` mantienen el contexto de sesión transaccional entre llamadas asíncronas de Node.js; `errors.ts` provee clases de excepción estandarizadas (`NotFoundError`, `ValidationError`, `AuthError`); `pagination.ts` gestiona codificación y decodificación de cursores en Base64 (`offset/limit`); y `logger.ts` ofrece registro estructurado de eventos en consola.
10. **`src/scripts/` & Raíz de Mantenimiento:**  
    Scripts utilitarios ejecutables vía `ts-node` o pre-producción para el mantenimiento operativo de la base de datos (migraciones SQL manuales, corrección de hashes de contraseñas, llenado de códigos QR y parches de nombres de host).

---

## 2. ARQUITECTURA CENTRAL Y CONFIGURACIÓN DEL SERVIDOR

### 2.1 Flujo de Inicialización (`src/index.ts`)
El ciclo de vida del servidor arranca de manera determinista mediante la función asíncrona `bootstrap()`:
1. **Conexión Resiliente a Base de Datos:** Se invoca `connectDatabase()`, la cual implementa un bucle de reintentos (*exponential/timed backoff* con 5 reintentos y esperas de 3000 ms) antes de habilitar el servidor HTTP. Si la base de datos no responde, el proceso aborta tempranamente (`process.exit(1)`).
2. **Configuración del Proxy y Seguridad HTTP:** Se inicializa `express()`. Se activa `app.set('trust proxy', 1)` debido a que el sistema opera típicamente detrás de un reverse proxy (IIS o Nginx en Windows Server). Se monta **Helmet** configurado con políticas de inserción cruzada adaptadas a GraphQL (`crossOriginEmbedderPolicy: false`).
3. **Control de Tráfico (Rate Limiting):** Se aplica `express-rate-limit` exclusivamente al endpoint `/graphql` protegiendo contra ataques de denegación de servicio (DDoS) o fuerza bruta, permitiendo un máximo configurable de peticiones por ventana de tiempo (`RATE_LIMIT_MAX` en `RATE_LIMIT_WINDOW_MS`).
4. **Sondeo de Salud (`/health`):** Endpoint REST ultra-ligero para monitoreo de infraestructura (uptime, estado y entorno) sin sobrecargar el motor GraphQL.
5. **Instanciación del Apollo Server:** Se inicializa `ApolloServer<GraphQLContext>` pasando `typeDefs`, `resolvers`, reglas de validación de profundidad (`graphql-depth-limit` fijado en 7 niveles para prevenir ataques de consultas anidadas recursivas), y plugins de cierre seguro de HTTP (`ApolloServerPluginDrainHttpServer`).
6. **Enmascaramiento de Errores en Producción (`formatError`):** Intercepta todas las excepciones devueltas por los resolutores. Si el entorno es `production`, se eliminan las trazas de pila (`stacktrace`) y detalles internos del motor antes de enviar el payload al cliente HTTP.

### 2.2 Gestión del Contexto (`context.ts` y `asyncLocalStorage.ts`)
Por cada petición HTTP entrante al endpoint `/graphql`, Express ejecuta `expressMiddleware` delegando a `buildContext({ req })`:
* **Instanciación de DataLoaders:** Se crea una nueva instancia fresca de `DataLoaders` por petición. Esto asegura aislamiento absoluto de caché entre peticiones de usuarios concurrentes.
* **Extracción de Identidad:** Se extrae el token `Bearer` de la cabecera `Authorization`. Si es válido, se verifica con `jwt.verify` y se inyecta el `payload` (ID de usuario, rol, matrícula, clave de unidad/zona) dentro del contexto.
* **Trazabilidad de Origen e IP:** Se extrae `x-origen` (ej. `WEB` o `WIN` para el agente de inventario en C#), sanitizando direcciones IPv4 enmascaradas sobre IPv6 (`::ffff:`).
* **Propagación Asíncrona Global (`sessionContext`):** Para evitar pasar el objeto `context` manualmente hasta las capas más profundas de persistencia de TypeORM, se utiliza el plugin `asyncContextPlugin`. Durante el hook `executionDidStart`, se invoca `sessionContext.enterWith({ usuarioId, origen })`. A partir de ese momento, cualquier función de Node.js en esa cadena de promesas (incluyendo triggers e interceptores de TypeORM) puede leer el usuario activo llamando a `sessionContext.getStore()`.

### 2.3 Variables de Entorno (`environment.ts`)
El archivo centraliza la configuración en un objeto inmutable exportado `env`. Utiliza validaciones estrictas con la función `requireEnv(key)` que lanza un error fatal durante el tiempo de arranque si variables críticas como `DB_PASSWORD` o `JWT_SECRET` no están presentes en el archivo `.env` o en el entorno del sistema operativo.

---

## 3. CAPA DE DATOS Y MODELADO ENTIDAD-RELACIÓN (SQL SERVER)

El backend interactúa de manera nativa con **Microsoft SQL Server (MSSQL)** utilizando **TypeORM**. El mapeo se realiza respetando la nomenclatura y tipos exactos de la base de datos institucional relacional.

### 3.1 Mapeo de Entidades Principales (`src/entities/`)
* **`Bien` (`Bienes`):** Entidad núcleo del sistema. Su clave primaria `id_bien` es un `uniqueidentifier` (UUID v4). Modela el inventario físico e informático, con campos para número de serie (`num_serie`), número de inventario (`num_inv`), estatus operativo, hash único de QR (`qr_hash`), y claves presupuestales autogeneradas por triggers en la base de datos.
* **`Usuario` (`Usuarios`):** Gestiona los accesos y resguardos. Contiene clave primaria autoincremental `id_usuario`, matrícula única, hash de contraseña (bcrypt), referencia a su rol institucional (`id_rol`), adscripción por clave de unidad (`clave_unidad`) y zona de cobertura (`clave_zona`).
* **`Incidencia` (`Incidencias`):** Modela tickets de reportes técnicos o fallas operativas de un activo. Relaciona el activo (`id_bien`), el usuario reportante y el técnico asignado, almacenando el historial textual y el estatus de reparación.
* **`Inmueble` (`unidades`):** Entidad que representa la estructura organizacional y física institucional (Centros de Trabajo, Residencias, Departamentos), identificada por una `clave` alfanumérica única (`varchar(50)`).

### 3.2 Patrones de Relaciones SQL
Para garantizar la integridad y trazabilidad institucional, el modelo ORM implementa relaciones complejas:
* **Many-to-One (Muchos a Uno):** Prácticamente todas las transacciones apuntan al catálogo central. Un `Bien` tiene relaciones `@ManyToOne` hacia `CatCategoriaActivo`, `CatUnidadMedida`, `Inmueble` (unidad de adscripción), `CatModelo`, `Usuario` (resguardante), `Segmento` y `Ubicacion`.
* **One-to-Many (Uno a Muchos):** Un `Bien` se vincula hacia múltiples `EspecificacionTI` (historial de escaneos de hardware), `Garantia` (pólizas activas o vencidas), `Nota` (observaciones e incidencias administrativas), `MovimientoInventario` (bitácora histórica de transferencias de unidad o resguardante) y `PrestamoBien`.
* **Relaciones Especializadas de Hardware (`BienMonitor`, `CuentaPC`, `ProgramasPC`):** El sistema modela los equipos de cómputo (PC/Laptops) vinculándolos con sus monitores perimetrales a través de la tabla relacional `Bien_Monitores`. Asimismo, almacena las cuentas locales/dominio detectadas por scripts en `Cuentas_PC` y el inventario de software instalado en `Programas_PC`.

### 3.3 Motor SQL Server y Configuración del Pool (`database.ts`)
La conexión al motor SQL Server impone parámetros críticos en `AppDataSource`:
* **Driver MSSQL y Seguridad:** Se configura con opciones nativas `encrypt: env.db.encrypt` y `trustServerCertificate: true` para permitir conexiones seguras en entornos locales y corporativos sobre redes institucionales.
* **Manejo de Aritmética SQL (`enableArithAbort: true`):** Parámetro esencial en SQL Server para evitar desbordamientos numéricos o advertencias de truncamiento al calcular agrupaciones o métricas de inventario en consultas pesadas.
* **Connection Pooling:** El pool se gestiona limitando conexiones concurrentes (`max: 20`, `min: 2`) con un tiempo de inactividad programado (`idleTimeoutMillis: 30000`). Esto previene el agotamiento de sockets (*Connection Pool Starvation*) cuando múltiples terminales o escáneres realizan consultas concurrentes en el sistema.

---

## 4. CAPA DE API (ESQUEMAS Y RESOLVERS GRAPHQL)

### 4.1 Diseño del Esquema (`src/graphql/schema/index.ts`)
El contrato GraphQL agrupa las operaciones en torno a flujos transaccionales claros:
* **Tipos de Dominio:** `Bien`, `Usuario`, `Incidencia`, `Garantia`, `Salida`, `Prestamo`, `SolicitudCambio`, cada uno exponiendo tanto atributos nativos como campos resueltos dinámicamente (ej. `bien.especificaciones_ti`, `bien.garantias`).
* **Entradas (Inputs):** Separación estricta entre operaciones de creación (`CreateBienInput`) y actualización (`UpdateBienInput`), asegurando que campos inmutables (como `id_bien` o la auditoría original) no puedan ser alterados mediante mutaciones arbitrarias.
* **Consultas Paginadas (`BienConnection`):** Las consultas de inventario no devuelven arreglos crudos infinitos; implementan el estándar Relay/Cursor de paginación o estructuras de respuesta compuestas conteniendo `items: [Bien!]!`, `totalCount: Int!`, y metadata de paginación (`pageInfo`).

### 4.2 Lógica en Resolvers (`bienes.resolver.ts`, etc.)
Los resolutores interceptan las peticiones y aplican una arquitectura transaccional defensiva:
1. **Verificación de Guarda:** Cada resolver protegido inicia llamando a `requireAuth(context)` y, en operaciones administrativas, a `requireRole(context, [ROLES.SUPERADMIN, ROLES.ADMIN])`.
2. **Filtrado Multidimensional Dinámico:** En `bienes.resolver.ts`, la consulta de inventario construye un `QueryBuilder` dinámico. Aplica filtros condicionales complejos (por categoría, segmento, ubicación, resguardo, marca, rangos de memoria RAM, almacenamiento y búsqueda full-text inteligente por IP, número de serie o cuenta de usuario).
3. **Control de Zona Estándar:** Si el usuario es de nivel `ESTANDAR`, el resolver inyecta automáticamente un `innerJoin` obligatorio hacia la tabla `unidades` forzando que `_ugz.clave_zona = context.user.clave_zona`, impidiendo físicamente que un operador visualice o modifique activos fuera de su zona geográfica asignada.
4. **Mutaciones Atómicas (`AppDataSource.transaction`):** Operaciones compuestas, como la creación o alta masiva de un bien con monitor y software (`createBien`, `registrarSalida`), se encapsulan dentro de un bloque `AppDataSource.transaction(async (manager) => { ... })`. Si el alta del monitor o la generación del QR falla, se realiza un *rollback* total garantizando cero inconsistencias en SQL.
5. **Calidad de Datos Institucional y Agregación Analítica (`movimientos.resolver.ts`):** Para la generación de indicadores y métricas analíticas (`dashboardStats` y `dashboardMetrics`), los resolutores aplican un filtrado de negocio institucional estricto sobre la entidad `Bien` (`createQueryBuilder`):
   - **Adscripción Delegacional Oficial (`clave_unidad_ref LIKE '19%'`):** Se impone un `INNER JOIN` al catálogo de `unidades` exigiendo que el activo pertenezca a claves oficiales de la Delegación 19 Nayarit (además de validar la `clave_zona` en usuarios Estándar).
   - **Exclusión de Almacenes y Resguardo Temporal:** Mediante `LEFT JOIN` con `Ubicaciones`, se descartan automáticamente equipos situados en áreas de almacenamiento o depósito (`LOWER(ub.nombre_ubicacion) NOT LIKE '%bodega%' OR ub.nombre_ubicacion IS NULL`).
   - **Purificación de Número de Inventario (`num_inv` numérico):** Se ejecuta la validación SQL `num_inv LIKE '%[0-9]%' AND num_inv NOT LIKE '%[^0-9]%'`, garantizando la contabilidad exclusiva de bienes patrimoniales debidamente capitalizados con número de inventario puro.
   - **Consolidación Operativa de Estatus:** Las agregaciones principales de activos en operación regular (`bienesActivos`) unifican los estatus `'ACTIVO'` y `'PRESTAMO'` (`PRÉSTAMO`) mediante `UPPER(b.estatus_operativo) IN ('ACTIVO', 'PRESTAMO', 'PRÉSTAMO')`, mientras que las métricas desglosadas por unidad y tipo (`dashboardMetrics`) mantienen e incluyen en paralelo el estatus `'INACTIVO'`, asegurando una total consistencia entre contadores globales y tablas desglosadas.

### 4.3 Optimización con DataLoaders (`dataloaders/index.ts`)
Cuando una consulta solicita un listado de 100 bienes e incluye el campo `categoria`, `marca` o `usuario_resguardo`, un servidor GraphQL ingenuo ejecutaría 1 consulta principal + 300 consultas individuales SQL (*Problema N+1*).
En nuestro backend, cada propiedad relacional de las entidades en el resolver delega a `context.loaders`:
```typescript
// En Bienes Resolver para el campo marca:
marca: (parent, _, { loaders }) => parent.clave_modelo ? loaders.catModeloLoader.load(parent.clave_modelo).then(mod => mod ? loaders.marcaLoader.load(mod.clave_marca) : null) : null
```
El `DataLoader` recolecta todas las llamadas de un ciclo de evento, extrae las claves únicas (`keys`) y ejecuta una única consulta SQL altamente eficiente:
```sql
SELECT * FROM Marcas WHERE clave_marca IN (1, 5, 12, 18);
```
Esto reduce la sobrecarga del motor de base de datos de cientos de queries a solo 2 o 3 consultas por cada petición GraphQL.

---

## 5. LÓGICA DE NEGOCIO, UTILIDADES Y MANEJO DE ERRORES

### 5.1 Sistema Transversal de Auditoría Automática (`BitacoraSubscriber.ts`)
A diferencia de sistemas tradicionales donde el programador debe insertar manualmente un registro en la tabla de log en cada endpoint, esta arquitectura utiliza un **Suscriptor Transaccional de TypeORM**:
* Al ocurrir un `afterInsert`, `afterUpdate` o `afterRemove` en cualquier tabla del sistema (excepto tablas de ruido interno como notificaciones o auditoría misma), el suscriptor se dispara.
* Interroga a `sessionContext.getStore()` para extraer el `usuarioId` y el `origen` (`WEB` o `WIN`) que provocó la petición en Express.
* Genera una fotografía JSON de los datos cambiados (`estadoAnterior`, `estadoNuevo`, `columnasModificadas`).
* **Ejecución Transaccional Aclopada:** Llama a `event.manager.save(Bitacora, bitacora)`. Al utilizar el `manager` del propio evento, la inserción en la tabla `Bitacora` viaja dentro de la misma transacción SQL que la modificación de inventario. Si la auditoría falla, toda la transacción se revierte; no es posible alterar un dato institucional sin dejar huella indeleble.

### 5.2 Estructura y Sanear Errores (`errors.ts` & `logger.ts`)
El backend prohíbe el envío de stacktraces en crudo al cliente en entornos productivos.
* Se definen clases herederas de `Error`: `AppError`, `NotFoundError`, `ValidationError`, `AuthError`, `ForbiddenError`. Cada una lleva consigo un código HTTP o GraphQL extension exacto (ej. `NOT_FOUND`, `UNAUTHENTICATED`).
* Cuando un resolver lanza `throw new NotFoundError('Bien')`, el formateador global en `src/index.ts` intercepta la respuesta, registra los detalles técnicos confidenciales en los archivos o consola del servidor usando `logger.ts`, y responde al cliente frontend con una estructura JSON limpia y segura sin exponer esquemas ni rutas de archivos internos del servidor.

---

## 6. SCRIPTS, MIGRACIONES Y ENTORNO DE PRODUCCIÓN

### 6.1 Scripts Auxiliares (`src/scripts/`)
* **`hash-passwords.ts`:** Script de mantenimiento administrativo utilizado en implementaciones iniciales o rotaciones de seguridad. Recorre la tabla de usuarios en SQL Server, identifica contraseñas en texto plano o legacy y las somete a hash criptográfico seguro con `bcrypt` (10 rondas de sal).
* **`create-prestamos-table.ts` & Scripts SQL:** Automatizan la creación idempotente de nuevas estructuras de tablas relacionales (`Prestamos_Bienes`) y sus restricciones de llaves foráneas (`CONSTRAINT FK_...`) sin depender de la sincronización insegura (`synchronize: true`) de TypeORM en producción.

### 6.2 Preparación para Producción en Windows Server
El sistema está diseñado para operar permanentemente 24/7 sobre infraestructura de servidores institucionales Windows Server:
* **Compilación a JavaScript Limpio:** Mediante el comando `npm run build` (`tsc`), todo el código TypeScript de `src/` se transpila a código JavaScript optimizado CommonJS en la carpeta `/dist`.
* **Gestor de Procesos PM2 / IIS Node:** En producción, la aplicación se ejecuta desde `dist/index.js` gestionada por **PM2** en clúster o como servicio de Windows (*Windows Service* vía `node-windows`). Esto asegura reinicio automático en caso de fallo del sistema operativo, recolección de logs rotativos en `/logs` y aprovechamiento multi-núcleo de la CPU del servidor.

---

## 7. FRAGMENTOS DE CÓDIGO ESENCIALES (DEEP DIVE SNIPPETS)

A continuación se presentan 4 bloques de código reales, críticos y representativos del núcleo del backend, detallados línea por línea.

### Snippet 1: Inyección de Identidad Transaccional por Petición (`src/utils/apolloPlugin.ts`)
Este bloque muestra cómo el backend logra vincular de forma invisible al usuario autenticado con los hooks transaccionales de bajo nivel del ORM sin pasar parámetros por toda la aplicación.

```typescript
import { ApolloServerPlugin } from '@apollo/server';
import { GraphQLContext } from '../middleware/context';
import { sessionContext } from './asyncLocalStorage';

// Plugin de Apollo Server que intercepta el ciclo de vida de cada petición GraphQL
export const asyncContextPlugin: ApolloServerPlugin<GraphQLContext> = {
  async requestDidStart() {
    return {
      // Hook disparado exactamente antes de comenzar la ejecución del resolver GraphQL solicitado
      async executionDidStart({ contextValue }) {
        // Línea 10: Extraemos el ID del usuario decodificado previamente del JWT por el middleware de Express
        const usuarioId = contextValue.user?.id_usuario;
        // Línea 11: Extraemos el origen del cliente (ej. 'WEB' o cliente de escritorio 'WIN')
        const origen = contextValue.origen;
        
        // Línea 14: CRÍTICO. Entramos al almacén asíncrono (AsyncLocalStorage).
        // Aísla este contexto en el hilo de Node.js durante toda la vida útil de la promesa del resolver.
        // Permite que BitacoraSubscriber lea el autor del cambio sin recibir 'context' como argumento.
        sessionContext.enterWith({ usuarioId, origen });
      },
    };
  },
};
```

### Snippet 2: Auditoría Automática Inviolable dentro de la Misma Transacción (`src/subscribers/BitacoraSubscriber.ts`)
Muestra el motor de auditoría que garantiza la trazabilidad 100% confiable de cualquier modificación en la base de datos institucional.

```typescript
  // Hook disparado por TypeORM justo después de actualizar un registro en cualquier tabla
  async afterUpdate(event: UpdateEvent<any>) {
    // Línea 39: Prevención de bucle infinito. Si el cambio ocurrió en la tabla Bitacora, ignorar.
    if (event.metadata.targetName === 'Bitacora' || !event.entity) return;

    // Línea 41: Extraemos el usuario actual del contexto asíncrono global (inyectado por Snippet 1)
    const store = sessionContext.getStore();
    const usuarioId = store?.usuarioId || event.queryRunner?.data?.usuarioId;
    if (!usuarioId) return; // Si fue una tarea automatica interna sin sesión de usuario, no se audita.

    // Línea 47: Extrae la clave primaria real de la entidad afectada sin importar cómo se llame la columna PK
    const entityId = this.obtenerIdEntidad(event);

    // Línea 49: Estructuramos el payload forense con la fotografía del Antes, el Después y las columnas alteradas
    const detalles = JSON.stringify({
      estadoAnterior: event.databaseEntity,
      estadoNuevo: event.entity,
      columnasModificadas: event.updatedColumns.map((col) => col.propertyName),
    });

    // Línea 55: Persistencia del registro de auditoría en la base de datos
    await this.guardarEnBitacora(
      event,
      usuarioId,
      'EDICION',
      event.metadata.tableName,
      entityId,
      detalles
    );
  }

  private async guardarEnBitacora(...) {
    const bitacora = new Bitacora();
    bitacora.id_usuario = usuarioId;
    bitacora.accion = accion;
    bitacora.tabla_afectada = tablaAfectada;
    bitacora.registro_afectado = registroAfectado || 'N/A';
    bitacora.detalles_movimiento = detalles;
    bitacora.origen = store?.origen || 'WEB';

    // Línea 159: PILAR CRÍTICO DE ARQUITECTURA.
    // Usamos 'event.manager.save' en vez de un repositorio externo independiente.
    // Esto obliga a que la escritura del registro de Bitácora se acople a la transacción SQL activa.
    // Si la tabla de auditoría falla, la modificación del activo se cancela por Rollback automático.
    await event.manager.save(Bitacora, bitacora);
  }
```

### Snippet 3: Resolución Transaccional Compleja y Detección de Conflictos de Hardware (`src/graphql/resolvers/bienes.resolver.ts`)
Muestra la lógica algorítmica de sincronización y validación de inventario cuando un escáner o agente reporta componentes de hardware (monitores WMI) vinculados a una terminal PC.

```typescript
/**
 * Helper compartido: procesa la lista de monitores WMI detectados en una terminal PC.
 * Realiza verificaciones cruzadas en SQL para evitar monitores duplicados o robados entre terminales.
 */
export async function procesarMonitoresHelper(
  manager: EntityManager, // Recibe el EntityManager de la transacción activa principal
  id_bien_pc: string,
  monitoresWmi: MonitorWmiInput[],
  forzar: boolean
): Promise<{ ok: boolean; conflictos: MonitorConflicto[] }> {
  const bienRepo = manager.getRepository(Bien);
  const bmRepo = manager.getRepository(BienMonitor);

  // Línea 74: Verificamos la existencia del equipo de cómputo principal
  const pc = await bienRepo.findOne({ where: { id_bien: id_bien_pc } });
  if (!pc) throw new NotFoundError('PC (bien)');

  const conflictos: MonitorConflicto[] = [];

  // Línea 80: Iteramos sobre los periféricos reportados por el agente físico
  for (const mon of monitoresWmi) {
    if (!mon.num_serie) continue;
    // Buscamos si el monitor ya existe en el inventario general por su número de serie
    const bienMon = await bienRepo.findOne({ where: { num_serie: mon.num_serie } });
    if (!bienMon) continue;

    // Verificamos si ese monitor ya está en la tabla relacional Bien_Monitores
    const relExistente = await bmRepo.findOne({ where: { id_monitor: bienMon.id_bien } });
    // Línea 86: Si el monitor está conectado en base de datos a un ID de PC distinto al actual -> ALERTA DE CONFLICTO
    if (relExistente && relExistente.id_bien !== id_bien_pc) {
      const equipoAnterior = await bienRepo.findOne({ where: { id_bien: relExistente.id_bien } });
      conflictos.push({
        num_serie: mon.num_serie,
        num_inv_equipo_anterior: equipoAnterior?.num_inv ?? undefined,
        num_serie_equipo_anterior: equipoAnterior?.num_serie ?? relExistente.id_bien,
      });
    }
  }

  // Línea 98: Si se detectaron colisiones físicas de inventario y el operador no activó la bandera 'forzar'
  // se aborta el flujo transaccional retornando los conflictos al frontend para decisión humana.
  if (conflictos.length > 0 && !forzar) {
    return { ok: false, conflictos };
  }
  
  // Continuación: Si no hay conflictos o se fuerza, se actualizan las relaciones dentro de la transacción...
```

### Snippet 4: Seguridad Geográfica y Aislamiento de Zona por Nivel de Acceso (`src/graphql/resolvers/transaccionales.resolver.ts`)
Demuestra cómo el backend impone restricciones de visibilidad de datos al vuelo basándose en el contexto del usuario en consultas de incidencias y garantías.

```typescript
    // Consulta GraphQL para listado de garantías activas
    garantias: async (_: unknown, { id_bien, estado_garantia }: any, context: GraphQLContext) => {
      // Línea 19: Guarda obligatoria de autenticación JWT
      requireAuth(context);
      
      // Línea 20: Construcción dinámica de consulta sobre la tabla Garantias usando TypeORM QueryBuilder
      const qb = AppDataSource.getRepository(Garantia).createQueryBuilder('g');
      if (id_bien) qb.andWhere('g.id_bien = :id_bien', { id_bien });
      if (estado_garantia) qb.andWhere('g.estado_garantia = :e', { e: estado_garantia });
      
      // Línea 24: PILAR DE SEGURIDAD GEOGRÁFICA (Multi-Tenancy por Zona Institucional).
      // Evaluamos si el usuario tiene rol 'ESTANDAR' (operador local de residencia/unidad).
      if (isEstandar(context) && context.user?.clave_zona) {
        // Forzamos un INNER JOIN contra la tabla Bienes y la tabla unidades.
        // Restringimos la consulta SQL para que solo retorne garantías cuyo bien pertenezca a la clave_zona del usuario.
        qb.innerJoin('Bienes', '_bgz', '_bgz.id_bien = g.id_bien')
          .innerJoin('unidades', '_ugz', `_ugz.clave = _bgz.clave_unidad_ref AND _ugz.clave_zona = :_gz`, { _gz: context.user.clave_zona });
      } else if (isEstandar(context)) {
        // Línea 27: Si el usuario estándar por alguna anomalía no tiene clave_zona asignada en su JWT,
        // inyectamos una condición falsa inquebrantable ('1 = 0') retornando 0 registros por seguridad por defecto.
        qb.andWhere('1 = 0');
      }
      
      return qb.orderBy('g.fecha_fin', 'ASC').getMany();
    },
```

---
*Fin del Manual Técnico Oficial de Arquitectura Backend.*
