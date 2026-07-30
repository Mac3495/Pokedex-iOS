# Clean Architecture SwiftUI — Reference & Implementation Roadmap

Fuente: https://github.com/nalexn/clean-architecture-swiftui (CountriesSwiftUI, rama `master`, revamp 2024).
Stack de referencia: SwiftUI + Swift 6.2 (concurrencia estricta) + async/await + SwiftData + Observation.
Combine se conserva solo para casos puntuales (`Store`/`updates(for:)`, debounce de búsqueda) — no para
networking ni para el flujo principal de datos.

Este documento describe, en orden de implementación, todo lo necesario para llevar cualquier
proyecto SwiftUI (partiendo de un template vacío, como Pokedex) a esta misma arquitectura.
Está pensado para que cada paso pueda convertirse en un agente/skill independiente.

> **Nota de versión**: la base es el repo de nalexn (`master`, revamp 2024). Este documento la
> actualiza con el stack descrito en `architecture-ios-template.md` (Swift 6 estricto, `@Observable`,
> Swift Testing, `AppError` tipado, Swift Packages locales). Los bloques marcados **[2026]** son
> modernización sobre el repo original, no parte de su código fuente.

---

## 0. Visión general de capas

```
Presentation (SwiftUI Views [+ ViewModel @Observable opcional])
        │  llama a
        ▼
Business Logic (Interactors / UseCases)
        │  lee/escribe
        ▼
AppState (Store, fuente única de verdad)   ◄──┐
        │                                     │
        ▼                                     │
Data Access (Repositories: Web + DB) ─────────┘
```

Reglas clave:
- Las **Views** no contienen lógica de negocio; son función del estado. Disparan efectos secundarios
  llamando a `Interactors` desde acciones del usuario o `onAppear`/`.task`.
- Los **Interactors** reciben peticiones de trabajo pero **nunca devuelven datos directamente**:
  escriben el resultado en `AppState` o en un `Binding` (vía `Loadable`).
- Los **Repositories** exponen operaciones CRUD asíncronas sobre red o base de datos local. No
  contienen lógica de negocio ni mutan `AppState`. Solo los usan los Interactors.
- Todo se inyecta vía `@Environment`, sin frameworks de DI externos.
- **[2026] Regla de dependencia hacia adentro**: las flechas de conocimiento solo apuntan hacia el
  dominio. La capa de negocio (Interactors/UseCases) no sabe nada de SwiftUI ni de SwiftData; la
  capa de datos (Repositories) no sabe nada de Views ni de `AppState`. Si se adoptan Swift Packages
  locales (ver §2), esta regla la impone el propio compilador.

---

## 1. Orden de implementación (roadmap para agentes)

Cada paso es una unidad de trabajo razonable para un agente dedicado. Los pasos están ordenados
por dependencia: no se puede implementar un paso sin que el anterior exista.

### Paso 1 — Utilidades base (sin dependencias de dominio)
Carpeta: `Utilities/` (o package `Core/Sources/Core/Utilities/` si se adoptan packages, ver §2)
1. `CancelBag.swift` — clase para agrupar y cancelar `Cancellable`/`Task` en bloque.
2. `Store.swift` — `typealias Store<State> = CurrentValueSubject<State, Never>` + helpers:
   - subscript por `WritableKeyPath` con chequeo de igualdad antes de emitir.
   - `bulkUpdate(_:)` para mutaciones múltiples en un solo emit.
   - `updates(for:)` → `AnyPublisher` filtrado por keyPath con `removeDuplicates()`.
   - extensión de `Binding` (`dispatched(to:_:)`, `onSet(_:)`) para enlazar bindings locales al Store.
   - **[2026]** Bajo Swift 6 en modo estricto, `Store` debe aislarse a `@MainActor` (o marcarse
     `Sendable` explícitamente); `AppState` y sus sub-structs deben ser `Sendable`.
3. `Loadable.swift` — enum genérico `notRequested / isLoading(last:cancelBag:) / loaded / failed`.
   - `value`, `error` computed properties.
   - `setIsLoading(cancelBag:)`, `cancelLoading()`, `map(_:)`, `unwrap()` (para `Loadable<T?>`).
   - `LoadableSubject<T> = Binding<Loadable<T>>` + `load(_:)` que ejecuta una `Task` async y
     actualiza el binding con `.loaded`/`.failed`.
   - Conformar `Equatable` cuando `T: Equatable` (necesario para tests de UI y diffing de estado);
     **[2026]** conformar también `Sendable` cuando `T: Sendable`.
4. `Helpers.swift` — utilidades varias (p.ej. `ProcessInfo.isRunningTests`).

> Agente sugerido: "swiftui-clean-arch-utilities" — genera estos 4 archivos sin conocer aún el dominio.

