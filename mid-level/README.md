# Flutter Interview Questions 2026 - Mid-Level (Questions 38-60)

**Experience: 2-4 Years**

[![Back to Main](https://img.shields.io/badge/←_Back_to_Main-blue?style=flat-square)](/README.md)

---

## What Makes Mid-Level Different?

At mid-level, interviewers assume you can build things. Now they want to know if you can build them *well*. The questions shift from "what is X?" to "when would you use X over Y?" and "what happens under the hood?"

You're expected to:
- Have opinions backed by experience
- Understand trade-offs, not just syntax
- Debug problems you've never seen before
- Write code that others can maintain

---

## Table of Contents

| Section | Questions | Topics |
|---------|-----------|--------|
| [Advanced State Management](#advanced-state-management) | 38-45 | Provider, Riverpod, BLoC, DI |
| [Concurrency & Performance](#concurrency--performance) | 46-48 | WidgetsBindingObserver, Tree shaking, Singletons |
| [Animations](#animations) | 49-51 | Error handling, Vsync, Implicit vs Explicit |
| [Networking & Security](#networking--security) | 52-60 | Dio, Certificate pinning, Three trees, Impeller |

---

## Advanced State Management

### Question 38: Provider In-Depth

**How does Provider actually work under the hood?**

Provider isn't magic — it's a sophisticated wrapper around InheritedWidget that solves its pain points.

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     HOW PROVIDER WORKS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ChangeNotifierProvider                                        │
│         │                                                       │
│         ▼                                                       │
│   InheritedProvider (extends InheritedWidget)                   │
│         │                                                       │
│         ├──► Stores value in _InheritedProviderScope            │
│         │                                                       │
│         └──► Notifies dependents via InheritedWidget mechanism  │
│                                                                 │
│   context.watch<T>()  ──► Subscribes to changes                 │
│   context.read<T>()   ──► One-time access, no subscription      │
│   context.select<T>() ──► Subscribes to specific property only  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Understanding the difference between read, watch, and select

class CounterNotifier extends ChangeNotifier {
  int _count = 0;
  String _label = 'Counter';

  int get count => _count;
  String get label => _label;

  void increment() {
    _count++;
    notifyListeners(); // This triggers ALL watchers
  }

  void updateLabel(String newLabel) {
    _label = newLabel;
    notifyListeners();
  }
}

// ❌ BAD: Rebuilds on ANY change (count OR label)
class BadWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final counter = context.watch<CounterNotifier>();
    return Text('${counter.count}');
  }
}

// ✅ GOOD: Only rebuilds when count changes
class GoodWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final count = context.select<CounterNotifier, int>((c) => c.count);
    return Text('$count');
  }
}

// ✅ GOOD: For event handlers, use read (no subscription)
class ButtonWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () => context.read<CounterNotifier>().increment(),
      child: Text('Increment'),
    );
  }
}
```

**Common Mistakes:**

1. Using `watch` inside callbacks (causes errors)
2. Using `read` in build method when you need reactivity
3. Not using `select` for large state objects

**Follow-up Questions:**
- "How would you test a widget that uses Provider?"
- "What's the difference between Provider and InheritedWidget's updateShouldNotify?"

---

### Question 39: Riverpod Deep Dive

**Why Riverpod over Provider? What problems does it solve?**

**Theory:**

Riverpod was created by the same author as Provider (Remi Rousselet) to fix Provider's fundamental limitations.

```
┌─────────────────────────────────────────────────────────────────┐
│                 PROVIDER vs RIVERPOD                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   PROVIDER LIMITATIONS          RIVERPOD SOLUTIONS              │
│   ──────────────────────        ─────────────────               │
│   • Requires BuildContext       • No context needed             │
│   • Runtime errors for          • Compile-time safety           │
│     missing providers                                           │
│   • Can't have 2 providers      • Each provider has unique ref  │
│     of same type                                                │
│   • Depends on widget tree      • Providers are global,         │
│                                   independent of tree           │
│   • Testing requires widget     • Test providers directly       │
│     pump                                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Riverpod provider types and when to use them

// 1. Provider - for values that never change
final configProvider = Provider<AppConfig>((ref) {
  return AppConfig(apiUrl: 'https://api.example.com');
});

// 2. StateProvider - for simple mutable state
final counterProvider = StateProvider<int>((ref) => 0);

// 3. StateNotifierProvider - for complex state with logic
class TodosNotifier extends StateNotifier<List<Todo>> {
  TodosNotifier() : super([]);

  void add(Todo todo) => state = [...state, todo];

  void remove(String id) => state = state.where((t) => t.id != id).toList();
}

final todosProvider = StateNotifierProvider<TodosNotifier, List<Todo>>((ref) {
  return TodosNotifier();
});

// 4. FutureProvider - for async data
final userProvider = FutureProvider<User>((ref) async {
  final api = ref.watch(apiClientProvider);
  return api.fetchUser();
});

// 5. StreamProvider - for reactive streams
final messagesProvider = StreamProvider<List<Message>>((ref) {
  final socket = ref.watch(socketProvider);
  return socket.messageStream;
});

