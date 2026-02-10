# Flutter Interview Questions 2026 - Senior Level (Questions 61-85)

**Experience: 4-6 Years**

[![Back to Main](https://img.shields.io/badge/←_Back_to_Main-blue?style=flat-square)](/README.md)

---

## What Makes Senior Different?

At senior level, you're not just writing code — you're making decisions that affect the entire team and product. Interviewers expect you to think in systems, not screens. You should be able to explain *why* something works the way it does, debug things you've never seen, and design solutions that scale.

You're expected to:
- Own technical decisions and defend trade-offs
- Understand platform-level behavior (iOS/Android internals)
- Write code that is testable, performant, and maintainable by others
- Mentor and unblock other developers
- Think about the user, not just the code

---

## Table of Contents

| Section | Questions | Topics |
|---------|-----------|--------|
| [Platform Integration & Testing](#platform-integration--testing) | 61-68 | Platform Channels, FFI, Deep linking, Testing |
| [Advanced Performance](#advanced-performance) | 69-78 | Memory leaks, Custom RenderObjects, Pagination, Background tasks |
| [Architecture & Infrastructure](#architecture--infrastructure) | 79-85 | Feature flags, Design systems, CI/CD, Modular architecture |

---

## Platform Integration & Testing

### Question 61: Platform Channels

**How do Platform Channels work? When would you use them?**

**Theory:**

Platform Channels are Flutter's bridge to native iOS/Android code. They use message passing — not code generation — to communicate across the Dart/native boundary.

```
┌─────────────────────────────────────────────────────────────────┐
│                   PLATFORM CHANNELS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Dart (Flutter)              Native (iOS/Android)              │
│   ──────────────              ────────────────────              │
│                                                                 │
│   ┌──────────────┐            ┌──────────────────┐              │
│   │ MethodChannel│ ◄────────► │ MethodChannel    │              │
│   │              │  async     │ (Swift/Kotlin)   │              │
│   └──────────────┘  message   └──────────────────┘              │
│                                                                 │
│   Types:                                                        │
│   • MethodChannel      → Request/response (most common)         │
│   • EventChannel       → Native to Dart streams                 │
│   • BasicMessageChannel → Low-level custom encoding             │
│                                                                 │
│   Codec (serialization):                                        │
│   • StandardMessageCodec (default) → int, String, List, Map    │
│   • JSONMessageCodec                                            │
│   • BinaryCodec                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Dart side
class BatteryService {
  static const _channel = MethodChannel('com.app.battery');

  Future<int> getBatteryLevel() async {
    try {
      final level = await _channel.invokeMethod<int>('getBatteryLevel');
      return level ?? -1;
    } on PlatformException catch (e) {
      throw BatteryException('Failed to get battery level: ${e.message}');
    }
  }
}

// EventChannel for continuous data (e.g., sensor streams)
class AccelerometerService {
  static const _eventChannel = EventChannel('com.app.accelerometer');

  Stream<AccelerometerData> get stream {
    return _eventChannel
        .receiveBroadcastStream()
        .map((event) => AccelerometerData.fromMap(event));
  }
}
```

```kotlin
// Android side (Kotlin)
class MainActivity : FlutterActivity() {
    private val CHANNEL = "com.app.battery"

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)

        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL)
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "getBatteryLevel" -> {
                        val level = getBatteryLevel()
                        if (level != -1) result.success(level)
                        else result.error("UNAVAILABLE", "Battery level not available", null)
                    }
                    else -> result.notImplemented()
                }
            }
    }

    private fun getBatteryLevel(): Int {
        val batteryManager = getSystemService(BATTERY_SERVICE) as BatteryManager
        return batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
    }
}
```

```swift
// iOS side (Swift)
@UIApplicationMain
class AppDelegate: FlutterAppDelegate {
    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        let controller = window?.rootViewController as! FlutterViewController
        let channel = FlutterMethodChannel(
            name: "com.app.battery",
            binaryMessenger: controller.binaryMessenger
        )

        channel.setMethodCallHandler { (call, result) in
            if call.method == "getBatteryLevel" {
                let device = UIDevice.current
                device.isBatteryMonitoringEnabled = true
                let level = Int(device.batteryLevel * 100)
                result(level)
            } else {
                result(FlutterMethodNotImplemented)
            }
        }

        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
}
```

**Common Mistakes:**

1. Not handling `PlatformException` on the Dart side
2. Calling `result` more than once on the native side (crashes)
3. Heavy computation on the platform channel thread (blocks UI)

**Follow-up Questions:**
- "How would you test platform channel code?"
- "What's the difference between MethodChannel and Pigeon?"

---

### Question 62: FFI (Foreign Function Interface)

**What is FFI and when would you use it over Platform Channels?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│            PLATFORM CHANNELS vs FFI                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Platform Channels                FFI (dart:ffi)               │
│   ─────────────────                ──────────────               │
│   • Async message passing         • Synchronous direct calls    │
│   • Higher overhead               • Near-zero overhead          │
│   • Works with Swift/Kotlin       • Works with C/C++ only       │
│   • Easy to use                   • More complex setup          │
│   • Good for: API calls,          • Good for: Image processing, │
│     platform features               crypto, audio, ML           │
│                                                                 │
│   Use Platform Channels when:     Use FFI when:                 │
│   • Calling platform APIs         • Performance-critical code   │
│   • One-off native features       • Existing C/C++ libraries    │
│   • Async is acceptable           • Sync calls needed           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
import 'dart:ffi';
import 'package:ffi/ffi.dart';

// Binding to a native C function
// C: int add(int a, int b) { return a + b; }

typedef AddNative = Int32 Function(Int32 a, Int32 b);
typedef AddDart = int Function(int a, int b);

class NativeMath {
  late final DynamicLibrary _lib;
  late final AddDart add;

  NativeMath() {
    _lib = Platform.isAndroid
        ? DynamicLibrary.open('libnative_math.so')
        : DynamicLibrary.process();

    add = _lib
        .lookupFunction<AddNative, AddDart>('add');
  }
}

// Usage
final math = NativeMath();
print(math.add(2, 3)); // 5 — synchronous, no async overhead
```

**When to use FFI:**
- Image/video processing (OpenCV, FFmpeg)
- Cryptographic operations
- Existing C/C++ codebases you need to reuse
- Performance-critical algorithms (audio DSP, physics)

---

### Question 63: Deep Linking with go_router

**How do you implement deep linking in Flutter?**

**Theory:**

Deep links let users navigate directly to specific content in your app from URLs, push notifications, or other apps.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DEEP LINKING FLOW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   User taps: myapp.com/products/123                             │
│        │                                                        │
│        ▼                                                        │
│   ┌─────────────┐                                               │
│   │ App closed?  │──Yes──► OS launches app with URL             │
│   │              │──No───► OS sends URL to running app          │
│   └─────────────┘                                               │
│        │                                                        │
│        ▼                                                        │
│   go_router parses URL → navigates to ProductScreen(id: 123)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
final router = GoRouter(
  initialLocation: '/',
  debugLogDiagnostics: true,

  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => HomeScreen(),
      routes: [
        GoRoute(
          name: 'product',
          path: 'products/:id',
          builder: (context, state) {
            final id = state.pathParameters['id']!;
            return ProductScreen(productId: id);
          },
        ),
        GoRoute(
          path: 'profile',
          builder: (context, state) => ProfileScreen(),
          routes: [
            GoRoute(
              path: 'settings',
              builder: (context, state) => SettingsScreen(),
            ),
          ],
        ),
      ],
    ),
  ],

  // Redirect for auth guards
  redirect: (context, state) {
    final isLoggedIn = AuthService.instance.isLoggedIn;
    final isLoginRoute = state.matchedLocation == '/login';

    if (!isLoggedIn && !isLoginRoute) return '/login';
    if (isLoggedIn && isLoginRoute) return '/';
    return null; // No redirect
  },

  // Custom error page
  errorBuilder: (context, state) => NotFoundScreen(),
);

// Navigation
context.go('/products/123');          // Replaces stack
context.push('/products/123');        // Pushes on stack
context.goNamed('product', pathParameters: {'id': '123'});
```

**Platform setup required:**
- **Android:** Intent filters in `AndroidManifest.xml`
- **iOS:** Associated Domains in `Runner.entitlements`
- **Web:** Works out of the box with URL-based routing

**Common Mistakes:**

1. Forgetting platform-specific configuration (deep links just don't work)
2. Not handling auth state in redirect (users land on protected pages)
3. Using `go` when you should use `push` (loses back navigation)

---

### Question 64: App Startup Optimization

**How would you optimize Flutter app startup time?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                APP STARTUP TIMELINE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Cold Start Phases:                                            │
│   ──────────────────                                            │
│                                                                 │
│   1. Native init    │ OS loads app binary, starts Dart VM       │
│   2. Flutter init   │ Framework setup, binding init             │
│   3. App init       │ Your runApp(), first build                │
│   4. First frame    │ Layout, paint, rasterize                  │
│                                                                 │
│   ──►──►──►──►──►──►──►──►──►──► time                          │
│   │ native │ flutter │ app init │ first frame │                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Optimization strategies:**

```dart
// 1. Defer heavy initialization
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // ❌ BAD: Blocking startup
  // await Firebase.initializeApp();
  // await Hive.initFlutter();
  // await loadConfig();

  // ✅ GOOD: Show app immediately, init in background
  runApp(MyApp());
}

class _MyAppState extends State<MyApp> {
  late final Future<void> _initFuture;

  @override
  void initState() {
    super.initState();
    _initFuture = _initializeApp();
  }

  Future<void> _initializeApp() async {
    await Firebase.initializeApp();
    await Hive.initFlutter();
    await loadConfig();
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder(
      future: _initFuture,
      builder: (context, snapshot) {
        if (snapshot.connectionState != ConnectionState.done) {
          return SplashScreen(); // Native splash transitions here
        }
        return HomeScreen();
      },
    );
  }
}

// 2. Use const constructors everywhere possible
const MyWidget(); // Cached by framework, skips rebuild

// 3. Reduce main isolate work
// Move JSON parsing, image processing to separate isolate
final parsed = await compute(parseJson, rawData);

// 4. Lazy-load routes
GoRoute(
  path: '/settings',
  builder: (context, state) => const SettingsScreen(),
  // Screen only loads when navigated to
),

// 5. Measure startup time
flutter run --trace-startup
// Outputs: timeToFirstFrameMicros, timeToFrameworkInitMicros
```

**Key metrics to track:**
- Time to first frame (TTFF)
- Time to interactive (TTI)
- Use `flutter run --trace-startup` and DevTools timeline

---

### Question 65: Complex Async State with Riverpod

**How do you handle complex async state that depends on multiple sources?**

**Code Example:**

```dart
// Scenario: Dashboard that needs user, orders, and notifications
// All fetched from different APIs, some depending on others

// Step 1: Base providers
final authProvider = StateNotifierProvider<AuthNotifier, AuthState>((ref) {
  return AuthNotifier();
});

// Step 2: Dependent provider — only fetches when authenticated
final userProfileProvider = FutureProvider<UserProfile>((ref) async {
  final auth = ref.watch(authProvider);
  if (auth is! Authenticated) throw UnauthorizedException();

  final api = ref.watch(apiClientProvider);
  return api.fetchProfile(auth.userId);
});

// Step 3: Provider that depends on user profile
final ordersProvider = FutureProvider<List<Order>>((ref) async {
  final profile = await ref.watch(userProfileProvider.future);
  final api = ref.watch(apiClientProvider);
  return api.fetchOrders(profile.id);
});

// Step 4: Combine everything for the dashboard
final dashboardProvider = FutureProvider<DashboardData>((ref) async {
  // These run in parallel since they don't depend on each other
  final results = await Future.wait([
    ref.watch(userProfileProvider.future),
    ref.watch(ordersProvider.future),
    ref.watch(notificationsProvider.future),
  ]);

  return DashboardData(
    profile: results[0] as UserProfile,
    orders: results[1] as List<Order>,
    notifications: results[2] as List<Notification>,
  );
});

// Step 5: Refresh with pull-to-refresh
class DashboardScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final dashboard = ref.watch(dashboardProvider);

    return RefreshIndicator(
      onRefresh: () async {
        // Invalidate triggers re-fetch of all dependents
        ref.invalidate(userProfileProvider);
        ref.invalidate(ordersProvider);
        ref.invalidate(notificationsProvider);
        // Wait for new data
        await ref.read(dashboardProvider.future);
      },
      child: dashboard.when(
        loading: () => DashboardSkeleton(),
        error: (err, stack) => ErrorView(error: err, onRetry: () {
          ref.invalidate(dashboardProvider);
        }),
        data: (data) => DashboardContent(data: data),
      ),
    );
  }
}
```

**Follow-up Questions:**
- "How do you handle stale data while refreshing?"
- "What happens if one of the parallel requests fails?"

---

### Question 66: Mocking Dependencies

**How do you mock dependencies for testing in Flutter?**

**Code Example:**

```dart
// Approach 1: Interface-based mocking with Mockito
abstract class UserRepository {
  Future<User> getUser(String id);
  Future<void> updateUser(User user);
}

class ApiUserRepository implements UserRepository {
  final ApiClient _api;
  ApiUserRepository(this._api);

  @override
  Future<User> getUser(String id) => _api.get('/users/$id');

  @override
  Future<void> updateUser(User user) => _api.put('/users/${user.id}', user.toJson());
}

// In test file
@GenerateMocks([UserRepository])
import 'user_test.mocks.dart';

void main() {
  late MockUserRepository mockRepo;
  late UserBloc bloc;

  setUp(() {
    mockRepo = MockUserRepository();
    bloc = UserBloc(mockRepo);
  });

  test('emits UserLoaded when getUser succeeds', () async {
    final user = User(id: '1', name: 'Alice');
    when(mockRepo.getUser('1')).thenAnswer((_) async => user);

    bloc.add(LoadUser('1'));

    await expectLater(
      bloc.stream,
      emitsInOrder([
        isA<UserLoading>(),
        isA<UserLoaded>().having((s) => s.user.name, 'name', 'Alice'),
      ]),
    );

    verify(mockRepo.getUser('1')).called(1);
  });

  test('emits UserError when getUser fails', () async {
    when(mockRepo.getUser('1')).thenThrow(NetworkException('timeout'));

    bloc.add(LoadUser('1'));

    await expectLater(
      bloc.stream,
      emitsInOrder([
        isA<UserLoading>(),
        isA<UserError>(),
      ]),
    );
  });
}

// Approach 2: Riverpod overrides (no mock classes needed)
void main() {
  test('dashboard loads data', () async {
    final container = ProviderContainer(
      overrides: [
        apiClientProvider.overrideWithValue(FakeApiClient()),
        userProfileProvider.overrideWith((ref) async {
          return UserProfile(id: '1', name: 'Test User');
        }),
      ],
    );

    final dashboard = await container.read(dashboardProvider.future);
    expect(dashboard.profile.name, 'Test User');

    container.dispose();
  });
}
```

**Common Mistakes:**

1. Mocking too many layers (test becomes meaningless)
2. Not verifying interactions (`verify` calls)
3. Mocking concrete classes instead of abstractions

---

### Question 67: Integration Tests

**How do integration tests differ from widget tests? When do you use them?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   TESTING PYRAMID                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│         /\          Integration Tests                           │
│        /  \         • Full app, real device/emulator            │
│       /    \        • Slow, expensive                           │
│      /──────\       • Test user flows end-to-end                │
│     /        \                                                  │
│    / Widget    \    Widget Tests                                │
│   /   Tests     \   • Single widget or screen                   │
│  /───────────────\  • Fast, no device needed                    │
│ /                 \ • Test UI behavior                          │
│/    Unit Tests     \                                            │
│─────────────────────  Unit Tests                                │
│                       • Pure logic, no Flutter                  │
│                       • Fastest                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:my_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Login Flow', () {
    testWidgets('user can login and see home screen', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Find and fill email field
      final emailField = find.byKey(Key('email_field'));
      await tester.enterText(emailField, 'test@example.com');

      // Find and fill password field
      final passwordField = find.byKey(Key('password_field'));
      await tester.enterText(passwordField, 'password123');

      // Tap login button
      await tester.tap(find.byKey(Key('login_button')));
      await tester.pumpAndSettle(Duration(seconds: 3));

      // Verify we're on home screen
      expect(find.text('Welcome'), findsOneWidget);
      expect(find.byType(HomeScreen), findsOneWidget);
    });

    testWidgets('shows error on invalid credentials', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      await tester.enterText(find.byKey(Key('email_field')), 'wrong@email.com');
      await tester.enterText(find.byKey(Key('password_field')), 'wrong');
      await tester.tap(find.byKey(Key('login_button')));
      await tester.pumpAndSettle(Duration(seconds: 3));

      expect(find.text('Invalid credentials'), findsOneWidget);
    });
  });
}
```

```bash
# Run integration tests
flutter test integration_test

# Run on specific device
flutter test integration_test --device-id=<device_id>
```

**When to use integration tests:**
- Critical user flows (login, checkout, onboarding)
- Flows that cross multiple screens
- Flows that interact with platform features
- Regression testing before releases

---

### Question 68: Golden Testing

**What are golden tests and when are they useful?**

**Theory:**

Golden tests compare widget renders against saved reference images ("goldens"). If a single pixel changes, the test fails — catching unintended visual regressions.

**Code Example:**

```dart
void main() {
  testWidgets('LoginScreen matches golden', (tester) async {
    await tester.pumpWidget(
      MaterialApp(
        home: LoginScreen(),
      ),
    );
    await tester.pumpAndSettle();

    // Compare against saved golden file
    await expectLater(
      find.byType(LoginScreen),
      matchesGoldenFile('goldens/login_screen.png'),
    );
  });

  testWidgets('Button states match golden', (tester) async {
    await tester.pumpWidget(
      MaterialApp(
        home: Scaffold(
          body: Column(
            children: [
              ElevatedButton(onPressed: () {}, child: Text('Enabled')),
              ElevatedButton(onPressed: null, child: Text('Disabled')),
            ],
          ),
        ),
      ),
    );

    await expectLater(
      find.byType(Scaffold),
      matchesGoldenFile('goldens/button_states.png'),
    );
  });
}
```

```bash
# Generate/update golden files
flutter test --update-goldens

# Run golden tests (compare against saved)
flutter test
```

**Common Mistakes:**

1. Goldens differ across platforms (Mac vs Linux CI) — use a consistent environment
2. Not committing golden files to version control
3. Testing too much in one golden (hard to tell what broke)

**Follow-up Questions:**
- "How do you handle golden tests in CI where the rendering environment differs?"
- "When would you choose golden tests over widget tests?"

---

## Advanced Performance

### Question 69: Memory Leak Detection

**How do you detect and fix memory leaks in Flutter?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│              COMMON MEMORY LEAKS IN FLUTTER                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. Forgotten listeners                                        │
│      StreamSubscription not cancelled                           │
│      AnimationController not disposed                           │
│      ScrollController not disposed                              │
│                                                                 │
│   2. Closures holding references                                │
│      Timer callbacks referencing disposed widget state           │
│      Async operations completing after widget disposal           │
│                                                                 │
│   3. Static/global references                                   │
│      Singletons holding widget references                       │
│      Static lists that grow forever                             │
│                                                                 │
│   4. Platform channel leaks                                     │
│      Native resources not released                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// ❌ LEAK: StreamSubscription never cancelled
class LeakyWidget extends StatefulWidget {
  @override
  State<LeakyWidget> createState() => _LeakyWidgetState();
}

class _LeakyWidgetState extends State<LeakyWidget> {
  @override
  void initState() {
    super.initState();
    // This subscription lives forever!
    FirebaseFirestore.instance
        .collection('messages')
        .snapshots()
        .listen((snapshot) {
      setState(() {}); // Widget may be disposed!
    });
  }
}

// ✅ FIXED: Properly manage subscription lifecycle
class FixedWidget extends StatefulWidget {
  @override
  State<FixedWidget> createState() => _FixedWidgetState();
}

class _FixedWidgetState extends State<FixedWidget> {
  StreamSubscription? _subscription;

  @override
  void initState() {
    super.initState();
    _subscription = FirebaseFirestore.instance
        .collection('messages')
        .snapshots()
        .listen((snapshot) {
      if (mounted) setState(() {});
    });
  }

  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }
}

// ❌ LEAK: Async gap — setState after dispose
Future<void> _loadData() async {
  final data = await api.fetchData(); // Takes 3 seconds
  setState(() => _data = data); // Widget may be disposed!
}

// ✅ FIXED: Check mounted before setState
Future<void> _loadData() async {
  final data = await api.fetchData();
  if (mounted) setState(() => _data = data);
}
```

**Detection tools:**
- **DevTools Memory tab** — heap snapshots, allocation tracking
- **`flutter run --observatory-port`** — connect memory profiler
- **Leak tracker** — `leak_tracker` package for automated detection

---

### Question 70: Custom RenderObjects

**When would you create a custom RenderObject?**

**Theory:**

Custom RenderObjects are the lowest level of Flutter's rendering system. You'd use them when existing widgets can't achieve the layout or painting behavior you need.

```
┌─────────────────────────────────────────────────────────────────┐
│               WHEN TO GO CUSTOM                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Try first:         Then try:           Last resort:           │
│   ──────────         ──────────          ────────────           │
│   Existing widgets → CustomPaint      → Custom RenderObject     │
│   Stack/Positioned → CustomMulti      → Custom RenderBox        │
│                      ChildLayout                                │
│                                                                 │
│   Custom RenderObject gives you:                                │
│   • Full control over layout (size, position of children)       │
│   • Full control over painting                                  │
│   • Full control over hit testing                               │
│   • Maximum performance (skip framework overhead)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Custom RenderObject: A simple gap widget (like SizedBox but semantic)
class Gap extends LeafRenderObjectWidget {
  final double size;

  const Gap(this.size, {super.key});

  @override
  RenderObject createRenderObject(BuildContext context) {
    return RenderGap(size);
  }

  @override
  void updateRenderObject(BuildContext context, RenderGap renderObject) {
    renderObject.gapSize = size;
  }
}

class RenderGap extends RenderBox {
  double _gapSize;

  RenderGap(this._gapSize);

  set gapSize(double value) {
    if (_gapSize != value) {
      _gapSize = value;
      markNeedsLayout(); // Tell framework to re-layout
    }
  }

  @override
  void performLayout() {
    // Determine this widget's size based on parent constraints
    final direction = _parentDirection;
    if (direction == Axis.vertical) {
      size = constraints.constrain(Size(0, _gapSize));
    } else {
      size = constraints.constrain(Size(_gapSize, 0));
    }
  }

  Axis get _parentDirection {
    final parent = this.parent;
    if (parent is RenderFlex) return parent.direction;
    return Axis.vertical;
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    // Nothing to paint — it's just a gap
  }
}
```

**Follow-up Questions:**
- "What's the difference between RenderBox and RenderSliver?"
- "How does Flutter's constraint-based layout differ from CSS flexbox?"

---

### Question 71: Efficient Pagination

**How do you implement efficient infinite scroll pagination?**

**Code Example:**

```dart
// Using Riverpod for paginated data
class PaginatedNotifier extends StateNotifier<PaginatedState<Post>> {
  final ApiClient _api;
  static const _pageSize = 20;

  PaginatedNotifier(this._api) : super(PaginatedState.initial()) {
    loadNextPage();
  }

  Future<void> loadNextPage() async {
    if (state.isLoading || !state.hasMore) return;

    state = state.copyWith(isLoading: true);

    try {
      final posts = await _api.getPosts(
        page: state.currentPage + 1,
        limit: _pageSize,
      );

      state = state.copyWith(
        items: [...state.items, ...posts],
        currentPage: state.currentPage + 1,
        hasMore: posts.length == _pageSize,
        isLoading: false,
      );
    } catch (e) {
      state = state.copyWith(isLoading: false, error: e.toString());
    }
  }

  Future<void> refresh() async {
    state = PaginatedState.initial();
    await loadNextPage();
  }
}

class PaginatedState<T> {
  final List<T> items;
  final int currentPage;
  final bool hasMore;
  final bool isLoading;
  final String? error;

  const PaginatedState({
    required this.items,
    required this.currentPage,
    required this.hasMore,
    required this.isLoading,
    this.error,
  });

  factory PaginatedState.initial() => PaginatedState(
    items: [],
    currentPage: 0,
    hasMore: true,
    isLoading: false,
  );

  PaginatedState<T> copyWith({
    List<T>? items,
    int? currentPage,
    bool? hasMore,
    bool? isLoading,
    String? error,
  }) {
    return PaginatedState(
      items: items ?? this.items,
      currentPage: currentPage ?? this.currentPage,
      hasMore: hasMore ?? this.hasMore,
      isLoading: isLoading ?? this.isLoading,
      error: error,
    );
  }
}

// Widget with scroll detection
class PaginatedListView extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(paginatedProvider);

    return NotificationListener<ScrollNotification>(
      onNotification: (notification) {
        // Load more when 80% scrolled
        if (notification.metrics.pixels >=
            notification.metrics.maxScrollExtent * 0.8) {
          ref.read(paginatedProvider.notifier).loadNextPage();
        }
        return false;
      },
      child: RefreshIndicator(
        onRefresh: () => ref.read(paginatedProvider.notifier).refresh(),
        child: ListView.builder(
          itemCount: state.items.length + (state.hasMore ? 1 : 0),
          itemBuilder: (context, index) {
            if (index == state.items.length) {
              return Center(child: CircularProgressIndicator());
            }
            return PostCard(post: state.items[index]);
          },
        ),
      ),
    );
  }
}
```

---

### Question 72: API Response Caching

**How do you implement API response caching in Flutter?**

**Code Example:**

```dart
// Multi-layer caching strategy
class CachedApiClient {
  final Dio _dio;
  final Box _cacheBox; // Hive box for persistent cache

  CachedApiClient(this._dio, this._cacheBox);

  Future<T> get<T>(
    String path, {
    required T Function(Map<String, dynamic>) fromJson,
    Duration maxAge = const Duration(minutes: 5),
    bool forceRefresh = false,
  }) async {
    final cacheKey = path.hashCode.toString();

    // 1. Check memory/disk cache first
    if (!forceRefresh) {
      final cached = _cacheBox.get(cacheKey);
      if (cached != null) {
        final cacheEntry = CacheEntry.fromJson(cached);
        if (!cacheEntry.isExpired(maxAge)) {
          return fromJson(cacheEntry.data);
        }
      }
    }

    // 2. Fetch from network
    try {
      final response = await _dio.get(path);

      // 3. Store in cache
      _cacheBox.put(cacheKey, CacheEntry(
        data: response.data,
        timestamp: DateTime.now(),
      ).toJson());

      return fromJson(response.data);
    } on DioException {
      // 4. Fallback to stale cache on network error
      final cached = _cacheBox.get(cacheKey);
      if (cached != null) {
        return fromJson(CacheEntry.fromJson(cached).data);
      }
      rethrow;
    }
  }
}

class CacheEntry {
  final Map<String, dynamic> data;
  final DateTime timestamp;

  CacheEntry({required this.data, required this.timestamp});

  bool isExpired(Duration maxAge) {
    return DateTime.now().difference(timestamp) > maxAge;
  }

  Map<String, dynamic> toJson() => {
    'data': data,
    'timestamp': timestamp.toIso8601String(),
  };

  factory CacheEntry.fromJson(Map<dynamic, dynamic> json) => CacheEntry(
    data: Map<String, dynamic>.from(json['data']),
    timestamp: DateTime.parse(json['timestamp']),
  );
}
```

**Caching strategies:**

| Strategy | When to Use |
|----------|-------------|
| Cache-first, network-fallback | Rarely changing data (config, categories) |
| Network-first, cache-fallback | Frequently changing data (feed, messages) |
| Stale-while-revalidate | Show cached data immediately, update in background |
| Cache with max-age | Time-sensitive data with known freshness window |

---

### Question 73: Runtime Permissions

**How do you handle runtime permissions in Flutter?**

**Code Example:**

```dart
// Using permission_handler package
class PermissionService {
  /// Request a single permission with rationale
  Future<bool> requestPermission(
    Permission permission, {
    required String rationale,
    required BuildContext context,
  }) async {
    var status = await permission.status;

    if (status.isGranted) return true;

    // Show rationale before requesting (Android best practice)
    if (status.isDenied) {
      final shouldRequest = await _showRationale(context, rationale);
      if (!shouldRequest) return false;

      status = await permission.request();
      return status.isGranted;
    }

    // Permanently denied — must open settings
    if (status.isPermanentlyDenied) {
      final opened = await _showSettingsDialog(context, rationale);
      if (opened) {
        // Check again after returning from settings
        return (await permission.status).isGranted;
      }
      return false;
    }

    return false;
  }

  /// Request multiple permissions at once
  Future<Map<Permission, bool>> requestMultiple(
    List<Permission> permissions,
  ) async {
    final statuses = await permissions.request();
    return statuses.map((key, value) =>
        MapEntry(key, value.isGranted));
  }

  Future<bool> _showRationale(BuildContext context, String message) async {
    return await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Permission Required'),
        content: Text(message),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: Text('Deny'),
          ),
          FilledButton(
            onPressed: () => Navigator.pop(context, true),
            child: Text('Allow'),
          ),
        ],
      ),
    ) ?? false;
  }

  Future<bool> _showSettingsDialog(BuildContext context, String message) async {
    final result = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Permission Required'),
        content: Text('$message\n\nPlease enable this in Settings.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: Text('Cancel'),
          ),
          FilledButton(
            onPressed: () async {
              Navigator.pop(context, true);
              await openAppSettings();
            },
            child: Text('Open Settings'),
          ),
        ],
      ),
    ) ?? false;
    return result;
  }
}

// Usage
final granted = await permissionService.requestPermission(
  Permission.camera,
  rationale: 'We need camera access to scan QR codes.',
  context: context,
);
if (granted) {
  // Proceed with camera
}
```

---

### Question 74: Background Tasks (WorkManager)

**How do you run background tasks in Flutter?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                BACKGROUND TASK OPTIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Task Type               Solution                              │
│   ─────────               ────────                              │
│   Short async work        Isolate (compute)                     │
│   Periodic sync           workmanager package                   │
│   Background fetch        workmanager + background_fetch        │
│   Location tracking       geolocator (foreground service)       │
│   Music playback          audio_service                         │
│   Push notification       firebase_messaging (onBackgroundMsg)  │
│                                                                 │
│   Platform constraints:                                         │
│   • iOS: ~30s background time, BGTaskScheduler for periodic     │
│   • Android: WorkManager handles doze, battery optimization     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Using workmanager package
void main() {
  WidgetsFlutterBinding.ensureInitialized();

  Workmanager().initialize(callbackDispatcher, isInDebugMode: true);

  // Register periodic task
  Workmanager().registerPeriodicTask(
    'sync-data',
    'syncDataTask',
    frequency: Duration(hours: 1),
    constraints: Constraints(
      networkType: NetworkType.connected,
      requiresBatteryNotLow: true,
    ),
    inputData: {'userId': 'user_123'},
  );

  runApp(MyApp());
}

// This MUST be a top-level function (not a method)
@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((taskName, inputData) async {
    switch (taskName) {
      case 'syncDataTask':
        try {
          final userId = inputData?['userId'] ?? '';
          await SyncService.syncUserData(userId);
          return true; // Success
        } catch (e) {
          return false; // Retry later
        }
      default:
        return false;
    }
  });
}
```

**Common Mistakes:**

1. Assuming background tasks run reliably on iOS (they don't — OS decides when)
2. Not handling the case where the app was killed between task scheduling and execution
3. Heavy Dart initialization in background callback (keep it minimal)

---

### Question 75: Push Notifications

**How do you implement push notifications in Flutter?**

**Code Example:**

```dart
// Using firebase_messaging
class NotificationService {
  final FirebaseMessaging _messaging = FirebaseMessaging.instance;

