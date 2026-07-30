# CLAUDE.md — Pokedex

Proyecto iOS. Toda generación de código debe seguir la arquitectura descripta en
`Pokedex/references/architecture-ios-template.md`. Ese documento es la referencia
completa (capas, ejemplos de código, convenciones); acá solo un resumen operativo.

## Arquitectura: Clean Architecture + MVVM

Presentation → Domain → Data. Las flechas de conocimiento apuntan hacia adentro:
Domain no conoce SwiftUI/SwiftData; Data no conoce ViewModels.

- **Presentation**: SwiftUI Views (sin lógica) + ViewModels `@Observable`, `@MainActor`.
- **Domain**: UseCases (protocolo + impl real + mock), un caso de uso por acción.
- **Data**: Repositories (protocolo + impl), RemoteDataSource (`URLSession` + DTOs
  Codable), LocalDataSource (SwiftData `@Model`), Mappers DTO ↔ Entity.
- **Core**: entidades de dominio puras, protocolos de Repository/UseCase, `AppError`.

## Stack

- Swift 6.2, concurrencia estricta desde el día uno (`Sendable`, `actor`).
- SwiftUI, sin UIKit salvo interoperabilidad puntual.
- `@Observable` (Observation) — nunca `ObservableObject`/`@Published` en código nuevo.
- `async/await` para todo; nada de completion handlers ni Combine para networking.
- `URLSession` propio protocol-oriented, sin librerías de terceros (sin Alamofire).
- SwiftData para persistencia (`@Model`, `ModelContainer`, `ModelContext`).
- `NavigationStack` + Router propio (`@Observable`), sin `NavigationLink` dispersos.
- DI: contenedor propio inyectado vía `@Environment`, sin frameworks (sin Swinject/Factory).
- Testing: Swift Testing (`@Test`, `#expect`) para unit tests; XCUITest solo para UI tests.
- iOS mínimo 17+ (idealmente 18+).

## Estructura de carpetas

Empezar simple (carpetas dentro de un solo target) y migrar a Swift Packages locales
(Core, Domain, Data, DesignSystem, DI, Features/*) cuando el proyecto crezca. La
separación conceptual de capas debe respetarse desde el inicio igual.

## Convenciones

- Naming: `XxxUseCase`/`XxxUseCaseImpl`, `XxxRepository`/`XxxRepositoryImpl`.
- Un tipo por archivo, nada de "God files".
- Cero lógica de negocio en las Views.
- Todo async es `async throws`.
- Sendable por defecto.
- Sin singletons mutables globales (`.shared` con estado mutable) — usar DI.
- Cada protocolo de Core tiene su `Mock` correspondiente para tests/previews.

## Checklist antes de dar por terminado un feature

- [ ] View sin lógica de negocio.
- [ ] ViewModel `@Observable` corriendo en `@MainActor`.
- [ ] Errores tipados (`AppError` o específicos del dominio).
- [ ] Mock de cada protocolo usado en tests/previews.
- [ ] Compila con Swift 6 en modo estricto sin warnings.
- [ ] Al menos un test Swift Testing para el UseCase principal.