// 6. NotifierProvider (Riverpod 2.0+) - the modern approach
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;

  void increment() => state++;
}

// Usage in widget
class MyWidget extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // .when() handles loading/error/data states elegantly
    final userAsync = ref.watch(userProvider);

    return userAsync.when(
      loading: () => CircularProgressIndicator(),
      error: (err, stack) => Text('Error: $err'),
      data: (user) => Text('Hello, ${user.name}'),
    );
  }
}
```

**Provider Modifiers:**

```dart
// autoDispose - clean up when no longer listened to
final searchProvider = FutureProvider.autoDispose<List<Result>>((ref) async {
  // Automatically disposed when widget unmounts
  final query = ref.watch(searchQueryProvider);
  return api.search(query);
});

// family - parameterized providers
final userByIdProvider = FutureProvider.family<User, String>((ref, userId) async {
  return api.fetchUser(userId);
});

// Usage
final user = ref.watch(userByIdProvider('user_123'));
```

**Common Mistakes:**

1. Forgetting `autoDispose` for screen-specific data (memory leaks)
2. Not using `family` when you need parameterized providers
3. Watching providers in callbacks instead of reading

**Follow-up Questions:**
- "How do you handle provider dependencies in Riverpod?"
- "Explain ref.listen vs ref.watch"

---

### Question 40: BLoC Pattern

**Explain the BLoC pattern. When would you choose it over other solutions?**

**Theory:**

BLoC (Business Logic Component) enforces strict separation through streams. Events go in, states come out.

```
┌─────────────────────────────────────────────────────────────────┐
│                      BLoC ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   UI Layer                 BLoC                  Data Layer     │
│   ────────                 ────                  ──────────     │
│                                                                 │
│   ┌─────────┐    Event    ┌─────────┐    Call   ┌───────────┐  │
│   │         │ ──────────► │         │ ────────► │           │  │
│   │  Widget │             │  BLoC   │           │ Repository│  │
│   │         │ ◄────────── │         │ ◄──────── │           │  │
│   └─────────┘    State    └─────────┘   Data    └───────────┘  │
│                                                                 │
│   • Widgets dispatch      • Transforms events  • Fetches data  │
│     events                  to states          • Caches        │
│   • React to states       • Contains all       • API calls     │
│   • NO business logic       business logic                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Modern BLoC with sealed classes (Dart 3+)

// Events
sealed class AuthEvent {}
class LoginRequested extends AuthEvent {
  final String email;
  final String password;
  LoginRequested(this.email, this.password);
}
class LogoutRequested extends AuthEvent {}

// States
sealed class AuthState {}
class AuthInitial extends AuthState {}
class AuthLoading extends AuthState {}
class AuthSuccess extends AuthState {
  final User user;
  AuthSuccess(this.user);
}
class AuthFailure extends AuthState {
  final String message;
  AuthFailure(this.message);
}

// BLoC
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  final AuthRepository _authRepository;

  AuthBloc(this._authRepository) : super(AuthInitial()) {
    on<LoginRequested>(_onLoginRequested);
    on<LogoutRequested>(_onLogoutRequested);
  }

  Future<void> _onLoginRequested(
    LoginRequested event,
    Emitter<AuthState> emit,
  ) async {
    emit(AuthLoading());

    try {
      final user = await _authRepository.login(
        email: event.email,
        password: event.password,
      );
      emit(AuthSuccess(user));
    } catch (e) {
      emit(AuthFailure(e.toString()));
    }
  }

  Future<void> _onLogoutRequested(
    LogoutRequested event,
    Emitter<AuthState> emit,
  ) async {
    await _authRepository.logout();
    emit(AuthInitial());
  }
}

// Usage in Widget with pattern matching
class LoginScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocConsumer<AuthBloc, AuthState>(
      listener: (context, state) {
        // Side effects (navigation, snackbars)
        if (state is AuthSuccess) {
          Navigator.pushReplacementNamed(context, '/home');
        }
        if (state is AuthFailure) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text(state.message)),
          );
        }
      },
      builder: (context, state) {
        return switch (state) {
          AuthLoading() => Center(child: CircularProgressIndicator()),
          _ => LoginForm(),
        };
      },
    );
  }
}
```

**Cubit - Simplified BLoC:**

```dart
// When events are overkill, use Cubit
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
  void decrement() => emit(state - 1);
}
```

**Common Mistakes:**

1. Putting UI logic in BLoC (navigation belongs in BlocListener)
2. Creating one massive BLoC instead of feature-specific ones
3. Not using Cubit for simple state (over-engineering)

**Follow-up Questions:**
- "How do you handle BLoC-to-BLoC communication?"
- "When would you use Cubit vs full BLoC?"

---

### Question 41: InheritedWidget Internals

**How does InheritedWidget actually propagate data down the tree?**

**Theory:**

InheritedWidget is the foundation of ALL state management in Flutter. Understanding it unlocks everything else.

```
┌─────────────────────────────────────────────────────────────────┐
│               INHERITEDWIDGET MECHANISM                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. Registration (context.dependOnInheritedWidgetOfExactType)  │
│   ──────────────                                                │
│   Widget calls dependOn... → Framework registers dependency     │
│   in Element._dependencies map                                  │
│                                                                 │
│   2. Notification (updateShouldNotify returns true)             │
│   ────────────────                                              │
│   InheritedWidget rebuilds → Framework checks all dependents    │
│   → Calls didChangeDependencies() → Marks for rebuild           │
│                                                                 │
│   ┌──────────────────┐                                          │
│   │ InheritedWidget  │                                          │
│   │   (holds data)   │                                          │
│   └────────┬─────────┘                                          │
│            │ updateShouldNotify()                               │
│            ▼                                                    │
│   ┌──────────────────┐                                          │
│   │ Dependent Widget │ ◄── dependOnInheritedWidgetOfExactType() │
│   │ (consumes data)  │                                          │
│   └──────────────────┘                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Building a mini state management solution

