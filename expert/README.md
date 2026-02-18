# Flutter Interview Questions 2026 - Expert Level (Questions 86-100)

**Experience: 6+ Years**

[![Back to Main](https://img.shields.io/badge/←_Back_to_Main-blue?style=flat-square)](/README.md)

---

## What Makes Expert Different?

At this level, the interview isn't about what you know. It's about how you think.

You won't get syntax questions. You'll get ambiguous problems with no single right answer. Interviewers want to see how you break down complexity, communicate trade-offs, make decisions under uncertainty, and lead teams through hard problems.

You're expected to:
- Design systems, not just features
- Make decisions that survive at scale
- Own outcomes across teams and timelines
- Communicate technical trade-offs to non-technical stakeholders
- Know when NOT to build something

---

## Table of Contents

| Section | Questions | Topics |
|---------|-----------|--------|
| [System Design](#system-design) | 86-92 | Architecture, Offline-first, WebSockets, Performance |
| [Leadership & Behavioral](#leadership--behavioral) | 93-100 | Communication, Mentoring, Estimation, Incidents |

---

## System Design

### Question 86: Large-Scale App Architecture

**Design the architecture for a Flutter app with 100+ screens, 10+ developers, and millions of users.**

**Theory:**

This is an open-ended system design question. There's no single right answer — they want to see your decision-making process.

```
┌─────────────────────────────────────────────────────────────────┐
│            LARGE-SCALE FLUTTER ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Presentation Layer                                            │
│   ──────────────────                                            │
│   • Feature modules (each team owns 1-3 features)              │
│   • Shared design system package                                │
│   • Router module (centralized navigation)                      │
│                                                                 │
│   Domain Layer                                                  │
│   ────────────                                                  │
│   • Use cases / interactors (business logic)                    │
│   • Domain models (pure Dart, no dependencies)                  │
│   • Repository interfaces                                       │
│                                                                 │
│   Data Layer                                                    │
│   ──────────                                                    │
│   • Repository implementations                                  │
│   • Remote data sources (API clients)                           │
│   • Local data sources (DB, cache, secure storage)              │
│   • DTOs and mappers                                            │
│                                                                 │
│   Core / Infrastructure                                         │
│   ─────────────────────                                         │
│   • Networking (Dio + interceptors)                              │
│   • Authentication                                              │
│   • Analytics / Crash reporting                                 │
│   • Feature flags                                               │
│   • Logging                                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key decisions to discuss:**

```dart
// 1. MONOREPO WITH FEATURE PACKAGES
// Each feature is an independent Dart package

// features/checkout/lib/checkout.dart (public API)
export 'src/checkout_screen.dart';
export 'src/checkout_routes.dart';
// Everything in src/ is private to the package

// features/checkout/lib/src/checkout_screen.dart
class CheckoutScreen extends ConsumerWidget {
  // Only depends on: core, design_system, domain
  // NEVER depends on another feature directly
}

// 2. DEPENDENCY RULE
// Dependencies only flow inward:
// Presentation → Domain ← Data
// Domain has ZERO external dependencies

// domain/lib/src/repositories/order_repository.dart
abstract class OrderRepository {
  Future<Order> getOrder(String id);
  Future<void> placeOrder(Cart cart);
  Stream<OrderStatus> watchOrderStatus(String id);
}
// Pure Dart. No Flutter. No packages. No implementations.

// data/lib/src/repositories/order_repository_impl.dart
class OrderRepositoryImpl implements OrderRepository {
  final ApiClient _api;
  final OrderCache _cache;
  final OrderDb _db;

  OrderRepositoryImpl(this._api, this._cache, this._db);

  @override
  Future<Order> getOrder(String id) async {
    // Try cache → try DB → fetch from API → cache result
    final cached = _cache.get(id);
    if (cached != null) return cached;

    try {
      final dto = await _api.getOrder(id);
      final order = dto.toDomain();
      await _db.save(order);
      _cache.put(id, order);
      return order;
    } catch (e) {
      final local = await _db.getOrder(id);
      if (local != null) return local;
      rethrow;
    }
  }
}

// 3. FEATURE COMMUNICATION
// Features don't import each other. They communicate through:

// Option A: Shared domain events
abstract class AppEvent {}
class OrderPlaced extends AppEvent {
  final String orderId;
  OrderPlaced(this.orderId);
}

// Option B: Router-based navigation with typed parameters
context.push('/order-confirmation', extra: OrderConfirmationArgs(orderId: '123'));

// Option C: Shared state via providers
final cartProvider = StateNotifierProvider<CartNotifier, Cart>((ref) {
  return CartNotifier(ref.watch(cartRepositoryProvider));
});
// Both checkout and cart features can watch this
```

**What interviewers look for:**
- Clear separation of concerns
- How features communicate without coupling
- How you handle shared state across features
- Testing strategy at each layer
- How you onboard new developers to this architecture

---

### Question 87: Offline-First Architecture

**Design an offline-first Flutter app that syncs when connectivity returns.**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│              OFFLINE-FIRST ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   User Action                                                   │
│       │                                                         │
│       ▼                                                         │
│   ┌──────────────┐                                              │
│   │ Write to     │ ← All writes go to local DB first            │
│   │ Local DB     │                                              │
│   └──────┬───────┘                                              │
│          │                                                      │
│          ├──────────────────────────┐                            │
│          ▼                          ▼                            │
│   ┌──────────────┐          ┌──────────────┐                    │
│   │ Update UI    │          │ Queue sync   │                    │
│   │ immediately  │          │ operation    │                    │
│   └──────────────┘          └──────┬───────┘                    │
│                                    │                            │
│                              ┌─────▼─────┐                      │
│                              │ Online?    │                      │
│                              ├─────┬──Yes─┤                      │
│                              │ No  │      │                      │
│                              │     ▼      │                      │
│                              │  Sync to   │                      │
│                              │  server    │                      │
│                              └────────────┘                      │
│                                                                 │
│   Conflict Resolution:                                          │
│   • Last-write-wins (simple, data loss risk)                    │
│   • Server-wins (safe, may lose local changes)                  │
│   • Merge (complex, best UX)                                    │
│   • CRDTs (automatic, limited data types)                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Sync queue that persists across app restarts
class SyncQueue {
  final Box<SyncOperation> _box;

  SyncQueue(this._box);

  Future<void> enqueue(SyncOperation operation) async {
    await _box.add(operation);
  }

  Future<void> processQueue(ApiClient api) async {
    final pending = _box.values.toList()
      ..sort((a, b) => a.createdAt.compareTo(b.createdAt));

    for (final op in pending) {
      try {
        await _executeSyncOperation(api, op);
        await op.delete(); // Remove from queue on success
      } on ConflictException catch (e) {
        await _resolveConflict(op, e.serverVersion);
      } on NetworkException {
        break; // Stop processing, will retry when online
      }
    }
  }

  Future<void> _executeSyncOperation(ApiClient api, SyncOperation op) async {
    switch (op.type) {
      case SyncType.create:
        await api.post(op.endpoint, data: op.payload);
      case SyncType.update:
        await api.put(op.endpoint, data: op.payload);
      case SyncType.delete:
        await api.delete(op.endpoint);
    }
  }

  Future<void> _resolveConflict(SyncOperation local, dynamic serverVersion) async {
    // Strategy: server-wins with user notification
    await _localDb.update(local.entityId, serverVersion);
    await local.delete();
    _conflictNotifier.add(ConflictResolved(
      entity: local.entityId,
      resolution: 'Server version kept',
    ));
  }
}

@HiveType(typeId: 0)
class SyncOperation extends HiveObject {
  @HiveField(0)
  final SyncType type;

  @HiveField(1)
  final String endpoint;

  @HiveField(2)
  final Map<String, dynamic>? payload;

  @HiveField(3)
  final DateTime createdAt;

  @HiveField(4)
  final String entityId;

  SyncOperation({
    required this.type,
    required this.endpoint,
    this.payload,
    required this.entityId,
    DateTime? createdAt,
  }) : createdAt = createdAt ?? DateTime.now();
}

// Connectivity-aware sync trigger
class SyncManager {
  final SyncQueue _queue;
  final ApiClient _api;
  StreamSubscription? _connectivitySub;

  SyncManager(this._queue, this._api);

  void startListening() {
    _connectivitySub = Connectivity()
        .onConnectivityChanged
        .listen((result) {
      if (result != ConnectivityResult.none) {
        _queue.processQueue(_api);
      }
    });
  }

  void dispose() {
    _connectivitySub?.cancel();
  }
}

// Repository that writes locally first
class OfflineFirstTodoRepository implements TodoRepository {
  final TodoLocalDb _localDb;
  final SyncQueue _syncQueue;

  OfflineFirstTodoRepository(this._localDb, this._syncQueue);

  @override
  Future<void> addTodo(Todo todo) async {
    // 1. Write to local DB immediately
    await _localDb.insert(todo);

    // 2. Queue sync operation
    await _syncQueue.enqueue(SyncOperation(
      type: SyncType.create,
      endpoint: '/todos',
      payload: todo.toJson(),
      entityId: todo.id,
    ));
  }

  @override
  Stream<List<Todo>> watchTodos() {
    // Always read from local DB — it's the source of truth
    return _localDb.watchAll();
  }
}
```

**Key discussion points:**
- Local DB is the source of truth, not the server
- Optimistic UI — user sees changes immediately
- Conflict resolution strategy depends on the domain
- Queue must persist across app restarts (Hive, SQLite)
- Idempotency — retried operations must be safe to repeat

---

### Question 88: Real-Time with WebSockets

**Design a real-time chat feature using WebSockets in Flutter.**

**Code Example:**

```dart
// WebSocket service with automatic reconnection
class WebSocketService {
  WebSocketChannel? _channel;
  final _messageController = StreamController<ChatMessage>.broadcast();
  final _connectionState = BehaviorSubject<ConnectionState>.seeded(
    ConnectionState.disconnected,
  );

  Timer? _reconnectTimer;
  Timer? _heartbeatTimer;
  int _reconnectAttempts = 0;
  static const _maxReconnectAttempts = 10;

  Stream<ChatMessage> get messages => _messageController.stream;
  Stream<ConnectionState> get connectionState => _connectionState.stream;

  Future<void> connect(String url, String authToken) async {
    _connectionState.add(ConnectionState.connecting);

    try {
      _channel = WebSocketChannel.connect(
        Uri.parse(url),
        protocols: ['chat-v1'],
      );

      // Send auth on connect
      _channel!.sink.add(jsonEncode({
        'type': 'auth',
        'token': authToken,
      }));

      _channel!.stream.listen(
        _handleMessage,
        onError: _handleError,
        onDone: _handleDisconnect,
      );

      _connectionState.add(ConnectionState.connected);
      _reconnectAttempts = 0;
      _startHeartbeat();
    } catch (e) {
      _connectionState.add(ConnectionState.disconnected);
      _scheduleReconnect(url, authToken);
    }
  }

  void _handleMessage(dynamic data) {
    final json = jsonDecode(data as String);

    switch (json['type']) {
      case 'message':
        _messageController.add(ChatMessage.fromJson(json['payload']));
      case 'pong':
        // Heartbeat acknowledged
        break;
      case 'typing':
        // Handle typing indicators
        break;
    }
  }

  void sendMessage(String roomId, String content) {
    _channel?.sink.add(jsonEncode({
      'type': 'message',
      'payload': {
        'roomId': roomId,
        'content': content,
        'clientId': _generateClientId(), // For deduplication
        'timestamp': DateTime.now().toIso8601String(),
      },
    }));
  }

  void _startHeartbeat() {
    _heartbeatTimer?.cancel();
    _heartbeatTimer = Timer.periodic(Duration(seconds: 30), (_) {
      _channel?.sink.add(jsonEncode({'type': 'ping'}));
    });
  }

  void _handleDisconnect() {
    _connectionState.add(ConnectionState.disconnected);
    _heartbeatTimer?.cancel();
    _scheduleReconnect(_lastUrl, _lastToken);
  }

  void _scheduleReconnect(String url, String token) {
    if (_reconnectAttempts >= _maxReconnectAttempts) {
      _connectionState.add(ConnectionState.failed);
      return;
    }

    // Exponential backoff with jitter
    final delay = Duration(
      milliseconds: (pow(2, _reconnectAttempts) * 1000).toInt() +
          Random().nextInt(1000),
    );

    _reconnectTimer = Timer(delay, () {
      _reconnectAttempts++;
      connect(url, token);
    });
  }

  void _handleError(dynamic error) {
    debugPrint('WebSocket error: $error');
  }

  void dispose() {
    _reconnectTimer?.cancel();
    _heartbeatTimer?.cancel();
    _channel?.sink.close();
    _messageController.close();
    _connectionState.close();
  }
}

enum ConnectionState { disconnected, connecting, connected, failed }

// Chat UI with optimistic updates
class ChatScreen extends ConsumerStatefulWidget {
  final String roomId;
  const ChatScreen({required this.roomId});

  @override
  ConsumerState<ChatScreen> createState() => _ChatScreenState();
}

class _ChatScreenState extends ConsumerState<ChatScreen> {
  final _controller = TextEditingController();
  final _scrollController = ScrollController();

  @override
  Widget build(BuildContext context) {
    final messages = ref.watch(chatMessagesProvider(widget.roomId));
    final connectionState = ref.watch(connectionStateProvider);

    return Column(
      children: [
        // Connection status banner
        if (connectionState != ConnectionState.connected)
          _ConnectionBanner(state: connectionState),

        // Message list
        Expanded(
          child: ListView.builder(
            controller: _scrollController,
            reverse: true, // Latest messages at bottom
            itemCount: messages.length,
            itemBuilder: (context, index) {
              final message = messages[index];
              return MessageBubble(
                message: message,
                isMine: message.senderId == currentUserId,
                isPending: message.status == MessageStatus.pending,
              );
            },
          ),
        ),

        // Input
        _ChatInput(
          controller: _controller,
          onSend: () {
            final text = _controller.text.trim();
            if (text.isEmpty) return;

            // Optimistic: add to local list immediately
            ref.read(chatMessagesProvider(widget.roomId).notifier)
                .addOptimistic(text);

            // Send via WebSocket
            ref.read(webSocketProvider).sendMessage(widget.roomId, text);

            _controller.clear();
          },
        ),
      ],
    );
  }
}
```

**Key discussion points:**
- Heartbeat/ping-pong to detect dead connections
- Exponential backoff with jitter for reconnection
- Optimistic UI — message appears before server confirms
- Client-generated IDs for deduplication
- Message ordering and eventual consistency
- Offline queue — queue messages when disconnected, send on reconnect

---

### Question 89: Complex Async State Patterns

**How do you handle async operations that can be cancelled, retried, and debounced?**

**Code Example:**

```dart
// A search feature with: debounce, cancellation, caching, and error retry

@riverpod
class SearchNotifier extends _$SearchNotifier {
  CancelToken? _cancelToken;
  Timer? _debounceTimer;

  // In-memory cache
  final _cache = <String, List<SearchResult>>{};

  @override
  AsyncValue<List<SearchResult>> build() {
    // Cleanup on dispose
    ref.onDispose(() {
      _cancelToken?.cancel();
      _debounceTimer?.cancel();
    });

    return AsyncValue.data([]);
  }

  void search(String query) {
    // 1. Debounce — wait for user to stop typing
    _debounceTimer?.cancel();
    _debounceTimer = Timer(Duration(milliseconds: 300), () {
      _executeSearch(query);
    });
  }

  Future<void> _executeSearch(String query) async {
    if (query.length < 2) {
      state = AsyncValue.data([]);
      return;
    }

    // 2. Check cache
    if (_cache.containsKey(query)) {
      state = AsyncValue.data(_cache[query]!);
      return;
    }

    // 3. Cancel previous in-flight request
    _cancelToken?.cancel();
    _cancelToken = CancelToken();

    state = AsyncValue.loading();

    try {
      final api = ref.read(apiClientProvider);
      final results = await api.search(
        query,
        cancelToken: _cancelToken,
      );

      // 4. Cache results
      _cache[query] = results;

      // 5. Only update state if this is still the latest request
      if (!_cancelToken!.isCancelled) {
        state = AsyncValue.data(results);
      }
    } on DioException catch (e) {
      if (e.type == DioExceptionType.cancel) {
        // Cancelled — do nothing, newer request is in progress
        return;
      }
      state = AsyncValue.error(e, StackTrace.current);
    }
  }

  void retry() {
    final lastQuery = _lastQuery;
    if (lastQuery != null) {
      _cache.remove(lastQuery); // Clear stale cache
      _executeSearch(lastQuery);
    }
  }
}

// Cancellable image upload with progress
class UploadNotifier extends StateNotifier<UploadState> {
  final ApiClient _api;
  CancelToken? _cancelToken;

  UploadNotifier(this._api) : super(UploadState.idle());

  Future<void> upload(File file) async {
    _cancelToken = CancelToken();
    state = UploadState.uploading(progress: 0);

    try {
      final formData = FormData.fromMap({
        'file': await MultipartFile.fromFile(file.path),
      });

      final response = await _api.post(
        '/upload',
        data: formData,
        cancelToken: _cancelToken,
        onSendProgress: (sent, total) {
          state = UploadState.uploading(progress: sent / total);
        },
      );

      state = UploadState.success(url: response.data['url']);
    } on DioException catch (e) {
      if (e.type == DioExceptionType.cancel) {
        state = UploadState.idle();
      } else {
        state = UploadState.error(message: e.message ?? 'Upload failed');
      }
    }
  }

  void cancel() {
    _cancelToken?.cancel();
  }
}

sealed class UploadState {
  factory UploadState.idle() = UploadIdle;
  factory UploadState.uploading({required double progress}) = UploadUploading;
  factory UploadState.success({required String url}) = UploadSuccess;
  factory UploadState.error({required String message}) = UploadError;
}
```

---

### Question 90: Testing Strategies at Scale

**How do you design a testing strategy for a large Flutter app?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│               TESTING STRATEGY AT SCALE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Layer              What to Test            How                │
│   ─────              ────────────            ───                │
│                                                                 │
│   Domain             Business logic          Unit tests          │
│                      Use cases               100% coverage       │
│                      Validation rules         Pure Dart, fast    │
│                                                                 │
│   Data               Repository impls        Unit + integration  │
│                      API parsing              Mock API responses  │
│                      Cache behavior           Test edge cases    │
│                                                                 │
│   State Mgmt         BLoC/Notifier           Unit tests          │
│                      State transitions        Mock repositories   │
│                      Error handling           Test all states    │
│                                                                 │
│   Widgets            Individual screens       Widget tests       │
│                      Component behavior       Mock state          │
│                      User interactions        Golden tests       │
│                                                                 │
│   Flows              Critical user paths     Integration tests   │
│                      Login, Checkout          Real device/emu    │
│                      Onboarding               Run before release │
│                                                                 │
│   Contract           API contracts            Contract tests     │
│                      Feature boundaries       Verify interfaces  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Testing a complex use case with multiple dependencies

class PlaceOrderUseCase {
  final OrderRepository _orderRepo;
  final PaymentService _paymentService;
  final InventoryService _inventoryService;
  final AnalyticsService _analytics;

  PlaceOrderUseCase(
    this._orderRepo,
    this._paymentService,
    this._inventoryService,
    this._analytics,
  );

  Future<OrderResult> execute(Cart cart, PaymentMethod payment) async {
    // 1. Validate inventory
    final available = await _inventoryService.checkAvailability(cart.items);
    if (!available.allAvailable) {
      return OrderResult.itemsUnavailable(available.unavailableItems);
    }

    // 2. Process payment
    final paymentResult = await _paymentService.charge(
      amount: cart.total,
      method: payment,
    );
    if (!paymentResult.success) {
      return OrderResult.paymentFailed(paymentResult.error);
    }

    // 3. Create order
    try {
      final order = await _orderRepo.create(
        items: cart.items,
        paymentId: paymentResult.transactionId,
      );

      _analytics.track('order_placed', {'orderId': order.id, 'total': cart.total});
      return OrderResult.success(order);
    } catch (e) {
      // Refund on order creation failure
      await _paymentService.refund(paymentResult.transactionId);
      return OrderResult.failed('Order creation failed. Payment refunded.');
    }
  }
}

// Comprehensive test suite
@GenerateMocks([OrderRepository, PaymentService, InventoryService, AnalyticsService])
void main() {
  late PlaceOrderUseCase useCase;
  late MockOrderRepository mockOrderRepo;
  late MockPaymentService mockPayment;
  late MockInventoryService mockInventory;
  late MockAnalyticsService mockAnalytics;

  setUp(() {
    mockOrderRepo = MockOrderRepository();
    mockPayment = MockPaymentService();
    mockInventory = MockInventoryService();
    mockAnalytics = MockAnalyticsService();
    useCase = PlaceOrderUseCase(
      mockOrderRepo, mockPayment, mockInventory, mockAnalytics,
    );
  });

  group('PlaceOrderUseCase', () {
    test('succeeds when all steps pass', () async {
      // Arrange
      when(mockInventory.checkAvailability(any))
          .thenAnswer((_) async => AvailabilityResult(allAvailable: true));
      when(mockPayment.charge(amount: anyNamed('amount'), method: anyNamed('method')))
          .thenAnswer((_) async => PaymentResult(success: true, transactionId: 'tx_1'));
      when(mockOrderRepo.create(items: anyNamed('items'), paymentId: anyNamed('paymentId')))
          .thenAnswer((_) async => Order(id: 'order_1'));

      // Act
      final result = await useCase.execute(testCart, testPayment);

      // Assert
      expect(result, isA<OrderSuccess>());
      verify(mockAnalytics.track('order_placed', any)).called(1);
    });

    test('returns unavailable when items out of stock', () async {
      when(mockInventory.checkAvailability(any))
          .thenAnswer((_) async => AvailabilityResult(
            allAvailable: false,
            unavailableItems: ['item_2'],
          ));

      final result = await useCase.execute(testCart, testPayment);

      expect(result, isA<OrderItemsUnavailable>());
      verifyNever(mockPayment.charge(amount: anyNamed('amount'), method: anyNamed('method')));
    });

    test('refunds payment when order creation fails', () async {
      when(mockInventory.checkAvailability(any))
          .thenAnswer((_) async => AvailabilityResult(allAvailable: true));
      when(mockPayment.charge(amount: anyNamed('amount'), method: anyNamed('method')))
          .thenAnswer((_) async => PaymentResult(success: true, transactionId: 'tx_1'));
      when(mockOrderRepo.create(items: anyNamed('items'), paymentId: anyNamed('paymentId')))
          .thenThrow(ServerException('DB error'));

      final result = await useCase.execute(testCart, testPayment);

      expect(result, isA<OrderFailed>());
      verify(mockPayment.refund('tx_1')).called(1); // Ensure refund happened
    });
  });
}
```

**What interviewers look for:**
- You test behavior, not implementation details
- You know what belongs at each level of the pyramid
- You test the sad paths (errors, edge cases, race conditions)
- You have a strategy for what NOT to test (generated code, trivial getters)

---

### Question 91: Performance Monitoring

**How do you monitor and optimize performance in a production Flutter app?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│             PRODUCTION PERFORMANCE MONITORING                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   What to Monitor        Tool / Approach                        │
│   ──────────────         ──────────────                         │
│                                                                 │
│   Frame rendering        CustomPerformanceOverlay               │
│   • Jank (>16ms frames)  Firebase Performance                   │
│   • Shader compilation   Sentry Performance                     │
│                                                                 │
│   App startup time       flutter run --trace-startup             │
│   • TTFF, TTI            Custom trace in main()                 │
│                                                                 │
│   Network performance    Dio interceptor logging                 │
│   • Latency, errors      Firebase Performance HTTP traces       │
│   • Payload size                                                │
│                                                                 │
│   Memory                 DevTools Memory tab                    │
│   • Heap growth          Custom leak detection                  │
│   • Object retention                                            │
│                                                                 │
│   App size               flutter build --analyze-size            │
│   • Per-package size     --split-debug-info                      │
│                                                                 │
│   User-perceived         Custom traces                          │
│   • Screen load time     Measure widget build to data rendered  │
│   • Interaction latency                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Custom performance tracing
class PerformanceTracer {
  static final _traces = <String, Stopwatch>{};

  static void startTrace(String name) {
    _traces[name] = Stopwatch()..start();
  }

  static Duration endTrace(String name) {
    final stopwatch = _traces.remove(name);
    if (stopwatch == null) return Duration.zero;

    stopwatch.stop();
    final duration = stopwatch.elapsed;

    // Report to monitoring service
    FirebasePerformance.instance
        .newTrace(name)
        .then((trace) async {
      trace.setMetric('duration_ms', duration.inMilliseconds);
      await trace.start();
      await trace.stop();
    });

    // Warn on slow operations
    if (duration > Duration(seconds: 2)) {
      debugPrint('SLOW: $name took ${duration.inMilliseconds}ms');
    }

    return duration;
  }
}

// Usage: Measure screen load time
class ProductScreen extends ConsumerStatefulWidget {
  @override
  ConsumerState<ProductScreen> createState() => _ProductScreenState();
}

class _ProductScreenState extends ConsumerState<ProductScreen> {
  @override
  void initState() {
    super.initState();
    PerformanceTracer.startTrace('product_screen_load');
  }

  @override
  Widget build(BuildContext context) {
    final product = ref.watch(productProvider(widget.id));

    return product.when(
      loading: () => ProductSkeleton(),
      error: (e, s) => ErrorView(error: e),
      data: (product) {
        // End trace when data is rendered
        WidgetsBinding.instance.addPostFrameCallback((_) {
          PerformanceTracer.endTrace('product_screen_load');
        });
        return ProductContent(product: product);
      },
    );
  }
}

// Frame performance monitoring
class FrameMonitor {
  static void start() {
    SchedulerBinding.instance.addTimingsCallback((timings) {
      for (final timing in timings) {
        final buildDuration = timing.buildDuration;
        final rasterDuration = timing.rasterDuration;
        final totalDuration = timing.totalSpan;

        // Flag jank: frame took longer than 16ms (60fps target)
        if (totalDuration > Duration(milliseconds: 16)) {
          FirebaseCrashlytics.instance.log(
            'Jank: build=${buildDuration.inMilliseconds}ms, '
            'raster=${rasterDuration.inMilliseconds}ms',
          );
        }
      }
    });
  }
}
```

---

### Question 92: State Sync Across Devices

**How would you sync state across multiple devices for the same user?**

**Theory:**

```
┌─────────────────────────────────────────────────────────────────┐
│              STATE SYNC STRATEGIES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Strategy             Pros                   Cons              │
│   ────────             ────                   ────              │
│                                                                 │
│   Polling              Simple to implement    Battery drain     │
│   (periodic fetch)     Works everywhere       Delayed updates   │
│                                                                 │
│   WebSockets           Real-time              Complex server    │
│   (push)               Low latency            Connection mgmt  │
│                                                                 │
│   Firebase RTDB /      Real-time              Vendor lock-in    │
│   Firestore            Built-in offline       Cost at scale     │
│                        Conflict handling                        │
│                                                                 │
│   Server-Sent Events   Simple server impl     One-directional   │
│   (SSE)                Auto-reconnect         No client→server  │
│                                                                 │
│   CRDTs                No conflicts           Limited types     │
│   (conflict-free)      Works offline           Complex impl     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Code Example:**

```dart
// Using Firestore for real-time cross-device sync
class CrossDeviceSyncService {
  final FirebaseFirestore _firestore;
  final String _userId;

  CrossDeviceSyncService(this._firestore, this._userId);

  // Real-time sync of user preferences
  Stream<UserPreferences> watchPreferences() {
    return _firestore
        .collection('users')
        .doc(_userId)
        .snapshots()
        .map((snapshot) {
      if (!snapshot.exists) return UserPreferences.defaults();
      return UserPreferences.fromJson(snapshot.data()!);
    });
  }

  // Optimistic write with conflict detection
  Future<void> updatePreferences(
    Map<String, dynamic> updates,
  ) async {
    await _firestore
        .collection('users')
        .doc(_userId)
        .set(
          {
            ...updates,
            'lastModified': FieldValue.serverTimestamp(),
            'lastModifiedBy': _deviceId,
          },
          SetOptions(merge: true), // Merge, don't overwrite
        );
  }

  // Sync complex state with version vectors
  Stream<List<Todo>> watchTodos() {
    return _firestore
        .collection('users')
        .doc(_userId)
        .collection('todos')
        .orderBy('updatedAt', descending: true)
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => Todo.fromFirestore(doc))
            .toList());
  }

  // Atomic batch operations for consistency
  Future<void> reorderTodos(List<String> orderedIds) async {
    final batch = _firestore.batch();

    for (var i = 0; i < orderedIds.length; i++) {
      batch.update(
        _firestore
            .collection('users')
            .doc(_userId)
            .collection('todos')
            .doc(orderedIds[i]),
        {'sortOrder': i},
      );
    }

    await batch.commit();
  }
}

// In the app — combine local and remote state
final todosProvider = StreamProvider<List<Todo>>((ref) {
  final syncService = ref.watch(syncServiceProvider);
  return syncService.watchTodos();
  // Firestore SDK handles offline caching automatically
  // UI stays reactive even without network
});
```

**Key discussion points:**
- Firestore gives you offline support + real-time sync out of the box
- For custom backends, WebSockets + event sourcing is common
- Version vectors or timestamps for conflict detection
- Batch writes for atomic multi-document updates
- Consider bandwidth — don't sync everything, sync what matters

---

## Leadership & Behavioral

### Question 93: Tell Me About a Challenging Bug

**Tell me about the hardest bug you've ever debugged.**

**How to structure your answer (STAR method):**

```
┌─────────────────────────────────────────────────────────────────┐
│                    STAR FRAMEWORK                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   S - Situation   What was the context?                         │
│                   "In our fintech app with 500K users..."       │
│                                                                 │
│   T - Task        What was your responsibility?                 │
│                   "I was the lead mobile dev responsible for..." │
│                                                                 │
│   A - Action      What did YOU specifically do?                 │
│                   "I added custom performance tracing..."       │
│                   "I bisected the git history..."               │
│                   "I reproduced it by..."                       │
│                                                                 │
│   R - Result      What was the measurable outcome?              │
│                   "Crash rate dropped from 2.3% to 0.01%"      │
│                   "We shipped the fix in 4 hours"               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example answer structure:**

> **Situation:** Our e-commerce app had a bug where users' carts would randomly empty. It happened to ~3% of users but we couldn't reproduce it.
>
> **Task:** As the senior Flutter dev, I owned the investigation.
>
> **Action:**
> 1. Added telemetry to track every cart state change with timestamps and triggers
> 2. Discovered the pattern: it only happened when users backgrounded the app during checkout
> 3. Root cause: The `WidgetsBindingObserver` was triggering a state reset on `AppLifecycleState.resumed`, and our state management wasn't persisting cart state to disk
> 4. Fix: Implemented local persistence of cart state using Hive, with a hydration step on app resume
>
> **Result:** Cart abandonment dropped 12%. Fix shipped in 2 days. Added integration test to prevent regression.

**What interviewers look for:**
- Systematic debugging approach (not random guessing)
- Use of tools (DevTools, logging, crash reports)
- How you narrowed down the problem
- That you prevented it from happening again

---

### Question 94: Handling Technical Disagreements

**Tell me about a time you disagreed with a technical decision. How did you handle it?**

**Framework:**

```
┌─────────────────────────────────────────────────────────────────┐
│           HANDLING TECHNICAL DISAGREEMENTS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. Understand their position fully first                      │
│      "Help me understand why you prefer X..."                   │
│                                                                 │
│   2. Acknowledge valid points                                   │
│      "You're right that X is simpler to implement..."           │
│                                                                 │
│   3. Present your view with data                                │
│      "I ran a benchmark and Y handles 3x more concurrent..."   │
│                                                                 │
│   4. Propose a way to decide                                    │
│      "Can we build a small prototype of both approaches?"       │
│      "Let's write down our assumptions and test them."          │
│                                                                 │
│   5. Disagree and commit                                        │
│      If the decision goes the other way, support it fully.      │
│      "I still think X, but I'll commit 100% to Y."             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Example:**

> We were choosing between BLoC and Riverpod for a new app. I preferred Riverpod for its simpler API and better testing story. The architect preferred BLoC for its strict event/state pattern.
>
> Instead of arguing, I proposed we both build the same feature — user authentication — in both approaches. We compared: lines of code, testability, onboarding time for junior devs, and debugging experience.
>
> BLoC won on traceability (event log), Riverpod won on everything else. We went with Riverpod but adopted BLoC's event naming conventions for our state notifiers.
>
> The key was making it about the code, not about who was right.

---

### Question 95: Mentoring Junior Developers

**How do you mentor junior developers?**

**Framework:**

```
┌─────────────────────────────────────────────────────────────────┐
│               MENTORING APPROACH                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. Code reviews as teaching moments                           │
│      • Don't just say "change this"                             │
│      • Explain WHY — link to docs, show examples                │
│      • Ask questions: "What happens if this is null?"           │
│                                                                 │
│   2. Pair programming on hard problems                          │
│      • Let them drive, you navigate                             │
│      • Think out loud so they learn your process                │
│                                                                 │
│   3. Gradually increase scope                                   │
│      • Week 1: Bug fixes with clear reproduction steps          │
│      • Month 1: Small features with design already decided      │
│      • Month 3: Features where they propose the design          │
│      • Month 6: They own a feature end-to-end                   │
│                                                                 │
│   4. Create psychological safety                                │
│      • "I broke production last year, here's what happened"     │
│      • Celebrate questions, not just answers                    │
│      • Never make someone feel bad for not knowing              │
│                                                                 │
│   5. Measure growth, not output                                 │
│      • Can they debug independently now?                        │
│      • Are their PRs getting smaller and more focused?          │
│      • Are they helping other juniors?                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Question 96: Task Estimation

**How do you estimate development tasks?**

**Framework:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 ESTIMATION APPROACH                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. Break it down                                              │
│      Never estimate a task bigger than 3 days.                  │
│      If it's bigger, decompose it first.                        │
│                                                                 │
│   2. Identify unknowns                                          │
│      • Have I done this before?                                 │
│      • Are there third-party dependencies?                      │
│      • Is the API designed yet?                                 │
│      • Does it need platform-specific work?                     │
│                                                                 │
│   3. Multiply by uncertainty factor                             │
│      • Done it before:           estimate × 1.2                 │
│      • Similar to past work:     estimate × 1.5                 │
│      • Never done before:        estimate × 2-3                 │
│      • Depends on external team: estimate × 3+                  │
│                                                                 │
│   4. Include non-coding work                                    │
│      • Code review cycles                                       │
│      • Testing (unit + integration)                             │
│      • Documentation                                            │
│      • QA back-and-forth                                        │
│                                                                 │
│   5. Communicate ranges, not points                             │
│      "I think this is 3-5 days"                                 │
│      NOT "this will take 3 days"                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What interviewers look for:**
- You've been burned by bad estimates before and learned from it
- You break work down before estimating
- You factor in unknowns and communicate uncertainty
- You don't give estimates under pressure without understanding scope

---

### Question 97: Prioritizing Technical Debt

**How do you prioritize technical debt?**

**Framework:**

```
┌─────────────────────────────────────────────────────────────────┐
│              TECHNICAL DEBT PRIORITIZATION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Priority    Criteria                   Example                │
│   ────────    ────────                   ───────                │
│                                                                 │
│   Fix NOW     • Causes production bugs   Memory leak in chat    │
│               • Blocks feature work      Can't add tests due    │
│               • Security vulnerability   to tight coupling      │
│                                                                 │
│   Plan for    • Slows development        No DI, hard to mock    │
│   next sprint • Growing pain             Manual deployments     │
│               • Measurable cost          Slow CI (45 min)       │
│                                                                 │
│   Track but   • Cosmetic/style issues    Inconsistent naming    │
│   don't rush  • "Nice to have"           Old package versions   │
│               • No measurable impact     (still working)        │
│                                                                 │
│   Ignore      • Hypothetical future      "We might need this"   │
│               • Rewrite fantasies         "Let's rewrite in X"  │
│               • No user/dev impact                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**How to sell it to stakeholders:**

> "Our CI takes 45 minutes. That means every PR takes 45 minutes to verify. With 8 PRs per day, that's 6 hours of developer wait time daily. I can cut it to 12 minutes in a 2-day spike. That's 4.6 hours saved per day, every day."

**Key principle:** Quantify the cost. "Technical debt" means nothing to a PM. "2 hours lost per developer per day" means everything.

---

### Question 98: Handling Production Incidents

**Walk me through how you handle a production incident.**

**Framework:**

```
┌─────────────────────────────────────────────────────────────────┐
│              INCIDENT RESPONSE FRAMEWORK                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Phase 1: DETECT (minutes)                                     │
│   ─────────────────────────                                     │
│   • Crash reporting alerts (Crashlytics, Sentry)                │
│   • Error rate spike in monitoring                              │
│   • User reports (support, social media, reviews)               │
│                                                                 │
│   Phase 2: ASSESS (5-15 minutes)                                │
│   ──────────────────────────────                                │
│   • How many users affected? (% of DAU)                         │
│   • What's the business impact? (revenue, trust)                │
│   • Is it getting worse or stable?                              │
│   • Can users work around it?                                   │
│                                                                 │
│   Phase 3: MITIGATE (minutes to hours)                          │
│   ────────────────────────────────────                          │
│   • Feature flag: disable broken feature                        │
│   • Server-side: fix API, update config                         │
│   • Client-side: publish hotfix, staged rollout                 │
│   • Communicate: status page, in-app message                    │
│                                                                 │
│   Phase 4: FIX (hours to days)                                  │
│   ────────────────────────────                                  │
│   • Root cause analysis                                         │
│   • Fix + tests that cover the exact failure                    │
│   • Code review with extra scrutiny                             │
│   • Staged rollout (1% → 10% → 50% → 100%)                    │
│                                                                 │
│   Phase 5: LEARN (within 1 week)                                │
│   ──────────────────────────────                                │
│   • Blameless post-mortem                                       │
│   • Timeline of events                                          │
│   • What went well, what didn't                                 │
│   • Action items to prevent recurrence                          │
│   • Share learnings with the team                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key things interviewers want to hear:**
- You stay calm and prioritize mitigation over root cause
- You communicate proactively (stakeholders, team, users)
- You do blameless post-mortems
- You put systems in place so the same thing can't happen twice

---

### Question 99: Questions to Ask Interviewers

**What questions should you ask at the end of an interview?**

```
┌─────────────────────────────────────────────────────────────────┐
│            QUESTIONS THAT SHOW SENIORITY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   About the codebase:                                           │
│   • "What's the biggest technical challenge the team faces      │
│      right now?"                                                │
│   • "How do you handle breaking changes across modules?"        │
│   • "What does your testing strategy look like? What's the      │
│      coverage target?"                                          │
│   • "What's your CI/CD pipeline? How long does a deploy take?"  │
│                                                                 │
│   About the team:                                               │
│   • "How are technical decisions made? Who has the final say?"  │
│   • "How do you handle technical debt? Is there dedicated       │
│      time for it?"                                              │
│   • "What does the code review process look like?"              │
│   • "How do you onboard new developers?"                        │
│                                                                 │
│   About growth:                                                 │
│   • "What does success look like in this role at 6 months?"     │
│   • "What's the most impactful project the team shipped         │
│      recently?"                                                 │
│   • "How do you balance feature work with engineering quality?" │
│                                                                 │
│   Red flag questions (ask diplomatically):                      │
│   • "What's the on-call rotation like?"                         │
│   • "How often do production incidents happen?"                 │
│   • "What's the typical sprint velocity vs. planned capacity?"  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**What NOT to ask:**
- Anything you could find on the company website
- Salary/benefits in a technical round
- "Do you like working here?" (too generic)

---

### Question 100: Principles for Maintainable Flutter Apps

**What are your core principles for building maintainable Flutter apps?**

```
┌─────────────────────────────────────────────────────────────────┐
│         10 PRINCIPLES FOR MAINTAINABLE FLUTTER APPS             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. Boring is good                                             │
│      Use well-known patterns. Clever code is hard to debug.     │
│      The best architecture is the one your whole team           │
│      understands.                                               │
│                                                                 │
│   2. Dependencies flow one direction                            │
│      UI → State → Domain → Data                                 │
│      Never the other way around.                                │
│                                                                 │
│   3. Make invalid states unrepresentable                        │
│      Use sealed classes. If a state shouldn't exist,            │
│      make it impossible to construct.                           │
│                                                                 │
│   4. Prefer composition over inheritance                        │
│      Small, focused widgets composed together beat              │
│      deep class hierarchies every time.                         │
│                                                                 │
│   5. Test behavior, not implementation                          │
│      Tests should survive refactoring.                          │
│      "When user taps X, Y happens"                              │
│      NOT "method Z was called with parameter W."                │
│                                                                 │
│   6. Fail fast, fail loudly                                     │
│      Assert preconditions. Don't silently swallow errors.       │
│      A crash in development beats a bug in production.          │
│                                                                 │
│   7. Optimize for deletion                                      │
│      Code should be easy to remove.                             │
│      Feature flags, modular packages, clear boundaries.         │
│                                                                 │
│   8. Measure before you optimize                                │
│      Don't guess at bottlenecks. Use DevTools, traces,          │
│      and profiling. Premature optimization is real.             │
│                                                                 │
│   9. Automate what you repeat                                   │
│      CI/CD, code generation, linting rules, formatting.         │
│      If a human has to remember it, a human will forget it.     │
│                                                                 │
│  10. Write code for the reader                                  │
│      You write code once. Others read it hundreds of times.     │
│      Clarity beats brevity. Names matter.                       │
│      The best comment is code that doesn't need one.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary

At expert level, you should be able to:

| Area | Expectation |
|------|-------------|
| System Design | Architect apps that scale to millions of users and dozens of developers |
| Offline-First | Design sync, conflict resolution, and queue-based architectures |
| Real-Time | Build WebSocket systems with reconnection, dedup, and optimistic UI |
| Performance | Instrument, monitor, and optimize production apps |
| Leadership | Mentor, estimate, prioritize debt, handle incidents, communicate trade-offs |
| Principles | Articulate and defend your engineering philosophy |

---

## Final Words

If you've made it through all 100 questions — congratulations. That's not just interview prep. That's a deep understanding of Flutter, software engineering, and technical leadership.

Remember:
- **Junior:** Show you can learn
- **Mid-Level:** Show you can build
- **Senior:** Show you can lead
- **Expert:** Show you can think

Good luck.

---

**Made with by [Debasmita Sarkar](https://github.com/debasmitasarkar)**

[![Back to Main](https://img.shields.io/badge/←_Back_to_Main-blue?style=flat-square)](/README.md)
[![Previous: Senior Level](https://img.shields.io/badge/←_Previous:_Senior_Level-orange?style=flat-square)](/senior/README.md)