### Paso 2 — AppState (fuente única de verdad)
Carpeta: `Core/AppState.swift`
- `struct AppState: Equatable, Sendable` con sub-structs anidados (**[2026]**: `Sendable` obligatorio
  bajo concurrencia estricta):
  - `ViewRouting` — un sub-struct de routing **por pantalla** (p.ej. `CountriesList.Routing`),
    todos `Equatable`, para navegación programática (deep links).
  - `System` — flags de ciclo de vida (`isActive`, `keyboardHeight`, etc).
  - `Permissions` — estado de permisos del sistema (push, cámara, ubicación...).
- Método estático `permissionKeyPath(for:)` que mapea un enum de permiso a su `WritableKeyPath`
  dentro de `AppState`, para poder observar/actualizar permisos genéricamente.
- `#if DEBUG static var preview: AppState { ... }` con datos de ejemplo para Previews.

> Este archivo se reescribe según el dominio de la app, pero la forma (routing/system/permissions
> anidados y Equatable) se mantiene igual.

### Paso 3 — Modelos de datos (tres niveles, no dos)
Carpeta: `Repositories/Models/` (o package `Data/Sources/Data/Models/` + `Core/Sources/Core/Entities/`)
- **[2026]** El repo original de nalexn separa solo `ApiModel`/`DBModel`. Aquí se adopta el modelo de
  tres niveles del template 2026, que aísla mejor el dominio:
  - `DTO` (`enum ApiModel { }` extendido, o `struct XDTO: Codable`) — el contrato de red tal cual
    llega del backend. Vive en `Repositories/WebAPI` o `Repositories/Models`.
  - **Entity de dominio** (`struct X` puro, sin `Codable`) — el tipo que usan Interactors y Views.
    No conoce nada de red ni de persistencia.
  - `DBModel` (`@Model final class DBModel.X`) — el modelo persistido con SwiftData.
- Mapeo explícito y unidireccional en `Mappers/`:
  - `DTO.toEntity() -> X` (Data → Dominio).
  - `X.dbModel() -> DBModel.X` (Dominio → Persistencia), y su inverso si se necesita rehidratar.
- `MockedData.swift` — datos de ejemplo reutilizables en Previews y Tests.
- Nunca reutilizar el mismo tipo para red, dominio y persistencia: mantiene desacoplada la capa de
  datos del esquema remoto y del dominio de negocio.

### Paso 4 — Capa de Red (WebAPI Repositories)
Carpeta: `Repositories/WebAPI/`
1. `WebRepository.swift`:
   - `protocol WebRepository { var session: URLSession { get }; var baseURL: String { get } }`
   - método genérico `call<Value: Decodable>(endpoint: APICall, decoder:, httpCodes:) async throws -> Value`
     usando `session.data(for:)` + validación de `HTTPURLResponse` contra `HTTPCodes` (`Range<Int>`).
   - `protocol APICall { path, method, headers, body() throws -> Data? }` + `urlRequest(baseURL:)`.
   - `enum APIError: LocalizedError` (`invalidURL`, `httpCode`, `unexpectedResponse`, ...).
2. **[2026] Alternativa moderna, protocol-oriented y con `actor`** (preferible en proyectos nuevos,
   del template 2026 §5.3):
   ```swift
   public protocol APIClient: Sendable {
       func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T
   }

   public actor URLSessionAPIClient: APIClient {
       private let session: URLSession
       private let decoder: JSONDecoder

       public init(session: URLSession = .shared, decoder: JSONDecoder = .init()) {
           self.session = session
           self.decoder = decoder
       }

       public func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
           let (data, response) = try await session.data(for: endpoint.urlRequest())
           guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
               throw AppError.network(underlying: "status code inválido")
           }
           do {
               return try decoder.decode(T.self, from: data)
           } catch {
               throw AppError.decoding
           }
       }
   }
   ```
   El `actor` aísla la sesión y el decoder sin necesitar `@MainActor` ni locks manuales.
3. **[2026] Errores unificados en `AppError`** en vez de `APIError` suelto por repositorio:
   ```swift
   public enum AppError: LocalizedError, Sendable {
       case network(underlying: String)
       case decoding
       case notFound
       case unauthorized
       case unknown

       public var errorDescription: String? {
           switch self {
           case .network(let underlying): "No se pudo conectar: \(underlying)"
           case .decoding: "Los datos recibidos no son válidos."
           case .notFound: "No se encontró el recurso."
           case .unauthorized: "Sesión expirada, iniciá sesión de nuevo."
           case .unknown: "Ocurrió un error inesperado."
           }
       }
   }
   ```