class AppState extends InheritedWidget {
  final int counter;
  final VoidCallback increment;

  const AppState({
    super.key,
    required this.counter,
    required this.increment,
    required super.child,
  });

  // This is called when a new AppState is placed in the tree
  @override
  bool updateShouldNotify(AppState oldWidget) {
    // Only notify if counter actually changed
    return counter != oldWidget.counter;
  }

  // Convenience method (the Provider pattern!)
  static AppState of(BuildContext context) {
    final result = context.dependOnInheritedWidgetOfExactType<AppState>();
    assert(result != null, 'No AppState found in context');
    return result!;
  }

  // For when you don't want to subscribe to changes
  static AppState read(BuildContext context) {
    final result = context.getInheritedWidgetOfExactType<AppState>();
    assert(result != null, 'No AppState found in context');
    return result!;
  }
}

// The stateful wrapper that triggers rebuilds
class AppStateProvider extends StatefulWidget {
  final Widget child;

  const AppStateProvider({super.key, required this.child});

  @override
  State<AppStateProvider> createState() => _AppStateProviderState();
}

class _AppStateProviderState extends State<AppStateProvider> {
  int _counter = 0;

  void _increment() {
    setState(() => _counter++);
    // setState rebuilds this widget, which creates new AppState
    // New AppState's updateShouldNotify returns true
    // Framework notifies all dependents
  }

  @override
  Widget build(BuildContext context) {
    return AppState(
      counter: _counter,
      increment: _increment,
      child: widget.child,
    );
  }
}

// Usage
class CounterDisplay extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // This registers a dependency
    final state = AppState.of(context);
    return Text('Count: ${state.counter}');
  }
}
```

**Common Mistakes:**

1. Using `getInheritedWidgetOfExactType` when you need reactivity
2. Always returning `true` from `updateShouldNotify` (unnecessary rebuilds)
3. Storing mutable objects in InheritedWidget

**Follow-up Questions:**
- "What's the time complexity of finding an InheritedWidget in the tree?"
- "How does Flutter know which widgets depend on an InheritedWidget?"

---

### Question 42: BehaviorSubject (RxDart)

**What is BehaviorSubject and when would you use RxDart over native Dart streams?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                STREAM TYPES COMPARISON                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   StreamController        │  BehaviorSubject (RxDart)           │
│   ─────────────────       │  ────────────────────────           │
│   • No initial value      │  • Has initial/current value        │
│   • Only future events    │  • New listeners get last value     │
│   • Basic operators       │  • Rich operators (debounce, etc)   │
│   • Can be single/        │  • Always broadcast                 │
│     broadcast             │                                     │
│                                                                 │
│   Timeline:               │  Timeline:                          │
│   ─────────               │  ─────────                          │
│   ───●───●───●───► t      │  ──[A]─●───●───●───► t              │
│      ▲                    │     ▲                               │
│      └─ subscriber joins, │     └─ subscriber joins,            │
│         sees nothing      │        immediately gets 'A'         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
import 'package:rxdart/rxdart.dart';

// Real-world search with debouncing and distinct
class SearchBloc {
  final _searchSubject = BehaviorSubject<String>.seeded('');
  final ApiClient _api;

  late final Stream<List<SearchResult>> results;

  SearchBloc(this._api) {
    results = _searchSubject
        .debounceTime(Duration(milliseconds: 300))  // Wait for typing to stop
        .distinct()                                   // Ignore duplicate queries
        .where((query) => query.length >= 2)         // Min 2 characters
        .switchMap((query) => _api                   // Cancel previous request
            .search(query)
            .asStream()
            .startWith([])                           // Show empty while loading
            .onErrorReturn([]))                      // Handle errors gracefully
        .share();                                    // Share among subscribers
  }

  // Current value is always accessible
  String get currentQuery => _searchSubject.value;

  void search(String query) => _searchSubject.add(query);

  void dispose() => _searchSubject.close();
}

// Combining multiple streams
class DashboardBloc {
  final _user = BehaviorSubject<User?>();
  final _notifications = BehaviorSubject<List<Notification>>.seeded([]);
  final _settings = BehaviorSubject<Settings>();

  // Combine latest values from multiple streams
  late final Stream<DashboardState> state = Rx.combineLatest3(
    _user,
    _notifications,
    _settings,
    (user, notifications, settings) => DashboardState(
      user: user,
      notifications: notifications,
      settings: settings,
    ),
  );
}
```

