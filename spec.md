# weActa — Especificación técnica

> Versión 0.1 · 2026-10-18 · Estado: **borrador para aprobación**
> Nombre provisional. Código e identificadores en inglés; UI en español (es-CO).
> Este documento es la fuente de verdad del producto. `AGENTS.md` define cómo trabajamos; `docs/slices/` define cada entrega.

---

## 0. Cómo leer este documento

- **DEBE / NO DEBE**: requisito obligatorio. **DEBERÍA**: requisito salvo justificación escrita en un ADR.
- Las secciones marcadas **[Verificar en Slice N]** dependen de una medición o de una API que aún no hemos probado en un iPhone real. No se implementan como definitivas hasta cerrar esa verificación.
- La sección 13 recoge las decisiones tomadas y **en qué nos apartamos del brief original, y por qué**.

---

## 1. Visión y alcance

App nativa de iPhone que graba reuniones presenciales, transcribe en el dispositivo, separa hablantes y genera, al terminar:

1. Acta formal (formato colombiano, sección 9).
2. Resumen ejecutivo (máximo 5 líneas).
3. Conclusiones.
4. Decisiones tomadas.
5. Compromisos: qué, responsable, fecha, prioridad y estado.
6. Hoja de ruta por horizonte (esta semana / 30 días / 90 días), con dependencias.
7. Riesgos y bloqueos.
8. Temas sin resolver y temas para la próxima reunión.
9. Correo de seguimiento listo para enviar.

**Uso inicial:** herramienta interna del autor y su equipo, distribuida por TestFlight (interno). Cada persona tiene sus propios datos en su iPhone; **no hay datos compartidos entre miembros del equipo** en el MVP (no hay backend). Se comparte el acta exportada (PDF, correo).

**Producto futuro:** la arquitectura (módulos y protocolos, sección 6) debe permitir agregar motores, sincronización, importación y un backend proxy sin reescribir el núcleo.

### Fuera de alcance del MVP (Slices 0–4)
- iPad, Mac, Apple Watch (el Watch entra en el Slice 5).
- Reuniones virtuales grabadas desde el sistema (Teams/Meet/Zoom): solo se importan transcripciones en el Slice 5.
- Colaboración en tiempo real sobre un acta.
- Reconocimiento de hablantes entre reuniones (prohibido por diseño, sección 10.3).

---

## 2. Principios, en orden de prioridad

1. **Seguridad y privacidad.** Ante conflicto con cualquier otro principio, gana este.
2. **Confiabilidad: nunca perder audio.** El audio es la fuente de verdad; todo lo demás se puede regenerar desde él.
3. **Calidad del acta.** Ningún dato inventado; todo dato del acta es trazable a la transcripción.
4. **Experiencia de uso.**

Reglas derivadas:
- Ningún contenido de la reunión (audio, transcripción, nombres, acta) sale del iPhone sin **consentimiento explícito previo, por motor y revocable** (sección 10.2). Esto incluye Private Cloud Compute (ver D-04).
- Toda etapa del pipeline va detrás de un protocolo y es **reprocesable**: se puede volver a correr la transcripción, la diarización o la síntesis con otro motor sin perder lo anterior.
- Nada de tamaños, umbrales o límites "inventados": cada número de la configuración tiene un origen documentado (API, medición del Slice 0 o decisión registrada).

---

## 3. Usuarios y escenarios

- **Secretario/a de la reunión** (usuario principal): crea la reunión, lee el aviso de grabación, graba, marca momentos, revisa y emite el acta.
- **Asistentes**: no usan la app; reciben el acta y el correo. Son **titulares de datos** según la Ley 1581.

Escenarios críticos que la app DEBE soportar:
- Reunión de 2–3 h con el iPhone sobre la mesa y la pantalla bloqueada.
- Entra una llamada a mitad de la reunión (contestada o rechazada).
- La app se cierra inesperadamente (crash, falta de memoria, reinicio del teléfono) durante la grabación.
- Poco espacio en disco o batería baja durante la grabación.
- Procesamiento al final con el teléfono bloqueado en el bolsillo.

---

## 4. Requisitos funcionales

### 4.1 Reuniones
- RF-01 Crear reunión con: título, tipo (`committee`, `client`, `followUp`, `handover`, `other` → UI: comité, cliente, seguimiento, empalme, otro), serie opcional, fecha, lugar o medio, asistentes, orden del día.
- RF-02 El número de acta es **consecutivo por serie** y se asigna al pasar a estado `final`; el usuario puede editarlo (ver D-09).
- RF-03 Estados: `draft` → `recording` → `processing` → `inReview` → `final`. Volver de `final` a `inReview` crea una nueva versión del acta (la anterior se conserva).

### 4.2 Aviso y consentimiento (Ley 1581)
- RF-10 Antes de iniciar cada grabación se muestra el aviso de tratamiento con el texto sugerido para leer en voz alta (sección 10.1). La grabación no empieza hasta que el usuario confirma "Leí el aviso".
- RF-11 Se registra la fecha y hora en que se mostró y se confirmó el aviso, la versión del texto y el offset del audio en que empieza la grabación (la lectura en voz alta queda grabada como evidencia).
- RF-12 Los inicios rápidos (Control Center, botón de Acción, Siri; Slice 3) abren la app directamente en la pantalla del aviso. No existe un inicio "a ciegas" sin aviso (ver D-11).