  Future<void> initialize() async {
    // 1. Request permission
    final settings = await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
      provisional: false,
    );

    if (settings.authorizationStatus != AuthorizationStatus.authorized) {
      return;
    }

    // 2. Get FCM token (send to your server)
    final token = await _messaging.getToken();
    await _sendTokenToServer(token);

    // 3. Listen for token refresh
    _messaging.onTokenRefresh.listen(_sendTokenToServer);

    // 4. Handle foreground messages
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // 5. Handle background message taps (app was in background)
    FirebaseMessaging.onMessageOpenedApp.listen(_handleMessageTap);

    // 6. Handle terminated state tap (app was killed)
    final initialMessage = await _messaging.getInitialMessage();
    if (initialMessage != null) {
      _handleMessageTap(initialMessage);
    }
  }

  void _handleForegroundMessage(RemoteMessage message) {
    // Show local notification (firebase_messaging doesn't show UI in foreground)
    LocalNotificationService.show(
      title: message.notification?.title ?? '',
      body: message.notification?.body ?? '',
      payload: message.data,
    );
  }

  void _handleMessageTap(RemoteMessage message) {
    final route = message.data['route'];
    if (route != null) {
      NavigationService.navigateTo(route);
    }
  }
}

// Background message handler — MUST be top-level
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  // Process data-only messages (silent notifications)
  await DataSyncService.processBackgroundMessage(message.data);
}