**When to use RxDart:**

| Use RxDart When | Stick with Dart Streams When |
|-----------------|------------------------------|
| Need current value synchronously | Simple one-time async |
| Complex stream transformations | Basic StreamBuilder |
| Combining multiple streams | Single stream source |
| Debounce/throttle/distinct | No complex operators needed |

**Common Mistakes:**

1. Forgetting to close BehaviorSubject (memory leak)
2. Not using `.share()` when multiple widgets listen
3. Over-using RxDart when simple setState would work

---

### Question 43: Sealed Classes for State

**How do sealed classes improve state management?**

**Theory:**

Sealed classes ensure the compiler checks you've handled all possible states.

```dart
// The power of exhaustive pattern matching

sealed class NetworkState<T> {}
class Initial<T> extends NetworkState<T> {}
class Loading<T> extends NetworkState<T> {}
class Success<T> extends NetworkState<T> {
  final T data;
  Success(this.data);
}
class Failure<T> extends NetworkState<T> {
  final String message;
  final Exception? exception;
  Failure(this.message, [this.exception]);
}

// The compiler ensures you handle ALL cases
Widget buildUI(NetworkState<User> state) {
  return switch (state) {
    Initial() => Text('Press button to load'),
    Loading() => CircularProgressIndicator(),
    Success(:final data) => UserCard(data),
    Failure(:final message) => ErrorWidget(message),
    // No default needed - compiler knows all cases covered!
  };
}

// If you add a new state later...
class Refreshing<T> extends NetworkState<T> {
  final T staleData;
  Refreshing(this.staleData);
}

// Compiler ERROR: The switch is not exhaustive
// You MUST handle Refreshing - can't forget!
```

---

### Question 44: Dependency Injection in Flutter

**How would you implement dependency injection in Flutter?**

**Code Example:**

```dart
// Approach 1: Service Locator (get_it)
final getIt = GetIt.instance;

void setupDependencies() {
  // Singleton - one instance for app lifetime
  getIt.registerSingleton<ApiClient>(ApiClient());

  // Lazy Singleton - created on first access
  getIt.registerLazySingleton<Database>(() => Database());

  // Factory - new instance each time
  getIt.registerFactory<Logger>(() => Logger());

  // Factory with parameters
  getIt.registerFactoryParam<UserRepository, String, void>(
    (userId, _) => UserRepository(userId),
  );
}

// Usage
class MyService {
  final apiClient = getIt<ApiClient>();
}

// Approach 2: Riverpod (recommended for Flutter)
final apiClientProvider = Provider<ApiClient>((ref) => ApiClient());

final userRepositoryProvider = Provider<UserRepository>((ref) {
  final api = ref.watch(apiClientProvider);  // Automatic dependency
  return UserRepository(api);
});

// Approach 3: Manual DI with InheritedWidget
class Dependencies extends InheritedWidget {
  final ApiClient apiClient;
  final Database database;

  const Dependencies({
    required this.apiClient,
    required this.database,
    required super.child,
  });

  static Dependencies of(BuildContext context) =>
      context.dependOnInheritedWidgetOfExactType<Dependencies>()!;

  @override
  bool updateShouldNotify(Dependencies old) => false;
}
```

---

### Question 45: Mixins

**When would you use a mixin vs inheritance vs composition?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                WHEN TO USE WHAT                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Inheritance (extends)                                         │
│   • "Is-a" relationship                                         │
│   • Only ONE superclass                                         │
│   • Example: CustomPainter IS-A Listenable                      │
│                                                                 │
│   Mixin (with)                                                  │
│   • "Can-do" relationship                                       │
│   • Multiple mixins allowed                                     │
│   • Reusable behavior across unrelated classes                  │
│   • Example: SingleTickerProviderStateMixin                     │
│                                                                 │
│   Composition (has-a)                                           │
│   • "Has-a" relationship                                        │
│   • Most flexible, preferred approach                           │
│   • Example: Widget HAS-A AnimationController                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Mixin with 'on' constraint
mixin Swimmer on Animal {
  void swim() => print('${name} is swimming');
}

mixin Flyer on Animal {
  void fly() => print('${name} is flying');
}

abstract class Animal {
  String get name;
}

// Duck can do multiple things
class Duck extends Animal with Swimmer, Flyer {
  @override
  String get name => 'Duck';
}

// Real Flutter example: Multiple ticker providers
class MyWidgetState extends State<MyWidget>
    with TickerProviderStateMixin {  // Multiple animations need this
  late final AnimationController _controller1;
  late final AnimationController _controller2;

  @override
  void initState() {
    super.initState();
    _controller1 = AnimationController(vsync: this, duration: Duration(seconds: 1));
    _controller2 = AnimationController(vsync: this, duration: Duration(milliseconds: 500));
  }
}

