# Pokédex — Plan de construcción por specs

> Documento vivo. Cada sección es una **spec ejecutable**: se implementa, se verifica con su
> criterio de aceptación y se marca. Arquitectura de referencia:
> `architecture-ios-template.md`. Resumen operativo: `CLAUDE.md` en la raíz.

---

## 0. Contexto y decisiones

Punto de partida: plantilla vacía de Xcode (target único `Pokedex`, `ContentView.swift`
"Hello, world!", `Assets.xcassets` vacío, sin target de tests).

| Decisión | Valor | Motivo |
|---|---|---|
| Alcance | MVP: lista paginada + detalle + cache offline | Ejercita las 3 capas completas sin inflar el scope |
| Módulos | Carpetas dentro del target único | pbxproj es objectVersion 77 con `PBXFileSystemSynchronizedRootGroup`: las carpetas nuevas se incluyen solas, sin tocar el project file. Migrable a Swift Packages locales después |
| Diseño | Tokens propios: 18 colores de tipo + artwork oficial de PokeAPI | Cero assets o fuentes que conseguir; íconos vía SF Symbols |
| Tests | Sin target de tests en esta iteración | Decisión explícita del usuario. Se aparta del checklist de `CLAUDE.md`; los mocks igual se escriben (los usan los `#Preview`), así que agregar el target después es trivial |

**API**: PokeAPI — `https://pokeapi.co/api/v2`. Pública, sin API key, HTTPS (no requiere ATS).

- `GET /pokemon?limit=&offset=` → `{ count, next, results: [{ name, url }] }`
- `GET /pokemon/{id}` → `id, name, height, weight, types[], stats[], abilities[], sprites`

⚠️ El endpoint de lista **no devuelve tipos ni sprite**: hay que resolver el detalle de cada
Pokémon de la página en paralelo (ver Spec 04).

**Fuera de alcance** (iteraciones siguientes, reutilizando Core/DesignSystem/DI): búsqueda,
filtros por tipo, favoritos, cadena de evoluciones, movimientos, regiones, comparador.

---

## Spec 00 — Preparación del proyecto

- [ ] `SWIFT_VERSION` de `5.0` → `6.0` en Debug y Release (`Pokedex.xcodeproj/project.pbxproj`).
- [ ] `SWIFT_STRICT_CONCURRENCY = complete` en ambas configs.
- [ ] `IPHONEOS_DEPLOYMENT_TARGET = 26.0` — se deja como está.
- [ ] Borrar `Pokedex/ContentView.swift`.
- [ ] Mover `PokedexApp.swift` a `Pokedex/App/`.
- [ ] Crear el árbol de carpetas de la sección siguiente.

**Aceptación**: el proyecto compila vacío con Swift 6 estricto y sin warnings.

### Árbol de carpetas (dentro de `Pokedex/`)

```
App/            PokedexApp.swift, AppRootView.swift
Core/
  Entities/     Pokemon.swift, PokemonSummary.swift, PokemonType.swift, PokemonStat.swift, PokemonPage.swift
  Errors/       AppError.swift
  Protocols/    PokemonRepository.swift, FetchPokemonPageUseCase.swift, FetchPokemonDetailUseCase.swift
Data/
  Remote/       APIClient.swift, Endpoint.swift, PokedexEndpoint.swift, URLSessionAPIClient.swift
  Remote/DTOs/  PokemonListResponseDTO.swift, PokemonDetailDTO.swift
  Local/        PokemonModel.swift, PokemonLocalDataSource.swift, ModelContainer+Pokedex.swift
  Repositories/ PokemonRepositoryImpl.swift
  Mappers/      PokemonDetailDTO+Mapping.swift, PokemonModel+Mapping.swift
Domain/
  UseCases/     FetchPokemonPageUseCaseImpl.swift, FetchPokemonDetailUseCaseImpl.swift
DesignSystem/
  Tokens/       PokedexColor.swift, PokedexSpacing.swift, PokedexTypography.swift
  Components/   TypeBadge.swift, PokemonCardView.swift, StatBarView.swift, RemoteImageView.swift,
                LoadingStateView.swift, ErrorStateView.swift
DI/             DIContainer.swift, EnvironmentValues+DIContainer.swift
Presentation/
  Navigation/   AppRouter.swift, Route.swift
  PokemonList/  PokemonListView.swift, PokemonListViewModel.swift
  PokemonDetail/ PokemonDetailView.swift, PokemonDetailViewModel.swift
Mocks/          MockAPIClient.swift, MockPokemonRepository.swift, MockFetchPokemonPageUseCase.swift,
                MockFetchPokemonDetailUseCase.swift, Pokemon+Fixture.swift
```