### 4.3 Captura
- RF-20 Grabar en segundo plano y con pantalla bloqueada durante al menos 3 h continuas.
- RF-21 Segmentos de 5 min en un contenedor que sobreviva a un cierre abrupto (ver 7.1 y D-05).
- RF-22 Ante interrupciones y cambios de ruta: pausar, registrar el hueco (inicio, fin, motivo) y reanudar. Si el sistema no permite reanudar en segundo plano, avisar con la Live Activity y una notificación local "Toca para reanudar la grabación". **[Verificar en Slice 0]**
- RF-23 Indicador de nivel de entrada y alerta si hay silencio sostenido (micrófono tapado o ruta equivocada).
- RF-24 Alertas preventivas por espacio en disco y batería, con umbrales derivados de la tasa de bits medida en el Slice 0.
- RF-25 Recuperación al relanzar: si hay una grabación sin cerrar, la app la reconstruye desde el journal y los archivos, y lo informa.

### 4.4 Transcripción
- RF-30 Transcripción final sobre los archivos guardados, en el dispositivo.
- RF-31 Transcripción en vivo **opcional** (apagada por defecto), para verla durante la reunión. Si el Slice 0 demuestra que su calidad es igual a la de la pasada final, el resultado en vivo se reutiliza y no se repite el trabajo. **[Verificar en Slice 0]**
- RF-32 Cada segmento guarda: inicio, fin, texto, etiqueta de hablante opcional, motor y versión.
- RF-33 Reprocesar con otro motor crea una nueva `TranscriptionRun`; el usuario elige cuál es la activa.

### 4.5 Hablantes (Slice 3)
- RF-40 Diarización offline al terminar: "Hablante 1", "Hablante 2"…
- RF-41 En la revisión, el usuario asigna cada etiqueta a un asistente (o la deja sin asignar).
- RF-42 Se puede desactivar la diarización por reunión (por ejemplo, si un asistente se opone).

### 4.6 Marcadores (Slice 3)
- RF-50 Botones Decisión / Compromiso / Riesgo / Nota en la app, la Live Activity y (si aplica) el Control.
- RF-51 El marcador guarda el tiempo en la línea de audio y un texto opcional. Funciona con la pantalla bloqueada (ver 7.1.4).

### 4.7 Acta y salidas (Slice 2)
- RF-60 Generar las 9 salidas de la sección 1 con el pipeline de 7.4.
- RF-61 Cada decisión y compromiso muestra sus fuentes (rangos de tiempo) y permite reproducir el audio desde ahí.
- RF-62 Campos no dichos en la reunión (responsable, fecha) quedan vacíos y marcados como **alerta**; el acta no puede pasar a `final` con alertas sin revisar (el usuario puede aceptarlas explícitamente).
- RF-63 Edición manual de todo el contenido; se marca `editedByUser`.
- RF-64 Exportar a PDF (formato de la sección 9) y compartir; copiar o abrir el correo de seguimiento en Mail.

### 4.8 Seguimiento (Slice 4)
- RF-70 Compromisos a Recordatorios; próxima reunión al Calendario.
- RF-71 Historial y búsqueda.
- RF-72 Compromisos abiertos de la reunión anterior de la misma serie.
- RF-73 Sincronización con iCloud, **opcional y apagada por defecto** (ver D-10).

### 4.9 Extensiones (Slice 5)
- RF-80 Importar transcripciones .vtt, .txt, .docx.
- RF-81 Motor Claude vía backend proxy, con pantalla de consentimiento.
- RF-82 Apple Watch.

---

## 5. Requisitos no funcionales

| Id | Requisito | Origen del valor |
|---|---|---|
| RNF-01 | Pérdida máxima de audio ante crash: ≤ el intervalo de fragmento del contenedor elegido | Slice 0 (E3) |
| RNF-02 | 0 s de audio perdido en una grabación de 3 h sin incidentes | Brief |
| RNF-03 | Consumo de batería en grabación (sin transcripción en vivo) | Medir en Slice 0; meta propuesta ≤ 8 %/h en iPhone 15 Pro o posterior |
| RNF-04 | Tiempo de proceso de 60 min de reunión (transcripción + diarización + acta) | Medir en Slice 0; meta propuesta ≤ 10 min |
| RNF-05 | Sin estado térmico `.serious` durante la grabación | Slice 0 |
| RNF-06 | Accesibilidad: Dynamic Type hasta AX5, VoiceOver completo, modo oscuro, contraste AA, Reduce Motion | Brief |
| RNF-07 | Localización: String Catalog, es-CO como idioma base | Brief |
| RNF-08 | Compila sin warnings en nuestro código; warnings de dependencias se documentan y no se ocultan | Brief + D-14 |

---

## 6. Arquitectura

### 6.1 Plataforma
- iOS 27.0 mínimo, solo iPhone. Swift 6 con concurrencia estricta (`SWIFT_STRICT_CONCURRENCY=complete`, modo de lenguaje Swift 6). SwiftUI + Observation (`@Observable`). SwiftData.
- Proyecto generado con XcodeGen (`project.yml`). Compilación y pruebas con `xcodebuild`. El `.xcodeproj` no se versiona.
- **Las funciones de IA de Apple requieren un iPhone compatible con Apple Intelligence** (iPhone 15 Pro o posterior) aunque iOS 27 corre desde el iPhone 11. En equipos sin Apple Intelligence la app graba, transcribe y diariza, pero no genera el acta con IA (ver D-03).

### 6.2 Módulos

```
WeActa (app, SwiftUI)                  ← solo UI, navegación, composición de dependencias
WeActaWidgets (extensión)              ← Live Activity, Control (Slice 3)
WeActaSpike (app, solo Slice 0)        ← experimentos; se elimina o archiva al cerrar el Slice 0
Packages/ActaKit (paquete SPM local)
 ├─ ActaDomain        Foundation pura: tipos de valor, protocolos, chunking, presupuesto de tokens,
 │                    validación anti-invención, WER/DER, carga de prompts. Sin frameworks de Apple
 │                    de UI ni de IA → sus pruebas corren en macOS y en Linux.
 ├─ ActaPersistence   Esquema SwiftData versionado, migraciones, repositorios.
 ├─ ActaCapture       AVAudioSession/AVAudioEngine, segmentos, journal, recuperación.
 ├─ ActaTranscription Adaptadores: SpeechTranscriber (+ WhisperKit / Parakeet solo si ganan en el Slice 0).
 ├─ ActaDiarization   Adaptador FluidAudio.
 ├─ ActaMinutes       Pipeline map→reduce→redacción sobre Foundation Models.
 └─ ActaExport        PDF, correo, Recordatorios, Calendario.
```

