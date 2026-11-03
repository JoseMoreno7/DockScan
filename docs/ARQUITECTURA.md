# DockScan — Arquitectura y base técnica del proyecto

> **Nombre confirmado (2026-11-02):** `DockScan`. El paquete `dev.morenojos.dockscan` sigue provisional: confirmarlo **antes de crear el proyecto Android**, porque cambiarlo después de publicar es muy caro.
>
> **Qué es este documento:** la referencia **estable** del proyecto (qué es, con qué se construye, cómo se organiza y bajo qué reglas se trabaja). Cambia poco. El avance, el roadmap y las decisiones del día a día van en `PROGRESO.md`, que se actualiza seguido.
>
> **Cómo usarlo:** pégalo completo al abrir una sesión nueva con cualquier asistente (Claude u otro) junto con `PROGRESO.md`, para retomar sin perder el hilo.

---

## 1. Qué es el producto

Una app móvil Android para **recepcionar equipos comprados y registrar cambios de equipos** escaneando códigos, en vez de verificar y anotar a mano. Nace de un proceso real de un almacén de TI: la mercadería llega con una guía o factura, los asistentes la comparan uno por uno visualmente con lapicero, y los cambios de equipo se registran después en un formulario de ocho campos escritos a mano.

Se resuelven dos casos de uso, y solo esos:

### 1.1 Recepción de compra
1. Se carga una **guía de compra** con los equipos esperados (series o cantidades por modelo).
2. Al bajar la mercadería, cada equipo se **escanea** (código de barras o número de serie por OCR, con el celular; en el futuro también con una pistola lectora).
3. La app responde **al instante** con un color, igual que los controles de acceso de exámenes masivos:

| Color | Estado | Significado |
|---|---|---|
| 🟢 Verde | `RECEIVED` | Primera vez que se escanea y pertenece a la guía. Se da por recibido. |
| 🟡 Amarillo | `DUPLICATE` | Ya se escaneó antes en esta recepción. |
| 🔴 Rojo | `NOT_IN_ORDER` | No pertenece a esta compra (serie o modelo que no está en la guía). |
| ⚪ Gris | `PENDING` | Está en la guía pero aún no se escaneó. Es la lista de "falta por recibir". |

4. Al terminar, la recepción muestra el cuadre: recibidos, faltantes, duplicados y no pertenecientes. Los equipos recibidos pasan al inventario.

### 1.2 Cambio de equipo (y repotenciación, y alta)
Se escanea el equipo que **sale** y el que **entra**. La app rellena marca, modelo, serie y código. El usuario solo agrega ubicación, fotos y fecha. Todo se guarda como **un único movimiento** con trazabilidad de ambos equipos.

Campos que el proceso manual exigía y cómo los resuelve la app:

| Campo del proceso manual | En la app |
|---|---|
| Correo | Automático (usuario autenticado) |
| Persona de soporte | Automático (usuario autenticado) |
| Tipo de operación (cambio / repotenciación / alta) | Selector |
| Dispositivo (categoría) | Se deduce del equipo escaneado; editable |
| Fecha | Hoy por defecto |
| Marca, modelo, serie, código (equipo anterior) | **Escaneo** |
| Marca, modelo, serie, código (equipo nuevo) | **Escaneo** |
| Bloque, piso, ambiente | Ubicación (selector jerárquico) |
| Usuario | Ver regla de negocio 1.3 |
| Fotos antes / después (hasta 5, 10 MB c/u) | Cámara integrada; se suben a almacenamiento |

### 1.3 Reglas de negocio ya confirmadas
- **Usuario del equipo:** se registra `autonoma` (valor genérico) cuando el equipo opera por DHCP, como aulas y PCs compartidas de docentes. Cuando el usuario tiene cuenta e IP propias (administrativos), se registra el usuario real al que se atiende.
- **El estado de un escaneo lo decide la base de datos, no el celular** (ver sección 5.2).
- **Una guía puede traer series por equipo o solo cantidades por modelo.** El modelo de datos soporta ambas.
- **Varios celulares pueden escanear la misma recepción a la vez**; lo que uno escanea se ve en los demás en tiempo real.

### 1.4 Fuera de alcance (por ahora)
Kardex completo con todos los tipos de movimiento, roles y permisos finos, indicadores/Power BI, importadores de otros sistemas, equipos a granel sin serie. Quedan como visión futura; no se diseña para ellos más allá de no cerrarles la puerta.

---

## 2. Stack tecnológico