// Mixin for reusable functionality
mixin AutoDisposeController<T extends StatefulWidget> on State<T> {
  final _controllers = <ChangeNotifier>[];

  T createController<T extends ChangeNotifier>(T controller) {
    _controllers.add(controller);
    return controller;
  }

  @override
  void dispose() {
    for (final c in _controllers) {
      c.dispose();
    }
    super.dispose();
  }
}
```

---

## Concurrency & Performance

### Question 46: WidgetsBindingObserver

**What is WidgetsBindingObserver and when do you need it?**

**Code Example:**

```dart
class AppLifecycleManager extends StatefulWidget {
  final Widget child;
  const AppLifecycleManager({required this.child});

  @override
  State<AppLifecycleManager> createState() => _AppLifecycleManagerState();
}

class _AppLifecycleManagerState extends State<AppLifecycleManager>
    with WidgetsBindingObserver {

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.resumed:
        // App visible, refresh data
        _refreshData();
        _connectWebSocket();
        break;
      case AppLifecycleState.inactive:
        // App partially visible (incoming call, etc)
        break;
      case AppLifecycleState.paused:
        // App not visible, save state
        _saveState();
        _disconnectWebSocket();
        break;
      case AppLifecycleState.detached:
        // App about to be terminated
        _cleanup();
        break;
      case AppLifecycleState.hidden:
        // App hidden but still running (desktop)
        break;
    }
  }

  @override
  void didChangeMetrics() {
    // Screen size changed (rotation, keyboard, etc)
  }

  @override
  void didChangePlatformBrightness() {
    // System dark/light mode changed
  }

  @override
  Widget build(BuildContext context) => widget.child;
}
```

---

### Question 47: Tree Shaking

**What is tree shaking and how does it work in Flutter?**

**Theory:**

Tree shaking removes unused code during compilation, reducing app size.

```
┌─────────────────────────────────────────────────────────────────┐
│                      TREE SHAKING                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Your Code              After Tree Shaking                     │
│   ─────────              ─────────────────                      │
│                                                                 │
│   import 'utils.dart';   // Only used function remains          │
│   // has 10 functions                                           │
│                                                                 │
│   main() {               main() {                               │
│     formatDate();          formatDate();  // kept               │
│   }                      }                                      │
│                                                                 │
│   // 9 functions removed automatically                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What breaks tree shaking:**

```dart
// ❌ BAD: dart:mirrors prevents tree shaking
import 'dart:mirrors'; // Never use in Flutter!

// ❌ BAD: Dynamic access prevents static analysis
void callMethod(dynamic obj, String methodName) {
  // Compiler can't know what's used
}

// ✅ GOOD: Static imports
import 'package:intl/intl.dart' show DateFormat;

// ✅ GOOD: Conditional imports for platform-specific code
import 'stub.dart'
    if (dart.library.io) 'mobile.dart'
    if (dart.library.html) 'web.dart';
```

---

### Question 48: Singleton Pattern Drawbacks

**What are the drawbacks of the Singleton pattern in Flutter?**

```dart
// The classic Singleton
class ApiClient {
  static final ApiClient _instance = ApiClient._internal();
  factory ApiClient() => _instance;
  ApiClient._internal();
}

// Problems:
// 1. Hard to test - can't substitute mock
// 2. Hidden dependencies - not visible in constructor
// 3. Global mutable state - hard to reason about
// 4. Lifecycle issues - no way to reset/dispose

// Better approach: Dependency injection
class ApiClient {
  final String baseUrl;
  ApiClient(this.baseUrl);
}

// Register once
final apiProvider = Provider<ApiClient>((ref) => ApiClient('https://api.example.com'));

// Easy to override in tests
void main() {
  testWidgets('test', (tester) async {
    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          apiProvider.overrideWithValue(MockApiClient()),
        ],
        child: MyApp(),
      ),
    );
  });
}
```

---

## Animations

### Question 49: Global Error Handling

**How do you implement global error handling in Flutter?**

**Code Example:**

```dart
void main() {
  // Catch Flutter framework errors
  FlutterError.onError = (FlutterErrorDetails details) {
    FlutterError.presentError(details);
    // Log to crash reporting service
    CrashReporting.recordFlutterError(details);
  };

  // Catch async errors not handled by Flutter
  PlatformDispatcher.instance.onError = (error, stack) {
    CrashReporting.recordError(error, stack);
    return true; // Prevents app crash
  };

  // Wrap entire app for widget-level errors
  runApp(
    ErrorBoundary(
      child: MyApp(),
      onError: (error, stack) => ErrorScreen(error: error),
    ),
  );
}

// Custom Error Boundary
class ErrorBoundary extends StatefulWidget {
  final Widget child;
  final Widget Function(Object error, StackTrace? stack) onError;

  const ErrorBoundary({required this.child, required this.onError});

  @override
  State<ErrorBoundary> createState() => _ErrorBoundaryState();
}

class _ErrorBoundaryState extends State<ErrorBoundary> {
  Object? _error;
  StackTrace? _stack;

  @override
  void initState() {
    super.initState();
    FlutterError.onError = (details) {
      setState(() {
        _error = details.exception;
        _stack = details.stack;
      });
    };
  }

  @override
  Widget build(BuildContext context) {
    if (_error != null) {
      return widget.onError(_error!, _stack);
    }
    return widget.child;
  }
}
```

