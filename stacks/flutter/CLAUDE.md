# CLAUDE.md — Flutter (Mobile)
### Forge Stack Rules | Cross-Platform Mobile Apps

---

This file extends the universal CLAUDE.md. Read that first.

---

## WHEN TO USE FLUTTER

- Cross-platform mobile (iOS + Android from one codebase)
- Apps needing near-native performance
- Apps with custom UIs that need full control
- When React Native's JS bridge limitations are a problem

---

## TECHNOLOGY STANDARDS

### Required
- **Flutter 3.x+** (latest stable)
- **Dart 3.x+** with null safety
- **Material 3** design system (or Cupertino for iOS-only look)

### State Management — Pick One
- **Riverpod** (recommended) — modern, type-safe, testable
- **Bloc/Cubit** — good for complex apps with clear event flows
- **Provider** — simpler apps, but Riverpod is a better evolution

**Never use setState for anything beyond single-widget local state.**

### Required Packages
- **go_router** for navigation (not Navigator 2.0 directly)
- **dio** for HTTP (better than http package)
- **flutter_secure_storage** for tokens and secrets
- **freezed** + **json_serializable** for immutable models
- **get_it** or Riverpod for dependency injection
- **intl** for localization
- **logger** for structured logging

---

## FOLDER STRUCTURE

```
lib/
├── main.dart
├── app.dart                       (MaterialApp configuration)
├── core/
│   ├── config/
│   │   ├── env.dart               (environment configuration)
│   │   └── theme.dart
│   ├── errors/                    (custom exceptions)
│   ├── network/
│   │   ├── api_client.dart        (dio instance with interceptors)
│   │   └── interceptors/
│   ├── router/                    (go_router configuration)
│   └── storage/                   (secure storage wrapper)
├── features/                      (organize by feature)
│   └── users/
│       ├── data/
│       │   ├── models/            (freezed models)
│       │   ├── datasources/       (API calls)
│       │   └── repositories/
│       ├── domain/
│       │   ├── entities/
│       │   └── repositories/      (abstract interfaces)
│       └── presentation/
│           ├── screens/
│           ├── widgets/
│           └── providers/         (Riverpod state)
├── shared/
│   ├── widgets/                   (reusable widgets)
│   └── utils/
└── generated/                     (code-generated files)
```

---

## WIDGETS

### Rules
- Small widgets — under 150 lines ideally
- Extract repeated UI into dedicated widget classes
- Const constructors where possible (`const MyWidget()`)
- Stateless by default, Stateful only when needed
- With Riverpod, use `ConsumerWidget` / `ConsumerStatefulWidget`

### Avoid
- Never build complex UI in a single `build` method
- Never put business logic in widgets
- Never fetch data directly in widgets — use providers

---

## STATE MANAGEMENT — RIVERPOD

### Provider Types
- `Provider` — read-only values
- `StateProvider` — simple mutable state
- `NotifierProvider` — complex state with methods
- `FutureProvider` — async computed state
- `StreamProvider` — stream-based state

### Example
```dart
// Repository provider
final usersRepositoryProvider = Provider<UsersRepository>((ref) {
  return UsersRepositoryImpl(ref.watch(apiClientProvider));
});

// State notifier
class UsersNotifier extends AsyncNotifier<List<User>> {
  @override
  Future<List<User>> build() async {
    final repository = ref.read(usersRepositoryProvider);
    return repository.getAll();
  }
  
  Future<void> createUser(CreateUserInput input) async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      final repository = ref.read(usersRepositoryProvider);
      await repository.create(input);
      return repository.getAll();
    });
  }
}

final usersProvider = AsyncNotifierProvider<UsersNotifier, List<User>>(() {
  return UsersNotifier();
});

// In widget
class UsersScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final usersAsync = ref.watch(usersProvider);
    
    return usersAsync.when(
      data: (users) => UserList(users: users),
      loading: () => const CircularProgressIndicator(),
      error: (error, stack) => ErrorView(error: error),
    );
  }
}
```

---

## NETWORKING

### Dio Client Setup
```dart
final dio = Dio(BaseOptions(
  baseUrl: Env.apiUrl,
  connectTimeout: const Duration(seconds: 10),
  receiveTimeout: const Duration(seconds: 10),
));

dio.interceptors.addAll([
  AuthInterceptor(),
  LoggingInterceptor(),
  ErrorInterceptor(),
]);
```

