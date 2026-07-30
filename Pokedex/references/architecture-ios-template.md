# Arquitectura iOS — Clean Architecture + MVVM (Stack 2026)

> Documento de referencia para construir nuevos proyectos iOS con Claude Code.
> Basado en el análisis de `nalexn/clean-architecture-swiftui` (rama `master`, revamp 2024)
> y `codetoanbug/Clean-architecture-SwiftUI`, actualizado con las herramientas nativas
> actuales de Apple (Swift 6, Observation, SwiftData, Swift Testing).

---

## 1. Objetivo

Definir una arquitectura de referencia, consistente y reproducible, para apps iOS nuevas que:

- Separe claramente **Presentación**, **Dominio (lógica de negocio)** y **Datos**.
- Use exclusivamente herramientas **nativas de Apple** (sin RxSwift, Alamofire, Swinject, etc.,
  salvo que el proyecto lo justifique explícitamente).
- Sea 100% testeable con **Swift Testing**.
- Adopte **Swift 6 en modo de concurrencia estricta** desde el día uno.
- Sirva como plantilla replicable: cualquier feature nueva sigue el mismo patrón.

---

## 2. Stack tecnológico

| Área | Herramienta | Notas |
|---|---|---|
| Lenguaje | Swift 6.2 | Modo de concurrencia estricta (`Sendable`, actors) activado desde el inicio |
| UI | SwiftUI | Sin UIKit salvo interoperabilidad puntual |
| Estado observable | `@Observable` (Observation framework) | **No** usar `ObservableObject` / `@Published` en código nuevo |
| Concurrencia | `async/await`, `actor`, `Task`, `TaskGroup` | Nada de Combine para networking; Combine solo si se necesita para casos puntuales (ej. debounce de búsqueda) |
| Networking | `URLSession` + async/await | Capa propia, protocol-oriented, sin librerías de terceros |
| Persistencia | `SwiftData` | Reemplaza CoreData; usar `@Model`, `ModelContainer`, `ModelContext` |
| Navegación | `NavigationStack` + Router/Coordinator propio | Navegación programática, deep-linking |
| Testing unitario | **Swift Testing** (`@Test`, `#expect`) | XCTest solo para UI Tests (`XCUITest`) que aún lo requieren |
| Testing de vistas | `ViewInspector` (opcional) | Para validar lógica condicional de SwiftUI Views |
| DI | Contenedor propio + `@Environment` | Sin frameworks (Swinject, Factory) salvo justificación |
| Mínimo iOS | iOS 17+ (idealmente 18+) | Necesario para `@Observable` y SwiftData maduro |
| Gestión de dependencias | Swift Package Manager | Módulos locales (local Swift Packages) para forzar límites de capas |

---

## 3. Visión general de la arquitectura

Mantenemos el modelo de 3 capas de Clean Architecture (Presentation → Domain → Data),
pero con el **estado centralizado opcional** inspirado en el patrón Redux-like de nalexn,
usado solo cuando el dato es compartido por múltiples pantallas.

```
┌──────────────────────────────────────────────────────────────┐
│                      PRESENTATION LAYER                       │
│  SwiftUI Views (sin lógica) + ViewModels (@Observable)         │
│  Reciben Interactors/UseCases vía @Environment                 │
└───────────────────────────┬────────────────────────────────────┘
                             │ acciones del usuario (tap, onAppear)
┌───────────────────────────▼────────────────────────────────────┐
│                    DOMAIN LAYER (Business Logic)               │
│  UseCases / Interactors (protocolo + implementación real +      │
│  implementación mock para tests y previews)                     │
│  Entidades de dominio (structs puros, sin Codable de red)        │
└───────────────────────────┬────────────────────────────────────┘
                             │ async/await
┌───────────────────────────▼────────────────────────────────────┐
│                       DATA LAYER                                │
│  Repositories (protocolo + implementación)                       │
│  ├── RemoteDataSource → URLSession + async/await (DTOs Codable)  │
│  └── LocalDataSource  → SwiftData (@Model)                        │
│  Mappers: DTO ↔ Entidad de dominio ↔ @Model                       │
└──────────────────────────────────────────────────────────────┘
```

**Regla de dependencia:** las flechas de conocimiento solo apuntan hacia adentro.
La capa de Dominio no sabe nada de SwiftUI ni de SwiftData. La capa de Datos no sabe
nada de ViewModels.

