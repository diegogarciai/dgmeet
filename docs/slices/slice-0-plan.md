# Slice 0 — Prueba de viabilidad (plan)

> Estado: **propuesto, pendiente de aprobación**. No se escribe código hasta el OK del dueño del producto.
> Entregable final: `SPIKE_REPORT.md` con resultados medidos y recomendación de motores. El slice termina ahí.

## 1. Objetivo

Responder con datos medidos en un iPhone real, antes de construir producto:

| # | Pregunta | Experimento |
|---|---|---|
| Q1 | ¿Podemos grabar ≥ 3 h con la pantalla bloqueada sin perder audio? | E1 |
| Q2 | ¿Qué pasa con cada interrupción? ¿Se reanuda sola en segundo plano? | E2 |
| Q3 | ¿Cuánto audio se pierde si la app muere? ¿CAF o MP4 fragmentado? | E3 |
| Q4 | ¿El esquema de protección de archivos (base `complete` + journal `completeUnlessOpen`) funciona bloqueado? | E4 |
| Q5 | ¿Qué español ofrece `SpeechTranscriber` para es-CO y con qué calidad? | E5 |
| Q6 | ¿WhisperKit o Parakeet superan a Apple lo suficiente para justificar la dependencia? | E6 |
| Q7 | ¿FluidAudio diariza bien reuniones en español y en cuánto tiempo, sin persistir embeddings? | E7 |
| Q8 | ¿Cuánto contexto real tiene el modelo local, cuántos tokens ocupa una hora de reunión en español? | E8 |
| Q9 | ¿PCC está disponible para nuestra app y con qué límites? | E9 |
| Q10 | ¿Podemos procesar al terminar con el teléfono bloqueado? | E10 |
| Q11 | ¿Batería, temperatura y memoria son aceptables? | Transversal |

Fuera de alcance: diseño, modelo de datos final, pipeline del acta, calidad del texto generado (eso es el Slice 2).

## 2. Prerrequisitos (los pone el dueño del producto)

- [ ] Mac con **Xcode 27** (última versión estable) y XcodeGen; o acceso a un runner macOS de CI.
- [ ] Cuenta del Apple Developer Program (Team ID) y bundle ID definido.
- [ ] **Solicitar ya** el entitlement de PCC en <https://developer.apple.com/private-cloud-compute/> (lo hace el Account Holder; la aprobación puede tardar). Verificar en el portal si *Background Inference* también requiere solicitud.
- [ ] iPhone con Apple Intelligence (15 Pro o posterior) en iOS 27.x, con Apple Intelligence activado en español. Ideal: un segundo modelo para comparar.
- [ ] Audio de prueba (ver §6).

## 3. Qué se construye

Un target `WeActaSpike` sin diseño, más el esqueleto mínimo de `ActaKit/ActaDomain` para lo que el Spike necesita y que sí sobrevivirá (WER, DER, journal, tipos de tiempo). Todo lo demás del Spike es descartable.

```
project.yml                         targets: WeActaSpike (+ ActaKit local)
Spike/Sources/
  SpikeApp.swift                    TabView: Grabar · Motores · Diagnóstico · Resultados
  Capture/SpikeRecorder.swift       AVAudioEngine + escritor CAF | MP4 fragmentado (conmutable)
  Capture/SpikeJournal.swift        journal JSONL con completeUnlessOpen
  Engines/AppleSpeechRunner.swift   SpeechAnalyzer + SpeechTranscriber (archivo y en vivo)
  Engines/WhisperKitRunner.swift
  Engines/ParakeetRunner.swift      FluidAudio ASR
  Engines/DiarizationRunner.swift   FluidAudio OfflineDiarizerManager
  Engines/FoundationModelsProbe.swift   contextSize, tokenCount, supportedLanguages, PCC availability/quota
  Background/ContinuedProcessing.swift  BGContinuedProcessingTask
  Metrics/MetricsRecorder.swift     JSONL: batería, térmico, memoria, tiempos
  Metrics/ResultsExporter.swift     comparte métricas (sin contenido de reuniones reales)
Packages/ActaKit/Sources/ActaDomain/
  Time/TimeRange.swift  Journal/JournalEvent.swift  Metrics/WER.swift  Metrics/DER.swift
Packages/ActaKit/Tests/ActaDomainTests/
  WERTests.swift  DERTests.swift  JournalEventTests.swift
```