// Register in main()
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);
  runApp(MyApp());
}
```

---

### Question 76: In-App Purchases

**How do you implement in-app purchases in Flutter?**

**Code Example:**

```dart
// Using in_app_purchase package (official Flutter plugin)
class PurchaseService {
  final InAppPurchase _iap = InAppPurchase.instance;
  StreamSubscription<List<PurchaseDetails>>? _subscription;

  static const _productIds = {'premium_monthly', 'premium_yearly', 'remove_ads'};

  Future<void> initialize() async {
    final available = await _iap.isAvailable();
    if (!available) return;

    // Listen to purchase updates
    _subscription = _iap.purchaseStream.listen(
      _handlePurchaseUpdates,
      onError: (error) => debugPrint('Purchase error: $error'),
    );
  }

  Future<List<ProductDetails>> loadProducts() async {
    final response = await _iap.queryProductDetails(_productIds);
    if (response.error != null) {
      throw PurchaseException(response.error!.message);
    }
    return response.productDetails;
  }

  Future<void> buy(ProductDetails product) async {
    final purchaseParam = PurchaseParam(productDetails: product);

    // For consumables (coins, gems)
    // await _iap.buyConsumable(purchaseParam: purchaseParam);

    // For subscriptions and non-consumables
    await _iap.buyNonConsumable(purchaseParam: purchaseParam);
  }