---

## 4. Estructura de carpetas / módulos

Usar **Swift Packages locales** (no solo carpetas) para que el compilador *obligue* a
respetar los límites de capas — si Presentation intenta importar algo de Data
directamente, no compila.

```
MyApp/
├── MyApp.xcodeproj
├── App/
│   ├── MyAppApp.swift          # @main, arma el DIContainer raíz
│   └── AppRootView.swift       # Punto de entrada visual + Router raíz
│
├── Packages/
│   ├── Core/                   # Sin dependencias de ningún otro módulo
│   │   ├── Sources/Core/
│   │   │   ├── Entities/       # Modelos de dominio puros (structs)
│   │   │   ├── Errors/         # AppError, DomainError (enum LocalizedError)
│   │   │   └── Protocols/      # Protocolos de Repository y UseCase
│   │   └── Tests/CoreTests/
│   │
│   ├── Domain/                 # Depende de Core
│   │   ├── Sources/Domain/
│   │   │   └── UseCases/       # Implementaciones reales de los protocolos de Core
│   │   └── Tests/DomainTests/  # Swift Testing, con mocks de Repository
│   │
│   ├── Data/                   # Depende de Core
│   │   ├── Sources/Data/
│   │   │   ├── Remote/         # APIClient, Endpoints, DTOs
│   │   │   ├── Local/          # SwiftData Models, ModelContainer setup
│   │   │   ├── Repositories/   # Implementación real de los protocolos
│   │   │   └── Mappers/        # DTO → Entity, @Model → Entity
│   │   └── Tests/DataTests/
│   │
│   ├── DesignSystem/           # Sin dependencias de negocio
│   │   └── Sources/DesignSystem/
│   │       ├── Components/     # Botones, TextFields, etc. reutilizables
│   │       ├── Tokens/         # Colores, tipografía, spacing
│   │       └── Extensions/
│   │
│   └── DI/                     # Depende de todo lo anterior
│       └── Sources/DI/
│           └── DIContainer.swift
│
└── Features/                   # Un módulo por feature (Presentation layer)
    ├── ItemList/
    │   ├── Sources/ItemList/
    │   │   ├── ItemListView.swift
    │   │   ├── ItemListViewModel.swift    # @Observable
    │   │   └── Components/
    │   └── Tests/ItemListTests/
    └── ItemDetail/
        └── ...
```

> Si el proyecto es muy chico (prototipo, MVP), está bien empezar con carpetas simples
> dentro de un solo target y migrar a packages locales cuando el equipo/código crezca.
> La separación conceptual (Core/Domain/Data/Presentation) debe respetarse igual.

---

## 5. Capa por capa

### 5.1 Core (Entidades, Errores, Protocolos)

- Contiene los `struct` de dominio, **sin** `Codable` de red mezclado (eso vive en los DTOs de Data).
- Define los `protocol` de Repository y UseCase — Domain los implementa, Data también los implementa
  para el lado de repositorios.
- Errores tipados por dominio:

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

### 5.2 Domain (Casos de uso / Interactors)

- Un `UseCase` por acción de negocio (Single Responsibility). Ejemplo: `FetchItemsUseCase`,
  no un `ItemsInteractor` gigante con 10 métodos (aceptable en apps chicas, pero preferí
  UseCases granulares en apps que van a crecer).
- Siempre expuesto como protocolo, con una implementación real y una `Mock` para tests/previews.

```swift
public protocol FetchItemsUseCase: Sendable {
    func execute() async throws -> [Item]
}

public struct FetchItemsUseCaseImpl: FetchItemsUseCase {
    private let repository: ItemsRepository

    public init(repository: ItemsRepository) {
        self.repository = repository
    }

    public func execute() async throws -> [Item] {
        try await repository.fetchItems()
    }
}
```

### 5.3 Data (Repositories, Networking, Persistencia)

**Networking (protocol-oriented, sin librerías de terceros):**

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

**Persistencia con SwiftData:**

```swift
@Model
public final class ItemModel {
    @Attribute(.unique) public var id: String
    public var title: String
    public var updatedAt: Date

    public init(id: String, title: String, updatedAt: Date) {
        self.id = id
        self.title = title
        self.updatedAt = updatedAt
    }
}
```