4. Un repositorio concreto por dominio (`CountriesWebRepository` → `PokemonWebRepository`, etc.):
   - `protocol X: WebRepository { func ... async throws -> ... }`
   - `struct RealX: X { let session; let baseURL; init(session:) }`
   - Endpoints como `enum API: APICall` anidado dentro del repo `Real`.
5. Configurar `URLSession` centralizado en `AppEnvironment` (timeouts, `waitsForConnectivity`,
   `httpMaximumConnectionsPerHost`, `urlCache`).

### Paso 5 — Capa de Persistencia (DB Repositories con SwiftData)
Carpeta: `Repositories/Database/`
1. `ModelContainer.swift` — helper `ModelContainer.appModelContainer()` (schema real) y
   `ModelContainer.stub` (in-memory, para tests/previews/fallback si falla la migración).
2. `AppSchema.swift` — lista explícita de todos los `@Model` que forman el `Schema`.
3. Un `protocol XDBRepository` por dominio + implementación como `extension MainDBRepository: XDBRepository`
   (un único `MainDBRepository` con `let modelContainer: ModelContainer` central, extendido por
   dominio para mantener cada archivo enfocado en su propia entidad).
   - Operaciones típicas: `fetch`/`store` usando `FetchDescriptor` + `#Predicate`.
   - Mutaciones envueltas en `modelContext.transaction { ... }`.
   - Métodos que tocan `mainContext` marcados `@MainActor`.
   - **[2026]** Campos únicos con `@Attribute(.unique)` (ver `ItemModel` de ejemplo en el template);
     `ModelContext` no es `Sendable` — todo acceso cruza explícitamente por `@MainActor` o por un
     `ModelActor` dedicado, nunca compartiendo el contexto entre tareas concurrentes sin aislar.

### Paso 6 — Interactors / UseCases (lógica de negocio)
Carpeta: `Interactors/` (o `Domain/Sources/Domain/UseCases/` si se adoptan packages)
- Un protocolo + `RealX` + `StubX` por dominio de negocio:
  ```swift
  protocol XInteractor {
      func refreshX() async throws
      func loadXDetails(x: DBModel.X, forceReload: Bool) async throws -> DBModel.XDetails
  }
  struct RealXInteractor: XInteractor {
      let webRepository: XWebRepository
      let dbRepository: XDBRepository
      // orquesta: llama web → guarda en DB → relee de DB → devuelve
  }
  struct StubXInteractor: XInteractor {
      // implementaciones vacías/throw, usadas en Previews y como DIContainer.stub
  }
  ```
- Reglas: el Interactor decide la política de cache (`forceReload`), combina múltiples
  repositorios, y es el único lugar con lógica de negocio real. Nunca accede a SwiftUI.
- Todo método es `async throws`; nunca completion handlers ni `Result` como valor de retorno.
- `UserPermissionsInteractor` — patrón especial: toma `appState: Store<AppState>` y una closure
  (`openAppSettings`) para resolver y solicitar permisos del sistema, escribiendo el resultado
  directamente en `AppState.permissions`.
- **[2026] Interactor por dominio vs UseCase granular** (template §5.2): el patrón `XInteractor`
  con varios métodos (arriba) es válido y reduce boilerplate en apps chicas. En apps que van a
  crecer, se recomienda dividir cada acción de negocio en su propio UseCase de responsabilidad
  única:
  ```swift
  public protocol FetchXUseCase: Sendable {
      func execute() async throws -> [X]
  }

  public struct FetchXUseCaseImpl: FetchXUseCase {
      private let repository: XRepository
      public init(repository: XRepository) { self.repository = repository }
      public func execute() async throws -> [X] { try await repository.fetchX() }
  }
  ```
  Ambos estilos comparten reglas (nunca retornan por callback, siempre `async throws`, siempre con
  `Stub`/`Mock`); la elección es de granularidad, no de arquitectura.

### Paso 7 — Dependency Injection
Carpeta: `DependencyInjection/` (o package `DI/Sources/DI/`)
1. `DIContainer.swift`:
   - `struct DIContainer: Sendable { let appState: Store<AppState>; let interactors: Interactors }`
   - Sub-structs anidados `WebRepositories`, `DBRepositories`, `Interactors` (uno por dominio cada uno).
   - `Interactors.stub` estático que instancia todos los `StubX` (para Previews/Tests).
   - `extension EnvironmentValues { @Entry var injected: DIContainer = DIContainer(appState: AppState(), interactors: .stub) }`
   - `extension View { func inject(_ container: DIContainer) -> some View }`.
   - **[2026]** Añadir además los estáticos `.live` y `.preview` del template (junto al `.stub`
     existente), para separar composición real de datos de ejemplo en Previews:
     ```swift
     public extension DIContainer {
         static let live = DIContainer(/* wiring real */)
         static let preview = DIContainer(/* mocks ligeros para #Preview */)
     }
     ```