Regla transversal: **un tipo por archivo**.

---

## Spec 01 — Core: entidades, errores, protocolos

Sin imports de SwiftUI ni SwiftData. Todo `Sendable`.

- [ ] `PokemonType`: `enum: String, CaseIterable, Sendable` con los 18 tipos
      (`normal`…`fairy`) + `unknown` como fallback de decoding.
- [ ] `PokemonStat`: `struct Sendable` — `kind` (hp, attack, defense, spAtk, spDef, speed), `value`.
- [ ] `PokemonSummary`: `struct Sendable, Identifiable, Hashable` — id, name, types, artworkURL.
      Es lo que consume la grilla.
- [ ] `Pokemon`: `struct Sendable, Identifiable, Hashable` — id, name, types, height (decímetros),
      weight (hectogramos), stats, abilities, artworkURL.
- [ ] `PokemonPage`: `{ items: [PokemonSummary], nextOffset: Int? }`.
- [ ] `AppError`: `enum LocalizedError, Sendable` — `.network(underlying:)`, `.decoding`,
      `.notFound`, `.offline`, `.unknown`; mensajes en español (ver template §5.1).
- [ ] Protocolos `Sendable`:
      `PokemonRepository { fetchPage(offset:limit:) async throws -> PokemonPage;
      fetchDetail(id:) async throws -> Pokemon }`,
      `FetchPokemonPageUseCase`, `FetchPokemonDetailUseCase`.

**Aceptación**: Core compila aislado, sin conocer capas externas.

---

## Spec 02 — Data / Remote: networking

- [ ] `Endpoint`: struct con `path`, `queryItems`, `method`; `urlRequest()` armado con
      `URLComponents` sobre la baseURL.
- [ ] `PokedexEndpoint`: factories `.pokemonList(offset:limit:)`, `.pokemonDetail(id:)`.
- [ ] `APIClient` (protocolo `Sendable`) + `URLSessionAPIClient` como `actor` (template §5.3):
      valida status 2xx, mapea `URLError.notConnectedToInternet` → `.offline`,
      404 → `.notFound`, `DecodingError` → `.decoding`, resto → `.network`.
- [ ] DTOs `Decodable, Sendable` con `keyDecodingStrategy = .convertFromSnakeCase`.
      El artwork sale de `sprites.other["official-artwork"].front_default` — clave con guiones,
      requiere `CodingKeys` explícito; fallback: construir la URL a partir del id.

**Aceptación**: un `#Preview` o un scratch call devuelve el DTO de `/pokemon/1` decodificado.

---

## Spec 03 — Data / Local: SwiftData

- [ ] `PokemonModel`: `@Model final class` con `@Attribute(.unique) var id: Int`, name,
      `typeRawValues: [String]`, height, weight, `statValues: [Int]`, `artworkURLString`, `updatedAt`.
- [ ] `PokemonLocalDataSource`: **`@ModelActor`** (obligatorio: `ModelContext` no es `Sendable`
      bajo concurrencia estricta) con `save(_:)`, `page(offset:limit:)`, `pokemon(id:)`.
- [ ] `ModelContainer+Pokedex.swift`: helper `static func pokedex(inMemory:)` para live y previews.

**Aceptación**: guardar y releer un `Pokemon` sobrevive a un reinicio de la app.

---

## Spec 04 — Data / Repository + Mappers

- [ ] Mappers en archivos aparte: `PokemonDetailDTO → Pokemon`, `Pokemon ↔ PokemonModel`.
- [ ] `PokemonRepositoryImpl` (struct `Sendable`), estrategia **cache-first**:
  - `fetchPage`: si hay cache fresco para ese rango, lo devuelve. Si no, pide
    `/pokemon?limit&offset`, resuelve los detalles de la página **en paralelo con
    `withThrowingTaskGroup`** (el endpoint de lista no trae tipos ni sprite), persiste vía el
    `@ModelActor` y devuelve `PokemonPage`.
  - Si la red falla **y hay cache**, devuelve el cache sin lanzar error.
  - `fetchDetail(id:)`: local primero, red después.

**Aceptación**: la lista se sirve desde disco en modo avión.

---

## Spec 05 — Domain: casos de uso