**Repository (combina remoto + local, decide estrategia de cache):**

```swift
public protocol ItemsRepository: Sendable {
    func fetchItems() async throws -> [Item]
}

public struct ItemsRepositoryImpl: ItemsRepository {
    private let apiClient: APIClient
    private let modelContext: ModelContext

    public func fetchItems() async throws -> [Item] {
        let dtos: [ItemDTO] = try await apiClient.request(.items)
        let entities = dtos.map { $0.toEntity() }
        // Persistir localmente (opcional según el feature)
        return entities
    }
}
```

### 5.4 Presentation (Views + ViewModels)

- Las `View` **no** contienen lógica de negocio ni llamadas directas a repos.
- El ViewModel es `@Observable`, vive en `@MainActor`, y expone estado + funciones `async`.

```swift
@MainActor
@Observable
public final class ItemListViewModel {
    public private(set) var items: [Item] = []
    public private(set) var isLoading = false
    public var errorMessage: String?

    private let fetchItemsUseCase: FetchItemsUseCase

    public init(fetchItemsUseCase: FetchItemsUseCase) {
        self.fetchItemsUseCase = fetchItemsUseCase
    }

    public func onAppear() async {
        isLoading = true
        defer { isLoading = false }
        do {
            items = try await fetchItemsUseCase.execute()
        } catch {
            errorMessage = (error as? LocalizedError)?.errorDescription ?? "Error inesperado"
        }
    }
}
```

```swift
public struct ItemListView: View {
    @State private var viewModel: ItemListViewModel

    public init(viewModel: ItemListViewModel) {
        _viewModel = State(initialValue: viewModel)
    }

    public var body: some View {
        List(viewModel.items) { item in
            Text(item.title)
        }
        .task { await viewModel.onAppear() }
        .overlay { if viewModel.isLoading { ProgressView() } }
        .alert("Error", isPresented: .constant(viewModel.errorMessage != nil)) {
            Button("OK") { viewModel.errorMessage = nil }
        } message: {
            Text(viewModel.errorMessage ?? "")
        }
    }
}
```

---

## 6. Flujo completo de un feature (ejemplo: listar items)

```
1. ItemListView aparece → .task { await viewModel.onAppear() }
2. ViewModel llama a FetchItemsUseCase.execute()
3. UseCase llama a ItemsRepository.fetchItems()
4. Repository llama a APIClient.request(.items) → URLSession async/await
5. Respuesta se decodifica a [ItemDTO]
6. Mapper convierte [ItemDTO] → [Item] (entidad de dominio)
7. (Opcional) Repository persiste en SwiftData vía ModelContext
8. UseCase devuelve [Item] al ViewModel
9. ViewModel actualiza `items` (@Observable dispara refresco de la View)
10. View se redibuja automáticamente
```

Si hay un error en cualquier paso, se propaga como `AppError` (tipado) hasta el
ViewModel, que lo traduce a un mensaje amigable para el usuario.

---

## 7. Dependency Injection

Contenedor simple, sin frameworks, inyectado vía `@Environment` en el árbol de vistas:

```swift
public struct DIContainer: Sendable {
    public let apiClient: APIClient
    public let itemsRepository: ItemsRepository
    public let fetchItemsUseCase: FetchItemsUseCase

    public init(apiClient: APIClient = URLSessionAPIClient()) {
        self.apiClient = apiClient
        self.itemsRepository = ItemsRepositoryImpl(apiClient: apiClient)
        self.fetchItemsUseCase = FetchItemsUseCaseImpl(repository: itemsRepository)
    }

    public static let live = DIContainer()
    public static let preview = DIContainer(apiClient: MockAPIClient())
}
```

```swift
@main
struct MyAppApp: App {
    let container = DIContainer.live

    var body: some Scene {
        WindowGroup {
            ItemListView(viewModel: .init(fetchItemsUseCase: container.fetchItemsUseCase))
        }
    }
}
```

---

## 8. Navegación

`NavigationStack` con un `Router` propio (`@Observable`), evitando `NavigationLink`
dispersos por toda la app:

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

Esto permite deep-linking (ej. desde push notifications) empujando rutas directamente
al `path` sin recorrer la jerarquía de vistas.

---

## 9. Estrategia de testing