2. `AppEnvironment.swift`:
   - `@MainActor struct AppEnvironment { isRunningTests, diContainer, modelContainer, systemEventsHandler }`
   - `static func bootstrap() -> AppEnvironment` — punto único de composición:
     `appState → session → webRepositories → modelContainer → dbRepositories → interactors → diContainer
     → deepLinksHandler → pushNotificationsHandler → systemEventsHandler`.
   - Funciones privadas `configuredURLSession()`, `configuredWebRepositories(session:)`,
     `configuredModelContainer()`, `configuredDBRepositories(modelContainer:)`,
     `configuredInteractors(appState:webRepositories:dbRepositories:)`.

### Paso 8 — Núcleo de la App (Core)
Carpeta: `Core/`
1. `App.swift` — `@main struct`, guarda `let environment = AppEnvironment.bootstrap()` y aplica
   `.inject(environment.diContainer)` a la vista raíz; conecta el `AppDelegate`/`SceneDelegate`.
2. `AppDelegate.swift` — reenvía eventos del sistema (`didRegisterForRemoteNotifications`,
   `didReceiveRemoteNotification`, deep links de escena) al `SystemEventsHandler`.
3. `SystemEventsHandler.swift`:
   - `@MainActor protocol SystemEventsHandler` con métodos de ciclo de vida
     (`sceneDidBecomeActive`, `sceneWillResignActive`, `sceneOpenURLContexts`,
     `handlePushRegistration`, `appDidReceiveRemoteNotification`).
   - `RealSystemEventsHandler` observa teclado (`NotificationCenter` → `appState[\.system.keyboardHeight]`),
     re-solicita permisos al volver a foreground, y delega deep links a `DeepLinksHandler`.
4. `DeepLinksHandler.swift`:
   - `enum DeepLink: Equatable` con `init?(url:)` parseando `URLComponents`.
   - `@MainActor protocol DeepLinksHandler { func open(deepLink:) }`.
   - `RealDeepLinksHandler` traduce el deep link en mutaciones sobre `appState.routing`, con el
     truco de "reset routing → delay → aplicar nueva ruta" cuando ya hay una ruta activa (SwiftUI
     no soporta bien descartar+presentar simultáneo).
5. `PushNotificationsHandler.swift` — homólogo para push, delega a `DeepLinksHandler` cuando el
   payload trae uno.
6. **[2026] Router `@Observable` complementario** (template §8): para pantallas que usan
   `NavigationStack` con push/pop programático simple, se puede sumar un router ligero además de
   (o en lugar de) rutas granulares en `AppState.routing`:
   ```swift
   @MainActor
   @Observable
   public final class AppRouter {
       public var path = NavigationPath()

       public func push(_ route: Route) { path.append(route) }
       public func pop() { path.removeLast() }
       public func popToRoot() { path.removeLast(path.count) }
   }

   public enum Route: Hashable {
       case itemDetail(id: String)
       case settings
   }
   ```
   Sigue siendo compatible con `DeepLinksHandler`: un deep link puede empujar directamente al
   `path` del router sin recorrer la jerarquía de vistas. Usar `AppState.routing` cuando la
   navegación debe ser 100% testeable sin UI (deep links complejos, múltiples rutas simultáneas);
   usar `AppRouter` cuando basta con push/pop simple de una pila.

### Paso 9 — Capa de Presentación (Views)
Carpeta: `UI/<Feature>/` (o `Features/<Feature>/Sources/<Feature>/` si se adoptan packages)
- Una carpeta por feature/pantalla (`CountriesList/`, `CountryDetails/`), no por tipo de archivo.
- Patrón por vista raíz de feature (sin ViewModel explícito — el patrón original del repo):
  ```swift
  struct CountriesList: View {
      @State private var data: [Model] = []
      @State private(set) var state: Loadable<Void>
      @State private var routingState: Routing = .init()
      private var routingBinding: Binding<Routing> {
          $routingState.dispatched(to: injected.appState, \.routing.countriesList)
      }
      @Environment(\.injected) private var injected: DIContainer
  }
  ```
  - `Routing: Equatable` anidado en la vista, reflejado 1:1 en `AppState.ViewRouting`.
  - `content` como `@ViewBuilder` que hace `switch` sobre el `Loadable` (`notRequested/isLoading/loaded/failed`).
  - Efectos secundarios (`loadX(forceReload:)`) en un `extension` privado bajo `// MARK: - Side Effects`,
    siempre llamando `$state.load { try await injected.interactors.x.refreshX() }`.
  - Sincronización de estado (`// MARK: - State Updates`) vía `onReceive(injected.appState.updates(for:))`.
  - `ErrorView` reutilizable con `retryAction` para el caso `.failed`.