  void _handlePurchaseUpdates(List<PurchaseDetails> purchases) {
    for (final purchase in purchases) {
      switch (purchase.status) {
        case PurchaseStatus.purchased:
        case PurchaseStatus.restored:
          _verifyAndDeliver(purchase);
          break;
        case PurchaseStatus.error:
          _handleError(purchase);
          break;
        case PurchaseStatus.pending:
          // Show pending UI
          break;
        case PurchaseStatus.canceled:
          break;
      }

      // CRITICAL: Always complete the purchase
      if (purchase.pendingCompletePurchase) {
        _iap.completePurchase(purchase);
      }
    }
  }

  Future<void> _verifyAndDeliver(PurchaseDetails purchase) async {
    // ALWAYS verify on your server, never trust the client
    final verified = await _serverVerify(purchase);
    if (verified) {
      await _deliverProduct(purchase.productID);
    }
  }

  void dispose() {
    _subscription?.cancel();
  }
}
```

**Common Mistakes:**

1. Not calling `completePurchase` (purchase stays pending forever)
2. Verifying purchases on the client side (easily bypassed)
3. Not handling `restored` purchases (users switching devices)

---

### Question 77: Analytics & Crash Reporting

**How do you set up analytics and crash reporting?**

**Code Example:**

```dart
// Unified analytics and crash reporting setup
class ObservabilityService {
  static Future<void> initialize() async {
    // Crash reporting
    FlutterError.onError = (details) {
      FirebaseCrashlytics.instance.recordFlutterFatalError(details);
    };

    PlatformDispatcher.instance.onError = (error, stack) {
      FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
      return true;
    };
  }