---

### Question 50: vsync and TickerProvider

**What is vsync in animations and why is it important?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      VSYNC EXPLAINED                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Without vsync:                                                │
│   ─────────────                                                 │
│   Animation ticks even when widget is not visible               │
│   → Wasted CPU cycles                                           │
│   → Battery drain                                               │
│   → Potential memory leaks                                      │
│                                                                 │
│   With vsync:                                                   │
│   ──────────                                                    │
│   Animation only ticks when:                                    │
│   1. Widget is mounted                                          │
│   2. Widget is visible on screen                                │
│   3. App is in foreground                                       │
│                                                                 │
│   Ticker lifecycle:                                             │
│   ────────────────                                              │
│   Widget builds → Ticker created → Animation starts             │
│        │                                                        │
│        ▼                                                        │
│   Widget off-screen → Ticker muted → No callbacks               │
│        │                                                        │
│        ▼                                                        │
│   Widget disposed → Ticker disposed → Animation stopped         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// SingleTickerProviderStateMixin - ONE animation
class SingleAnimationState extends State<MyWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,  // 'this' IS the TickerProvider
      duration: Duration(seconds: 1),
    );
  }

  @override
  void dispose() {
    _controller.dispose();  // Always dispose!
    super.dispose();
  }
}

// TickerProviderStateMixin - MULTIPLE animations
class MultiAnimationState extends State<MyWidget>
    with TickerProviderStateMixin {
  late final AnimationController _fadeController;
  late final AnimationController _slideController;

  @override
  void initState() {
    super.initState();
    _fadeController = AnimationController(vsync: this, duration: Duration(milliseconds: 300));
    _slideController = AnimationController(vsync: this, duration: Duration(milliseconds: 500));
  }
}
```

---

### Question 51: Implicit vs Explicit Animations

**When would you use implicit animations vs explicit animations?**

```
┌─────────────────────────────────────────────────────────────────┐
│              IMPLICIT vs EXPLICIT ANIMATIONS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   IMPLICIT (AnimatedFoo)       EXPLICIT (AnimationController)   │
│   ─────────────────────        ───────────────────────────      │
│   • Automatic                  • Manual control                 │
│   • Just change target value   • Start, stop, reverse, repeat   │
│   • Simple use cases           • Complex choreography           │
│   • Limited control            • Full control                   │
│                                                                 │
│   Use for:                     Use for:                         │
│   • Hover effects              • Page transitions               │
│   • Size changes               • Staggered lists                │
│   • Color transitions          • Looping animations             │
│   • Simple fades               • Gesture-driven motion          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Implicit - just change the value
class ImplicitExample extends StatefulWidget {
  @override
  State<ImplicitExample> createState() => _ImplicitExampleState();
}

class _ImplicitExampleState extends State<ImplicitExample> {
  bool _expanded = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _expanded = !_expanded),
      child: AnimatedContainer(
        duration: Duration(milliseconds: 300),
        curve: Curves.easeOutCubic,
        width: _expanded ? 200 : 100,
        height: _expanded ? 200 : 100,
        color: _expanded ? Colors.blue : Colors.red,
      ),
    );
  }
}

// Explicit - full control
class ExplicitExample extends StatefulWidget {
  @override
  State<ExplicitExample> createState() => _ExplicitExampleState();
}

class _ExplicitExampleState extends State<ExplicitExample>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _scaleAnimation;
  late final Animation<double> _rotationAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: Duration(seconds: 2),
    );

    // Staggered animations on single controller
    _scaleAnimation = Tween<double>(begin: 1.0, end: 1.5).animate(
      CurvedAnimation(
        parent: _controller,
        curve: Interval(0.0, 0.5, curve: Curves.easeOut),
      ),
    );

    _rotationAnimation = Tween<double>(begin: 0, end: 2 * pi).animate(
      CurvedAnimation(
        parent: _controller,
        curve: Interval(0.5, 1.0, curve: Curves.easeInOut),
      ),
    );

    _controller.repeat();  // Loop forever
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return Transform.scale(
          scale: _scaleAnimation.value,
          child: Transform.rotate(
            angle: _rotationAnimation.value,
            child: child,
          ),
        );
      },
      child: FlutterLogo(size: 100),  // child doesn't rebuild
    );
  }
}
```

---

## Networking & Security

### Question 52: Staggered Animations

**How do you create staggered animations in Flutter?**

**Code Example:**

```dart
class StaggeredListAnimation extends StatefulWidget {
  @override
  State<StaggeredListAnimation> createState() => _StaggeredListAnimationState();
}