### 2.1 Cliente (Android)
| Pieza | Elección | Nota |
|---|---|---|
| Lenguaje | Kotlin 2.2.10 | |
| UI | Jetpack Compose (BOM 2026.02.01) + Material 3 | |
| Arquitectura | MVVM + flujo unidireccional (`ViewModel` + `StateFlow`) | |
| Inyección de dependencias | **Hilt** (con KSP) desde el inicio | En el proyecto anterior se dejó para después; aquí va desde el día uno. |
| Navegación | Navigation Compose 2.9.8 | |
| Cámara | CameraX 1.4.2 | `ImageAnalysis` a 1920×1080 junto a `Preview` |
| Lectura de códigos | ML Kit Barcode Scanning 17.3.0 | |
| OCR | ML Kit Text Recognition **16.0.1** (variante *bundled*, funciona offline) | ⚠️ La 19.0.1 es de otro artefacto (`play-services-mlkit-text-recognition`); mezclarlas rompe el build. |
| Permisos | Accompanist Permissions 0.37.3 | |
| Backend SDK | supabase-kt 3.2.2 (`auth-kt`, `postgrest-kt`, `realtime-kt`, `storage-kt`, `functions-kt`) + Ktor 3.2.2 (OkHttp) | Verificar que la versión de Ktor coincida con la que usa supabase-kt. |
| Persistencia local | Room (fase posterior: cola offline de escaneos) | |
| Build | Gradle (Kotlin DSL) + version catalog `gradle/libs.versions.toml`; AGP 9.4.1; compileSdk/targetSdk 37; minSdk 26 | |
| Pruebas | JUnit, MockK, Turbine (flujos), Compose UI tests | |

### 2.2 Backend (Supabase, cuenta personal)
| Pieza | Uso |
|---|---|
| **Postgres** | Fuente de verdad. Esquema versionado con migraciones SQL. |
| **Auth** | Google (Credential Manager en Android) y correo/contraseña. |
| **Realtime** | Difundir escaneos entre celulares de una misma recepción. |
| **Storage** | Fotos de "antes/después" de los cambios y PDFs de guías. |
| **RLS (Row Level Security)** | Aislamiento por organización desde la primera tabla. |
| **Funciones SQL (RPC)** | Registro atómico de escaneos (ver 5.2). |
| **Edge Functions** (Deno/TypeScript) | Extracción de datos de PDF de guías (fase posterior), nunca con claves en el cliente. |

### 2.3 Herramientas de trabajo
- **Git + GitHub**, con la CLI `gh` para todo lo que se pueda (PR, merge, issues).
- **Supabase CLI** para migraciones, semillas y despliegue de funciones (todo por terminal, sin tocar el panel web salvo para consultar).
- **GitHub Actions** para CI: compilar, lint y pruebas unitarias en cada PR.
- **PowerShell** como única terminal (nunca `cmd.exe`).

---

## 3. Estructura del repositorio

Un solo repositorio (monorepo) con el cliente, el backend y la documentación. Un solo módulo Gradle al inicio: se modulariza solo si el tamaño lo pide.

```
dockscan/
├─ app/                                  # Aplicación Android
│  └─ src/
│     ├─ main/java/dev/morenojos/dockscan/
│     │  ├─ DockScanApp.kt               # Application + Hilt
│     │  ├─ MainActivity.kt
│     │  ├─ core/                        # Transversal
│     │  │  ├─ di/                       # Módulos Hilt
│     │  │  ├─ network/                  # Cliente Supabase
│     │  │  └─ util/                     # Normalización, formato, fechas
│     │  ├─ domain/                      # Lógica de negocio sin Android
│     │  │  ├─ model/                    # Asset, Reception, ScanResult...
│     │  │  └─ scan/                     # Reglas de estado de escaneo
│     │  ├─ data/
│     │  │  ├─ remote/                   # Acceso a Supabase (Postgrest, Realtime, Storage)
│     │  │  ├─ local/                    # Room (fase offline)
│     │  │  └─ repository/               # Interfaces + implementaciones
│     │  ├─ feature/                     # Una carpeta por funcionalidad
│     │  │  ├─ auth/
│     │  │  ├─ scanner/                  # Cámara, barcode, OCR, throttle, anti-rebote
│     │  │  ├─ reception/                # Lista de guía, colores, cuadre
│     │  │  ├─ change/                   # Cambio / repotenciación / alta
│     │  │  └─ assets/                   # Búsqueda y detalle de equipos
│     │  └─ ui/
│     │     ├─ theme/
│     │     ├─ components/               # Tarjetas, badges de color, bottom sheets
│     │     └─ navigation/
│     ├─ test/                           # Pruebas unitarias
│     └─ androidTest/                    # Pruebas instrumentadas / UI
├─ supabase/
│  ├─ config.toml
│  ├─ migrations/                        # SQL versionado (esquema, RLS, RPC)
│  ├─ functions/                         # Edge Functions (TypeScript)
│  └─ seed.sql                           # Datos de prueba INVENTADOS
├─ sample-data/                          # CSV/Excel de guías de ejemplo (sintéticos)
├─ docs/
│  ├─ ARQUITECTURA.md                    # Este documento
│  ├─ PROGRESO.md                        # Estado y roadmap (cambia seguido)
│  └─ decisiones/                        # Registros de decisiones (ADR), uno por archivo
├─ .github/workflows/                    # CI
├─ gradle/libs.versions.toml
├─ build.gradle.kts / settings.gradle.kts
└─ README.md                             # Presentación pública del proyecto
```