Regla: las dependencias externas (WhisperKit, FluidAudio) solo se importan en su módulo adaptador. Ningún otro módulo las ve.

### 6.3 Protocolos (borrador; los nombres finales se ajustan al implementar)

```swift
// ActaDomain
public protocol AudioCaptureService: Sendable {
    var events: AsyncStream<CaptureEvent> { get }            // nivel, segmento cerrado, hueco, error
    func start(meetingID: UUID) async throws
    func addMarker(_ kind: MarkerKind, note: String?) async   // escribe en el journal
    func stop() async throws -> RecordingManifest
}

public protocol TranscriptionEngine: Sendable {
    var descriptor: EngineDescriptor { get }                  // id, versión, local/remoto
    func availability(for locale: Locale) async -> EngineAvailability
    func prepare(locale: Locale) -> AsyncThrowingStream<PreparationProgress, Error>   // descarga de modelos
    func transcribe(_ audio: AudioSource) -> AsyncThrowingStream<TranscriptPiece, Error>
}

public protocol LiveTranscriptionEngine: TranscriptionEngine {
    func startLive(format: AudioFormatDescriptor) async throws -> LiveTranscriptionSession
}

public protocol DiarizationEngine: Sendable {
    var descriptor: EngineDescriptor { get }
    func diarize(_ audio: AudioSource) async throws -> [SpeakerTurn]   // nunca persiste embeddings
}

public protocol TokenBudgeting: Sendable {
    func contextSize() async throws -> Int
    func tokenCount(_ text: String) async throws -> Int
}

public protocol MinutesEngine: Sendable {
    var descriptor: EngineDescriptor { get }                  // incluye si el contenido sale del iPhone
    func generate(_ input: MinutesInput) -> AsyncThrowingStream<MinutesProgress, Error>
}
```

Nota: en iOS 27, Foundation Models abrió su capa de modelos (`LanguageModel`), de modo que `SystemLanguageModel`, `PrivateCloudComputeLanguageModel` y proveedores externos comparten `LanguageModelSession`. `ActaMinutes` implementa `MinutesEngine` una sola vez sobre esa abstracción y recibe el modelo inyectado. Mantenemos nuestro protocolo propio para no acoplar el resto de la app a Foundation Models.

### 6.4 Concurrencia
- Servicios con estado = `actor`. Modelos de vista = `@MainActor @Observable`.
- El callback del tap de `AVAudioEngine` corre en un hilo de tiempo real: no se usa `await`, no se asigna memoria innecesaria y no se toman locks largos; se copia el buffer a una cola sin bloqueo y otro contexto escribe a disco.
- Prohibido `@unchecked Sendable` y `nonisolated(unsafe)` salvo ADR que lo justifique.
- SwiftData: un `ModelContext` por actor (`@ModelActor`) para escrituras en segundo plano; nunca se pasan objetos `@Model` entre actores, solo `PersistentIdentifier`.

---

## 7. Pipeline

### 7.1 Captura (`ActaCapture`)

#### 7.1.1 Sesión de audio
- Categoría `.record` (o `.playAndRecord` si el Slice 0 muestra que la necesitamos para reproducir sin cortar), `UIBackgroundModes = audio`.
- Modo y fuente de micrófono a elegir en el Slice 0 comparando WER: `.default` vs `.measurement`, y patrón polar omnidireccional del micrófono integrado vía `AVAudioSession` data sources. **[Verificar en Slice 0]**
- Formato: AAC-LC mono, frecuencia de muestreo y tasa de bits elegidas en el Slice 0 (punto de partida: 48 kHz, 64 kb/s ≈ 29 MB/h). Los motores reciben 16 kHz mono por conversión.

#### 7.1.2 Segmentos y contenedor (D-05)
Un `.m4a` normal escribe su índice (`moov`) al cerrar: si la app muere a mitad de un segmento, **ese segmento queda ilegible** y se pierden hasta 5 min. Por eso el Slice 0 compara:
- (a) AAC en contenedor **CAF** (`AVAudioFile`), que admite tamaño de datos indeterminado y se lee tras un cierre abrupto.
- (b) **MPEG-4 fragmentado** con `AVAssetWriter` y `movieFragmentInterval` corto.

Se elige el que tenga menor pérdida medida y lo lean todos los motores y `AVPlayer`. Los segmentos siguen siendo de 5 min para acotar el tamaño de archivo y la recuperación.

#### 7.1.3 Journal de grabación
La base de datos no se puede escribir con el teléfono bloqueado si su archivo tiene protección `complete` (ver 7.1.5). Por eso, durante la grabación todo se escribe en un **journal append-only** por reunión (`journal.jsonl`, protección `completeUnlessOpen`), con `synchronize()` tras cada evento:

`recordingStarted · chunkOpened · chunkClosed(duration, bytes, sha256) · gap(start, end, reason) · marker(time, kind, note) · liveSegment(...) · noticeConfirmed · recordingStopped`

Al desbloquear o al relanzar, `RecordingReconciler` vuelca el journal en SwiftData, verifica los archivos contra el journal (duración real leída del archivo) y repara inconsistencias. El journal se conserva hasta que la reconciliación termina bien.