class _StaggeredListAnimationState extends State<StaggeredListAnimation>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  final List<String> items = ['Item 1', 'Item 2', 'Item 3', 'Item 4', 'Item 5'];

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: Duration(milliseconds: 1500),
    )..forward();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (context, index) {
        // Each item animates during its portion of the timeline
        final startInterval = index / items.length;
        final endInterval = (index + 1) / items.length;

        final animation = Tween<Offset>(
          begin: Offset(1, 0),  // Start from right
          end: Offset.zero,
        ).animate(
          CurvedAnimation(
            parent: _controller,
            curve: Interval(
              startInterval,
              endInterval,
              curve: Curves.easeOutCubic,
            ),
          ),
        );

        return SlideTransition(
          position: animation,
          child: FadeTransition(
            opacity: CurvedAnimation(
              parent: _controller,
              curve: Interval(startInterval, endInterval),
            ),
            child: ListTile(title: Text(items[index])),
          ),
        );
      },
    );
  }
}
```

---

### Question 53: HTTP Requests with Dio

**Why use Dio over the http package?**

**Code Example:**

```dart
class ApiClient {
  late final Dio _dio;

  ApiClient() {
    _dio = Dio(BaseOptions(
      baseUrl: 'https://api.example.com',
      connectTimeout: Duration(seconds: 5),
      receiveTimeout: Duration(seconds: 10),
      headers: {'Content-Type': 'application/json'},
    ));

    // Add interceptors
    _dio.interceptors.addAll([
      _AuthInterceptor(),
      _LoggingInterceptor(),
      _RetryInterceptor(),
    ]);
  }

  Future<User> getUser(String id) async {
    try {
      final response = await _dio.get('/users/$id');
      return User.fromJson(response.data);
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  AppException _handleError(DioException e) {
    return switch (e.type) {
      DioExceptionType.connectionTimeout => NetworkException('Connection timeout'),
      DioExceptionType.receiveTimeout => NetworkException('Server not responding'),
      DioExceptionType.badResponse => _handleResponseError(e.response),
      DioExceptionType.cancel => CancelledException(),
      _ => UnknownException(e.message),
    };
  }
}

// Auth interceptor for automatic token refresh
class _AuthInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final token = AuthStorage.getToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      // Try to refresh token
      final newToken = await AuthService.refreshToken();
      if (newToken != null) {
        // Retry with new token
        err.requestOptions.headers['Authorization'] = 'Bearer $newToken';
        final response = await Dio().fetch(err.requestOptions);
        return handler.resolve(response);
      }
    }
    handler.next(err);
  }
}
```

---

### Question 54: Certificate Pinning

**What is certificate pinning and how do you implement it?**

**Code Example:**

```dart
// Using dio with certificate pinning
class SecureApiClient {
  late final Dio _dio;

  SecureApiClient() {
    _dio = Dio();

    // Load pinned certificate
    (_dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
      final client = HttpClient();

      client.badCertificateCallback = (cert, host, port) {
        // Compare certificate fingerprint
        final fingerprint = sha256.convert(cert.der).toString();
        return fingerprint == PINNED_CERTIFICATE_FINGERPRINT;
      };

      return client;
    };
  }
}

// Alternative: Using public key pinning
const PINNED_PUBLIC_KEY_HASH = 'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=';

// With http_certificate_pinning package
final response = await SecureHttp.get(
  'https://api.example.com/data',
  pinnedCertificates: [PINNED_PUBLIC_KEY_HASH],
);
```

---

### Question 55: Secure Storage

**How do you securely store sensitive data in Flutter?**

**Code Example:**

```dart
// Using flutter_secure_storage
class SecureStorageService {
  final _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
    iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
  );

  // Store token
  Future<void> saveToken(String token) async {
    await _storage.write(key: 'auth_token', value: token);
  }

  // Retrieve token
  Future<String?> getToken() async {
    return await _storage.read(key: 'auth_token');
  }

  // Delete on logout
  Future<void> clearAll() async {
    await _storage.deleteAll();
  }
}

// What NOT to store securely (use SharedPreferences instead):
// - User preferences
// - App settings
// - Non-sensitive cached data

// What TO store securely:
// - Auth tokens
// - API keys
// - Encryption keys
// - Biometric data references
```

---

### Question 56: Retry Logic with Exponential Backoff

**How do you implement retry logic with exponential backoff?**

**Code Example:**

```dart
class RetryInterceptor extends Interceptor {
  final int maxRetries;
  final Duration initialDelay;