- **[2026] Alternativa con ViewModel `@Observable`**: para pantallas con estado local complejo
  (múltiples campos derivados, debounce, validación de formularios) es válido introducir un
  ViewModel explícito, siempre `@Observable` + `@MainActor`, nunca `ObservableObject`/`@Published`:
  ```swift
  @MainActor
  @Observable
  final class CountriesListViewModel {
      private(set) var state: Loadable<[Model]> = .notRequested
      private let interactor: CountriesInteractor
      init(interactor: CountriesInteractor) { self.interactor = interactor }
      func onAppear() async { /* ... */ }
  }
  ```
  Regla de decisión: si la vista solo dispara una carga y refleja un `Loadable`, el patrón
  `@State` + `@Environment(\.injected)` de arriba es suficiente y más simple. Si acumula lógica de
  presentación no trivial, se sube esa lógica a un `@Observable` ViewModel — nunca a la `View` ni a
  `ObservableObject`.
- `Common/` — componentes reutilizables (`ErrorView`, `ImageView` con cache, helpers de `Query`).
- `RootViewModifier.swift` — modifier aplicado a la vista raíz para reaccionar a `system.isActive`
  (blur al pasar a background, por privacidad).

### Paso 10 — Testing
Carpeta: `UnitTests/`
- **[2026] Swift Testing (`@Suite`, `@Test`, `#expect`) reemplaza a XCTest como framework
  principal** para tests unitarios. XCTest se conserva solo donde el framework aún lo exige
  (`XCUITest` para UI tests end-to-end). Ejemplo:
  ```swift
  import Testing
  @testable import Domain

  @Suite("XInteractor")
  struct XInteractorTests {
      @Test("refreshX guarda en DB lo que trae la web")
      func refreshXStoresFetchedData() async throws {
          let web = MockedXWebRepository(itemsToReturn: [.fixture()])
          let db = MockedXDBRepository()
          let sut = RealXInteractor(webRepository: web, dbRepository: db)

          try await sut.refreshX()

          #expect(db.storedItems.count == 1)
      }
  }
  ```

  | Tipo | Herramienta | Qué se testea |
  |---|---|---|
  | Interactors / UseCases | Swift Testing (`@Test`) | Lógica de negocio, con Repository mockeado |
  | Repositories | Swift Testing | Mapeo DTO → Entity → DBModel, manejo de errores, con `URLSession`/`APIClient` mockeado |
  | Views con estado local (`@Observable` ViewModel) | Swift Testing | Transiciones de `Loadable`, `isLoading`, mensajes de error |
  | UI end-to-end | XCUITest | Flujos críticos (login, checkout, etc.) |
  | Snapshot/inspección de vistas | ViewInspector (opcional) | Lógica condicional dentro del `body` |

1. `Mocks/` — mocks manuales (no frameworks): `MockedInteractors`, `MockedWebRepositories`,
   `MockedDBRepositories`, `MockedSystemEventsHandler`, `MockedSystemPermissions`.
   - `Mock.swift` — helper genérico para registrar/verificar invocaciones esperadas
     (`register(...)`, `verify()`), reescrito para Swift Testing en vez de `XCTestCase`.
2. `NetworkMocking/` — `RequestMocking.swift` + `MockedResponse.swift`: intercepta `URLProtocol`
   para servir respuestas fijas a los tests de `WebRepository`/`APIClient` sin red real.
3. `Repositories/` — tests de cada `WebRepository`/`DBRepository` contra datos mockeados.
4. `Mocks/Interactors/` — tests de cada Interactor/UseCase verificando que orquesta bien
   Web→DB y que el resultado final es el esperado.
5. `System/` — tests de `DeepLinksHandler` y `PushNotificationsHandler`.
6. `UI/` — tests de Views con **ViewInspector** (opcional, `https://github.com/nalexn/ViewInspector`):
   - patrón `Inspection<Self>` + `onReceive(inspection.notice)` insertado en cada vista para
     poder engancharse desde el test sin exponer estado interno.
7. `Utilities/` — tests de `Loadable`, `Helpers`, etc.

### Paso 11 — Calidad / CI (opcional pero presente en el repo original)
- `.swiftlint.yml` — reglas de estilo.
- Pipeline (`.travis.yml` en el original → adaptar a GitHub Actions): build + test + coverage
  (Codecov).
- **[2026]** Gate adicional: build debe pasar en modo de concurrencia estricta de Swift 6 **sin
  warnings** (ver checklist §"Calidad" más abajo).

---

## 2. Estructura de carpetas objetivo

