# AGENTS.md — Cómo trabajamos en weActa

Este archivo aplica a cualquier agente (Claude Code, subagentes, otras herramientas) y a cualquier persona que toque el repositorio. El **qué** está en `spec.md`; este archivo define el **cómo**.

## 1. Metodología

`spec.md` → `AGENTS.md` → **slices verticales**.

1. Cada slice tiene su documento en `docs/slices/slice-N-*.md` con objetivo, alcance, tareas, criterios de aceptación y pasos de demo.
2. Un slice no empieza sin plan aprobado por el dueño del producto, y no se pasa al siguiente sin su aprobación explícita del anterior.
3. Si una tarea obliga a apartarse de `spec.md`, **se detiene la tarea**, se escribe un ADR en `docs/adr/NNNN-titulo.md` (contexto, opciones, decisión, consecuencias) y se pide aprobación. Luego se actualiza `spec.md` (sección 13).
4. Nunca se inventan APIs. Antes de usar una API de Apple (Speech, Foundation Models, PCC, ActivityKit, App Intents, SwiftData, BackgroundTasks), se verifica el nombre exacto y la disponibilidad en la documentación oficial de iOS 27 y se cita la URL en el PR o en el código del adaptador. Si algo no existe como dice la spec, se informa y se propone la alternativa.

## 2. Equipo (subagentes en `.claude/agents/`)

| Agente | Responsable de | No toca |
|---|---|---|
| `backend` | `ActaKit` (dominio, persistencia, captura, motores de transcripción/diarización/IA, exportación, integraciones), entitlements, `Info.plist`, privacy manifest | Vistas SwiftUI (salvo vistas de depuración del Spike) |
| `ux-ui` | Pantallas SwiftUI, navegación, flujos, Live Activity, Controles, accesibilidad, textos de UI (String Catalog es-CO) | Lógica de motores y persistencia; solo consume protocolos y modelos de vista |
| `qa-pruebas` | Plan de pruebas, pruebas Swift Testing, fixtures, checklists de dispositivo, revisión de cada slice contra la definición de terminado, revisión de seguridad y privacidad | No implementa funcionalidad; si encuentra un defecto, lo reporta con reproducción y lo corrige el dueño del área |

Traspasos:
- `backend` publica protocolos y modelos de vista estables antes de que `ux-ui` construya la pantalla.
- Toda entrega pasa por `qa-pruebas` antes de declararse terminada. Su informe va en `docs/qa/slice-N-review.md`.

## 3. Estructura del repositorio

```
project.yml                     XcodeGen (el .xcodeproj no se versiona)
App/                            Target WeActa (SwiftUI): Sources/, Resources/ (Localizable.xcstrings, PrivacyInfo.xcprivacy)
Widgets/                        Target WeActaWidgets (Slice 3)
Spike/                          Target WeActaSpike (solo Slice 0)
Packages/ActaKit/               Paquete SPM local (ver spec §6.2)
  Sources/ActaDomain|ActaPersistence|ActaCapture|ActaTranscription|ActaDiarization|ActaMinutes|ActaExport
  Sources/ActaMinutes/Resources/Prompts/   Prompts versionados (*.vN.md)
  Tests/…                       Swift Testing
Tests/Fixtures/Audio/           Fixtures de audio + transcripción de referencia + LICENSE por fuente
docs/slices/  docs/adr/  docs/qa/
scripts/                        lint de privacidad, utilidades de CI
THIRD_PARTY_NOTICES.md          Licencias y atribuciones de dependencias y modelos
```

## 4. Estándares de código

- Swift 6, modo de lenguaje 6, concurrencia estricta completa. SwiftUI + Observation (`@Observable`); no `ObservableObject` en código nuevo.
- Servicios con estado: `actor`. Modelos de vista: `@MainActor @Observable final class`.
- Prohibido: `@unchecked Sendable`, `nonisolated(unsafe)`, `try!`, force unwrap (`!`) fuera de pruebas, `print`, `fatalError` en rutas de usuario. Excepciones solo con ADR.
- `@preconcurrency import` solo para frameworks de Apple que aún no estén anotados, con un comentario que lo justifique.
- Inyección de dependencias por protocolos definidos en `ActaDomain`. Las dependencias externas solo se importan en su módulo adaptador.
- Errores tipados por módulo (`enum …Error: Error`), con mensajes de usuario en el String Catalog, nunca en el error.
- Identificadores, comentarios y commits en inglés. Textos visibles en español es-CO vía `String(localized:)` / `LocalizedStringResource`. Nada de texto de UI hardcodeado.
- Comentarios solo donde el porqué no es obvio (especialmente: decisiones de audio en tiempo real, protección de archivos, límites de API con su URL).
- Fechas y horas: `Date` en UTC en persistencia; formato es-CO solo en presentación. Tiempos de audio en `meetingTime` (segundos, `Double`).