| Tipo | Herramienta | Qué se testea |
|---|---|---|
| Unit tests de UseCases | Swift Testing (`@Test`) | Lógica de negocio, con `Repository` mockeado |
| Unit tests de Repositories | Swift Testing | Mapeo DTO → Entity, manejo de errores, con `APIClient` mockeado |
| Unit tests de ViewModels | Swift Testing | Transiciones de estado (`isLoading`, `errorMessage`) |
| UI tests | XCUITest | Flujos críticos end-to-end (login, checkout, etc.) |
| Snapshot/inspección de vistas | ViewInspector (opcional) | Lógica condicional dentro del `body` |

Ejemplo con Swift Testing:

```swift
import Testing
@testable import Domain

@Suite("FetchItemsUseCase")
struct FetchItemsUseCaseTests {
    @Test("devuelve los items del repositorio")
    func fetchesItems() async throws {
        let repo = MockItemsRepository(itemsToReturn: [.fixture()])
        let sut = FetchItemsUseCaseImpl(repository: repo)

        let result = try await sut.execute()

        #expect(result.count == 1)
    }
}
```

Cada protocolo de `Core` debe tener su `Mock` correspondiente en un target de test
utilities, reutilizable entre Domain, Data y Presentation.

---

## 10. Convenciones de código

- **Naming:** `XxxUseCase` (protocolo) / `XxxUseCaseImpl` (implementación) — mismo patrón
  para `Repository` / `RepositoryImpl`.
- **Un archivo por tipo.** Nada de "God files" con 5 structs adentro.
- **Nada de lógica en las Views.** Si una `View` tiene un `if` que decide una llamada de red,
  está mal ubicado.
- **Todo async es `async throws`,** nunca completion handlers ni `Result` como valor de retorno
  de funciones async.
- **Sendable por defecto.** Con concurrencia estricta de Swift 6, el compilador te va a obligar
  a esto de todas formas.
- **Sin Singletons mutables globales** (`.shared` con estado mutable). Usar DI.

---

## 11. Cómo usar este documento con Claude Code

Al iniciar el proyecto en Claude Code, pegar este archivo como contexto y pedir algo así:

> "Usando la arquitectura descripta en `arquitectura-ios-template.md`, generá la estructura
> inicial del proyecto Xcode con Swift Packages locales (Core, Domain, Data, DesignSystem, DI)
> y un primer feature de ejemplo llamado `[NOMBRE_FEATURE]` que [DESCRIPCIÓN]."

**Orden recomendado de construcción (para pedirle a Claude Code paso a paso):**

1. Estructura de carpetas + Swift Packages locales vacíos con sus `Package.swift`
2. Capa `Core`: entidades + protocolos + errores del primer feature
3. Capa `Data`: DTOs, APIClient, SwiftData models, Repository real + Mock
4. Capa `Domain`: UseCase real + Mock
5. `DIContainer` con configuración `live` y `preview`
6. Capa `Presentation`: ViewModel `@Observable` + View
7. Tests con Swift Testing para UseCase y Repository
8. Router/Navigation si el feature lo requiere
9. Repetir 2-8 para cada feature nuevo, reutilizando Core/DesignSystem/DI

**Checklist de calidad antes de dar por terminado un feature:**

- [ ] ¿La View tiene cero lógica de negocio?
- [ ] ¿El ViewModel es `@Observable` y corre en `@MainActor`?
- [ ] ¿Los errores están tipados (`AppError` o específicos del dominio)?
- [ ] ¿Existe una implementación `Mock` de cada protocolo usado en tests/previews?
- [ ] ¿Compila con Swift 6 en modo de concurrencia estricta sin warnings?
- [ ] ¿Hay al menos un test de Swift Testing para el UseCase principal?

---

## 12. Referencias

- Clean Architecture for SwiftUI — Alexey Naumov: https://nalexn.github.io/clean-architecture-swiftui/
- Repo base (Clean Architecture, rama `master`): https://github.com/nalexn/clean-architecture-swiftui
- Repo de referencia modular con Swift 6 + Sendable: https://github.com/codetoanbug/Clean-architecture-SwiftUI
- Documentación oficial de Observation: https://developer.apple.com/documentation/observation
- Documentación oficial de SwiftData: https://developer.apple.com/documentation/swiftdata
- Documentación oficial de Swift Testing: https://developer.apple.com/documentation/testing