**Regla de dependencias entre capas:** `feature` → `domain` ← `data`. El `domain` no conoce Android ni Supabase. Las pantallas y los `ViewModel` solo hablan con interfaces de repositorio; no saben de dónde vienen los datos.

---

## 4. Modelo de dominio (borrador, se detalla en un paso aparte)

Entidades principales, sin campos finales:

- **Organization** y **Membership**: aislamiento por organización (`organization_id` en todas las tablas) y rol del usuario dentro de ella.
- **Location**: ubicación jerárquica (bloque → piso → ambiente).
- **Asset**: equipo con **identificador interno propio (UUID)**. Serie, código patrimonial y otros códigos son atributos opcionales; un equipo debe poder existir sin código.
- **PurchaseOrder** y **PurchaseOrderLine**: la guía/factura y sus líneas (serie prefijada o cantidad por modelo).
- **Reception**: una sesión de recepción asociada a una guía.
- **ScanEvent**: cada escaneo, **inmutable** (quién, cuándo, qué código, resultado). Nunca se edita ni se borra.
- **AssetMovement**: cambio, repotenciación o alta; referencia al equipo que sale y al que entra, ubicación, usuario atendido y fotos.
- **Attachment**: fotos y archivos en Storage.

Normalización de identificadores (heredada del proyecto anterior): código solo con dígitos y relleno a 8 con ceros cuando es numérico; serie en mayúsculas, sin espacios, y `S/N` o vacío como nulo.

---

## 5. Decisiones técnicas clave

### 5.1 Escáner (se reutiliza lo ya validado)
- **Barcode en cada frame y OCR concurrente, limitado a un intento por segundo desde el primer frame.** No se espera a que falle el barcode. Si el barcode se lee en el mismo frame, se ignora el OCR.
- **Resolución de análisis 1920×1080.** La predeterminada de CameraX (640×480) hace que el OCR confunda dígitos. No bajar.
- **Filtros por texto, evaluados por palabra suelta:** código numérico de 8 dígitos o serie alfanumérica de 5–20 caracteres con al menos una letra y un dígito. Evaluar por palabra evita falsos positivos como "INVENTARIO 2025". Si hay varios candidatos en cuadro, se elige el más cercano al centro.
- Anti-rebote de 3 s por código, vibración al detectar, linterna (solo si hay flash) y botón para forzar el siguiente intento de OCR.
- **Cola de procesamiento de un solo consumidor:** un escaneo a la vez, en orden, ninguno se descarta.

### 5.2 El estado de un escaneo lo decide la base de datos
El celular **nunca** decide si un escaneo es verde, amarillo o rojo. Llama a una función SQL (RPC) que, dentro de una transacción, hace tres cosas:
1. Comprueba si el código pertenece a la guía → si no, `NOT_IN_ORDER`.
2. Intenta marcarlo como recibido apoyándose en una **restricción única** `(reception_id, código normalizado)` → si ya existía, `DUPLICATE`.
3. Si todo va bien → `RECEIVED`.

Con esto, si dos celulares escanean la misma serie al mismo tiempo, exactamente uno recibe verde y el otro amarillo, sin condiciones de carrera. Realtime reenvía el resultado a los demás celulares para refrescar la lista.