Dependencias del Spike (versión exacta, fijada al iniciar tras revisar la última release y su licencia): `FluidAudio` (Apache 2.0; modelos CC-BY-4.0) y `WhisperKit` (MIT). Solo el target del Spike las enlaza.

Entitlements e Info.plist del Spike: `UIBackgroundModes: audio`, `NSMicrophoneUsageDescription`, `default-data-protection: NSFileProtectionComplete`, `BGTaskSchedulerPermittedIdentifiers`, *Background Inference* y PCC (si ya están aprobados). **Sin** `NSSpeechRecognitionUsageDescription`, para verificar que `SpeechAnalyzer` no lo necesita.

## 4. Experimentos

Cada experimento registra en `MetricsRecorder`: modelo de iPhone, build de iOS, modo bajo consumo, batería inicial/final (`UIDevice.batteryLevel`), transiciones de `ProcessInfo.thermalState`, pico de memoria (`phys_footprint`), duración de audio, tiempo de reloj y factor de tiempo real (RTF). La batería de iOS es aproximada: para E1 y E6 se complementa con una traza de *Power Profiler* en Instruments.

### E1 — Grabación larga bloqueada
- 3 h continuas con pantalla bloqueada, en tres variantes: (a) solo grabación, (b) grabación + transcripción en vivo con `SpeechTranscriber`, (c) grabación con modo de micrófono `.measurement`.
- Aceptación: 0 s perdidos (duración de audio = tiempo de reloj − huecos), todos los segmentos legibles, journal consistente, sin `thermalState` `.serious`.
- Se mide: %/h de batería, MB/h, temperatura.

### E2 — Interrupciones y rutas
Matriz, cada caso con la pantalla bloqueada y desbloqueada:

| Caso | Esperado |
|---|---|
| Llamada telefónica contestada / rechazada / ignorada | Hueco marcado; reanuda al terminar |
| Llamada de FaceTime y de WhatsApp | Ídem |
| Siri invocado | Ídem |
| Alarma o temporizador | Ídem |
| Otra app reproduce audio | Hueco o mezcla, según categoría |
| Conectar / desconectar AirPods; micrófono Bluetooth | Cambio de ruta registrado; sigue grabando del micrófono correcto |
| Modo bajo consumo activado durante la grabación | Sigue grabando |

Se registra para cada caso: si llegó `shouldResume`, si se pudo reanudar **en segundo plano** sin abrir la app, y cuánto duró el hueco. Si no se puede reanudar en segundo plano, se prueba la notificación local "Toca para reanudar".

### E3 — Cierre abrupto
- Matar la app durante la grabación (detener desde Xcode, `kill -9` desde el depurador, presión de memoria) y reiniciar el teléfono a mitad de un segmento.
- Comparar **CAF (AAC)** contra **MPEG-4 fragmentado** (`AVAssetWriter.movieFragmentInterval`).
- Aceptación: el segmento interrumpido es legible por `AVAudioFile`, `AVPlayer`, `SpeechAnalyzer`, WhisperKit y FluidAudio; pérdida ≤ intervalo de fragmento. Se elige contenedor.

### E4 — Protección de datos bloqueado
- Con el teléfono bloqueado durante una grabación: rotar segmentos, escribir el journal, intentar escribir en un store SwiftData `complete` (se espera que falle), leer modelos de ML con `completeUntilFirstUserAuthentication`, cambiar la clase de un segmento cerrado de `completeUnlessOpen` a `complete`.
- Verificar la clase final de cada archivo con `FileManager.attributesOfItem(atPath:)[.protectionKey]`.
- Aceptación: el esquema de `spec.md` §7.1.5 funciona; si no, propuesta alternativa en el informe.