  // Structured analytics events
  static void trackEvent(String name, {Map<String, Object>? params}) {
    FirebaseAnalytics.instance.logEvent(name: name, parameters: params);
  }

  // Screen tracking
  static void trackScreen(String screenName) {
    FirebaseAnalytics.instance.logScreenView(screenName: screenName);
  }

  // User properties
  static void setUserProperties({required String userId, String? plan}) {
    FirebaseCrashlytics.instance.setUserIdentifier(userId);
    FirebaseAnalytics.instance.setUserId(id: userId);
    if (plan != null) {
      FirebaseAnalytics.instance.setUserProperty(name: 'plan', value: plan);
    }
  }

  // Custom breadcrumbs for crash context
  static void log(String message) {
    FirebaseCrashlytics.instance.log(message);
  }
}

// Auto screen tracking with go_router
final router = GoRouter(
  observers: [
    FirebaseAnalyticsObserver(analytics: FirebaseAnalytics.instance),
  ],
  routes: [...],
);
```

---

### Question 78: Code Coverage

**How do you measure and enforce code coverage?**

```bash
# Run tests with coverage
flutter test --coverage

# Generate HTML report
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html

# Check coverage threshold in CI
flutter test --coverage
lcov --summary coverage/lcov.info | grep "lines" | awk '{print $2}' | \
  awk -F'%' '{ if ($1 < 80) exit 1 }'