### 5.3 Carga de la guía
- **Fase 1:** importar Excel o CSV (columnas definidas en `sample-data/`).
- **Fase 2:** subir PDF o foto de la orden de compra y extraer las líneas con una Edge Function. Como la extracción de tablas es probabilística, **siempre hay una pantalla de revisión y corrección antes de guardar**.

### 5.4 Offline (fase posterior)
Un camión descargando puede estar en un lugar sin buena señal. Los escaneos se guardan primero en Room con una marca de "pendiente de sincronizar" y se envían cuando hay red. Mientras tanto la app muestra el estado provisional con un indicador visible de que no está confirmado.

### 5.5 Seguridad
- **Cero secretos en el repositorio.** URLs y claves públicas (anon key) en `local.properties` → `buildConfigField`. La `service_role` jamás sale del servidor.
- **RLS activada en todas las tablas**, con políticas por `organization_id`. Una tabla sin política no se publica.
- Las funciones sensibles se ejecutan en el servidor; el cliente solo tiene la clave anónima.
- **El repositorio es público (es parte del portafolio).** Por eso, ningún dato real ni nombre institucional entra al código, a la base de datos de prueba ni a la documentación pública.

---

## 6. Convenciones de trabajo (para personas y asistentes)

**Ritmo.** Se trabaja **un solo paso por mensaje**: se entrega un paso, se pega el resultado y recién entonces se da el siguiente. Antes de entregar bloques grandes de código se confirma el enfoque. Si un cambio toca muchas partes de un archivo, se entrega el **archivo completo**, no ediciones por número de línea. El objetivo es **entender y aprender**: toda decisión se acompaña del porqué.

**Git.**
- Ramas `feat/...`, `fix/...`, `docs/...`, `chore/...` desde `main` actualizada.
- **Conventional Commits.** Un commit por avance con sentido.
- Se agregan los archivos **por nombre**; nunca `git add .`. No se commitea `.idea/misc.xml`.
- Flujo: rama → `git push -u origin` → `gh pr create` → `gh pr merge N --merge --delete-branch`.
- `main` protegida: solo por PR, sin push directo ni force-push.
- Atribución: solo la línea `Co-Authored-By`, sin enlaces de sesión ni pies de "generado con".

**Datos de prueba.** Siempre sintéticos e inventados (marcas, series, personas, ubicaciones). Nunca datos de una institución real.

**Calidad antes de seguir.** Cada paso compila y se prueba antes del siguiente. Cuando algo falla se explica **por qué**, y si es un problema conocido de una herramienta, se dice.

**Terminal.** PowerShell 5.1:
- `Set-Content -Encoding utf8` añade un BOM invisible y rompe `libs.versions.toml`. Editar esos archivos desde Android Studio o con `[System.IO.File]::WriteAllText($ruta, $texto, (New-Object System.Text.UTF8Encoding($false)))`.
- `git diff` con paginador se queda en `:` → salir con `q` o usar `git --no-pager diff`.
- Un `BUILD SUCCESSFUL` con muchas tareas `up-to-date` no es un error: el IDE ya compiló ese código.

---

## 7. Lecciones heredadas del proyecto anterior

- La resolución por defecto de CameraX es demasiado baja para leer dígitos finos.
- El barcode es el camino fiable; el OCR es probabilístico y cambia entre frames.
- Filtrar el OCR **por línea completa** es demasiado estricto o demasiado permisivo según el patrón; **por palabra suelta** funciona.
- Las etiquetas del almacén están pegadas unas junto a otras: elegir el candidato más cercano al centro evita leer la etiqueta vecina.
- Una lista horizontal con columnas largas se trunca mal en el celular; **tarjetas compactas con detalle en un bottom sheet** funcionan mejor.
- Los nombres largos se muestran completos reduciendo el tamaño de letra (`TextAutoSize`), no cortándolos con "...".
- Diseñar el acceso a datos detrás de interfaces de repositorio permitió cambiar la fuente de datos sin tocar la interfaz de usuario. Se mantiene aquí desde el inicio.

---

## 8. Decisiones abiertas (cerrar antes de escribir código)

1. **Nombre definitivo de la app y del paquete.** Hoy provisionales.
2. **Cuenta de GitHub y visibilidad del repositorio** (se asume pública, con la regla de cero secretos y cero datos reales).
3. **Nuevo proyecto de Supabase** en la cuenta personal (región y nombre).
4. **Cuenta de Google Cloud personal** para el inicio de sesión con Google (nuevos clientes OAuth: Web y Android).
5. **Equipo de pruebas:** el celular físico para probar el escáner con etiquetas reales (barcode y serie).