#### 7.1.4 Línea de tiempo
- **Tiempo de reunión** (`meetingTime`, segundos): posición en el audio concatenado de todos los segmentos. Es la referencia de segmentos de transcripción, marcadores, turnos de hablante y fuentes del acta.
- Los huecos guardan hora de reloj de inicio y fin y el `meetingTime` en que ocurren (duración de audio cero).

#### 7.1.5 Protección de archivos (D-06)
- Entitlement `com.apple.developer.default-data-protection = NSFileProtectionComplete`: todo archivo nuevo nace `complete`.
- Excepciones explícitas, creadas con su clase:
  - Segmento en grabación y `journal.jsonl`: `completeUnlessOpen` (se pueden crear archivos nuevos con el teléfono bloqueado, condición necesaria para rotar segmentos bloqueado).
  - Archivos de resultados intermedios del procesamiento en segundo plano (staging): `completeUnlessOpen`.
  - Modelos de ML descargados (no contienen datos personales): `completeUntilFirstUserAuthentication`, para que el procesamiento en segundo plano pueda cargarlos con el teléfono bloqueado.
- Al cerrar un segmento y con el teléfono desbloqueado, se cambia a `complete`. **[Verificar en Slice 0 si el cambio de clase se puede hacer bloqueado]**
- El audio se excluye del backup de iCloud por defecto (`isExcludedFromBackup`); el usuario puede activarlo en Ajustes con una explicación clara del intercambio privacidad/confiabilidad (D-12).
- Nombres de archivos y carpetas: solo UUID. Nunca el título de la reunión ni nombres de personas.

#### 7.1.6 Interrupciones y rutas
- `AVAudioSession.interruptionNotification`: `began` → cerrar el buffer actual, registrar `gap` con motivo; `ended` con `.shouldResume` → reanudar en un **segmento nuevo**.
- `routeChangeNotification` (`oldDeviceUnavailable`, conexión de AirPods, etc.): registrar hueco, reconfigurar el motor y reanudar.
- `mediaServicesWereResetNotification`: reconstruir la sesión completa.
- Si no se puede reanudar en segundo plano: notificación local + estado en la Live Activity. **[Verificar en Slice 0]**

### 7.2 Transcripción (`ActaTranscription`)

APIs verificadas (Speech, iOS 26+): `SpeechAnalyzer`, `SpeechTranscriber`, `AssetInventory`, `AssetInstallationRequest`, `SpeechTranscriber.supportedLocales`, `.installedLocales`, `.supportedLocale(equivalentTo:)`, `.isAvailable`, `SpeechAnalyzer.bestAvailableAudioFormat(compatibleWith:)`, `AnalyzerInputConverter`, `DictationTranscriber` (respaldo). En iOS 27 se suma `CaptureInputSequenceProvider` (no lo usamos: queremos un solo tap de audio que alimente archivo y transcriptor).

- Locale: `SpeechTranscriber.supportedLocale(equivalentTo: Locale(identifier: "es-CO"))`. Apple no publica la lista; si no hay es-CO exacto, el método devuelve otro español (p. ej. es-MX o es-US). El Slice 0 registra la lista real en el dispositivo y el locale elegido. Ajustes debe permitir al usuario cambiarlo, como recomienda Apple.
- Descarga del modelo con `AssetInventory.assetInstallationRequest(supporting:)` → `downloadAndInstall()`, mostrando `progress`. Si no hay red, la grabación funciona igual y la transcripción queda en cola.
- `SpeechTranscriber.supportedLocales` está vacío en el simulador: **toda prueba de transcripción con Apple es en dispositivo**.
- Permisos: `SpeechAnalyzer` no requiere la autorización de `SFSpeechRecognizer` (Apple documenta que ese flujo aplica solo a `SFSpeechRecognizer`). **[Verificar en Slice 0 que no se pide permiso ni falla sin `NSSpeechRecognitionUsageDescription`]**
- Motores alternos (WhisperKit, Parakeet vía FluidAudio): solo en el Spike. Pasan a producción únicamente si ganan en el Slice 0 con un margen que justifique la dependencia y el tamaño de descarga.

### 7.3 Diarización (`ActaDiarization`)
- FluidAudio `OfflineDiarizerManager` (pipeline pyannote Community-1: segmentación + WeSpeaker + VBx), sobre audio en disco (`StreamingAudioSourceFactory.makeDiskBackedSource`) para no cargar 3 h en memoria.
- Los embeddings viven solo en memoria durante el proceso. Prohibido exportarlos, cachearlos o persistirlos (sección 10.3). El Slice 0 verifica con un diff del sistema de archivos que la librería no escribe nada más que sus modelos.
- Los modelos se descargan de Hugging Face en el primer uso: la UI informa el tamaño y pide confirmación. Licencia CC-BY-4.0: atribución obligatoria en la pantalla "Acerca de".
- No usar Sortformer (licencia NVIDIA Open Model License, revisar antes de considerarla).

### 7.4 Síntesis con IA (`ActaMinutes`)

#### 7.4.1 Motores (verificados en la documentación de iOS 27)

| Motor | Tipo | Contexto | Sale del iPhone | Notas |
|---|---|---|---|---|
| `SystemLanguageModel` (en el dispositivo) | Foundation Models | `contextSize`: 4096 en iOS 26; 8192 en iOS 27 en equipos recientes | No | Sin límite de uso. Requiere Apple Intelligence |
| `PrivateCloudComputeLanguageModel` | Foundation Models | 32 768 | **Sí (a PCC de Apple)** | Entitlement gestionado `com.apple.developer.private-cloud-compute`; cuota diaria por cuenta iCloud (`quotaUsage`); requiere red y Apple Intelligence; sin costo para apps con menos de 2 M de primeras descargas |
| Claude vía proxy (Slice 5) | Proveedor externo | Según modelo | **Sí (a Anthropic vía nuestro proxy)** | Consentimiento explícito (App Store 5.1.2(i)); API key solo en el servidor |