### Repository Pattern
```dart
abstract class UsersRepository {
  Future<List<User>> getAll();
  Future<User> getById(String id);
  Future<User> create(CreateUserInput input);
}

class UsersRepositoryImpl implements UsersRepository {
  final Dio _dio;
  
  UsersRepositoryImpl(this._dio);
  
  @override
  Future<List<User>> getAll() async {
    final response = await _dio.get('/users');
    return (response.data as List).map((json) => User.fromJson(json)).toList();
  }
  
  // ... other methods
}
```

### Error Handling
- Map Dio errors to custom exceptions
- Never expose raw Dio errors to the UI
- Handle offline state gracefully

---

## NAVIGATION — GO_ROUTER

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomeScreen(),
    ),
    GoRoute(
      path: '/login',
      builder: (context, state) => const LoginScreen(),
    ),
    GoRoute(
      path: '/dashboard',
      builder: (context, state) => const DashboardScreen(),
      redirect: (context, state) {
        final isAuthed = authNotifier.isAuthenticated;
        return isAuthed ? null : '/login';
      },
    ),
  ],
);
```

- Type-safe route parameters
- Centralized route definitions
- Guards for protected routes

---

## MODELS — FREEZED

```dart
@freezed
class User with _$User {
  const factory User({
    required String id,
    required String email,
    required String name,
    required DateTime createdAt,
  }) = _User;
  
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

- Immutable by default
- Automatic `copyWith`, `==`, `hashCode`, `toString`
- Automatic JSON serialization
- Run: `dart run build_runner build` after changes

---

## SECURE STORAGE

```dart
class SecureStorage {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
  );
  
  static Future<void> saveToken(String token) async {
    await _storage.write(key: 'auth_token', value: token);
  }
  
  static Future<String?> getToken() async {
    return _storage.read(key: 'auth_token');
  }
  
  static Future<void> clear() async {
    await _storage.deleteAll();
  }
}
```

**Never use SharedPreferences for tokens or sensitive data.**

---

## PLATFORM-SPECIFIC CODE

- Keep platform-specific code minimal
- When needed, use platform channels with clear interfaces
- Document platform differences in code comments

---

## TESTING

### Stack
- **flutter_test** for unit and widget tests
- **mocktail** for mocking
- **integration_test** for E2E tests
- **golden_toolkit** for golden file tests (UI snapshots)

### What to Test
- Providers and notifiers (business logic)
- Repositories (with mocked API)
- Widgets (rendering, interaction)
- Critical user flows (integration tests)

---

## PERFORMANCE

- Use `const` constructors aggressively
- Use `ListView.builder` for long lists — never `ListView` with `children`
- Use `Image.network` with caching headers, or `cached_network_image` package
- Avoid rebuilds — use `select` in Riverpod to watch only specific fields
- Profile with DevTools before optimizing

---

## BUILD AND RELEASE

### Android
- Signing key in environment, never committed
- Use `--obfuscate` and `--split-debug-info` for release builds
- App Bundle (`.aab`) for Play Store, not APK

### iOS
- Code signing via Xcode with a proper provisioning profile
- Use App Store Connect API for automation

### Commands
```bash
# Development
flutter run

# Android release
flutter build appbundle --release --obfuscate --split-debug-info=build/symbols

# iOS release
flutter build ipa --release --obfuscate --split-debug-info=build/symbols

# Run tests
flutter test
flutter test integration_test/
```

---

## ENVIRONMENT CONFIGURATION

```dart
// lib/core/config/env.dart
class Env {
  static const apiUrl = String.fromEnvironment('API_URL', defaultValue: 'http://localhost:8000');
  static const environment = String.fromEnvironment('ENV', defaultValue: 'development');
}
```

Run with: `flutter run --dart-define=API_URL=https://api.example.com --dart-define=ENV=production`

---

## SPECIFIC DO NOTS

- Never use print() — use logger
- Never hardcode API URLs — use environment config
- Never store tokens in SharedPreferences
- Never use setState for global state
- Never mutate provider state directly — always create new state
- Never block the UI thread with heavy computation (use isolates)
- Never ignore null safety with `!` unless you've verified it's non-null

---

## SESSION COMPLETION CHECKLIST — FLUTTER SPECIFIC

- [ ] `flutter analyze` shows zero issues
- [ ] `flutter test` passes
- [ ] App builds for both iOS and Android
- [ ] No unhandled async errors
- [ ] Loading, error, and empty states exist for every screen
- [ ] Accessibility: proper semantic labels on interactive widgets
- [ ] Environment variables documented
- [ ] Assets properly declared in `pubspec.yaml`

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