## 5. Reglas de privacidad (no negociables)

1. Ningún contenido de la reunión sale del iPhone sin consentimiento explícito previo por motor (`spec.md` §10.2). Un motor remoto nunca es fallback automático.
2. Logs: solo `os.Logger`; `.private` por defecto; `.public` solo para enums, conteos, duraciones y códigos. Nunca texto de transcripción, nombres, títulos, correos ni rutas con datos personales.
3. Archivos y carpetas nombrados solo con UUID.
4. Prohibido persistir embeddings o huellas de voz.
5. Protección de archivos según `spec.md` §7.1.5. Todo archivo nuevo de audio, journal o staging declara su clase de protección de forma explícita.
6. Nada de SDKs de analítica, crash reporting o publicidad.
7. Secretos (API keys, tokens) nunca en el repositorio ni en la app. El `.gitignore` incluye `*.p8`, `*.mobileprovision`, `*.xcconfig.local`.

## 6. Dependencias

- Solo SPM, con **versión exacta** (`exactVersion`) en `project.yml` / `Package.swift`.
- Antes de agregar una: ADR con justificación, alternativa sin dependencia, licencia del código **y de los modelos** que descarga, tamaño en la app y en descarga, y si hace llamadas de red.
- Registrar en `THIRD_PARTY_NOTICES.md` y en la pantalla "Acerca de" (atribuciones CC-BY obligatorias).
- Dependencias aprobadas: ninguna para producción todavía. Para el Spike: `FluidAudio` (Apache 2.0; modelos CC-BY-4.0) y `WhisperKit` (MIT; modelos Whisper MIT).

## 7. Pruebas

- Swift Testing (`import Testing`, `@Test`, `#expect`) en todo código nuevo.
- Obligatorias: chunking, presupuesto de tokens, parseo/validación de salidas, migraciones de datos, WER/DER, reconciliación del journal.
- `ActaDomain` debe compilar y probarse sin frameworks de UI/IA de Apple (para correr en Linux).
- Lo que depende de micrófono, segundo plano, protección de datos o Neural Engine se prueba **en iPhone real** con checklist en `docs/qa/`.

## 8. Comandos

```bash
# Generar el proyecto
xcodegen generate

# Compilar (sin warnings: el build falla si aparece uno en nuestro código)
xcodebuild -project WeActa.xcodeproj -scheme WeActa \
  -destination 'generic/platform=iOS' build

# Pruebas en simulador
xcodebuild -project WeActa.xcodeproj -scheme WeActa \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' test

# Pruebas de dominio (macOS o Linux)
swift test --package-path Packages/ActaKit --filter ActaDomainTests

# Lint de privacidad (Slice 1)
scripts/privacy-lint.sh
```

El nombre del simulador se ajusta a los disponibles en Xcode 27 (`xcrun simctl list devices`).

## 9. Git

- Ramas: `slice-N/<tema>` o la que asigne la sesión. Commits pequeños, en inglés, en imperativo (`Add journal reconciler`).
- Nunca se suben `.xcodeproj`, `DerivedData`, audios de reuniones reales ni transcripciones reales. Los fixtures solo con licencia documentada.
- Un PR por tarea, con: qué cambia, cómo se probó (simulador / dispositivo y modelo), riesgos de privacidad, APIs verificadas (URLs).

## 10. Definición de terminado (por slice)

- [ ] Compila sin warnings en nuestro código (warnings de dependencias listados en `THIRD_PARTY_NOTICES.md`).
- [ ] Pruebas en verde (simulador + `swift test` de dominio).
- [ ] Revisión de `qa-pruebas` (funcional, seguridad y privacidad) en `docs/qa/slice-N-review.md`, sin hallazgos bloqueantes abiertos.
- [ ] Probado en iPhone real: checklist firmado con modelo, versión de iOS y fecha.
- [ ] Pasos de demo en `docs/slices/slice-N-*.md`.
- [ ] `spec.md` actualizado si cambió alguna decisión.
- [ ] Aprobación del dueño del producto antes de iniciar el siguiente slice.