- Siempre se lee `contextSize` en tiempo de ejecución y se cuenta con `tokenCount(for:)` (disponible desde iOS 26.4). Nunca se asume un tamaño fijo.
- PCC no elimina el map-reduce: 2 h de reunión rondan las decenas de miles de tokens, por encima de 32 K con instrucciones y respuesta. **[Medir en Slice 0 la relación palabras→tokens en español]**
- Selección de motor: on-device por defecto. PCC solo con consentimiento (D-04) y cuando esté disponible; ante falta de red, cuota agotada (`quotaLimitReached`) o `deviceNotEligible`, se cae al on-device con map-reduce, avisando en la UI sin alertas modales (como recomienda Apple).

#### 7.4.2 Resúmenes parciales durante la reunión (D-07)
El brief pide resúmenes parciales cada ~10 min con el modelo local. Riesgos: con el teléfono bloqueado la app está en segundo plano, donde Foundation Models puede limitar las peticiones (`GenerationError.rateLimited`) y donde el acceso al Neural Engine exige el entitlement *Background Inference* (iOS 27); además suma consumo y calor a una grabación larga. Decisión: el "map" de cada bloque se ejecuta **durante la reunión solo si la app está en primer plano**, y si no, al terminar. Se reevalúa con las mediciones del Slice 0.

#### 7.4.3 Presupuesto de tokens y bloques
```
presupuesto_entrada = contextSize
                    − tokens(instrucciones)
                    − tokens(esquema Generable)          // medido; ver nota
                    − reserva_salida(paso)               // calibrada con salidas reales
                    − margen_seguridad                   // % configurable, valor inicial justificado en ADR
```
- Los bloques se arman con segmentos completos (nunca se parte un segmento), en orden, con un solapamiento de los últimos segmentos del bloque anterior medido en tokens.
- La reserva de salida por paso se calibra en el Slice 2 midiendo con `tokenCount` las respuestas reales; queda registrada con su origen.
- Nota: verificar en el Slice 2 qué sobrecargas de `tokenCount(for:)` existen (instrucciones, prompt, herramientas, transcript) y cómo contar el esquema de generación guiada. Si no hay sobrecarga directa, se mide con una sesión de prueba.

#### 7.4.4 Tres pasos
1. **Map — extracción estructurada por bloque** (`@Generable ChunkExtraction`): decisiones, compromisos (descripción, responsable opcional, fecha textual opcional, prioridad opcional), riesgos, temas abiertos, temas para la próxima reunión, avance por punto del orden del día. Cada ítem cita IDs de segmento (`S0042`) que el prompt incluye.
2. **Reduce — consolidación**: une duplicados entre bloques, resuelve contradicciones (gana lo dicho más tarde, se conserva la fuente), prioriza ítems respaldados por marcadores manuales.
3. **Redacción**: genera las 9 salidas a partir de la consolidación (no de la transcripción cruda), con tipos `Generable` por sección.

#### 7.4.5 Marcadores
Los segmentos dentro de una ventana alrededor de cada marcador (tamaño a calibrar en el Slice 3) se anotan en el prompt (`⟦MARCADOR: decisión⟧`), nunca se descartan al compactar y su ítem resultante se marca `fromMarker` para que la consolidación lo priorice.

#### 7.4.6 Regla anti-invención y validación
Tras cada paso, `MinutesValidator` (en `ActaDomain`, con pruebas) aplica:
- Todo ítem debe citar al menos una fuente existente; si no, se descarta y se registra (sin contenido) en el log de diagnóstico.
- **Responsable**: debe coincidir con un asistente o con texto presente en sus fuentes; si no, el campo queda vacío y se crea una alerta.
- **Fecha**: debe aparecer en sus fuentes. Si es relativa ("el viernes"), se resuelve contra la fecha de la reunión y se marca `inferredDate` para revisión. Si no se dijo, queda vacía con alerta.
- Las fuentes se guardan como **rangos de tiempo** (`meetingTime`), no como IDs de segmento, para que sobrevivan a un reprocesamiento de la transcripción.

#### 7.4.7 Prompts
- En español, en `ActaKit/Sources/ActaMinutes/Resources/Prompts/<paso>.v<N>.md`, con cabecera YAML: `id`, `version`, `models` (para cuáles se probó), `changelog`.
- `PromptLibrary` los carga; `Minutes` guarda `promptSetVersion`.
- Cambiar un prompt = nueva versión + evaluación con el framework *Evaluations* (iOS 27) sobre los fixtures. Nunca se edita una versión publicada.
- Guardrails: manejar `guardrailViolation` y negativas con un mensaje claro y la opción de reintentar por bloques; nunca se rellena con texto inventado.

---

## 8. Modelo de datos (SwiftData)

Reglas de compatibilidad con CloudKit desde el día 1:
- Sin `@Attribute(.unique)`; todas las propiedades con valor por defecto o opcionales; **todas las relaciones opcionales y con inversa explícita**; borrado `.cascade` desde `Meeting`.
- Enums persistidos como `String` (raw value) para migraciones y predicados.
- Esquema versionado desde la v1 (`VersionedSchema` + `SchemaMigrationPlan`); cada cambio de esquema trae su prueba de migración.
- Estructuras anidadas que no necesitan consultarse ni sincronizarse por separado se guardan como valores `Codable` (iOS 27 permite persistir tipos `Codable` como atributo). **[Verificar en Slice 4 su compatibilidad con CloudKit]**
- Rutas de archivo **relativas** al contenedor de la app (las absolutas cambian entre instalaciones y restauraciones).

