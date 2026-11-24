# Fuentes pendientes: resultados

Fecha de la comprobación: 2026-11-24.

## Resumen

- La decisión provisional (Three.js + react-three-fiber + WebGL + BVHEcctrl + catálogo JSON) se mantiene. Ningún punto la tumba.
- WebGPURenderer sigue siendo "experimental" según el manual oficial. Dos benchmarks de terceros lo miden más lento en CPU que WebGLRenderer con muchas mallas sueltas. Conviene quedarse con WebGLRenderer.
- Jellyfin: el HLS maestro no sale en la especificación OpenAPI. Sale en el código fuente (`Videos/{itemId}/master.m3u8`). El parámetro `api_key` solo se acepta con la autorización heredada activada. Lo seguro es `ApiKey` o la cabecera `Authorization: MediaBrowser ...`.
- CORS de Jellyfin: el ajuste existe (`CorsHosts`, por defecto `*`). No pude confirmar que haya un campo para él en el panel de administración.
- Punto 6 no se pudo cerrar. El navegador automatizado no puede bloquear el puntero y las pestañas estaban ocultas. No hay pruebas de fluidez ni de movimiento reales.

## Resultados por punto

### 1. Aura Stacks Library 3D

**Estado:** comprobado en parte.

**Hallazgos**
- Es un proyecto de Devpost de un estudiante de 11º curso, principiante en programación.
- Es una biblioteca virtual donde se suben PDFs y se colocan en estanterías. El libro se ilumina al pasar el ratón. Tiene lector de PDF integrado, buscador y estanterías que crecen solas.
- Tecnologías declaradas: TypeScript, CSS y Three.js. "Lovable" hizo la mayor parte del montaje.
- Los libros salen de PDFs que sube el usuario, no de carpetas del ordenador.
- Reto declarado: cargar los PDFs dentro del 3D. Lo resolvió con una página de lector aparte con su propio marco.
- Repositorio público, licencia y demo: no constan en la página.
- No comprobé si Devpost enlaza a un repo o a una demo desde sus botones "Try it out". La extracción solo devolvió el texto.

**Fuentes abiertas**
- https://devpost.com/software/aura-stacks-library-3d

### 2. Estado de WebGPURenderer en Three.js

**Estado:** comprobado.