```

**What to measure:**

| Metric | Target | What It Means |
|--------|--------|---------------|
| Line coverage | 80%+ | % of lines executed by tests |
| Branch coverage | 70%+ | % of if/else branches tested |
| Function coverage | 90%+ | % of functions called by tests |

**What NOT to cover (diminishing returns):**
- Generated code (`.g.dart`, `.freezed.dart`)
- Simple data classes with no logic
- Platform channel glue code
- UI layout code (use golden tests instead)

```bash
# Exclude generated files from coverage in lcov
lcov --remove coverage/lcov.info \
  '**/*.g.dart' \
  '**/*.freezed.dart' \
  '**/generated/**' \
  -o coverage/lcov_filtered.info
```

---

## Architecture & Infrastructure

### Question 79: Feature Flags & A/B Testing

**How do you implement feature flags in a Flutter app?**

**Code Example:**

```dart
// Feature flag service with remote config
class FeatureFlagService {
  final FirebaseRemoteConfig _remoteConfig;

  FeatureFlagService(this._remoteConfig);

  Future<void> initialize() async {
    await _remoteConfig.setDefaults({
      'new_checkout_flow': false,
      'max_upload_size_mb': 10,
      'onboarding_variant': 'control',
    });

    await _remoteConfig.setConfigSettings(RemoteConfigSettings(
      fetchTimeout: Duration(seconds: 10),
      minimumFetchInterval: Duration(hours: 1),
    ));

    await _remoteConfig.fetchAndActivate();
  }

  bool isEnabled(String flag) => _remoteConfig.getBool(flag);
  String getString(String flag) => _remoteConfig.getString(flag);
  int getInt(String flag) => _remoteConfig.getInt(flag);
}

// Usage with Riverpod
final featureFlagProvider = Provider<FeatureFlagService>((ref) {
  return FeatureFlagService(FirebaseRemoteConfig.instance);
});