| Entidad | Campos principales | Cambios frente al brief |
|---|---|---|
| `MeetingSeries` | `uuid`, `name`, `lastNumber` | **Nueva**: consecutivo por serie y "reunión anterior de la misma serie" |
| `Meeting` | `uuid`, `number: Int?`, `title`, `kindRaw`, `statusRaw`, `city`, `scheduledDate`, `startedAt?`, `endedAt?`, `locationOrMedium`, `localeIdentifier`, `series?`, `nextMeetingDate?` | Número opcional hasta `final`; ciudad explícita para el acta |
| `ConsentRecord` | `noticeVersion`, `shownAt`, `confirmedAt?`, `audioOffsetAtStart`, `diarizationEnabled` | **Nueva**: separa la evidencia del aviso de la reunión |
| `AgendaItem` | `order`, `title`, `discussionSummary?` | Se agrega el desarrollo del punto |
| `Participant` | `name`, `jobTitle`, `organization`, `attendanceRaw` (`attended`, `absent`, `guest`), `email?`, `roleRaw?` (`chair`, `secretary`) | Rol para las firmas |
| `SpeakerLabel` | `label` ("Hablante 1"), `participant?` | **Nueva**: la asignación se hace una vez por etiqueta |
| `AudioChunk` | `index`, `relativePath`, `meetingTimeOffset`, `duration`, `byteCount`, `sha256`, `statusRaw` (`recording`, `closed`, `verified`, `recovered`) | Ruta relativa, integridad |
| `RecordingGap` | `startedAt`, `endedAt?`, `meetingTime`, `reasonRaw` | **Nueva**: huecos por interrupción |
| `TranscriptionRun` | `engineID`, `engineVersion`, `localeIdentifier`, `createdAt`, `isActive` | **Nueva**: reprocesar sin perder |
| `TranscriptSegment` | `start`, `end`, `text`, `speakerLabel?`, `run?` | Cuelga de la corrida, no de la reunión |
| `Marker` | `meetingTime`, `kindRaw` (`decision`, `commitment`, `risk`, `note`), `note?` | — |
| `Minutes` | `version`, `executiveSummary`, `conclusions: [String]`, `risks`, `openTopics`, `nextMeetingTopics`, `followUpEmail`, `engineID`, `promptSetVersion`, `generatedAt`, `editedByUser`, `alerts` | Riesgos y temas como `Codable` con fuentes; versión del acta |
| `Decision` | `text`, `sources: [TimeRange]`, `fromMarker` | **Entidad propia** (antes dentro de Minutes) |
| `Commitment` | `summary`, `ownerName?`, `owner: Participant?`, `dueDate?`, `dueDateIsInferred`, `priorityRaw?`, `statusRaw`, `sources: [TimeRange]`, `reminderIdentifier?` | Responsable texto + vínculo opcional; fecha inferida |
| `RoadmapItem` | `action`, `horizonRaw` (`thisWeek`, `days30`, `days90`), `dependsOn: [UUID]`, `commitment?` | Dependencias por `uuid` (no relación muchos-a-muchos) |

`TimeRange` = `struct { start: Double; end: Double }` en `meetingTime`.

---

## 9. Formato del acta

Orden obligatorio:
1. **Acta No.** (consecutivo de la serie)
2. **Ciudad y fecha**
3. **Hora de inicio y fin**
4. **Lugar o medio**
5. **Asistentes** (nombre y cargo)
6. **Ausentes**
7. **Invitados**
8. **Orden del día**
9. **Desarrollo** de cada punto del orden del día
10. **Decisiones**
11. **Compromisos** (tabla: compromiso, responsable, fecha, estado)
12. **Próxima reunión**
13. **Firmas** (quien preside y secretario/a)

Reglas: fechas en formato largo es-CO ("18 de octubre de 2026"), horas en formato de 12 h con "a. m./p. m." **[confirmar preferencia del usuario]**; un campo vacío se muestra como "Por definir", nunca se omite ni se rellena. Los temas sin punto del orden del día van a "Varios". El resumen ejecutivo, la hoja de ruta, los riesgos y el correo **no** forman parte del acta formal: se exportan como anexo opcional.

---

## 10. Privacidad y cumplimiento

> Aviso: esta sección es una guía técnica, no asesoría legal. El texto del aviso y la política de tratamiento deben validarse con un abogado en Colombia antes de usar la app con terceros.

### 10.1 Aviso de grabación (texto sugerido, versión `aviso-v1`)
> "Antes de empezar: voy a grabar el audio de esta reunión para elaborar el acta y hacer seguimiento a los compromisos. La grabación y su transcripción se procesan en mi teléfono y no se comparten con terceros. Para saber quién habla, la app analiza las voces solo durante el procesamiento de esta reunión y no guarda huellas de voz. El tratamiento se hace conforme a la Ley 1581 de 2012 y a la política de tratamiento de datos de [organización]. Pueden conocer, actualizar, rectificar o pedir la supresión de sus datos escribiendo a [correo]. Si alguien no está de acuerdo con que se grabe, por favor díganlo ahora."

- `[organización]` y `[correo]` se configuran en Ajustes antes de la primera grabación (bloqueante).
- Si en la reunión se va a usar un motor que envía contenido fuera del iPhone (PCC, Claude), el aviso cambia a la variante que lo menciona.

### 10.2 Consentimiento por motor remoto
- Pantalla de consentimiento por motor (PCC en Slice 2, Claude en Slice 5): qué se envía (texto de la transcripción, no audio), a quién, para qué, retención declarada por el proveedor. Se guarda fecha y versión del texto; se revoca en Ajustes.
- Un motor remoto nunca se usa por defecto ni como fallback automático de otro motor.

### 10.3 Biometría
- Prohibido guardar embeddings, huellas de voz o perfiles de hablante entre reuniones, ni siquiera cifrados.
- La diarización se puede desactivar por reunión (RF-42).

