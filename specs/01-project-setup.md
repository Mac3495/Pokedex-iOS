# SPEC 01 — Preparación del proyecto y esqueleto de la app

> **Estado:** Implementado
> **Depende de:** —
> **Fecha:** 2026-07-30
> **Objetivo:** Dejar el proyecto Xcode configurado en Swift 6 estricto sobre iOS 18 y arrancando en una pantalla propia vacía, listo para construir las capas encima.

## Alcance

**Adentro:**

- `SWIFT_VERSION` de `5.0` a `6.0` en Debug y Release.
- `SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated` en ambas configs.
- `SWIFT_STRICT_CONCURRENCY = complete` explícito en ambas configs.
- `IPHONEOS_DEPLOYMENT_TARGET` de `26.0` a `18.0`.
- Carpeta `Pokedex/App/` con `PokedexApp.swift` (movido) y `AppRootView.swift` (nuevo).
- Borrado de `Pokedex/ContentView.swift`.

**Fuera de alcance (specs futuros):**

- Carpetas y código de Core, Data, Domain, DesignSystem, DI, Presentation y Mocks: cada capa nace en su propio spec, con archivos reales adentro.
- Router y `NavigationStack`: van en el spec de Presentation.
- Target de tests unitarios: diferido por decisión del usuario.
- Cualquier llamada a PokeAPI, SwiftData o `ModelContainer`.
- App icon, accent color y assets: `Assets.xcassets` queda como está.

## Modelo de datos

Este spec no introduce estructuras de datos. Solo configuración de build y el esqueleto de arranque de la app.

## Plan de implementación

1. **Build settings del proyecto.** En `Pokedex.xcodeproj/project.pbxproj`, cambiar `IPHONEOS_DEPLOYMENT_TARGET = 26.0` → `18.0` en los dos bloques de nivel proyecto (Debug y Release).
   *Verificación:* `xcodebuild ... build` sigue compilando la plantilla sin errores.

2. **Build settings del target.** En los dos bloques de nivel target (Debug y Release), poner `SWIFT_VERSION = 6.0` y agregar `SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated` y `SWIFT_STRICT_CONCURRENCY = complete`.
   *Verificación:* compila sin warnings. Abrir Xcode y confirmar que Build Settings muestra Swift Language Version 6 y Default Actor Isolation `nonisolated`.

3. **Carpeta App/.** Crear `Pokedex/App/` y mover `PokedexApp.swift` adentro. No hay que tocar el project file: el target usa `PBXFileSystemSynchronizedRootGroup`, que toma las carpetas del disco automáticamente.
   *Verificación:* compila; el archivo aparece bajo `App` en el navegador de Xcode.

4. **AppRootView.** Crear `Pokedex/App/AppRootView.swift`: `struct AppRootView: View` con un contenedor vacío y el título "Pokédex", más su `#Preview`. Sin lógica, sin navegación, sin dependencias.

5. **Cerrar el arranque.** Apuntar el `WindowGroup` de `PokedexApp` a `AppRootView()` y borrar `Pokedex/ContentView.swift`.
   *Verificación:* correr en el simulador iPhone 17; la app abre en la pantalla nueva y no queda ninguna referencia a `ContentView` en el repo.

## Criterios de aceptación

- [ ] `xcodebuild -project Pokedex.xcodeproj -scheme Pokedex -destination 'platform=iOS Simulator,name=iPhone 17' build` termina en **BUILD SUCCEEDED** sin warnings.
- [ ] Los cuatro bloques de `XCBuildConfiguration` tienen los valores esperados: `SWIFT_VERSION = 6.0`, `SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated` y `SWIFT_STRICT_CONCURRENCY = complete` en los de target; `IPHONEOS_DEPLOYMENT_TARGET = 18.0` en los de proyecto.
- [ ] `Pokedex/ContentView.swift` no existe y `grep -r ContentView Pokedex/` no devuelve resultados.
- [ ] Existen `Pokedex/App/PokedexApp.swift` y `Pokedex/App/AppRootView.swift`, y no queda ningún `.swift` suelto en la raíz de `Pokedex/`.
- [ ] La app arranca en el simulador iPhone 17 y muestra la pantalla de `AppRootView` con el título "Pokédex", sin el "Hello, world!" de la plantilla.
- [ ] El `#Preview` de `AppRootView` renderiza en el canvas de Xcode.

## Decisiones

- **Sí:** iOS 18.0 como mínimo. `CLAUDE.md` pide "17+, idealmente 18+". iOS 26 recorta el parque de dispositivos sin dar nada a cambio: `@Observable` y SwiftData ya existen desde 17, y 18 trae SwiftData sin los bugs de `@ModelActor` de 17.0.
- **No:** iOS 17.0. Migraciones y `@ModelActor` inestables, y el spec de persistencia depende de eso.
- **Sí:** `SWIFT_DEFAULT_ACTOR_ISOLATION = nonisolated`, contra el default de Xcode 26. Domain y Data tienen que ser `Sendable` puros y correr fuera del main thread; el `@MainActor` se marca explícito solo en ViewModels y Router.
- **No:** aislamiento `MainActor` global. Ataría la capa de datos al main thread y dejaría el `actor` del APIClient y el `@ModelActor` como excepciones sueltas, en contra de "Sendable por defecto".
- **Sí:** `SWIFT_STRICT_CONCURRENCY = complete` explícito, aunque Swift 6 ya lo implica. Documenta la intención y sobrevive a un downgrade accidental de `SWIFT_VERSION`.
- **Sí:** crear solo `App/`. Las carpetas vacías no las trackea git ni aportan al build; cada capa nace en su spec con archivos reales adentro.
- **No:** sembrar el árbol completo con `.gitkeep`. Quince archivos basura para borrar después.
- **Sí:** editar `project.pbxproj` a mano. Son cuatro claves en bloques ya existentes; el formato objectVersion 77 no requiere tocar nada más.
- **No:** target de tests unitarios. Decisión explícita del usuario; se aparta del checklist de `CLAUDE.md` y se difiere.

## Riesgos

| Riesgo | Mitigación |
| --- | --- |
| Editar `project.pbxproj` a mano lo corrompe | Cambios acotados a cuatro claves dentro de bloques existentes. Verificar con `xcodebuild -list` antes de commitear; el repo está limpio, así que `git checkout` revierte. |
| Bajar el deployment target a 18.0 rompe alguna API de la plantilla | La plantilla solo usa `SwiftUI` básico. El build del paso 1 lo detecta enseguida. |
| `nonisolated` por defecto genera errores de concurrencia en specs siguientes | Es el resultado buscado: los errores aparecen al escribir Data y Presentation, que es donde hay que decidir el aislamiento. |

## Lo que **no** entra en este spec

- Carpetas y código de Core, Data, Domain, DesignSystem, DI, Presentation y Mocks.
- Router y `NavigationStack`.
- Target de tests unitarios.
- Llamadas a PokeAPI, SwiftData y `ModelContainer`.
- App icon, accent color y cualquier asset.

Cada uno de esos, cuando toque, va en su propio spec.
