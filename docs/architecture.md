# Architecture Guide – consistency-flutter

This document standardizes architectural decisions for the `consistency-flutter` project.
It is derived from the `clean-flutter-app` reference repository and adapted for this project's context.

---

## 1. Overview

The project follows **Clean Architecture** with a strict separation between domain, data, infra, presentation, UI, and composition layers.
State management is done via **GetX**. Dependency injection is **manual**, using factory functions.

---

## 2. Layer Structure

```
lib/
├── domain/
│   ├── entities/
│   ├── usecases/
│   └── helpers/
├── data/
│   ├── usecases/
│   ├── models/
│   └── http/
├── infra/
│   └── http/
├── presentation/
│   ├── presenters/
│   ├── protocols/
│   ├── mixins/
│   └── extensions/
├── ui/
│   ├── pages/
│   ├── components/
│   ├── helpers/
│   └── mixins/
├── validation/
│   ├── protocols/
│   └── validators/
└── main/
    ├── factories/
    ├── composites/
    ├── decorators/
    ├── builders/
    └── main.dart
```

---

## 3. Layers in Detail

### 3.1 domain/

The purest layer. Has **zero Flutter or external dependencies**.

#### entities/
- Plain Dart classes extending `Equatable`
- Represent core business objects
- No JSON serialization here

```dart
class HabitEntity extends Equatable {
  final String id;
  final String name;
  final int score;

  const HabitEntity({required this.id, required this.name, required this.score});

  @override
  List<Object> get props => [id, name, score];
}
```

#### usecases/
- **Abstract classes only** — they are contracts/interfaces, not implementations
- One file per use case
- Params classes defined alongside (extending Equatable when needed)

```dart
abstract class LoadHabits {
  Future<List<HabitEntity>> load();
}
```

```dart
abstract class ExecuteHabit {
  Future<void> execute(ExecuteHabitParams params);
}

class ExecuteHabitParams extends Equatable {
  final String habitId;
  final DateTime date;

  const ExecuteHabitParams({required this.habitId, required this.date});

  @override
  List<Object> get props => [habitId, date];
}
```

#### helpers/
- Pure Dart utilities used only within the domain (e.g., date helpers, score calculators)

---

### 3.2 data/

Contains **concrete implementations** of domain use cases and the abstract HTTP contract.

#### usecases/
- One folder per use case, one file per implementation
- Class named with prefix describing origin: `Remote` (API) or `Local` (cache)
- Implements the domain abstract class
- Has a companion model class (or uses one from `data/models/`) for JSON

```dart
class RemoteLoadHabits implements LoadHabits {
  final HttpClient httpClient;
  final String url;

  RemoteLoadHabits({required this.httpClient, required this.url});

  @override
  Future<List<HabitEntity>> load() async {
    try {
      final response = await httpClient.request(url: url, method: 'get');
      return (response as List).map((json) => RemoteHabitModel.fromJson(json).toEntity()).toList();
    } on HttpError catch (error) {
      throw error == HttpError.forbidden ? DomainError.accessDenied : DomainError.unexpected;
    }
  }
}
```

#### models/
- Data Transfer Objects for JSON parsing
- Named with `Remote` prefix: `RemoteHabitModel`
- Must implement `fromJson(Map)` factory and `toEntity()` method
- No domain logic

```dart
class RemoteHabitModel {
  final String id;
  final String name;
  final int score;

  RemoteHabitModel({required this.id, required this.name, required this.score});

  factory RemoteHabitModel.fromJson(Map json) => RemoteHabitModel(
    id: json['id'],
    name: json['name'],
    score: json['score'],
  );

  HabitEntity toEntity() => HabitEntity(id: id, name: name, score: score);
}
```

#### http/
- Contains the `HttpClient` abstract class only (interface for infra layer)

```dart
abstract class HttpClient {
  Future<dynamic> request({
    required String url,
    required String method,
    Map? headers,
    Map? body,
  });
}
```

---

### 3.3 infra/

Contains **concrete adapters** for external services.

#### http/
- `HttpAdapter` implements `HttpClient` using the `http` package
- Handles status codes → `HttpError` mapping
- Sets default headers (Content-Type, Accept: application/json)
- 3-second timeout on all requests

```
200/204 → return body (decoded JSON) or null
400     → HttpError.badRequest
401     → HttpError.unauthorized
403     → HttpError.forbidden
404     → HttpError.notFound
other   → HttpError.serverError
```

---

### 3.4 presentation/

Contains **presenters** and their abstract contracts. No Flutter widgets here.

#### protocols/
- Abstract classes defining the presenter interface for each page
- Named by feature: `LoginPresenter`, `HabitsPresenter`, etc.
- Expose streams (or `Rx`) for reactive state consumed by UI