### 10.4 Logs y diagnóstico
- Solo `os.Logger`. Interpolaciones con privacidad `.private` por defecto; `.public` solo para enums, conteos, duraciones y códigos de error.
- Prohibido registrar texto de transcripciones, nombres, títulos, correos o rutas con datos personales. Prohibido `print`. Revisión automática con un script de lint en CI (Slice 1).
- Sin SDK de analítica ni de crashes de terceros. Solo MetricKit.

### 10.5 Permisos (verificados por API)

| Permiso / clave | Cuándo | Notas |
|---|---|---|
| `NSMicrophoneUsageDescription` | Slice 0 | Único permiso para grabar |
| `UIBackgroundModes: audio` | Slice 0 | Grabación en segundo plano |
| `NSSpeechRecognitionUsageDescription` | **No** | `SpeechAnalyzer` no lo requiere (verificar en Slice 0) |
| `BGTaskSchedulerPermittedIdentifiers` | Slice 1 | `BGContinuedProcessingTask` para procesar al terminar |
| Entitlement *Background Inference* (`…continued-processing.inference`) | Slice 0 | Necesario para usar el Neural Engine en segundo plano (Core ML: FluidAudio, WhisperKit) |
| `com.apple.developer.private-cloud-compute` | Slice 0 (solicitud) | Entitlement gestionado: requiere solicitud aprobada por Apple |
| `NSSupportsLiveActivities` | Slice 3 | Live Activity; obligatoria con `AudioRecordingIntent` |
| `NSRemindersFullAccessUsageDescription` | Slice 4 | Recordatorios no tiene acceso de solo escritura |
| Calendario | Slice 4 | `EKEventEditViewController` no requiere permiso; evita pedir acceso completo |
| Contactos | — | Si se usa, `CNContactPickerViewController` (sin permiso) |

### 10.6 Privacy manifest (`PrivacyInfo.xcprivacy`)
- `NSPrivacyTracking = false`, sin dominios de tracking.
- Datos recopilados: ninguno mientras todo sea local (Apple considera "recopilado" lo que sale del dispositivo hacia el desarrollador o terceros). Se actualiza al activar Claude.
- APIs de razón requerida a declarar según uso real: `UserDefaults`, marcas de tiempo de archivos, espacio en disco (lo usamos para RF-24), tiempo de arranque del sistema (si se mide con `systemUptime`). Verificar también los manifests de cada dependencia.
- `ITSAppUsesNonExemptEncryption = NO` (solo cifrado del sistema).

### 10.7 Retención y derechos del titular
- Borrar una reunión elimina audio, journal, transcripciones y actas (los archivos con protección por archivo de APFS quedan irrecuperables al borrar su clave).
- Política de retención configurable para el audio después de que el acta pasa a `final` (propuesta: 90 días; decisión pendiente del usuario).
- Exportación completa de una reunión (para atender solicitudes de titulares).

---

## 11. Estrategia de pruebas

- **Swift Testing** para todo código nuevo. `ActaDomain` no depende de frameworks de Apple de UI/IA, así que sus pruebas corren con `swift test` en macOS y en Linux.
- Obligatorias (brief): chunking, presupuesto de tokens (con un `TokenBudgeting` falso determinista), parseo y validación de salidas (`MinutesValidator`), migraciones de esquema (cada versión con su fixture de store).
- Integración: fixtures de audio cortos en español en `Tests/Fixtures/Audio/` con su transcripción de referencia y licencia. Fuentes propuestas: grabaciones propias con consentimiento escrito; corpus abiertos con acento colombiano (p. ej. OpenSLR 72 "Crowdsourced high-quality Colombian Spanish", CC BY-SA 4.0 — verificar licencia antes de incluirlo); voces sintéticas solo para pruebas deterministas, nunca para medir calidad.
- Métricas de calidad: WER (normalización de español documentada) y DER (collar de 0,25 s), implementadas en `ActaDomain` con pruebas.
- Pruebas en dispositivo: checklist manual versionado por slice (`docs/qa/`), con modelo de iPhone, build de iOS y resultado.
- CI: GitHub Actions en macOS para `xcodegen generate`, build y pruebas en simulador; job en Linux para `swift test` de `ActaDomain`. Lo que depende de micrófono, segundo plano o Neural Engine **no** se da por probado en CI.

---

## 12. Slices y definición de terminado

| Slice | Contenido | Documento |
|---|---|---|
| 0 | Prueba de viabilidad → `SPIKE_REPORT.md` | `docs/slices/slice-0-plan.md` |
| 1 | Crear reunión, grabar en segundo plano, transcripción con tiempos, reproducir desde un segmento | Se escribe al aprobar el Slice 0 |
| 2 | Pipeline de IA, revisión y edición, PDF y compartir | — |
| 3 | Hablantes, marcadores, Live Activity, Control, botón de Acción, Siri | — |
| 4 | Recordatorios, Calendario, historial y búsqueda, serie, iCloud opcional | — |
| 5 | Importar transcripciones, Claude vía proxy, Apple Watch | — |

**Definición de terminado (cada slice):** compila sin warnings (RNF-08) · pruebas en verde · revisado por `qa-pruebas` (incluye seguridad y privacidad) · probado en un iPhone real con checklist firmado · pasos de demo en el documento del slice. **No se avanza al siguiente slice sin aprobación del dueño del producto.**

---

## 13. Registro de decisiones (y desviaciones del brief)