// In widgets
class CheckoutScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final flags = ref.watch(featureFlagProvider);

    if (flags.isEnabled('new_checkout_flow')) {
      return NewCheckoutFlow();
    }
    return LegacyCheckoutFlow();
  }
}

// A/B testing with analytics tracking
class OnboardingScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final flags = ref.watch(featureFlagProvider);
    final variant = flags.getString('onboarding_variant');

    // Track which variant the user sees
    ObservabilityService.trackEvent('onboarding_viewed', params: {
      'variant': variant,
    });

    return switch (variant) {
      'variant_a' => OnboardingVariantA(),
      'variant_b' => OnboardingVariantB(),
      _ => OnboardingControl(),
    };
  }
}
```

---

### Question 80: Design System Implementation

**How do you build a design system in Flutter?**

**Code Example:**

```dart
// 1. Tokens — the foundation
class AppTokens {
  // Spacing
  static const space4 = 4.0;
  static const space8 = 8.0;
  static const space12 = 12.0;
  static const space16 = 16.0;
  static const space24 = 24.0;
  static const space32 = 32.0;

  // Border radius
  static const radiusSmall = 4.0;
  static const radiusMedium = 8.0;
  static const radiusLarge = 16.0;
  static const radiusFull = 999.0;

  // Elevation
  static const elevationLow = 2.0;
  static const elevationMedium = 4.0;
  static const elevationHigh = 8.0;
}

// 2. Theme — colors and typography
class AppTheme {
  static ThemeData light() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: Color(0xFF1A73E8),
      brightness: Brightness.light,
    );

    return ThemeData(
      colorScheme: colorScheme,
      useMaterial3: true,
      textTheme: _textTheme,
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          padding: EdgeInsets.symmetric(
            horizontal: AppTokens.space24,
            vertical: AppTokens.space12,
          ),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(AppTokens.radiusMedium),
          ),
        ),
      ),
    );
  }

  static ThemeData dark() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: Color(0xFF1A73E8),
      brightness: Brightness.dark,
    );
    return ThemeData(colorScheme: colorScheme, useMaterial3: true);
  }

  static const _textTheme = TextTheme(
    displayLarge: TextStyle(fontSize: 32, fontWeight: FontWeight.bold),
    titleLarge: TextStyle(fontSize: 22, fontWeight: FontWeight.w600),
    bodyLarge: TextStyle(fontSize: 16, height: 1.5),
    bodyMedium: TextStyle(fontSize: 14, height: 1.5),
    labelLarge: TextStyle(fontSize: 14, fontWeight: FontWeight.w600),
  );
}

// 3. Components — reusable building blocks
class AppButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final AppButtonVariant variant;

  const AppButton({
    required this.label,
    this.onPressed,
    this.variant = AppButtonVariant.primary,
    super.key,
  });

  @override
  Widget build(BuildContext context) {
    return switch (variant) {
      AppButtonVariant.primary => FilledButton(
          onPressed: onPressed,
          child: Text(label),
        ),
      AppButtonVariant.secondary => OutlinedButton(
          onPressed: onPressed,
          child: Text(label),
        ),
      AppButtonVariant.text => TextButton(
          onPressed: onPressed,
          child: Text(label),
        ),
    };
  }
}