- [ ] `FetchPokemonPageUseCaseImpl` y `FetchPokemonDetailUseCaseImpl`: structs finos que
      delegan en el repositorio (template §5.2). Un caso de uso por acción.

**Aceptación**: los ViewModels no conocen `PokemonRepository`, solo los UseCases.

---

## Spec 06 — DI

- [ ] `DIContainer` (struct `Sendable`): apiClient, localDataSource, repository y los dos UseCases.
      `.live` y `.preview` (este último con `MockAPIClient` + contenedor SwiftData en memoria).
- [ ] `EnvironmentValues+DIContainer.swift`: `@Entry var diContainer: DIContainer`.

**Aceptación**: cero singletons mutables; toda dependencia entra por inicializador o `@Environment`.

---

## Spec 07 — DesignSystem

- [ ] `PokedexColor`: color canónico por cada `PokemonType` (definidos en código con
      `Color(red:green:blue:)`, sin assets) + superficies y texto que respetan light/dark.
- [ ] `PokedexSpacing`: escala 4 / 8 / 12 / 16 / 24 / 32.
- [ ] `PokedexTypography`: `Font` semánticas con Dynamic Type.
- [ ] Componentes **sin lógica de negocio**: `TypeBadge`, `PokemonCardView` (gradiente del tipo
      primario + artwork + número formateado `#0001`), `StatBarView`, `RemoteImageView`
      (wrapper de `AsyncImage` con placeholder), `LoadingStateView`, `ErrorStateView`
      (mensaje + botón reintentar).

**Aceptación**: cada componente tiene `#Preview` y se ve correcto en claro y oscuro.

---

## Spec 08 — Presentation

- [ ] `Route: Hashable` (`case detail(id: Int)`) y `AppRouter` `@MainActor @Observable`
      con `push` / `pop` / `popToRoot` (template §8).
- [ ] `PokemonListViewModel` `@MainActor @Observable`: `pokemon`, `isLoading`, `isLoadingMore`,
      `errorMessage`; `onAppear()` y `loadMoreIfNeeded(currentItem:)` (páginas de 30).
- [ ] `PokemonListView`: `NavigationStack(path:)` + `LazyVGrid` de 2 columnas de `PokemonCardView`,
      `.task`, overlays de loading/error, `navigationDestination(for: Route.self)`.
- [ ] `PokemonDetailViewModel` + `PokemonDetailView`: header con artwork sobre gradiente del tipo,
      badges, altura/peso formateados (dm→m, hg→kg), `StatBarView` por stat, habilidades.
- [ ] `AppRootView` arma el router y la vista raíz; `PokedexApp` crea `DIContainer.live` +
      `ModelContainer` y los inyecta por `@Environment`.
- [ ] Todas las Views con `#Preview` usando `DIContainer.preview`.

**Aceptación**: cero lógica de negocio en las Views; los ViewModels son `@Observable` en `@MainActor`.

---

## Spec 09 — Mocks

- [ ] `MockAPIClient` (respuestas JSON fijas), `MockPokemonRepository`, mocks de los dos UseCases
      con caso éxito y caso error, `Pokemon.fixture()`.
- [ ] Viven en el target de la app porque los consumen los `#Preview`.

**Aceptación**: un mock por cada protocolo de Core (checklist de `CLAUDE.md`).

---

## Verificación end-to-end

1. **Build estricto sin warnings**
   `xcodebuild -project Pokedex.xcodeproj -scheme Pokedex -destination 'platform=iOS Simulator,name=iPhone 17' build`
2. **Camino feliz**: la grilla carga los primeros 30 Pokémon con sprite y color de tipo;
   el scroll infinito trae la página siguiente; el tap abre el detalle con stats; volver funciona.
3. **Offline**: activar Airplane Mode, matar y reabrir la app → la lista sale de SwiftData
   en vez de mostrar error.
4. **Accesibilidad**: modo oscuro y Dynamic Type grande no rompen el layout.

---

## Checklist final (de `CLAUDE.md`)

- [ ] Views sin lógica de negocio.
- [ ] ViewModels `@Observable` en `@MainActor`.
- [ ] Errores tipados (`AppError`).
- [ ] Mock de cada protocolo usado en previews.
- [ ] Compila con Swift 6 estricto sin warnings.
- [ ] ~~Al menos un test Swift Testing por UseCase~~ — **diferido por decisión del usuario**
      (sin target de tests en esta iteración).