| Id | Decisión | Motivo |
|---|---|---|
| D-01 | Se mantiene iOS 27 mínimo | PCC (`PrivateCloudComputeLanguageModel`) es solo iOS 27; el modelo local pasa a 8K de contexto; entitlement *Background Inference*; mejoras de SwiftData. iOS 27 corre desde el iPhone 11 |
| D-02 | Se mantiene XcodeGen + `xcodebuild`, más un paquete SPM local `ActaKit` | Modularidad, pruebas rápidas y pruebas de dominio en Linux |
| D-03 | Apple Intelligence es requisito para generar el acta con IA en el MVP | No hay motor de IA local alternativo sin dependencias pesadas; se documenta en la UI |
| D-04 | **PCC envía la transcripción fuera del iPhone**: requiere consentimiento explícito previo, igual que Claude | Contradice "todo se procesa en el dispositivo". PCC no almacena datos y es verificable, pero el texto viaja a servidores de Apple. Por la prioridad 1, es opt-in |
| D-05 | El contenedor de audio se decide en el Slice 0 (CAF vs MP4 fragmentado), no `.m4a` clásico | Un `.m4a` sin cerrar es ilegible: un crash perdería hasta 5 min |
| D-06 | Base de datos `complete` + journal y staging `completeUnlessOpen` | Con `complete` la base no se puede escribir bloqueada: sin journal se perderían segmentos, huecos y marcadores de la Live Activity |
| D-07 | Resúmenes parciales solo en primer plano | Límites de Foundation Models en segundo plano, entitlement, batería y calor |
| D-08 | Fuentes del acta como rangos de tiempo | Sobreviven al reprocesamiento; el audio es la fuente de verdad |
| D-09 | Consecutivo por serie, asignado al finalizar y editable | Un acta formal se numera por órgano/serie; evita huecos por borradores descartados y conflictos de sincronización |
| D-10 | iCloud opcional, apagado por defecto, con campos sensibles cifrados (`allowsCloudEncryption`, verificar) y audio sin sincronizar por defecto | Contradice "nada sale del iPhone" si se activa sin consentimiento |
| D-11 | Inicio rápido (Control, Acción, Siri) abre la pantalla del aviso | Grabar sin aviso incumple la regla de la Ley 1581 del propio brief. Además, las frases de App Shortcuts deben incluir el nombre de la app ("Inicia acta en weActa") |
| D-12 | Audio excluido del backup de iCloud por defecto | Coherente con D-04/D-10; el usuario puede activarlo |
| D-13 | Se agrega Parakeet (vía FluidAudio, ya incluida) a la comparación del Slice 0 | Cero dependencias nuevas; soporta español; tercer punto de comparación |
| D-14 | "Sin warnings" aplica a nuestro código; warnings de dependencias se listan en `THIRD_PARTY_NOTICES.md` | No controlamos el código de terceros |

---

## 14. Riesgos abiertos

| Riesgo | Impacto | Mitigación / verificación |
|---|---|---|
| No es posible reanudar la grabación en segundo plano tras una llamada | Alto (huecos largos) | Slice 0 E2; notificación + Live Activity para reanudar con un toque |
| `SpeechTranscriber` no tiene es-CO y el español disponible rinde mal con acento colombiano | Alto | Slice 0 E5/E6: WER comparado contra WhisperKit y Parakeet |
| El entitlement de PCC no se aprueba o tarda | Medio | Fallback on-device con map-reduce desde el diseño |
| El modelo local no alcanza la calidad del acta formal | Alto | Evaluar en Slice 2 con *Evaluations*; PCC opcional; Claude en Slice 5 |
| Consumo y calor en reuniones de 3 h | Medio | Slice 0 E1; transcripción en vivo apagada por defecto |
| Cambios de API entre betas de iOS 27.x | Medio | Adaptadores aislados; verificar en cada slice |
| Descarga de modelos (Apple, Hugging Face) sin red en el sitio de la reunión | Medio | Preparar modelos en el onboarding; la grabación nunca depende de la red |

---

## 15. Referencias verificadas (2026-10-18)

- Foundation Models: <https://developer.apple.com/documentation/foundationmodels>
- Managing the context window: <https://developer.apple.com/documentation/foundationmodels/managing-the-context-window>
- PCC en Foundation Models: <https://developer.apple.com/documentation/foundationmodels/adding-server-side-intelligence-with-private-cloud-compute>
- Entitlement PCC: <https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.private-cloud-compute> · Solicitud: <https://developer.apple.com/private-cloud-compute/>
- WWDC26 "Build with the new Apple Foundation Model on Private Cloud Compute": <https://developer.apple.com/videos/play/wwdc2026/319/>
- WWDC26 "What's new in the Foundation Models framework": <https://developer.apple.com/videos/play/wwdc2026/241/>
- SpeechAnalyzer: <https://developer.apple.com/documentation/speech/speechanalyzer> · SpeechTranscriber: <https://developer.apple.com/documentation/speech/speechtranscriber> · AssetInventory: <https://developer.apple.com/documentation/speech/assetinventory>
- Permiso de reconocimiento de voz (solo SFSpeechRecognizer): <https://developer.apple.com/documentation/speech/asking-permission-to-use-speech-recognition>
- Background Inference: <https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.background-tasks.continued-processing.inference> · BGContinuedProcessingTask: <https://developer.apple.com/documentation/backgroundtasks/bgcontinuedprocessingtask>
- AudioRecordingIntent: <https://developer.apple.com/documentation/appintents/audiorecordingintent>
- App Review Guidelines 5.1.2(i): <https://developer.apple.com/app-store/review/guidelines/>
- What's new in SwiftData (WWDC26): <https://developer.apple.com/videos/play/wwdc2026/274/>
- FluidAudio (Apache 2.0): <https://github.com/FluidInference/FluidAudio> · Modelos de diarización (CC-BY-4.0): <https://huggingface.co/FluidInference/speaker-diarization-coreml>
- WhisperKit (MIT): <https://github.com/argmaxinc/WhisperKit>