enum AppButtonVariant { primary, secondary, text }
```

---

### Question 81: CI/CD Pipeline Setup

**How do you set up a CI/CD pipeline for a Flutter app?**

```yaml
# .github/workflows/flutter_ci.yml
name: Flutter CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  analyze-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.27.x'
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Verify formatting
        run: dart format --set-exit-if-changed .

      - name: Run analyzer
        run: flutter analyze --fatal-infos

      - name: Run tests with coverage
        run: flutter test --coverage

      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | awk '{print $2}' | sed 's/%//')
          echo "Coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage below 80%!"
            exit 1
          fi

  build-android:
    needs: analyze-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.27.x'

      - name: Build APK
        run: flutter build apk --release --obfuscate --split-debug-info=./debug-info

      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_JSON }}
          packageName: com.example.app
          releaseFiles: build/app/outputs/flutter-apk/app-release.apk
          track: internal

  build-ios:
    needs: analyze-and-test
    runs-on: macos-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.27.x'

      - name: Build iOS
        run: flutter build ipa --release --obfuscate --split-debug-info=./debug-info

      - name: Upload to TestFlight
        uses: apple-actions/upload-testflight-build@v1
        with:
          app-path: build/ios/ipa/*.ipa
          issuer-id: ${{ secrets.APPLE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPLE_API_KEY_ID }}
          api-private-key: ${{ secrets.APPLE_API_PRIVATE_KEY }}
```

**Key principles:**
- Fail fast: formatting and analysis before expensive builds
- Cache dependencies for speed
- Separate build and deploy stages
- Use secrets for signing keys and store credentials
- Run builds only on main branch (not on every PR)

---

### Question 82: Modular Architecture

**How do you structure a large Flutter app for multiple teams?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                MODULAR ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   app/                    ← Shell app (routing, DI, theme)      │
│   ├── lib/                                                      │
│   │   └── main.dart                                             │
│   │                                                             │
│   packages/               ← Shared packages                     │
│   ├── core/               ← Models, utils, constants            │
│   ├── design_system/      ← Shared UI components                │
│   ├── networking/         ← API client, interceptors            │
│   └── analytics/          ← Tracking, crash reporting           │
│   │                                                             │
│   features/               ← Feature modules (team-owned)        │
│   ├── auth/               ← Login, signup, password reset       │
│   ├── home/               ← Home feed, dashboard                │
│   ├── profile/            ← User profile, settings              │
│   ├── checkout/           ← Cart, payment, order tracking       │
│   └── search/             ← Search, filters, results            │
│                                                                 │
│   Each feature is a separate Dart package with:                 │
│   ├── lib/src/            ← Private implementation              │
│   ├── lib/feature.dart    ← Public API (barrel file)            │
│   ├── test/               ← Feature tests                       │
│   └── pubspec.yaml        ← Own dependencies                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- Teams own features independently
- Clear dependency boundaries
- Faster builds (only rebuild changed modules)
- Enforced API contracts between modules

**Melos for monorepo management:**

```yaml
# melos.yaml
name: my_app_workspace
packages:
  - app
  - packages/**
  - features/**

scripts:
  analyze:
    run: melos exec -- flutter analyze
  test:
    run: melos exec -- flutter test
  format:
    run: melos exec -- dart format --set-exit-if-changed .
```

```bash
# Bootstrap all packages
melos bootstrap

# Run tests across all packages
melos run test

# Publish only changed packages
melos version
```

---

### Question 83: App Update Management

**How do you handle forced and optional app updates?**

**Code Example:**

```dart
class AppUpdateService {
  final PackageInfo _packageInfo;
  final RemoteConfig _remoteConfig;

  AppUpdateService(this._packageInfo, this._remoteConfig);

  Future<UpdateAction> checkForUpdate() async {
    await _remoteConfig.fetchAndActivate();

    final currentVersion = Version.parse(_packageInfo.version);
    final minVersion = Version.parse(_remoteConfig.getString('min_app_version'));
    final latestVersion = Version.parse(_remoteConfig.getString('latest_app_version'));

    if (currentVersion < minVersion) {
      return UpdateAction.forceUpdate;
    }
    if (currentVersion < latestVersion) {
      return UpdateAction.optionalUpdate;
    }
    return UpdateAction.none;
  }
}

enum UpdateAction { none, optionalUpdate, forceUpdate }

// Show appropriate UI
class UpdateChecker extends StatelessWidget {
  final Widget child;

  const UpdateChecker({required this.child});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<UpdateAction>(
      future: context.read<AppUpdateService>().checkForUpdate(),
      builder: (context, snapshot) {
        if (snapshot.data == UpdateAction.forceUpdate) {
          return ForceUpdateScreen(); // No way to dismiss
        }

        // Show optional update as a banner, don't block the app
        return Column(
          children: [
            if (snapshot.data == UpdateAction.optionalUpdate)
              MaterialBanner(
                content: Text('A new version is available!'),
                actions: [
                  TextButton(
                    onPressed: () => _openStore(),
                    child: Text('Update'),
                  ),
                ],
              ),
            Expanded(child: child),
          ],
        );
      },
    );
  }
}
```

---

### Question 84: Localization (i18n)

**How do you implement localization in Flutter?**

**Code Example:**

```yaml
# pubspec.yaml
dependencies:
  flutter_localizations:
    sdk: flutter
  intl: any

flutter:
  generate: true
```

```yaml
# l10n.yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

```json
// lib/l10n/app_en.arb
{
  "@@locale": "en",
  "appTitle": "My App",
  "welcomeMessage": "Welcome, {name}!",
  "@welcomeMessage": {
    "placeholders": {
      "name": { "type": "String" }
    }
  },
  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "@itemCount": {
    "placeholders": {
      "count": { "type": "int" }
    }
  },
  "lastLogin": "Last login: {date}",
  "@lastLogin": {
    "placeholders": {
      "date": {
        "type": "DateTime",
        "format": "yMMMd"
      }
    }
  }
}
```

```json
// lib/l10n/app_es.arb
{
  "@@locale": "es",
  "appTitle": "Mi Aplicacion",
  "welcomeMessage": "Bienvenido, {name}!",
  "itemCount": "{count, plural, =0{Sin articulos} =1{1 articulo} other{{count} articulos}}",
  "lastLogin": "Ultimo inicio: {date}"
}
```

```dart
// App setup
MaterialApp(
  localizationsDelegates: AppLocalizations.localizationsDelegates,
  supportedLocales: AppLocalizations.supportedLocales,
  home: HomeScreen(),
);

// Usage
Text(AppLocalizations.of(context)!.welcomeMessage('Alice'));
Text(AppLocalizations.of(context)!.itemCount(5));
```

**Best practices:**
- Never hardcode strings — always use ARB files
- Use ICU message format for plurals, genders, selects
- Test with pseudolocalization to catch layout issues
- Support RTL languages (Arabic, Hebrew) from the start

---

### Question 85: Accessibility (a11y)

**How do you make a Flutter app accessible?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│                ACCESSIBILITY CHECKLIST                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. Semantics       → Screen readers (TalkBack, VoiceOver)     │
│   2. Contrast        → WCAG 2.1 AA (4.5:1 for text)            │
│   3. Touch targets   → Minimum 48x48 dp                        │
│   4. Text scaling    → Support up to 2x font size              │
│   5. Motion          → Respect reduce motion preference         │
│   6. Focus           → Keyboard and switch navigation           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// 1. Semantics for screen readers
Semantics(
  label: 'Delete item',
  hint: 'Double tap to delete this item',
  button: true,
  child: IconButton(
    icon: Icon(Icons.delete),
    onPressed: () => deleteItem(),
  ),
);

// For images
Semantics(
  image: true,
  label: 'Profile photo of Alice',
  child: CircleAvatar(backgroundImage: NetworkImage(url)),
);

// Exclude decorative elements
Semantics(
  excludeSemantics: true, // Screen reader skips this
  child: Icon(Icons.decoration),
);

// 2. Support text scaling
class AccessibleText extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Text(
      'Hello',
      style: Theme.of(context).textTheme.bodyLarge,
      // Don't clamp textScaleFactor — let users control it
      // ❌ textScaleFactor: 1.0  // Breaks accessibility
    );
  }
}

// 3. Respect reduced motion
class MotionAwareWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final reduceMotion = MediaQuery.of(context).disableAnimations;

    return AnimatedContainer(
      duration: reduceMotion
          ? Duration.zero          // No animation
          : Duration(milliseconds: 300),
      width: _expanded ? 200 : 100,
    );
  }
}

// 4. Minimum touch targets
SizedBox(
  width: 48, // Minimum 48x48
  height: 48,
  child: IconButton(
    icon: Icon(Icons.close),
    onPressed: () {},
  ),
);

// 5. Test accessibility
testWidgets('button has correct semantics', (tester) async {
  await tester.pumpWidget(MyWidget());

  final semantics = tester.getSemantics(find.byType(IconButton));
  expect(semantics.label, 'Delete item');
  expect(semantics.hasAction(SemanticsAction.tap), true);
});
```

**Testing tools:**
- `flutter test` with semantics assertions
- Android: Accessibility Scanner app
- iOS: Xcode Accessibility Inspector
- `SemanticsDebugger` widget for visual debugging

---

## Summary

At senior level, you should be able to:

| Area | Expectation |
|------|-------------|
| Platform Integration | Bridge Dart to native code, choose between channels and FFI |
| Testing | Write unit, widget, integration, and golden tests |
| Performance | Detect memory leaks, optimize startup, build custom RenderObjects |
| Infrastructure | Set up CI/CD, manage releases, implement feature flags |
| Architecture | Design modular systems for multiple teams |
| User Experience | Implement localization, accessibility, push notifications |

---

**Made with by [Debasmita Sarkar](https://github.com/debasmitasarkar)**

[![Back to Main](https://img.shields.io/badge/←_Back_to_Main-blue?style=flat-square)](/README.md)
[![Previous: Mid-Level](https://img.shields.io/badge/←_Previous:_Mid--Level-orange?style=flat-square)](/mid-level/README.md)
[![Next: Expert Level](https://img.shields.io/badge/Next:_Expert_Level_→-green?style=flat-square)](/expert/README.md)
