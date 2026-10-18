# CLAUDE.md

weActa (nombre provisional): app nativa de iPhone que graba reuniones presenciales, transcribe y separa hablantes en el dispositivo, y genera el acta formal colombiana y sus salidas de seguimiento.

## Lee primero
1. `spec.md` — qué construimos, decisiones y desviaciones del brief (§13).
2. `AGENTS.md` — cómo trabajamos: equipo, estándares, privacidad, pruebas, definición de terminado.
3. El documento del slice en curso en `docs/slices/`.

## Estado actual
- Slice 0 (prueba de viabilidad): **plan propuesto, pendiente de aprobación**. No se escribe código de la app hasta que el dueño del producto lo apruebe.

## Reglas para Claude
- Prioridades: seguridad y privacidad → no perder audio → calidad del acta → experiencia de uso.
- No inventes APIs. Verifica nombres y disponibilidad en la documentación de Apple para iOS 27 / Xcode 27 y cita la URL. Si algo no existe como lo describe la spec, detente y dilo.
- Si una tarea contradice `spec.md`, detente, propone un ADR y pide aprobación.
- No avances de slice sin aprobación explícita.
- Delegación: `backend` (ActaKit, motores, datos, integraciones), `ux-ui` (SwiftUI, Live Activity, accesibilidad), `qa-pruebas` (pruebas y revisión de cada slice, incluida la de seguridad y privacidad). Ver `.claude/agents/`.
- UI en español (es-CO) mediante String Catalog; código, identificadores y commits en inglés.
- Nunca registres en logs contenido de transcripciones, nombres ni títulos. Nunca subas audios ni transcripciones reales al repositorio.

## Entorno
- Compilar y probar la app requiere macOS con Xcode 27 y XcodeGen (comandos en `AGENTS.md` §8).
- Las sesiones en la nube de Claude Code corren en Linux: ahí **no** hay `xcodebuild` ni simulador. En Linux solo se puede editar código y, si hay toolchain de Swift instalado, correr `swift test` del módulo `ActaDomain`.
- Todo lo que dependa del micrófono, el segundo plano, la protección de datos o el Neural Engine se valida en un iPhone real.