```dart
abstract class HabitsPresenter {
  Stream<List<HabitViewModel>?> get habitsStream;
  Stream<UIError?> get mainErrorStream;
  Stream<bool> get isLoadingStream;
  Future<void> loadHabits();
}
```

#### presenters/
- Concrete implementations using GetX: `GetxHabitsPresenter`
- Extend `GetxController`
- Use `mixins` for shared behavior (loading, navigation, errors)
- Receive domain use cases via constructor injection

```dart
class GetxHabitsPresenter extends GetxController
    with LoadingManager, NavigationManager, UIErrorManager
    implements HabitsPresenter {

  final LoadHabits loadHabits;

  GetxHabitsPresenter({required this.loadHabits});

  // ...
}
```

#### mixins/
- Reusable presenter behaviors: `LoadingManager`, `NavigationManager`, `UIErrorManager`
- Use Rx observables internally

#### extensions/
- Dart extensions for presentation types (e.g., DomainError → UIError)

---

### 3.5 ui/

Contains all **Flutter widgets**.

#### pages/
- One folder per page/screen
- Page widget receives the presenter via constructor
- Subscribes to presenter streams with `StreamBuilder` or `Obx`
- Calls presenter methods on user interaction
- No business logic

```dart
class HabitsPage extends StatefulWidget {
  final HabitsPresenter presenter;
  // ...
}
```

#### components/
- Reusable widgets shared across pages

#### helpers/
- UI utilities (theme, i18n, error message strings from `UIError`)

#### mixins/
- Reusable UI behaviors (e.g., `LoadingManager` showing/hiding overlays)

---

### 3.6 validation/

#### protocols/
- `Validation` abstract class: `ValidationError? validate(String field, Map input)`

#### validators/
- Concrete validators: `RequiredFieldValidation`, `MinLengthValidation`, etc.
- Each validator handles one rule and implements `FieldValidation`

---

### 3.7 main/

Composition root. The only layer that knows all other layers.

#### factories/
- One factory function per page, presenter, use case, and http client
- Responsible for wiring all dependencies manually (no DI framework)

```dart
// factories/usecases/load_habits_factory.dart
RemoteLoadHabits makeRemoteLoadHabits() =>
    RemoteLoadHabits(httpClient: makeHttpAdapter(), url: makeApiUrl('habits'));

// factories/presenters/habits_presenter_factory.dart
GetxHabitsPresenter makeGetxHabitsPresenter() =>
    GetxHabitsPresenter(loadHabits: makeRemoteLoadHabits());

// factories/pages/habits_page_factory.dart
HabitsPage makeHabitsPage() =>
    HabitsPage(presenter: makeGetxHabitsPresenter());
```

#### composites/
- Composite pattern implementations (e.g., `ValidationComposite` running multiple validators)

#### decorators/
- Decorator pattern implementations (e.g., caching decorator wrapping a remote use case)

#### builders/
- Builder pattern for complex object creation

---

## 4. Data & Error Flow

```
UI (Page)
  ↕ streams / method calls
Presenter (GetxController)
  ↕ domain use case call
Domain UseCase (abstract) ← implemented by →
Data UseCase (Remote/Local)
  ↕ HttpClient (abstract) ← implemented by →
Infra (HttpAdapter)
  ↕ HTTP
Backend API (consistency)
```

**Error mapping:**
```
HttpError (infra) → DomainError (domain) → UIError (presentation)
```

---

## 5. Naming Conventions

| Layer | Prefix/Suffix | Example |
|-------|--------------|---------|
| Domain entity | `Entity` suffix | `HabitEntity` |
| Domain use case | none | `LoadHabits` (abstract) |
| Data use case (API) | `Remote` prefix | `RemoteLoadHabits` |
| Data use case (cache) | `Local` prefix | `LocalLoadHabits` |
| Data model | `Remote` prefix + `Model` suffix | `RemoteHabitModel` |
| Presenter (GetX) | `Getx` prefix + `Presenter` suffix | `GetxHabitsPresenter` |
| Presenter protocol | `Presenter` suffix | `HabitsPresenter` |
| Page widget | `Page` suffix | `HabitsPage` |
| Factory function | `make` prefix | `makeHabitsPage()` |

---

## 6. Key Rules

- **Domain has no Flutter, no http, no external dependencies**
- **Domain use cases are abstract** — never instantiate them directly
- **Data use cases** implement domain contracts and handle JSON/HTTP
- **Presenters** hold state and orchestrate use cases — no business logic
- **Pages** are dumb — they only render state and delegate events to presenter
- **Factories** are the only place where layers are wired together
- **No DI framework** — manual factory functions only
- **GetX** is the state management solution for all presenters
- **Equatable** for all entities and params classes
- Barrel files (`usecases.dart`, `pages.dart`, etc.) export all items in their folder