  RetryInterceptor({
    this.maxRetries = 3,
    this.initialDelay = const Duration(seconds: 1),
  });

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (_shouldRetry(err)) {
      final retryCount = err.requestOptions.extra['retryCount'] ?? 0;

      if (retryCount < maxRetries) {
        // Exponential backoff: 1s, 2s, 4s...
        final delay = initialDelay * pow(2, retryCount);

        await Future.delayed(delay);

        err.requestOptions.extra['retryCount'] = retryCount + 1;

        try {
          final response = await Dio().fetch(err.requestOptions);
          return handler.resolve(response);
        } catch (e) {
          return handler.next(err);
        }
      }
    }
    handler.next(err);
  }

  bool _shouldRetry(DioException err) {
    return err.type == DioExceptionType.connectionTimeout ||
           err.type == DioExceptionType.receiveTimeout ||
           (err.response?.statusCode ?? 0) >= 500;
  }
}
```

---

### Question 57: Three Trees (Widget/Element/RenderObject)

**Explain Flutter's three tree architecture.**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   FLUTTER'S THREE TREES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   WIDGET TREE          ELEMENT TREE        RENDER TREE          │
│   ───────────          ────────────        ───────────          │
│   (Configuration)      (Lifecycle)         (Layout/Paint)       │
│                                                                 │
│   Container ─────────► ComponentElement                         │
│      │                      │                                   │
│      ▼                      ▼                                   │
│   ColoredBox ────────► SingleChildRenderObjectElement           │
│      │                      │                    │              │
│      ▼                      ▼                    ▼              │
│   SizedBox ──────────► SingleChildRenderObjectElement           │
│      │                      │                    │              │
│      ▼                      ▼                    ▼              │
│   Text ──────────────► LeafRenderObjectElement ─► RenderParagraph
│                                                                 │
│   • Immutable            • Mutable              • Mutable       │
│   • Recreated often      • Long-lived           • Long-lived    │
│   • Lightweight          • Manages lifecycle    • Expensive     │
│   • Blueprint            • Glue between trees   • Actual work   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why this matters:**

```dart
// When you call setState(), Flutter:
// 1. Rebuilds widget tree (cheap - just creating objects)
// 2. Walks element tree, comparing old vs new widgets
// 3. Only updates render objects that actually changed

// This is why keys matter:
ListView(
  children: items.map((item) =>
    ItemWidget(key: ValueKey(item.id), item: item)
  ).toList(),
);

// Without keys: Flutter matches by index (wrong updates when list reorders)
// With keys: Flutter matches by key (correct identity preserved)
```

---

### Question 58: Impeller Rendering Engine

**What is Impeller and how does it differ from Skia?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  SKIA vs IMPELLER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   SKIA (Legacy)                IMPELLER (New Default)           │
│   ─────────────                ─────────────────────            │
│                                                                 │
│   • Compiles shaders at        • Precompiles ALL shaders        │
│     runtime                      at build time                  │
│                                                                 │
│   • First frame jank           • No shader compilation jank     │
│     (shader compilation)                                        │
│                                                                 │
│   • Uses OpenGL                • Uses Metal (iOS) /             │
│                                  Vulkan (Android)               │
│                                                                 │
│   • General purpose            • Built specifically for         │
│     2D graphics library          Flutter's rendering needs      │
│                                                                 │
│   Status (2026):                                                │
│   • iOS: Impeller default since Flutter 3.16                    │
│   • Android: Impeller default since Flutter 3.27                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Question 59: AOT vs JIT Compilation

**Explain AOT vs JIT compilation in Flutter.**

```
┌─────────────────────────────────────────────────────────────────┐
│                   JIT vs AOT COMPILATION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   JIT (Just-In-Time)           AOT (Ahead-Of-Time)              │
│   ──────────────────           ─────────────────                │
│                                                                 │
│   Used in: Debug mode          Used in: Release mode            │
│                                                                 │
│   • Compiles at runtime        • Compiles before runtime        │
│   • Enables hot reload         • No hot reload possible         │
│   • Slower startup             • Fast startup                   │
│   • Larger memory footprint    • Smaller memory footprint       │
│   • Includes debug symbols     • Optimized, stripped binary     │
│                                                                 │
│   flutter run                  flutter build apk --release      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Question 60: Code Obfuscation

**How and why do you obfuscate Flutter code?**

**Code Example:**

```bash
# Build with obfuscation
flutter build apk --obfuscate --split-debug-info=./debug-info

# The flags:
# --obfuscate: Renames classes, methods, variables to random characters
# --split-debug-info: Saves mapping file for crash report symbolication
```

**Why obfuscate:**
- Protects intellectual property
- Makes reverse engineering harder
- Hides API endpoints and business logic
- Required by some enterprise policies

**Important:** Keep the `debug-info` folder safe! You'll need it to decode crash reports from obfuscated builds.

---

## Summary

At mid-level, you should be able to:

| Area | Expectation |
|------|-------------|
| State Management | Explain trade-offs between Provider, Riverpod, BLoC |
| Performance | Understand the three trees and why they matter |
| Animations | Build staggered animations with proper vsync |
| Networking | Implement robust API clients with retry logic |
| Security | Know certificate pinning and secure storage |
| Architecture | Design testable, maintainable code |

---

**Made with by [Debasmita Sarkar](https://github.com/debasmitasarkar)**

[![Back to Main](https://img.shields.io/badge/←_Back_to_Main-blue?style=flat-square)](/README.md)
[![Next: Senior Level](https://img.shields.io/badge/Next:_Senior_Level_→-green?style=flat-square)](/senior/README.md)