### E5 — SpeechTranscriber
- Volcar `SpeechTranscriber.supportedLocales`, `installedLocales`, `isAvailable` y el resultado de `supportedLocale(equivalentTo: es-CO)`.
- Medir descarga del modelo con `AssetInventory` (tamaño, tiempo, comportamiento sin red).
- Transcribir los fixtures (§6) desde archivo: WER, RTF, memoria.
- Comparar WER de la transcripción **en vivo** (E1b) contra la pasada final sobre el mismo audio: ¿se puede reutilizar la de en vivo?
- Confirmar que no aparece ningún diálogo de permiso de reconocimiento de voz.

### E6 — WhisperKit y Parakeet
- WhisperKit: dos tamaños de modelo (uno grande cuantizado y uno mediano, elegidos según la release vigente y el dispositivo), idioma forzado a español.
- Parakeet (FluidAudio, modelo multilingüe por defecto de la release vigente).
- Mismos fixtures y métricas que E5, más tamaño de descarga, tiempo de carga del modelo y temperatura.
- Ejecutar también con la app en segundo plano para confirmar que, sin el entitlement *Background Inference*, el Neural Engine no está disponible, y con él, que sí.

### E7 — Diarización con FluidAudio
- `OfflineDiarizerManager` sobre los fixtures multihablante anotados: DER (collar 0,25 s), número de hablantes detectados vs real, tiempo y memoria para 10, 60 y 120 min (fuente en disco).
- Diff del contenedor de la app antes y después: confirmar que solo se escriben archivos de modelo (ningún embedding).
- Registrar desde dónde se descargan los modelos, su tamaño y su licencia.

### E8 — Foundation Models en el dispositivo
- `SystemLanguageModel.default.availability`, `contextSize`, `supportedLanguages` (¿incluye español?).
- `tokenCount(for:)` sobre transcripciones de 10, 30 y 60 min: tokens por minuto de reunión en español.
- Una extracción `@Generable` mínima sobre un bloque de un fixture público (prueba de funcionamiento, no de calidad), con la app en primer plano y en segundo plano (¿`rateLimited`?).
- Registrar errores de guardrails, si aparecen.

### E9 — Private Cloud Compute
- Con el entitlement: `PrivateCloudComputeLanguageModel().availability` (`.available`, `.unavailable(.deviceNotEligible)`, `.unavailable(.systemNotReady)`), `contextSize`, `quotaUsage.status`.
- Una petición con **texto sintético o de un corpus público, nunca con una reunión real** (aún no existe la pantalla de consentimiento).
- Sin el entitlement: documentar el error exacto y el estado de la solicitud.

### E10 — Procesamiento al terminar, bloqueado
- Iniciar desde primer plano un `BGContinuedProcessingTask` que transcribe y diariza 60 min de audio; bloquear el teléfono.
- Medir si termina, cuánto tarda, si el sistema lo cancela y cómo se ve el progreso en la Live Activity del sistema.
- Escribir resultados en staging (`completeUnlessOpen`) y verificar la importación posterior.

## 5. Criterios de decisión (propuestos; el dueño del producto puede ajustarlos)

| Decisión | Regla |
|---|---|
| Viabilidad de captura | **No-go** si E1 pierde audio o E3 pierde más que el intervalo de fragmento. Sin captura confiable no hay producto |
| Motor de transcripción | Por defecto `SpeechTranscriber` (sin dependencias, sin descarga propia). Se cambia a otro motor solo si mejora el WER en ≥ 3 puntos absolutos en los fixtures conversacionales es-CO **y** procesa 60 min en ≤ 10 min sin `thermalState` `.serious` |
| Transcripción en vivo | Se reutiliza como final si su WER no es peor en > 1 punto que la pasada final |
| Diarización | FluidAudio se adopta si DER ≤ 20 % en los fixtures de reunión; si no, el Slice 3 se replantea |
| Motor de acta | On-device por defecto; PCC opcional si E9 confirma disponibilidad |
| Contenedor de audio | El de menor pérdida medida en E3 que lean todos los consumidores |