**[2026]** Se adopta la estructura de **Swift Packages locales** del template 2026: el compilador
obliga a respetar los límites de capas (si Presentation intenta importar algo de Data
directamente, no compila). Mapeo respecto a las carpetas "clásicas" del repo original:

| Carpeta clásica | Package/carpeta nueva |
|---|---|
| `Utilities/` | `Core/Sources/Core/Utilities/` |
| `Core/AppState.swift` | `Core/Sources/Core/AppState.swift` |
| `Repositories/Models/` (Entities) | `Core/Sources/Core/Entities/` |
| `Repositories/WebAPI`, `Repositories/Database` | `Data/Sources/Data/{Remote,Local,Repositories,Mappers}/` |
| `Interactors/` | `Domain/Sources/Domain/UseCases/` (o `Interactors/`) |
| `DependencyInjection/` | `DI/Sources/DI/` |
| `UI/<Feature>/` | `Features/<Feature>/Sources/<Feature>/` |
| `UI/Common/` (componentes reutilizables) | `DesignSystem/Sources/DesignSystem/` |

```
MyApp/
├── MyApp.xcodeproj
├── App/
│   ├── MyAppApp.swift            # @main, guarda AppEnvironment.bootstrap(), aplica .inject(...)
│   ├── AppDelegate.swift
│   ├── SystemEventsHandler.swift
│   ├── DeepLinksHandler.swift
│   └── PushNotificationsHandler.swift
│
├── Packages/
│   ├── Core/                     # Sin dependencias de ningún otro módulo
│   │   ├── Sources/Core/
│   │   │   ├── Entities/         # Modelos de dominio puros (structs, sin Codable)
│   │   │   ├── Errors/           # AppError (enum LocalizedError, Sendable)
│   │   │   ├── Protocols/        # Protocolos de Repository e Interactor/UseCase
│   │   │   ├── AppState.swift
│   │   │   └── Utilities/        # CancelBag, Store, Loadable, Helpers
│   │   └── Tests/CoreTests/
│   │
│   ├── Domain/                   # Depende de Core
│   │   ├── Sources/Domain/
│   │   │   └── UseCases/         # o Interactors/ — implementaciones reales + Stub
│   │   └── Tests/DomainTests/    # Swift Testing, con mocks de Repository
│   │
│   ├── Data/                     # Depende de Core
│   │   ├── Sources/Data/
│   │   │   ├── Remote/           # APIClient/WebRepository, Endpoints, DTOs
│   │   │   ├── Local/            # SwiftData Models (DBModel), ModelContainer
│   │   │   ├── Repositories/     # Implementación real de los protocolos
│   │   │   └── Mappers/          # DTO → Entity, Entity → DBModel
│   │   └── Tests/DataTests/
│   │
│   ├── DesignSystem/             # Sin dependencias de negocio
│   │   └── Sources/DesignSystem/
│   │       ├── Components/       # ErrorView, ImageView, botones, etc.
│   │       ├── Tokens/           # Colores, tipografía, spacing
│   │       └── Extensions/
│   │
│   └── DI/                       # Depende de todo lo anterior
│       └── Sources/DI/
│           ├── DIContainer.swift
│           └── AppEnvironment.swift
│
└── Features/                     # Un módulo por feature (Presentation layer)
    ├── CountriesList/
    │   ├── Sources/CountriesList/
    │   │   ├── CountriesList.swift          # View
    │   │   ├── CountriesListViewModel.swift  # opcional, @Observable
    │   │   └── Components/
    │   └── Tests/CountriesListTests/
    └── CountryDetails/
        └── ...

UnitTests/ (o Tests/ por package si se separan como arriba)
├── Mocks/
│   ├── Interactors/
│   ├── NetworkMocking/
│   ├── Mock.swift
│   ├── MockedDBRepositories.swift
│   ├── MockedInteractors.swift
│   ├── MockedSystemEventsHandler.swift
│   ├── MockedSystemPermissions.swift
│   └── MockedWebRepositories.swift
├── Repositories/
├── System/
├── UI/
├── Utilities/
└── TestHelpers.swift
```

> Si el proyecto es muy chico (prototipo, MVP), está bien empezar con carpetas simples dentro de
> un solo target (la estructura "clásica" de la tabla de arriba) y migrar a packages locales
> cuando el equipo/código crezca. La separación conceptual (Core/Domain/Data/Presentation) debe
> respetarse igual en ambos casos — los packages solo la hacen cumplir en tiempo de compilación.

---

## 3. Flujo de datos de un caso de uso típico (ejemplo: cargar lista)

1. Vista aparece → `.task`/`onAppear` llama a `loadX(forceReload: false)` (o
   `viewModel.onAppear()` si la pantalla usa un ViewModel `@Observable`).