**Hallazgos**
- Madurez: el manual dice que el renderizador "is still in an experimental state although its maturity level has been greatly improved in the last years". Según el escenario puede haber funciones que faltan o mejor rendimiento con `WebGLRenderer`.
- Retroceso: usa WebGPU por defecto y cae a un backend WebGL 2 si el navegador no soporta WebGPU. Se fuerza con `forceWebGL: true`.
- Migración: no admite `ShaderMaterial`, `RawShaderMaterial` ni `onBeforeCompile()`. Tampoco `EffectComposer`. Hay que pasar a materiales de nodos y TSL.
- `WebGLRenderer` sigue mantenido y es la opción recomendada para aplicaciones WebGL 2 puras. No se prevén funciones grandes nuevas para él.
- Última versión de three en npm: 0.187.0 (publicada el 2026-10-30). Release r187 en GitHub del 2026-10-30. Las notas de r187 incluyen correcciones de BatchedMesh (por ejemplo #34828, #35272) y de la parte de nodos/WebGPU. No las leí enteras.
- Rendimiento (benchmarks de terceros, no oficiales):
  - 3d-asset.com, r187, portátil con RTX 4060, Chrome headless: con 4.176 mallas separadas, WebGPURenderer gastó 48 ms de CPU por fotograma frente a 28 ms de WebGLRenderer. El primer fotograma tardó 1,5 s frente a 665 ms. El retroceso a WebGL 2 de WebGPURenderer fue el más lento: 62 ms. Con 1.000 a 20.000 mallas sueltas, WebGPU tardó aproximadamente el doble que WebGL. Con instancias de vehículos: 13 ms (WebGPU) frente a 7,9 ms (WebGL). Con `BundleGroup` más instancias, WebGPU bajó a 3,0 ms.
  - digitalgarden.si, three 0.186.1, RTX 5070 Ti: con 1.000 y 10.000 `InstancedMesh` los tres caminos (WebGL clásico, WebGPU, WebGL 2 de respaldo) dieron 59,99 FPS. Con 100.000 dieron unos 20 FPS en los tres. El tiempo de CPU por llamada de render fue siempre menor en WebGL clásico (0,065 ms) que en WebGPU (0,22 ms). Con `BatchedMesh` y 100.000 elementos, el backend WebGPU declara 100.001 llamadas. El WebGL 2 de respaldo bajó a unos 9,9 FPS.
  - Son mediciones de una sola máquina cada una. Los ejes de CPU y de fotogramas no son iguales entre los dos estudios.
- El manual oficial no da cifras de rendimiento con miles de mallas, `InstancedMesh` ni `BatchedMesh`: no consta.

**Fuentes abiertas**
- https://threejs.org/manual/pages/webgpurenderer.html
- https://github.com/mrdoob/three.js/releases
- https://registry.npmjs.org/three
- https://3d-asset.com/guides/threejs-webgpurenderer-vs-webglrenderer-benchmark/
- https://digitalgarden.si/benchmarks/threejs-webgpu-vs-webgl-instancing/
- Nota: la dirección `https://threejs.org/manual/en/webgpurenderer` no se pudo abrir (devolvió otra página). La página actual es la de `/manual/pages/webgpurenderer.html`.

### 3. API de Jellyfin

**Estado:** comprobado en parte.

**Hallazgos**
- `https://api.jellyfin.org/` no se pudo extraer como texto (error del extractor, y el navegador automatizado se quedó sin respuesta). Usé el JSON de OpenAPI estable: `info.version` "12.4.0", descargado de `https://api.jellyfin.org/openapi/jellyfin-openapi-stable.json`. Contrasté lo que faltaba con el código fuente de Jellyfin en GitHub (rama master).
- Listar elementos de una biblioteca: `GET /Items`, con `parentId`, `userId`, `recursive`, `includeItemTypes`, `fields`, `startIndex`, `limit`, `sortBy`. Las bibliotecas del usuario salen con `GET /UserViews`. Ambas en OpenAPI.
- Vídeo en reproducción directa: `GET /Videos/{itemId}/stream` con `static=true` y `mediaSourceId`. El OpenAPI describe `static` como "the original file will be streamed statically without any encoding". También existe `/Videos/{itemId}/stream.{container}`. En OpenAPI.
- Lista HLS maestra: `GET /Videos/{itemId}/master.m3u8` (también `HEAD`). Hay además `Videos/{itemId}/main.m3u8` y los segmentos `Videos/{itemId}/hls1/{playlistId}/{segmentId}.{container}`. NO aparecen en el OpenAPI estable: el controlador `DynamicHlsController` lleva `[ApiExplorerSettings(IgnoreApi = true)]`. Los parámetros obligatorios de `master.m3u8` no los he comprobado.
- Carátula: `GET /Items/{itemId}/Images/{imageType}` (por ejemplo `Primary`), con `maxWidth`, `maxHeight`, `quality`, `format`, `tag`. En OpenAPI.
- Autenticación (OpenAPI): esquema `apiKey` en la cabecera `Authorization`.
- Autenticación (código fuente, `AuthorizationContext.cs`):
  - La cabecera debe empezar por `MediaBrowser ` (no distingue mayúsculas). Los campos que lee son `Token`, `Client`, `Device`, `DeviceId` y `Version`.
  - Si el token no viene en la cabecera, lo busca en el parámetro de consulta `ApiKey`.
  - El parámetro `api_key`, y las cabeceras `X-Emby-Token`, `X-MediaBrowser-Token` y `X-Emby-Authorization`, solo se aceptan si `EnableLegacyAuthorization` está activo. Por defecto el manual de configuración de Mintlify (no oficial) lo muestra en `false`. No lo confirmé en una fuente oficial.
- Una etiqueta `<video>` no puede mandar cabeceras. Eso apunta a usar `ApiKey` en la URL (afirmación mía, no de la fuente).

**Fuentes abiertas**
- https://api.jellyfin.org/openapi/jellyfin-openapi-stable.json
- https://raw.githubusercontent.com/jellyfin/jellyfin/master/Jellyfin.Api/Controllers/DynamicHlsController.cs
- https://raw.githubusercontent.com/jellyfin/jellyfin/master/Jellyfin.Server.Implementations/Security/AuthorizationContext.cs
- https://api.jellyfin.org/ (intentada, sin texto)

### 4. CORS en Jellyfin

**Estado:** comprobado en parte.

**Hallazgos**
- Existe un ajuste de orígenes permitidos: `CorsHosts` en la configuración del servidor. Por defecto es `["*"]`.
- Comportamiento (código `CorsPolicyProvider.cs`): si la lista está vacía o solo contiene `*`, permite cualquier origen. Si no, permite solo esos orígenes y activa `AllowCredentials`. Siempre permite cualquier método y cualquier cabecera.
- La página de documentación `https://jellyfin.org/docs/general/networking/cors` devuelve 404. La página de redes (`.../post-install/networking/`) que sí abrí no menciona CORS.
- Panel de administración: no lo abrí (no tengo servidor ni cuenta). Una búsqueda en el código de jellyfin-web con `gh search code CorsHosts` no devolvió resultados. No consta que haya un campo en el panel. Lo que está documentado es el fichero de configuración del servidor.
- Efecto en HLS: `AddCors()` y el proveedor de políticas se registran para toda la API. No comprobé que el middleware se aplique a las rutas HLS ni medí las cabeceras de una respuesta real. No consta.

**Fuentes abiertas**
- https://raw.githubusercontent.com/jellyfin/jellyfin/master/Jellyfin.Server/Configuration/CorsPolicyProvider.cs
- https://raw.githubusercontent.com/jellyfin/jellyfin/master/MediaBrowser.Model/Configuration/ServerConfiguration.cs
- https://raw.githubusercontent.com/jellyfin/jellyfin/master/Jellyfin.Server/Extensions/ApiServiceCollectionExtensions.cs
- https://jellyfin.org/docs/general/post-install/networking/
- https://jellyfin.org/docs/general/networking/cors (404)

### 5. BVHEcctrl y ecctrl

**Estado:** comprobado.

**Hallazgos**
- BVHEcctrl: licencia MIT, creado el 2025-04-01. Última versión en npm: 0.0.18, del 2025-08-14. GitHub no tiene releases ni tags. Último push: 2025-08-14.
- Primera persona en BVHEcctrl: el README no trae una propiedad ni un ejemplo de primera persona. Dice que se puede usar cualquier control de cámara (`OrbitControls`, `PointerLockControls`, `CameraControls`) y propone `CameraControls` siguiendo `ecctrlRef.current.group.position`.
- Escaleras en BVHEcctrl: cápsula flotante "for stairs, slopes, and uneven surfaces". Propiedades relacionadas: `floatHeight` 0,2, `maxSlope` 1 rad, `floatCheckType` `BOTH`. No hay propiedad específica de altura de escalón.
- BVHEcctrl no necesita motor de físicas y usa three-mesh-bvh. Colisionadores: `StaticCollider`, `KinematicCollider`, `InstancedStaticCollider`.
- Incidencias abiertas en BVHEcctrl (5): #7 colisionador personalizado y modo vehículo, #6 "Performance issue debug", #5 `transformMatrix` no aplicada en glb cuantizado, #3 actualizar `StaticCollider`, #1 animaciones. La #5 puede afectar si los modelos van cuantizados. La #3 afecta si el catálogo cambia en caliente.
- ecctrl: licencia MIT. Última versión en npm: 2.0.2, del 2026-09-06. Tags en GitHub: v2.0.2 y v2.0.1, sin releases. Necesita `@react-three/rapier` (motor de físicas Rapier).
- Primera persona en ecctrl: el README de la versión actual no la menciona. La incidencia abierta #63 "First person camera controls are faulty" (2024-10-04) sigue abierta. No leí su contenido ni si es de una versión anterior.
- Escaleras en ecctrl: el README habla de personaje ShapeCast flotante con manejo de pendientes y huecos. No menciona escaleras de forma explícita en la parte que leí. El medio del README (zona truncada) no lo leí entero.
- Otras incidencias de ecctrl abiertas: son 40 listadas, entre ellas #139 "Sloped map constant sliding and no jumping" y #148 "Rapier Sleeping Body wake up code misses 1 frame of gravity" (2026-11-18).

**Fuentes abiertas**
- https://github.com/pmndrs/BVHEcctrl
- https://github.com/pmndrs/ecctrl
- https://registry.npmjs.org/bvhecctrl
- https://registry.npmjs.org/ecctrl
- Incidencias consultadas con `gh issue list` sobre ambos repositorios.

### 6. Demos en vivo

**Estado:** no comprobado (solo carga inicial y texto de las pantallas).

**Qué pasó**
- Probé en el navegador de la nube y en el navegador local. En ambos, `document.visibilityState` era `hidden`, no había foco y el bucle de animación no corría (0 a 1 fps en el panel de MyShelves).
- MyShelves mostró el error "The browser refused to lock the pointer". Sin bloqueo de puntero no se puede entrar en el modo de primera persona.
- No caminé por ninguna demo ni medí fluidez durante dos minutos. No hay nota de tiempo de carga fiable. Hay que repetirlo en el navegador del usuario, con la pestaña en primer plano.

**Lo que sí vi**
- **MyShelves** (https://myshelves.eu). El menú carga en unos segundos. Ofrece "Enter the library", "Lounges", "Collection", "Guide", "Controls", "Settings" y "Sign in". Controles de la pantalla "Controls": W A S D para andar, ratón para mirar, Shift correr, Ctrl agacharse, Espacio saltar, Esc pausa, C abrir el libro, rueda o clic izquierdo para pasar página, E para interactuar o coger y dejar un libro, Q/R/F girar el libro, clic derecho girarlo libremente, rueda para zoom, clic izquierdo para lanzarlo, G soltar, N escribir una nota. Modo construcción con B (inventario I, tienda de muebles O). Tiene un panel de depuración (F3) con fps, draws y tris. La GPU que informó el navegador local fue Intel UHD Graphics y declaraba `navigator.gpu`. Escaleras y varios niveles: no consta. Copiaría el panel de estadísticas, la pantalla de controles y el gesto de coger, girar y leer el libro. No sé si evitar algo concreto sin haberla usado.
- **LinkWalk** (https://notbigmuzzy.github.io/linkwalk/). Museo virtual con datos de Wikipedia. La pantalla inicial pide "Click to play". Controles: WASD o flechas para moverse, ratón para mirar, Espacio saltar, Shift correr, Escape libera el ratón. Tiene selector de 14 idiomas. Hay un canvas WebGL en la página. Repositorio público en GitHub con licencia GPL-2.0 (la licencia limita copiar código a un proyecto no GPL). Escaleras: no consta. Cómo se ven y abren los objetos: no comprobado.
- **Babel Nexus** (https://babelnexus.com/n/mathematics). La página carga con "823 Classics", con niveles ("Level I — The Sciences", "Descend to Level II", "Bookstore and Book Apps on Level IV"), pestañas "3D Gallery" y "Book Page", fichas de libro con fuente (Internet Archive), fecha, verificación y obras conectadas. Había un aviso de analíticas opcionales que no acepté ni rechacé. No encontró ningún canvas en el momento de la carga (`canvas: 0`), y la página mostraba la pestaña "Book Page". No abrí la galería 3D. Los niveles son al menos un concepto de navegación. No consta cómo se sube entre ellos dentro del 3D.

**Capturas**
- `docs/investigacion/capturas/myshelves.png`
- `docs/investigacion/capturas/linkwalk.png`
- `docs/investigacion/capturas/babelnexus.png`
- Son capturas de la pantalla inicial de cada demo, no de la escena 3D.

**Fuentes abiertas**
- https://myshelves.eu
- https://notbigmuzzy.github.io/linkwalk/
- https://babelnexus.com/n/mathematics
- https://github.com/notbigmuzzy/linkwalk (licencia GPL-2.0)

## Impacto en el plan

| Punto | Cambia la recomendación (sí o no) | Qué habría que ajustar |
|---|---|---|
| 1. Aura Stacks | No | Sirve como idea (PDF en estantería con lector aparte). No es referencia técnica: es un proyecto de estudiante, sin repo ni licencia visibles. |
| 2. WebGPURenderer | No | Mantener WebGLRenderer. Usar `InstancedMesh` para libros repetidos. No usar `ShaderMaterial` si se quiere migrar luego. Medir en el hardware propio. |
| 3. API de Jellyfin | Sí, en parte | No fiarse solo del OpenAPI: el HLS maestro no sale ahí. Autenticar con `ApiKey` en la URL o con cabecera `MediaBrowser`. No usar `api_key`. Comprobar los parámetros de `master.m3u8` contra un servidor real. |
| 4. CORS en Jellyfin | No | Por defecto permite todo (`*`). Si el amigo lo restringe, habrá que añadir el origen de la biblioteca en `CorsHosts`. Probar HLS con un servidor real. |
| 5. BVHEcctrl / ecctrl | Sí, en parte | BVHEcctrl no trae primera persona: hay que montarla con `PointerLockControls` y mover la cámara a mano. Escaleras: sí, con cápsula flotante, pero hay que probar con la geometría real. Último release de BVHEcctrl: agosto de 2025, sin releases en GitHub. |
| 6. Demos en vivo | Sin datos | Repetir en el navegador del usuario. Hasta entonces no hay base para copiar o evitar nada concreto. |

## Actualización: pruebas en vivo (punto 6) y puntos cerrados

Método: navegador local de sevas (Edge), ventana de 774x498 salvo Babel Nexus (1280x800). El navegador automatizado no puede bloquear el puntero, así que se activó el foco por CDP y se simuló `requestPointerLock`; las teclas y el movimiento del ratón se enviaron por CDP. Capturas dentro de la escena en `docs/investigacion/capturas/`. Es una prueba corta de cada demo, no un recorrido completo.

### MyShelves (myshelves.eu)
- **Controles (de su pantalla "Controls"):** WASD caminar, ratón mirar, Shift correr, Ctrl agacharse, Espacio saltar, Esc pausa, C abrir libro, E interactuar o coger/colocar libro, Q/R/F girar el libro, clic izquierdo lanzarlo, G soltarlo, N escribir nota, B modo construcción, I inventario, O tienda de muebles.
- **Qué vi:** una sala única con parquet, alfombra, estantería vacía, atril de pedidos y un pedestal. Primera persona. Iluminación cálida con focos, sin escaleras ni segundo nivel en la sala inicial. Los libros llegan al pedir un título real en el atril (búsqueda) o pintar uno propio; se amplía la sala con el modo construcción.
- **Rendimiento:** unos 115-128 fps (pico con 91 draws y 9 mil triángulos), con caídas puntuales (53 fps y un frame de 152 ms al girar y cargar objetos). Trae un panel de depuración (F3) con fps, draws y triángulos. Con 0 libros no dice nada sobre escalado.
- **No probado:** pedir un libro (necesita su búsqueda externa y cuenta), modo construcción, el modo "Lounges".
- **Qué copiar:** el panel F3 de métricas, el modo construcción, el tutorial en 6 pasos y que cada libro sea un objeto manipulable. **Qué evitar:** una sala vacía al inicio no enseña nada.

### LinkWalk (notbigmuzzy.github.io/linkwalk)
- **Controles:** WASD o flechas, ratón, Espacio saltar, Shift correr, Esc libera el puntero. Selector de 14 idiomas y un clic para empezar.
- **Qué vi:** un vestíbulo con **dos escaleras con barandilla que suben a un piso superior** con puertas rotuladas por tema (Philosophy, Cosmology, Society, Culture, Geography...), un panel central con el índice y macetas. Estilo low-poly sencillo. Primera persona.
- **Colisiones:** subí por la escalera hasta el rellano. Me quedé atascado contra la barandilla al intentar llegar a una puerta, así que **no entré en ninguna sala**; eso lo achaco a mi navegación a ciegas, no a la demo.
- **Fluidez:** la escena fue fluida y sin cargas visibles; no medí fps (no trae contador).
- **Qué copiar:** vestíbulo con escaleras dobles y puertas por categoría, rótulos legibles de lejos. Licencia GPL-2.0: copiar código obliga a publicar el nuestro con la misma licencia.

### Babel Nexus (babelnexus.com/n/mathematics)
- **No es primera persona.** Es una vista **en tercera persona** con un avatar en una **sala hexagonal** con un minimapa ("Room overview"). Cada libro es un panel enmarcado con una tarjeta de título/autor/fecha; las puertas llevan a materias relacionadas y hay un disco para "Descend to Level II" (los niveles son temáticos: Level I, The Sciences).
- **Controles (de su panel de ayuda):** WASD o flechas, clic en el suelo para ir, Q/E girar cámara, R/F altura de cámara, rueda para cambiar de estantería, clic en un libro para abrir sus detalles. Clic en una puerta o escalera para cambiar de galería. Hay un modo de lectura en lista ("Book Page") y navegación solo con teclado, sin cuenta.
- **Qué copiar:** el modo dual (3D opcional + lista accesible), el minimapa, las tarjetas de datos junto a cada libro y los niveles por materia. **Qué evitar:** su peso web (todo el sitio alrededor de la escena) y que el 3D ocupe poco espacio.
- **Rendimiento:** no medido. El avatar se movió con W; la interacción con E no abrió nada (el libro se abre con clic).

### Puntos que se cierran
- **Aura Stacks (1):** la página de Devpost enlaza a una demo pública: `https://aura-stacks.lovable.app/`. No enlaza a repositorio.
- **Jellyfin HLS (3):** `GET /Videos/{itemId}/master.m3u8` exige `itemId` en la ruta y `mediaSourceId` en la query; el resto de parámetros (`playSessionId`, `segmentContainer`, `segmentLength`, `audioCodec`, etc.) son opcionales. Fuente: `DynamicHlsController.cs` en jellyfin/jellyfin (master).
- **ecctrl (5):** la incidencia #63 es de `pmndrs/ecctrl` (no de BVHEcctrl), sigue abierta y se titula "First person camera controls are faulty": al usar primera persona con `camInitDis`/`camMinDis` en -0.01 y `CameraBasedMovement`, la cámara y el personaje giran solos. El mantenedor pidió un vídeo y no hubo respuesta. Refuerza que la primera persona hay que montarla aparte.
- El README de ecctrl no menciona escaleras (búsqueda de "stair", "step", "slope": sin resultados).

## Lo que sigue sin comprobar

- Parámetros de `master.m3u8` más allá de los obligatorios, y si responde con `ApiKey` en segmentos y listas hijas.
- Panel de administración de Jellyfin (CORS) y cabeceras CORS reales de las respuestas HLS: necesita un servidor Jellyfin.
- Si `EnableLegacyAuthorization` viene en `false` por defecto, desde una fuente oficial.
- Entrar en una sala de LinkWalk, pedir un libro en MyShelves y rendimiento de Babel Nexus.
- La lectura completa de las notas de versión r187 de three.js.
- El extra (vídeos de YouTube y artículos académicos): no se hizo.