## 6. Fixtures de audio

| Conjunto | Uso | Fuente | Requisito |
|---|---|---|---|
| A. Lectura es-CO, 10 × 30 s | WER base, pruebas de integración cortas | Corpus abierto con acento colombiano (p. ej. OpenSLR 72) | Verificar licencia y atribución; transcripción de referencia incluida |
| B. Reunión simulada, 2–4 personas, 10 min | WER conversacional y DER | Grabación propia del equipo con consentimiento escrito | Transcripción y turnos de hablante anotados a mano (el equipo QA prepara la plantilla) |
| C. Reunión real, 60–120 min | E1, E7, E10 (tiempos, batería) | Reunión real con aviso leído | **No sale del iPhone ni se sube al repositorio**; solo se exportan métricas |

Los conjuntos A y B (si el consentimiento lo permite) se versionan en `Tests/Fixtures/Audio/` con su `LICENSE`.

## 7. Orden de trabajo

| Paso | Qué | Agente | Depende de |
|---|---|---|---|
| 1 | `project.yml`, target Spike, entitlements, `ActaDomain` mínimo (TimeRange, JournalEvent, WER, DER) con pruebas | backend + qa | Aprobación de este plan |
| 2 | Grabador con journal y dos contenedores; vista "Grabar" | backend | 1 |
| 3 | Checklist de dispositivo E1–E4 y plantilla de anotación de fixtures | qa | 1 |
| 4 | Ejecutar E1–E4 en el iPhone | dueño del producto (guiado) | 2, 3 |
| 5 | Runners de Apple, WhisperKit, Parakeet, FluidAudio; vista "Motores" | backend | 1 |
| 6 | Sondas de Foundation Models y PCC; vista "Diagnóstico" | backend | 1 |
| 7 | `BGContinuedProcessingTask` | backend | 5 |
| 8 | Ejecutar E5–E10 | dueño del producto (guiado) | 5–7, fixtures |
| 9 | `SPIKE_REPORT.md`, revisión de qa-pruebas | backend + qa | 4, 8 |

## 8. Pasos de demo para el dueño del producto

1. Instalar `WeActaSpike` desde Xcode o TestFlight en el iPhone.
2. **Grabar**: elegir contenedor, iniciar, bloquear el teléfono 3 h; recibir y colgar una llamada a mitad. Al volver: ver la lista de segmentos, huecos con motivo y duración total.
3. **Motores**: elegir un fixture o la grabación, correr Apple / WhisperKit / Parakeet / Diarización; ver WER, DER, RTF, batería y temperatura.
4. **Diagnóstico**: ver locales de Speech, `contextSize` local y de PCC, estado de PCC y cuota, entitlements activos.
5. **Resultados**: exportar el JSONL de métricas (sin contenido de reuniones) y compararlo con las tablas de `SPIKE_REPORT.md`.

## 9. Definición de terminado del Slice 0

- [ ] El Spike compila sin warnings en nuestro código; pruebas de `ActaDomain` en verde.
- [ ] E1–E10 ejecutados en iPhone real; checklist firmado en `docs/qa/slice-0-device-checklist.md`.
- [ ] `SPIKE_REPORT.md` con: tabla de resultados por experimento, motores recomendados, contenedor de audio elegido, cambios propuestos a `spec.md` y riesgos que siguen abiertos.
- [ ] Revisión de `qa-pruebas` en `docs/qa/slice-0-review.md` (incluye privacidad: la grabación C nunca salió del iPhone).
- [ ] Aprobación del dueño del producto. **Aquí nos detenemos.**