2. `loadX` invoca `$state.load { try await injected.interactors.x.refreshX() }`
   (definido en `Loadable.swift`), que marca `state = .isLoading` y lanza una `Task`.
3. El Interactor/UseCase (`RealXInteractor.refreshX()` o `FetchXUseCaseImpl.execute()`) llama al
   `WebRepository`/`APIClient` para obtener datos:
   - `APIClient.request(endpoint) async throws -> [XDTO]` (ver Paso 4).
   - Un `Mapper` convierte `[XDTO] → [X]` (entidad de dominio, Paso 3).
   - El Interactor persiste el resultado vía `DBRepository.store(...)`, mapeando `X → DBModel.X`.
4. La vista lee los datos reales no del resultado del Interactor sino de una `@Query` de SwiftData
   (o de `AppState`, según el volumen de datos — ver "Business Logic Layer" del README) — el
   Interactor solo confirma éxito/fracaso vía `Loadable`.
5. Al completar, `state = .loaded(())` o `.failed(error)` (con `error` tipado como `AppError`), y la
   vista reacciona vía el `switch`, traduciendo el error a un mensaje amigable
   (`error.errorDescription`).
6. Cualquier navegación/routing pasa por `AppState.routing` o por el `AppRouter` (Paso 8), nunca por
   `@State` de navegación fuera de ese circuito (excepto el `NavigationPath` local, sincronizado
   con el routing vía `onChange`).

---

## 4. Decisiones de diseño a replicar (y por qué)

- **AppState como Redux-like single source of truth**: permite testear routing y navegación
  programática (deep links) sin UI, y permite que `SystemEventsHandler`/`DeepLinksHandler`
  actúen sobre la app sin acoplarse a vistas concretas.
- **Interactors nunca retornan datos directamente**: fuerza a que el resultado sea observable
  (`AppState` o `Binding`), lo cual hace la UI reactiva y testeable sin mocks de callbacks.
  - Datos pequeños/de configuración → viven en `AppState`.
  - Datos masivos → viven en SwiftData, consultados on-demand vía `@Query`/`Binding`.
- **Loadable<T>**: unifica el manejo de estados de carga/error/éxito y soporta cancelación
  cooperativa vía `CancelBag`, evitando duplicar `isLoading`/`error`/`data` en cada vista.
- **DIContainer + @Environment**: evita frameworks de DI de terceros; todo el grafo de
  dependencias se construye una sola vez en `AppEnvironment.bootstrap()` y se inyecta por
  `Environment`, permitiendo swaps triviales a `Stub`/`Mock`/`.preview` en Previews y Tests.
- **Separación DTO/Entity/DBModel**: evita que cambios en el contrato de red rompan el esquema de
  persistencia o el dominio de negocio (y viceversa); el mapeo vive explícitamente en `Mappers/`.
- **Stubs junto a cada protocolo real**: cada `XInteractor`/`XWebRepository` trae su
  implementación fake, manteniendo Previews y `DIContainer.stub`/`.preview` triviales de construir.
- **[2026] Swift 6 en modo estricto desde el día uno**: detectar problemas de concurrencia
  (data races, captura de estado mutable no aislado) en tiempo de compilación en vez de en
  producción; migrarlo después sobre una base grande es mucho más caro que adoptarlo desde el
  inicio.
- **[2026] `@Observable` sobre `ObservableObject`**: menos boilerplate (`@Published` no aplica solo
  a properties leídas), refresco de vista más granular (solo se redibuja lo que realmente cambió),
  y es el camino que Apple mantiene activamente para SwiftUI moderno.
- **[2026] Errores tipados (`AppError`)**: un único punto de traducción error→mensaje de usuario,
  en vez de manejar `Error` genérico disperso por toda la capa de presentación.
- **[2026] Swift Packages locales**: el compilador impone los límites de capa — Presentation no
  puede importar Data por accidente. También acelera el build incremental al aislar módulos.

---

## 5. Convenciones de código

**[2026]** (tomado del template 2026, §10)

- **Naming**: `XxxUseCase`/`XxxInteractor` (protocolo) / `XxxUseCaseImpl`/`RealXxxInteractor`
  (implementación real) — mismo patrón para `Repository`/`RealRepository`.
- **Un archivo por tipo.** Nada de "God files" con varios structs/clases adentro.
- **Nada de lógica en las Views.** Si una `View` tiene un `if` que decide una llamada de red, está
  mal ubicado — debe vivir en el Interactor/UseCase o, a lo sumo, en un ViewModel `@Observable`.
- **Todo async es `async throws`**, nunca completion handlers ni `Result` como valor de retorno de
  funciones async.
- **`Sendable` por defecto.** Con concurrencia estricta de Swift 6, el compilador obliga a esto de
  todas formas.
- **Sin Singletons mutables globales** (`.shared` con estado mutable). Usar DI (`DIContainer`).

---

## 6. Checklist de calidad antes de dar por terminado un feature

**[2026]** (tomado del template 2026, §11)

- [ ] ¿La View tiene cero lógica de negocio?
- [ ] Si hay ViewModel, ¿es `@Observable` y corre en `@MainActor`? (nunca `ObservableObject`/`@Published`)
- [ ] ¿Los errores están tipados (`AppError` o específicos del dominio)?
- [ ] ¿Existe una implementación `Stub`/`Mock` de cada protocolo usado en tests/previews?
- [ ] ¿Compila con Swift 6 en modo de concurrencia estricta sin warnings?
- [ ] ¿Hay al menos un test con Swift Testing para el Interactor/UseCase principal?
- [ ] ¿El Interactor sigue sin devolver datos directamente (escribe en `AppState`/`Loadable`)?
- [ ] ¿La separación DTO/Entity/DBModel sigue intacta (sin tipos reusados entre capas)?

---

## 7. Cómo trocear esto en agentes

Sugerencia de agentes/skills especializados, cada uno operando sobre un paso del roadmap y
capaz de reutilizarse entre proyectos distintos (solo cambia el "dominio"). Nombres actualizados
al árbol de packages de §2:

1. `scaffold-core-package` → Paso 1 + 2 (Store, Loadable, CancelBag, Helpers, AppState) dentro del
   package `Core`.
2. `scaffold-entities` → Paso 3, a partir de un contrato de API (OpenAPI/JSON de ejemplo) genera
   `DTO`/`Entity`/`DBModel` + `Mappers/`.
3. `scaffold-data-package` → Paso 4 + 5, a partir de una lista de endpoints y las `Entity`
   generadas: `APIClient`/`WebRepository`, `DBRepository` con SwiftData.
4. `scaffold-usecase` → Paso 6, combina un Web + DB repository dados (como Interactor único o como
   UseCases granulares, según lo decidido con el equipo).
5. `scaffold-di` → Paso 7, genera `DIContainer` (`.live`/`.preview`/`.stub`) + `AppEnvironment` a
   partir del inventario de repos/interactors ya creados.
6. `scaffold-app-core` → Paso 8 (App, AppDelegate, SystemEventsHandler, DeepLinksHandler,
   `AppRouter` opcional).
7. `scaffold-feature` → Paso 9, a partir de un Interactor/UseCase + wireframe simple, genera la
   vista (con o sin ViewModel `@Observable`) con `Loadable` y `Routing`, dentro de su propio
   package en `Features/`.
8. `scaffold-tests-swift-testing` → Paso 10, genera mocks + tests con Swift Testing para cualquier
   protocolo (Interactor/UseCase, WebRepository, DBRepository) dado.

Cada agente debe recibir como contexto mínimo: nombre del dominio/feature, los tipos de datos
involucrados, y los archivos ya generados en pasos previos (para mantener nombres consistentes).

**Orden recomendado de construcción** (para pedirle a un agente paso a paso, alineado con el
template 2026 §11):
1. Estructura de carpetas + Swift Packages locales vacíos con sus `Package.swift`.
2. `Core`: entidades + protocolos + `AppState` + `AppError` del primer feature.
3. `Data`: DTOs, `APIClient`/`WebRepository`, SwiftData models, Repository real + Stub.
4. `Domain`: Interactor/UseCase real + Stub.
5. `DI`: `DIContainer` con configuración `.live`/`.preview`/`.stub`.
6. `Features/<Feature>`: View (+ ViewModel `@Observable` si aplica).
7. Tests con Swift Testing para Interactor/UseCase y Repository.
8. `AppRouter`/deep links si el feature lo requiere.
9. Repetir 2–8 para cada feature nuevo, reutilizando `Core`/`DesignSystem`/`DI`.

---

## 8. Referencias

- Repo base: https://github.com/nalexn/clean-architecture-swiftui (rama `master`, revamp 2024).
- Clean Architecture for SwiftUI — Alexey Naumov: https://nalexn.github.io/clean-architecture-swiftui/
- Template de arquitectura 2026 (fuente de las secciones `[2026]`): `architecture-ios-template.md`.
- Repo de referencia modular con Swift 6 + Sendable: https://github.com/codetoanbug/Clean-architecture-SwiftUI
- Documentación oficial de Observation: https://developer.apple.com/documentation/observation
- Documentación oficial de SwiftData: https://developer.apple.com/documentation/swiftdata
- Documentación oficial de Swift Testing: https://developer.apple.com/documentation/testing
