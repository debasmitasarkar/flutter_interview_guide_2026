# Flutter Interview Questions 2026 - Junior Level (Questions 1-37)

A comprehensive guide for Flutter developers with **0-2 years experience**. Each question includes detailed theory, code examples, common mistakes, and interview follow-ups.

---

## Table of Contents

1. [Dart Fundamentals (1-15)](#dart-fundamentals)
2. [Flutter Basics (16-25)](#flutter-basics)
3. [Widgets & UI (26-32)](#widgets--ui)
4. [State Management Basics (33-37)](#state-management-basics)

---

# Dart Fundamentals

### 1. What is the difference between `var`, `final`, and `const`?

**Theory:**

Dart provides three ways to declare variables, each with different mutability and initialization rules. Understanding these is fundamental to writing correct Dart code.

```
┌─────────────────────────────────────────────────────────────────┐
│                    VARIABLE DECLARATIONS                         │
├─────────────┬─────────────────┬─────────────────────────────────┤
│   Keyword   │   Reassignable  │   When Value is Determined      │
├─────────────┼─────────────────┼─────────────────────────────────┤
│    var      │       ✅        │   Runtime                       │
│    final    │       ❌        │   Runtime (first assignment)    │
│    const    │       ❌        │   Compile time                  │
└─────────────┴─────────────────┴─────────────────────────────────┘
```

**`var` - Mutable variable:**

```dart
var name = 'Alice';     // Type inferred as String
name = 'Bob';           // ✅ Can reassign
name = 42;              // ❌ Error: Can't assign int to String

var count;              // Type is dynamic (avoid this!)
count = 10;
count = 'ten';          // ✅ Works but dangerous - avoid dynamic
```

**`final` - Runtime constant:**

```dart
final name = 'Alice';   // Assigned once at runtime
name = 'Bob';           // ❌ Error: Can't reassign

final DateTime now = DateTime.now();  // ✅ Value determined at runtime
final list = [1, 2, 3];
list.add(4);            // ✅ Can modify contents
list = [5, 6];          // ❌ Can't reassign reference
```

**`const` - Compile-time constant:**

```dart
const pi = 3.14159;     // Value must be known at compile time
const DateTime now = DateTime.now();  // ❌ Error: Not a compile-time constant

const list = [1, 2, 3];
list.add(4);            // ❌ Error: Can't modify const list

// const creates canonicalized objects (same memory)
const a = [1, 2, 3];
const b = [1, 2, 3];
print(identical(a, b)); // true - same object in memory!
```

**When to use each:**

| Use Case | Keyword | Example |
|----------|---------|---------|
| Value changes over time | `var` | Loop counters, form inputs |
| Set once, never changes | `final` | User ID after login, API response |
| Known at compile time | `const` | App constants, widget constructors |

**Common mistakes:**

```dart
// ❌ Using var when final would be better
var userId = response.userId;  // Never reassigned

// ✅ Better
final userId = response.userId;

// ❌ Using final when const is possible
final padding = EdgeInsets.all(16);  // Known at compile time

// ✅ Better - enables widget optimization
const padding = EdgeInsets.all(16);
```

**Interview follow-up:** "Why does Flutter emphasize `const` constructors?"

> `const` widgets are canonicalized - Flutter reuses the same instance, skipping rebuilds entirely. This improves performance, especially in large widget trees.

---

### 2. Explain null safety in Dart

**Theory:**

Null safety, introduced in Dart 2.12, prevents null reference errors at compile time rather than runtime. It's a fundamental feature that makes Dart code safer and more reliable.

**The problem it solves:**

```dart
// Before null safety - runtime crash!
String name;           // Implicitly null
print(name.length);    // 💥 Null pointer exception at runtime

// With null safety - compile-time error!
String name;           // ❌ Error: Must be initialized
print(name.length);    // Never reaches here
```

**Nullable vs Non-nullable types:**

```dart
// Non-nullable (default) - cannot be null
String name = 'Alice';
name = null;           // ❌ Compile error

// Nullable - can be null (add ?)
String? name = 'Alice';
name = null;           // ✅ OK
print(name.length);    // ❌ Error: name might be null

// Must check before using
if (name != null) {
  print(name.length);  // ✅ Dart knows it's not null here
}
```

**Null safety operators:**

```dart
// 1. Null-aware access (?.)
String? name;
print(name?.length);   // null (no crash)

// 2. Null-aware assignment (??=)
String? name;
name ??= 'Default';    // Assigns only if null

// 3. Null coalescing (??)
String? name;
print(name ?? 'Unknown');  // 'Unknown'

// 4. Null assertion (!)
String? name = getName();
print(name!.length);   // Asserts name is not null (use carefully!)

// 5. Late initialization
late String name;      // Promise to initialize before use
void init() {
  name = 'Alice';      // Must be assigned before access
}
```

**Flow analysis (smart casting):**

```dart
void process(String? name) {
  if (name == null) {
    return;  // Early return
  }
  // Dart knows name is non-null here
  print(name.length);  // ✅ No error, no ! needed
}

// Also works with other checks
void process2(String? name) {
  if (name == null || name.isEmpty) {
    return;
  }
  print(name.length);  // ✅ Dart promoted type to String
}
```

**Common patterns:**

```dart
// Pattern 1: Required parameters with defaults
void greet({String name = 'Guest'}) {
  print('Hello, $name');
}

// Pattern 2: Late final for dependency injection
class UserService {
  late final ApiClient _client;
  
  void init(ApiClient client) {
    _client = client;
  }
}

// Pattern 3: Nullable fields with getters
class User {
  String? _nickname;
  
  String get displayName => _nickname ?? 'Anonymous';
}
```

**Common mistakes:**

```dart
// ❌ Overusing null assertion (!)
String? name = getName();
print(name!.length);  // Dangerous - crashes if null

// ✅ Better - handle the null case
print(name?.length ?? 0);

// ❌ Making everything nullable "just in case"
String? firstName;  // Will this ever be null?
String? lastName;   // Think about your data model

// ✅ Better - be intentional about nullability
String firstName;   // Required, never null
String? middleName; // Optional, can be null
```

**Interview follow-up:** "What is the `late` keyword and when should you use it?"

> `late` defers initialization but promises the variable will be assigned before use. Use it for:
> 1. Variables initialized in `initState()` or setup methods
> 2. Expensive computations you want to lazily evaluate
> 3. Non-nullable instance variables that can't be initialized in the constructor

---

### 3. What are the differences between `List`, `Set`, and `Map`?

**Theory:**

Dart provides three core collection types, each optimized for different use cases.

```
┌─────────────────────────────────────────────────────────────────┐
│                      COLLECTION TYPES                            │
├─────────────┬─────────────┬─────────────┬───────────────────────┤
│   Type      │   Ordered   │  Duplicates │   Access Pattern      │
├─────────────┼─────────────┼─────────────┼───────────────────────┤
│   List      │     ✅      │     ✅      │   By index [i]        │
│   Set       │     ❌*     │     ❌      │   By value            │
│   Map       │     ❌*     │  Keys: ❌   │   By key [key]        │
└─────────────┴─────────────┴─────────────┴───────────────────────┘
* LinkedHashSet/LinkedHashMap maintain insertion order
```

**List - Ordered, allows duplicates:**

```dart
// Creation
List<int> numbers = [1, 2, 3, 2, 1];
var names = <String>['Alice', 'Bob'];
final empty = <int>[];

// Operations
numbers.add(4);              // [1, 2, 3, 2, 1, 4]
numbers.addAll([5, 6]);      // [1, 2, 3, 2, 1, 4, 5, 6]
numbers.insert(0, 0);        // [0, 1, 2, 3, 2, 1, 4, 5, 6]
numbers.removeAt(0);         // [1, 2, 3, 2, 1, 4, 5, 6]
numbers.remove(2);           // Removes first 2: [1, 3, 2, 1, 4, 5, 6]

// Access
print(numbers[0]);           // 1
print(numbers.first);        // 1
print(numbers.last);         // 6
print(numbers.length);       // 7

// Useful methods
numbers.where((n) => n > 3); // (4, 5, 6)
numbers.map((n) => n * 2);   // (2, 6, 4, 2, 8, 10, 12)
numbers.any((n) => n > 5);   // true
numbers.every((n) => n > 0); // true

// Spread operator
var more = [0, ...numbers, 7]; // [0, 1, 3, 2, 1, 4, 5, 6, 7]
```

**Set - Unordered, no duplicates:**

```dart
// Creation
Set<int> numbers = {1, 2, 3, 2, 1};  // {1, 2, 3} - duplicates removed
var names = <String>{'Alice', 'Bob'};

// Operations
numbers.add(4);              // {1, 2, 3, 4}
numbers.add(1);              // {1, 2, 3, 4} - no change, 1 exists
numbers.addAll({5, 6});      // {1, 2, 3, 4, 5, 6}
numbers.remove(1);           // {2, 3, 4, 5, 6}

// Set operations
final a = {1, 2, 3};
final b = {2, 3, 4};
print(a.union(b));           // {1, 2, 3, 4}
print(a.intersection(b));    // {2, 3}
print(a.difference(b));      // {1}

// Membership check is O(1)
print(numbers.contains(3));  // true - very fast!

// Use case: Remove duplicates from list
final list = [1, 2, 2, 3, 3, 3];
final unique = list.toSet().toList(); // [1, 2, 3]
```

**Map - Key-value pairs:**

```dart
// Creation
Map<String, int> ages = {'Alice': 30, 'Bob': 25};
var scores = <String, double>{};

// Operations
ages['Charlie'] = 35;        // Add/update
ages.remove('Bob');          // Remove by key
ages.putIfAbsent('Dave', () => 40); // Add only if missing

// Access
print(ages['Alice']);        // 30 (nullable - returns null if missing)
print(ages['Unknown']);      // null
print(ages.containsKey('Alice')); // true

// Safe access
final age = ages['Unknown'] ?? 0; // 0

// Iteration
ages.forEach((name, age) {
  print('$name is $age');
});

for (final entry in ages.entries) {
  print('${entry.key}: ${entry.value}');
}

// Keys and values
print(ages.keys);            // (Alice, Charlie, Dave)
print(ages.values);          // (30, 35, 40)
```

**When to use each:**

| Collection | Use When |
|------------|----------|
| List | Order matters, need index access, duplicates OK |
| Set | Need uniqueness, fast contains check, set operations |
| Map | Need key-value association, fast lookup by key |

**Performance comparison:**

| Operation | List | Set | Map |
|-----------|------|-----|-----|
| Add | O(1)* | O(1) | O(1) |
| Access by index | O(1) | N/A | N/A |
| Access by key | N/A | N/A | O(1) |
| Contains | O(n) | O(1) | O(1) keys |
| Remove | O(n) | O(1) | O(1) |

*Amortized - occasional O(n) for resizing

---

### 4. What is the difference between `async/await` and `Future`?

**Theory:**

They're two syntaxes for the same thing - asynchronous programming. `Future` is the object representing an eventual value; `async/await` is syntactic sugar to work with Futures more readably.

**Future - The object:**

```dart
// A Future represents a value that will be available later
Future<String> fetchUserName() {
  return Future.delayed(
    Duration(seconds: 2),
    () => 'Alice',
  );
}

// Using .then() chains
void main() {
  print('Start');
  
  fetchUserName()
    .then((name) => print('Hello, $name'))
    .catchError((error) => print('Error: $error'))
    .whenComplete(() => print('Done'));
  
  print('End');  // Prints before "Hello, Alice"!
}

// Output:
// Start
// End
// Hello, Alice
// Done
```

**async/await - The syntax:**

```dart
// Same logic, but reads like synchronous code
Future<void> main() async {
  print('Start');
  
  try {
    final name = await fetchUserName();  // Pauses here
    print('Hello, $name');
  } catch (error) {
    print('Error: $error');
  } finally {
    print('Done');
  }
  
  print('End');  // Prints AFTER "Hello, Alice"
}

// Output:
// Start
// Hello, Alice
// Done
// End
```

**Key differences:**

```dart
// .then() chains can get messy
fetchUser()
  .then((user) => fetchOrders(user.id))
  .then((orders) => fetchOrderDetails(orders.first.id))
  .then((details) => print(details))
  .catchError((e) => print('Error: $e'));

// async/await is cleaner
Future<void> loadData() async {
  try {
    final user = await fetchUser();
    final orders = await fetchOrders(user.id);
    final details = await fetchOrderDetails(orders.first.id);
    print(details);
  } catch (e) {
    print('Error: $e');
  }
}
```

**Parallel execution:**

```dart
// Sequential - slow (5 seconds total)
Future<void> sequential() async {
  final user = await fetchUser();        // 2 seconds
  final posts = await fetchPosts();      // 3 seconds
  print('$user, $posts');
}

// Parallel - fast (3 seconds total)
Future<void> parallel() async {
  final results = await Future.wait([
    fetchUser(),   // Starts immediately
    fetchPosts(),  // Starts immediately
  ]);
  print('${results[0]}, ${results[1]}');
}

// Even cleaner with records (Dart 3)
Future<void> parallelRecords() async {
  final (user, posts) = await (
    fetchUser(),
    fetchPosts(),
  ).wait;
  print('$user, $posts');
}
```

**Common patterns:**

```dart
// Pattern 1: Timeout
final result = await fetchData().timeout(
  Duration(seconds: 5),
  onTimeout: () => 'Default value',
);

// Pattern 2: Retry logic
Future<T> retry<T>(Future<T> Function() fn, {int attempts = 3}) async {
  for (var i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i == attempts - 1) rethrow;
      await Future.delayed(Duration(seconds: 1 << i)); // Exponential backoff
    }
  }
  throw Exception('Should not reach here');
}

// Pattern 3: First completed
final fastest = await Future.any([
  fetchFromServer1(),
  fetchFromServer2(),
]);
```

**Interview follow-up:** "What happens if you forget `await`?"

```dart
Future<void> buggyCode() async {
  fetchData();  // ⚠️ No await - fire and forget!
  print('Done');  // Prints immediately, before fetch completes
}

// The Future runs but you can't:
// - Get the result
// - Handle errors (they're unhandled!)
// - Know when it completes
```

---

### 5. Explain Dart's `Stream` and when to use it

**Theory:**

While a `Future` represents a single async value, a `Stream` represents a sequence of async values over time. Think of it as an async iterator.

```
┌─────────────────────────────────────────────────────────────────┐
│                    FUTURE vs STREAM                              │
├─────────────────────────────────────────────────────────────────┤
│   Future<String>          │   Stream<String>                    │
│   ───────────────         │   ──────────────                    │
│   Single value            │   Multiple values over time         │
│   Completes once          │   Can emit many times               │
│   await keyword           │   await for / listen()              │
│                           │                                     │
│   Example:                │   Example:                          │
│   HTTP response           │   WebSocket messages                │
│   File read               │   User input events                 │
│   Database query          │   Real-time updates                 │
└─────────────────────────────────────────────────────────────────┘
```

**Creating Streams:**

```dart
// 1. Stream.periodic - emit at intervals
final ticker = Stream.periodic(
  Duration(seconds: 1),
  (count) => count,
).take(5);  // 0, 1, 2, 3, 4

// 2. Stream.fromIterable - convert list to stream
final numbers = Stream.fromIterable([1, 2, 3, 4, 5]);

// 3. StreamController - manual control
final controller = StreamController<String>();
controller.sink.add('Hello');
controller.sink.add('World');
controller.close();

// 4. async* generator function
Stream<int> countDown(int from) async* {
  for (var i = from; i >= 0; i--) {
    yield i;  // Emit value
    await Future.delayed(Duration(seconds: 1));
  }
}
```

**Consuming Streams:**

```dart
// Method 1: listen() - most common
stream.listen(
  (data) => print('Data: $data'),
  onError: (error) => print('Error: $error'),
  onDone: () => print('Stream closed'),
  cancelOnError: false,
);

// Method 2: await for - cleaner syntax
Future<void> consumeStream(Stream<int> stream) async {
  await for (final value in stream) {
    print(value);
  }
  print('Stream completed');
}

// Method 3: Collect all values
final list = await stream.toList();
```

**Stream transformations:**

```dart
final stream = Stream.fromIterable([1, 2, 3, 4, 5]);

// map - transform each value
stream.map((n) => n * 2);  // 2, 4, 6, 8, 10

// where - filter values
stream.where((n) => n.isEven);  // 2, 4

// take - limit count
stream.take(3);  // 1, 2, 3

// skip - ignore first n
stream.skip(2);  // 3, 4, 5

// distinct - remove consecutive duplicates
stream.distinct();

// debounce (with rxdart) - wait for pause in events
stream.debounceTime(Duration(milliseconds: 300));
```

**Single vs Broadcast streams:**

```dart
// Single-subscription (default) - one listener only
final single = StreamController<int>();
single.stream.listen(print);      // ✅ OK
single.stream.listen(print);      // ❌ Error: already listened

// Broadcast - multiple listeners
final broadcast = StreamController<int>.broadcast();
broadcast.stream.listen((n) => print('Listener 1: $n'));
broadcast.stream.listen((n) => print('Listener 2: $n'));  // ✅ OK

// Convert single to broadcast
final broadcastStream = singleStream.asBroadcastStream();
```

**Real-world example - Search with debounce:**

```dart
class SearchService {
  final _searchController = StreamController<String>();
  
  Stream<List<SearchResult>> get results => _searchController.stream
      .debounceTime(Duration(milliseconds: 300))  // Wait for typing pause
      .distinct()                                   // Ignore same query
      .where((query) => query.length >= 2)         // Min 2 characters
      .asyncMap((query) => _api.search(query));    // Make API call
  
  void search(String query) {
    _searchController.add(query);
  }
  
  void dispose() {
    _searchController.close();
  }
}
```

**Interview follow-up:** "When would you use a Stream instead of a Future?"

> Use Stream when:
> - Data arrives over time (WebSocket, user events)
> - You need to react to changes (state management)
> - Multiple values from one source (file reading chunks)
> - Real-time updates (Firebase, stock prices)

---

### 6. What is the difference between `extends`, `implements`, and `with`?

**Theory:**

Dart has three ways to reuse code from other classes, each with different semantics and use cases.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLASS RELATIONSHIPS                           │
├─────────────┬───────────────────────────────────────────────────┤
│   extends   │   Inheritance - IS-A relationship                 │
│             │   Inherit implementation + interface              │
│             │   Single inheritance only                         │
├─────────────┼───────────────────────────────────────────────────┤
│  implements │   Interface - CAN-DO relationship                 │
│             │   Must provide ALL implementations                │
│             │   Multiple interfaces allowed                     │
├─────────────┼───────────────────────────────────────────────────┤
│    with     │   Mixin - HAS-ABILITY relationship                │
│             │   Reuse code without inheritance                  │
│             │   Multiple mixins allowed                         │
└─────────────┴───────────────────────────────────────────────────┘
```

**`extends` - Inheritance:**

```dart
class Animal {
  String name;
  Animal(this.name);
  
  void breathe() => print('$name is breathing');
  void move() => print('$name is moving');
}

class Dog extends Animal {
  Dog(String name) : super(name);
  
  // Override parent method
  @override
  void move() => print('$name is running');
  
  // Add new method
  void bark() => print('$name says woof!');
}

void main() {
  final dog = Dog('Buddy');
  dog.breathe();  // Inherited: "Buddy is breathing"
  dog.move();     // Overridden: "Buddy is running"
  dog.bark();     // New: "Buddy says woof!"
}
```

**`implements` - Interface:**

```dart
// Any class can be used as an interface
class Flyable {
  void fly() => print('Flying');
}

class Swimmable {
  void swim() => print('Swimming');
}

// Must implement ALL methods from interfaces
class Duck implements Flyable, Swimmable {
  @override
  void fly() => print('Duck flying');  // MUST implement
  
  @override
  void swim() => print('Duck swimming');  // MUST implement
}

// Abstract class as interface
abstract class JsonSerializable {
  Map<String, dynamic> toJson();
  factory JsonSerializable.fromJson(Map<String, dynamic> json);
}
```

**`with` - Mixins:**

```dart
// Mixin - reusable code without inheritance hierarchy
mixin Swimming {
  void swim() => print('Swimming');
}

mixin Flying {
  void fly() => print('Flying');
}

mixin Walking {
  void walk() => print('Walking');
}

// Combine multiple abilities
class Duck with Swimming, Flying, Walking {
  void quack() => print('Quack!');
}

class Fish with Swimming {
  // Only has swimming ability
}

class Bird with Flying, Walking {
  // Can fly and walk
}

void main() {
  final duck = Duck();
  duck.swim();  // From Swimming mixin
  duck.fly();   // From Flying mixin
  duck.walk();  // From Walking mixin
  duck.quack(); // Own method
}
```

**Mixin restrictions:**

```dart
// Mixin with 'on' clause - requires a base class
mixin Barking on Animal {
  void bark() {
    print('${this.name} barks!');  // Can access Animal properties
  }
}

class Dog extends Animal with Barking {
  Dog(String name) : super(name);
}

// ❌ Error: Can't use Barking without Animal base
// class Robot with Barking {}
```

**Combining all three:**

```dart
abstract class Animal {
  String get name;
  void breathe();
}

mixin Swimming {
  void swim() => print('Swimming');
}

class Swimmer {
  void compete() => print('Competing');
}

// Extends Animal, implements Swimmer interface, mixes in Swimming
class Dolphin extends Animal with Swimming implements Swimmer {
  @override
  String get name => 'Dolphin';
  
  @override
  void breathe() => print('Surfacing to breathe');
  
  @override
  void compete() => print('Racing in dolphin show');
}
```

**When to use each:**

| Mechanism | Use When |
|-----------|----------|
| `extends` | True IS-A relationship, need parent's implementation |
| `implements` | Guarantee a class has certain methods |
| `with` | Add capabilities to unrelated classes |

---

### 7. How does Dart handle memory management?

**Theory:**

Dart uses automatic memory management with garbage collection. You don't manually allocate/free memory, but understanding the system helps avoid memory leaks.

**Garbage Collection basics:**

```dart
void example() {
  var user = User('Alice');  // Object allocated on heap
  
  // ... use user ...
  
}  // user goes out of scope, eligible for GC

// The garbage collector will:
// 1. Identify objects with no references
// 2. Reclaim their memory automatically
// 3. Run at appropriate times (not immediately)
```

**Object lifecycle:**

```
Object Created ─► Referenced ─► No References ─► GC Eligible ─► Collected
       │              │                                              │
       └──────────────┴──────────────────────────────────────────────┘
                               Heap Memory
```

**Reference types:**

```dart
// Strong reference (default) - prevents GC
class Parent {
  Child child;  // Strong reference
}

// Dart doesn't have weak references in the language,
// but the concept matters for understanding memory leaks
```

**Common memory leak patterns:**

```dart
// ❌ LEAK: Closure captures context
class MyWidget extends StatefulWidget {
  @override
  _MyWidgetState createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  @override
  void initState() {
    super.initState();
    // This closure captures 'this', keeping State alive!
    someService.onComplete = () {
      setState(() {});  // 'this' is captured
    };
  }
}

// ✅ FIXED: Clear the reference
@override
void dispose() {
  someService.onComplete = null;
  super.dispose();
}
```

**Isolates and memory:**

```dart
// Each isolate has its own heap
// Objects can't be shared between isolates
// Must pass data by copying (or transfer)

await Isolate.run(() {
  // This runs in a separate memory space
  return heavyComputation();
});
```

**Tips for memory efficiency:**

```dart
// 1. Use const to share instances
const widget1 = Text('Hello');  // Same object
const widget2 = Text('Hello');  // as this one

// 2. Dispose controllers
@override
void dispose() {
  _controller.dispose();
  _subscription.cancel();
  super.dispose();
}

// 3. Clear large objects when done
List<LargeObject>? cache;
void clearCache() {
  cache = null;  // Allow GC to collect
}

// 4. Use streams carefully
// Always cancel subscriptions
StreamSubscription? _sub;

void start() {
  _sub = stream.listen(handleData);
}

void stop() {
  _sub?.cancel();
}
```

**Interview follow-up:** "Can you force garbage collection in Dart?"

> No, Dart doesn't expose a way to force GC. The garbage collector runs automatically when it determines it's necessary. In DevTools, you can trigger GC for debugging, but this isn't available in production code.

---

### 8. What is the difference between `==` and `identical()`?

**Theory:**

`==` checks equality (do objects have the same value?), while `identical()` checks identity (are they the exact same object in memory?).

```dart
// == checks equality (can be overridden)
final a = User('Alice');
final b = User('Alice');
print(a == b);  // Depends on == implementation

// identical() checks memory identity
print(identical(a, b));  // false - different objects
```

**Default behavior:**

```dart
class Point {
  final int x, y;
  Point(this.x, this.y);
  // Default == compares identity (same as identical)
}

final p1 = Point(1, 2);
final p2 = Point(1, 2);
print(p1 == p2);         // false - different objects
print(identical(p1, p2)); // false - different objects
```

**Overriding ==:**

```dart
class Point {
  final int x, y;
  Point(this.x, this.y);
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;  // Quick check
    return other is Point && other.x == x && other.y == y;
  }
  
  @override
  int get hashCode => Object.hash(x, y);  // Must override if == is overridden
}

final p1 = Point(1, 2);
final p2 = Point(1, 2);
print(p1 == p2);         // true - same values
print(identical(p1, p2)); // false - still different objects
```

**With const:**

```dart
class Point {
  final int x, y;
  const Point(this.x, this.y);  // const constructor
}

const p1 = Point(1, 2);
const p2 = Point(1, 2);
print(p1 == p2);         // true
print(identical(p1, p2)); // true! - same object (canonicalized)

// Non-const creates different objects
final p3 = Point(1, 2);
final p4 = Point(1, 2);
print(identical(p3, p4)); // false
```

**Practical implications:**

```dart
// This is why const widgets are important!
const Text('Hello');  // Canonicalized
const Text('Hello');  // Same object - Flutter skips comparison

// Using identical for quick checks
@override
bool operator ==(Object other) {
  if (identical(this, other)) return true;  // Fast path
  // ... expensive comparison ...
}
```

---

### 9. Explain generics in Dart

**Theory:**

Generics allow you to write type-safe, reusable code that works with multiple types while maintaining type information at compile time.

```dart
// Without generics - lose type safety
class BoxDynamic {
  dynamic value;
  BoxDynamic(this.value);
}

final box = BoxDynamic('Hello');
String value = box.value;  // No compile-time type checking!

// With generics - type safe
class Box<T> {
  T value;
  Box(this.value);
}

final box = Box<String>('Hello');
String value = box.value;  // Compiler knows it's a String
```

**Generic classes:**

```dart
class Pair<K, V> {
  final K first;
  final V second;
  
  Pair(this.first, this.second);
  
  Pair<V, K> swap() => Pair(second, first);
}

final pair = Pair<String, int>('age', 25);
print(pair.first);   // String: 'age'
print(pair.second);  // int: 25

// Type inference
final inferred = Pair('name', 'Alice');  // Pair<String, String>
```

**Generic functions:**

```dart
T first<T>(List<T> items) {
  return items[0];
}

final number = first<int>([1, 2, 3]);    // int
final name = first(['Alice', 'Bob']);    // String (inferred)

// Multiple type parameters
Map<K, V> zipToMap<K, V>(List<K> keys, List<V> values) {
  return Map.fromIterables(keys, values);
}

final ages = zipToMap(['Alice', 'Bob'], [30, 25]);
// Map<String, int>
```

**Type constraints:**

```dart
// Unbounded - any type
class Box<T> { }

// Bounded - must be subtype of Comparable
class SortedList<T extends Comparable<T>> {
  final List<T> _items = [];
  
  void add(T item) {
    _items.add(item);
    _items.sort();  // Works because T is Comparable
  }
}

// Usage
final numbers = SortedList<int>();  // ✅ int is Comparable
// final things = SortedList<Object>();  // ❌ Object isn't Comparable
```

**Common Flutter generics:**

```dart
// ListView.builder with typed items
ListView.builder<User>(
  itemCount: users.length,
  itemBuilder: (context, index) {
    final User user = users[index];  // Type safe
    return UserTile(user: user);
  },
);

// FutureBuilder with type
FutureBuilder<List<Post>>(
  future: fetchPosts(),
  builder: (context, snapshot) {
    if (snapshot.hasData) {
      final List<Post> posts = snapshot.data!;  // Type safe
      return PostList(posts: posts);
    }
    return CircularProgressIndicator();
  },
);
```

---

### 10. What are extension methods and when would you use them?

**Theory:**

Extension methods let you add functionality to existing classes without modifying them or creating subclasses.

```dart
// Problem: Can't modify String class
// Solution: Extension methods!

extension StringExtensions on String {
  // Add new methods to String
  String capitalize() {
    if (isEmpty) return this;
    return '${this[0].toUpperCase()}${substring(1)}';
  }
  
  bool get isValidEmail {
    return RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(this);
  }
  
  String truncate(int maxLength) {
    if (length <= maxLength) return this;
    return '${substring(0, maxLength)}...';
  }
}

void main() {
  final name = 'alice';
  print(name.capitalize());  // 'Alice'
  
  final email = 'test@example.com';
  print(email.isValidEmail);  // true
  
  final text = 'Hello, World!';
  print(text.truncate(5));  // 'Hello...'
}
```

**Generic extensions:**

```dart
extension ListExtensions<T> on List<T> {
  T? get firstOrNull => isEmpty ? null : first;
  
  List<T> distinct() => toSet().toList();
  
  List<T> separatedBy(T separator) {
    if (isEmpty) return [];
    return expand((item) => [item, separator]).toList()..removeLast();
  }
}

final numbers = [1, 2, 2, 3, 3, 3];
print(numbers.distinct());  // [1, 2, 3]

final empty = <String>[];
print(empty.firstOrNull);  // null (no exception!)
```

**Flutter-specific extensions:**

```dart
extension BuildContextExtensions on BuildContext {
  // Easy access to theme
  ThemeData get theme => Theme.of(this);
  TextTheme get textTheme => theme.textTheme;
  ColorScheme get colorScheme => theme.colorScheme;
  
  // Easy access to media query
  MediaQueryData get mediaQuery => MediaQuery.of(this);
  Size get screenSize => mediaQuery.size;
  bool get isLandscape => mediaQuery.orientation == Orientation.landscape;
  
  // Navigation shortcuts
  void pop<T>([T? result]) => Navigator.of(this).pop(result);
  Future<T?> push<T>(Widget page) => Navigator.of(this).push(
    MaterialPageRoute(builder: (_) => page),
  );
  
  // Show snackbar easily
  void showSnackBar(String message) {
    ScaffoldMessenger.of(this).showSnackBar(
      SnackBar(content: Text(message)),
    );
  }
}

// Usage in widgets
class MyWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      color: context.colorScheme.primary,
      width: context.screenSize.width * 0.5,
      child: Text(
        'Hello',
        style: context.textTheme.headlineLarge,
      ),
    );
  }
}
```

**Best practices:**

```dart
// ✅ Good: Focused, related functionality
extension DateTimeFormatting on DateTime {
  String get formatted => '$day/$month/$year';
  String get timeAgo => _calculateTimeAgo();
}

// ❌ Bad: Unrelated functionality
extension DateTimeEverything on DateTime {
  String get formatted => ...;
  double calculateMortgage() => ...;  // Unrelated!
}

// ✅ Good: Named extensions (can be imported specifically)
extension IntValidation on int { ... }

// ⚠️ Caution: Anonymous extensions (always available)
extension on int { ... }  // Can't selectively import
```

---

### 11. What is a factory constructor?

**Theory:**

A `factory` constructor can return an existing instance, a subtype, or null (when returning nullable type). Unlike regular constructors, it doesn't automatically create a new instance.

```dart
// Regular constructor - always creates new instance
class Dog {
  String name;
  Dog(this.name);
}

// Factory constructor - controls instantiation
class Logger {
  static final Logger _instance = Logger._internal();
  
  factory Logger() {
    return _instance;  // Return existing instance
  }
  
  Logger._internal();  // Private constructor
}

void main() {
  final log1 = Logger();
  final log2 = Logger();
  print(identical(log1, log2));  // true - same instance!
}
```

**Common use cases:**

**1. Singleton pattern:**

```dart
class Database {
  static final Database _instance = Database._internal();
  
  factory Database() => _instance;
  
  Database._internal();
  
  void query(String sql) { }
}
```

**2. Return subtype based on input:**

```dart
abstract class Shape {
  factory Shape(String type) {
    switch (type) {
      case 'circle': return Circle();
      case 'square': return Square();
      default: throw ArgumentError('Unknown shape: $type');
    }
  }
  
  double get area;
}

class Circle implements Shape {
  @override
  double get area => 3.14159 * 10 * 10;  // radius = 10
}

class Square implements Shape {
  @override
  double get area => 10 * 10;  // side = 10
}

void main() {
  final shape = Shape('circle');  // Returns Circle instance
  print(shape.area);
}
```

**3. JSON deserialization:**

```dart
class User {
  final String id;
  final String name;
  final String email;
  
  User({required this.id, required this.name, required this.email});
  
  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] as String,
      name: json['name'] as String,
      email: json['email'] as String,
    );
  }
  
  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'email': email,
  };
}

// Usage
final json = {'id': '1', 'name': 'Alice', 'email': 'alice@example.com'};
final user = User.fromJson(json);
```

**4. Caching instances:**

```dart
class ExpensiveObject {
  static final Map<String, ExpensiveObject> _cache = {};
  
  final String id;
  
  factory ExpensiveObject(String id) {
    return _cache.putIfAbsent(id, () => ExpensiveObject._internal(id));
  }
  
  ExpensiveObject._internal(this.id) {
    print('Creating ExpensiveObject: $id');
  }
}

void main() {
  final obj1 = ExpensiveObject('abc');  // Prints: Creating...
  final obj2 = ExpensiveObject('abc');  // No print - cached
  print(identical(obj1, obj2));  // true
}
```

**Regular vs Factory constructor:**

| Feature | Regular Constructor | Factory Constructor |
|---------|---------------------|---------------------|
| Creates new instance | Always | Optional |
| Returns `this` | Implicitly | Explicitly |
| Can return subtype | No | Yes |
| Can be const | Yes | No |
| Can return null | No | Yes (if nullable) |
| Access to `this` | Yes | No |

---

### 12. Explain named constructors in Dart

**Theory:**

Named constructors provide alternative ways to create an object, each with a descriptive name explaining the construction method.

```dart
class Point {
  final double x;
  final double y;
  
  // Primary constructor
  Point(this.x, this.y);
  
  // Named constructors
  Point.origin() : x = 0, y = 0;
  
  Point.fromJson(Map<String, dynamic> json)
    : x = json['x'] as double,
      y = json['y'] as double;
  
  Point.polar(double distance, double angle)
    : x = distance * cos(angle),
      y = distance * sin(angle);
  
  @override
  String toString() => 'Point($x, $y)';
}

void main() {
  final p1 = Point(3, 4);
  final p2 = Point.origin();
  final p3 = Point.fromJson({'x': 1.0, 'y': 2.0});
  final p4 = Point.polar(5, pi / 4);
  
  print(p1);  // Point(3.0, 4.0)
  print(p2);  // Point(0.0, 0.0)
  print(p3);  // Point(1.0, 2.0)
}
```

**Common patterns in Flutter:**

```dart
class TextStyle {
  // Named constructors for common styles
  TextStyle.displayLarge() : this(...);
  TextStyle.bodyMedium() : this(...);
  TextStyle.caption() : this(...);
}

class EdgeInsets {
  // Named constructors for different padding patterns
  EdgeInsets.all(double value);
  EdgeInsets.only({...});
  EdgeInsets.symmetric({...});
  EdgeInsets.fromLTRB(...);
}

class BorderRadius {
  BorderRadius.circular(double radius);
  BorderRadius.vertical({...});
  BorderRadius.horizontal({...});
}
```

**Redirecting constructors:**

```dart
class Rectangle {
  final double width;
  final double height;
  
  Rectangle(this.width, this.height);
  
  // Redirecting constructor - calls primary constructor
  Rectangle.square(double size) : this(size, size);
}
```

---

### 13. What are sealed classes in Dart?

**Theory:**

Sealed classes restrict which classes can extend/implement them, enabling exhaustive pattern matching. The compiler knows all possible subtypes.

```dart
// Sealed class - subtypes must be in same file
sealed class Result<T> {}

class Success<T> extends Result<T> {
  final T value;
  Success(this.value);
}

class Failure<T> extends Result<T> {
  final String error;
  Failure(this.error);
}

class Loading<T> extends Result<T> {}

// Exhaustive pattern matching
String handleResult(Result<String> result) {
  return switch (result) {
    Success(:final value) => 'Got: $value',
    Failure(:final error) => 'Error: $error',
    Loading() => 'Loading...',
    // No default needed - compiler knows all cases!
  };
}
```

**Use cases:**

**1. State management:**

```dart
sealed class AuthState {}

class AuthInitial extends AuthState {}
class AuthLoading extends AuthState {}
class AuthAuthenticated extends AuthState {
  final User user;
  AuthAuthenticated(this.user);
}
class AuthError extends AuthState {
  final String message;
  AuthError(this.message);
}

Widget buildAuthUI(AuthState state) {
  return switch (state) {
    AuthInitial() => LoginButton(),
    AuthLoading() => CircularProgressIndicator(),
    AuthAuthenticated(:final user) => HomePage(user: user),
    AuthError(:final message) => ErrorWidget(message: message),
  };
}
```

**2. API responses:**

```dart
sealed class ApiResponse<T> {}

class ApiSuccess<T> extends ApiResponse<T> {
  final T data;
  final int statusCode;
  ApiSuccess(this.data, this.statusCode);
}

class ApiError<T> extends ApiResponse<T> {
  final String message;
  final int? statusCode;
  ApiError(this.message, [this.statusCode]);
}

// Handling is exhaustive
Future<void> handleResponse(ApiResponse<User> response) async {
  switch (response) {
    case ApiSuccess(:final data, :final statusCode):
      print('Got user: ${data.name} (status: $statusCode)');
    case ApiError(:final message, :final statusCode):
      print('Error: $message (status: $statusCode)');
  }
}
```

**Benefits:**

1. **Exhaustiveness checking** - Compiler warns if you miss a case
2. **Type safety** - No unexpected subtypes at runtime
3. **Pattern matching** - Destructure and switch in one step
4. **Refactoring safety** - Add a case, compiler shows where to update

---

### 14. Explain Dart records

**Theory:**

Records are anonymous, immutable data structures with positional and/or named fields. They're perfect for returning multiple values or creating lightweight data objects.

```dart
// Positional record
final point = (3, 4);
print(point.$1);  // 3
print(point.$2);  // 4

// Named record
final user = (name: 'Alice', age: 30);
print(user.name);  // Alice
print(user.age);   // 30

// Mixed
final mixed = (1, 2, name: 'test');
print(mixed.$1);    // 1
print(mixed.name);  // test
```

**Type annotations:**

```dart
// Positional record type
(int, int) getPoint() => (3, 4);

// Named record type
({String name, int age}) getUser() => (name: 'Alice', age: 30);

// Mixed
(int, {String name}) getMixed() => (1, name: 'test');
```

**Destructuring:**

```dart
// Positional destructuring
final (x, y) = getPoint();
print('x: $x, y: $y');

// Named destructuring
final (:name, :age) = getUser();
print('$name is $age years old');

// Partial destructuring
final (first, _, third) = (1, 2, 3);  // Ignore second value
```

**Pattern matching with records:**

```dart
String describe((int, int) point) {
  return switch (point) {
    (0, 0) => 'Origin',
    (0, _) => 'On Y-axis',
    (_, 0) => 'On X-axis',
    (var x, var y) when x == y => 'On diagonal',
    (var x, var y) => 'Point at ($x, $y)',
  };
}
```

**Practical use - multiple return values:**

```dart
// Before records - needed a class or Map
class MinMax {
  final int min, max;
  MinMax(this.min, this.max);
}

// With records - simple and clean
(int min, int max) findMinMax(List<int> numbers) {
  return (numbers.reduce(min), numbers.reduce(max));
}

void main() {
  final (min, max) = findMinMax([3, 1, 4, 1, 5, 9]);
  print('Min: $min, Max: $max');  // Min: 1, Max: 9
}
```

**Records vs Classes:**

| Feature | Record | Class |
|---------|--------|-------|
| Named | Anonymous | Named |
| Mutable | No | Configurable |
| Inheritance | No | Yes |
| Methods | No | Yes |
| == / hashCode | Auto | Manual |
| Best for | Simple data, tuples | Complex objects |

---

### 15. What is cascade notation (..)?

**Theory:**

Cascade notation allows you to perform multiple operations on the same object without repeating the object reference. It returns the object, not the operation result.

```dart
// Without cascade - repetitive
var button = Button();
button.text = 'Click me';
button.color = Colors.blue;
button.onClick = handleClick;
button.build();

// With cascade - clean
var button = Button()
  ..text = 'Click me'
  ..color = Colors.blue
  ..onClick = handleClick
  ..build();
```

**Common use cases:**

```dart
// 1. List operations
var numbers = <int>[]
  ..add(1)
  ..add(2)
  ..addAll([3, 4, 5])
  ..sort()
  ..removeWhere((n) => n.isEven);

// 2. Builder pattern
final paint = Paint()
  ..color = Colors.blue
  ..strokeWidth = 2.0
  ..style = PaintingStyle.stroke
  ..strokeCap = StrokeCap.round;

// 3. StringBuffer
final buffer = StringBuffer()
  ..write('Hello')
  ..write(' ')
  ..write('World')
  ..writeln('!');
String result = buffer.toString();

// 4. Stream controllers
final controller = StreamController<int>()
  ..add(1)
  ..add(2)
  ..add(3)
  ..close();
```

**Null-aware cascade (?..):**

```dart
// May be null - use ?..
User? user = getUser();
user
  ?..name = 'Alice'
  ..age = 30;  // Only executes if user != null
```

**Important: Cascade returns the object, not the result:**

```dart
// add() returns void, but cascade returns the list
final list = [1, 2]..add(3);
print(list);  // [1, 2, 3]

// Compare to normal call
final result = [1, 2].add(3);
print(result);  // void (null)
```

---

# Flutter Basics

### 16. What is the difference between StatelessWidget and StatefulWidget?

**Theory:**

The fundamental distinction is whether a widget needs to track mutable state that affects its build output.

```
┌─────────────────────────────────────────────────────────────────┐
│                    WIDGET TYPES                                  │
├──────────────────────┬──────────────────────────────────────────┤
│   StatelessWidget    │   StatefulWidget                         │
├──────────────────────┼──────────────────────────────────────────┤
│ • Immutable          │ • Has mutable State object               │
│ • build() only       │ • build() + lifecycle methods            │
│ • No internal state  │ • Can call setState()                    │
│ • Rebuilt by parent  │ • Can rebuild itself                     │
│ • Simple, fast       │ • More complex                           │
└──────────────────────┴──────────────────────────────────────────┘
```

**StatelessWidget:**

```dart
class Greeting extends StatelessWidget {
  final String name;  // Immutable - set by parent
  
  const Greeting({required this.name, super.key});
  
  @override
  Widget build(BuildContext context) {
    // Always builds the same output for the same input
    return Text('Hello, $name!');
  }
}

// Use when:
// - Display-only widgets (icons, labels)
// - Widgets whose output depends only on constructor params
// - Container/wrapper widgets
```

**StatefulWidget:**

```dart
class Counter extends StatefulWidget {
  const Counter({super.key});
  
  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;  // Mutable state
  
  void _increment() {
    setState(() {  // Trigger rebuild
      _count++;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_count'),
        ElevatedButton(
          onPressed: _increment,
          child: const Text('Increment'),
        ),
      ],
    );
  }
}

// Use when:
// - Widget needs to change over time
// - Responding to user interaction
// - Animations
// - Data that changes (timers, streams)
```

**Lifecycle comparison:**

```dart
// StatelessWidget lifecycle:
// 1. Constructor called
// 2. build() called
// 3. (Widget is destroyed)

// StatefulWidget lifecycle:
// 1. Constructor called (Widget)
// 2. createState() called
// 3. initState() called
// 4. didChangeDependencies() called
// 5. build() called
// 6. (setState triggers rebuild → build() called again)
// 7. didUpdateWidget() when parent rebuilds with new config
// 8. dispose() called when removed from tree
```

**Common mistake - using StatefulWidget unnecessarily:**

```dart
// ❌ Bad - no internal state, should be StatelessWidget
class MyWidget extends StatefulWidget {
  final String title;
  MyWidget({required this.title});
  
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  @override
  Widget build(BuildContext context) {
    return Text(widget.title);  // Just displays prop, no state!
  }
}

// ✅ Good - use StatelessWidget
class MyWidget extends StatelessWidget {
  final String title;
  const MyWidget({required this.title, super.key});
  
  @override
  Widget build(BuildContext context) {
    return Text(title);
  }
}
```

---

### 17. Explain the Flutter widget lifecycle

**Theory:**

StatefulWidget has a rich lifecycle that allows you to initialize resources, respond to changes, and clean up when done.

```
┌─────────────────────────────────────────────────────────────────┐
│                 STATEFULWIDGET LIFECYCLE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    createState()                                                 │
│         │                                                        │
│         ▼                                                        │
│    initState() ◄────────────────┐                               │
│         │                       │                                │
│         ▼                       │                                │
│    didChangeDependencies()      │                                │
│         │                       │                                │
│         ▼                       │                                │
│    build() ◄──────────┐         │                                │
│         │             │         │                                │
│         ▼             │         │                                │
│    [Widget Active]    │         │                                │
│         │             │         │                                │
│    ┌────┴────┐        │         │                                │
│    │         │        │         │                                │
│    ▼         ▼        │         │                                │
│ setState() didUpdateWidget()    │                                │
│    │         │        │         │                                │
│    └────┬────┘        │         │                                │
│         │             │         │                                │
│         └─────────────┘         │                                │
│                                 │                                │
│    deactivate()                 │                                │
│         │                       │                                │
│         ▼                       │                                │
│    dispose() ───────────────────┘ (if reinserted)               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Lifecycle methods:**

```dart
class LifecycleWidget extends StatefulWidget {
  final String data;
  const LifecycleWidget({required this.data, super.key});
  
  @override
  State<LifecycleWidget> createState() => _LifecycleWidgetState();
}

class _LifecycleWidgetState extends State<LifecycleWidget> {
  late StreamSubscription _subscription;
  late AnimationController _controller;
  
  @override
  void initState() {
    super.initState();
    // Called ONCE when State is created
    // Initialize controllers, subscriptions, one-time setup
    print('initState');
    _subscription = myStream.listen(handleData);
  }
  
  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // Called after initState and when dependencies change
    // InheritedWidget lookups safe here
    print('didChangeDependencies');
    final theme = Theme.of(context);  // Safe here
  }
  
  @override
  void didUpdateWidget(LifecycleWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    // Called when parent rebuilds with new config
    // Compare old and new widget properties
    print('didUpdateWidget');
    if (widget.data != oldWidget.data) {
      // React to prop changes
      fetchNewData(widget.data);
    }
  }
  
  @override
  Widget build(BuildContext context) {
    // Called whenever setState, didUpdateWidget, or didChangeDependencies
    print('build');
    return Text(widget.data);
  }
  
  @override
  void deactivate() {
    // Called when widget is removed from tree (may be reinserted)
    print('deactivate');
    super.deactivate();
  }
  
  @override
  void dispose() {
    // Called when State is permanently removed
    // Clean up: cancel subscriptions, dispose controllers
    print('dispose');
    _subscription.cancel();
    _controller.dispose();
    super.dispose();
  }
}
```

**What to do in each method:**

| Method | Use For |
|--------|---------|
| `initState` | Initialize controllers, subscriptions, one-time setup |
| `didChangeDependencies` | React to InheritedWidget changes |
| `didUpdateWidget` | React to parent configuration changes |
| `build` | Create widget tree (KEEP FAST!) |
| `deactivate` | Rarely used - cleanup before potential reinsertion |
| `dispose` | Cancel subscriptions, dispose controllers |

**Common mistakes:**

```dart
// ❌ Using context in initState
@override
void initState() {
  super.initState();
  final theme = Theme.of(context);  // May not work!
}

// ✅ Use didChangeDependencies or post-frame callback
@override
void initState() {
  super.initState();
  WidgetsBinding.instance.addPostFrameCallback((_) {
    final theme = Theme.of(context);  // Safe after build
  });
}

// ❌ Forgetting to dispose
@override
void initState() {
  super.initState();
  _controller = AnimationController(vsync: this);
  // Never disposed! Memory leak!
}

// ✅ Always dispose
@override
void dispose() {
  _controller.dispose();
  super.dispose();
}
```

---

### 18. What is BuildContext?

**Theory:**

`BuildContext` is a reference to a widget's location in the widget tree. It's actually an `Element` object, which is the link between the Widget and RenderObject trees.

```
┌─────────────────────────────────────────────────────────────────┐
│                    BUILDCONTEXT                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   BuildContext IS the Element                                    │
│                                                                  │
│   Widget Tree          Element Tree                              │
│   ───────────          ────────────                              │
│   MaterialApp    ◄───► MaterialAppElement (context)             │
│       │                     │                                    │
│       ▼                     ▼                                    │
│    Scaffold      ◄───► ScaffoldElement (context)                │
│       │                     │                                    │
│       ▼                     ▼                                    │
│    MyWidget      ◄───► StatefulElement (context)                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**What BuildContext provides:**

```dart
@override
Widget build(BuildContext context) {
  // 1. Access ancestors (InheritedWidgets)
  final theme = Theme.of(context);
  final mediaQuery = MediaQuery.of(context);
  final navigator = Navigator.of(context);
  final scaffold = Scaffold.of(context);
  
  // 2. Get widget information
  final widget = context.widget;
  final size = context.size;  // After layout
  
  // 3. Find ancestors/descendants
  final scaffold = context.findAncestorWidgetOfExactType<Scaffold>();
  final renderBox = context.findRenderObject() as RenderBox;
  
  return Container();
}
```

**Context rules:**

```dart
// ❌ Wrong context - Scaffold.of needs Scaffold ABOVE in tree
@override
Widget build(BuildContext context) {
  return Scaffold(
    body: ElevatedButton(
      onPressed: () {
        // context is ABOVE Scaffold, not below!
        Scaffold.of(context).showBottomSheet(...);  // Error!
      },
      child: Text('Show Sheet'),
    ),
  );
}

// ✅ Use Builder to get context BELOW Scaffold
@override
Widget build(BuildContext context) {
  return Scaffold(
    body: Builder(
      builder: (scaffoldContext) {  // This context is BELOW Scaffold
        return ElevatedButton(
          onPressed: () {
            Scaffold.of(scaffoldContext).showBottomSheet(...);  // Works!
          },
          child: Text('Show Sheet'),
        );
      },
    ),
  );
}
```

**Context and async:**

```dart
// ❌ Dangerous - context may be invalid after await
Future<void> doSomething(BuildContext context) async {
  await Future.delayed(Duration(seconds: 2));
  Navigator.of(context).pop();  // Widget might be disposed!
}

// ✅ Check if mounted
Future<void> doSomething(BuildContext context) async {
  await Future.delayed(Duration(seconds: 2));
  if (context.mounted) {
    Navigator.of(context).pop();
  }
}
```

---

### 19. How does the Flutter rendering pipeline work?

**Theory:**

Flutter renders frames through a pipeline that transforms widgets into pixels on screen.

```
┌─────────────────────────────────────────────────────────────────┐
│                 FLUTTER RENDERING PIPELINE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. BUILD PHASE                                                  │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  Widget.build() → Widget tree → Element tree        │     │
│     └─────────────────────────────────────────────────────┘     │
│                              │                                   │
│                              ▼                                   │
│  2. LAYOUT PHASE                                                 │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  RenderObject.performLayout()                       │     │
│     │  • Parent passes constraints down                   │     │
│     │  • Child returns size up                            │     │
│     │  • Parent positions child                           │     │
│     └─────────────────────────────────────────────────────┘     │
│                              │                                   │
│                              ▼                                   │
│  3. PAINT PHASE                                                  │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  RenderObject.paint()                               │     │
│     │  • Creates Layer tree                               │     │
│     │  • Draws to canvas                                  │     │
│     └─────────────────────────────────────────────────────┘     │
│                              │                                   │
│                              ▼                                   │
│  4. COMPOSITING PHASE                                            │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  Layer tree → Scene → GPU                           │     │
│     │  • Rasterize to pixels                              │     │
│     │  • Display on screen                                │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                  │
│  Target: Complete in <16.67ms for 60fps                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Layout constraints:**

```dart
// Parent passes down constraints
// Child returns size within those constraints

Container(
  width: 200,
  height: 100,
  child: Center(          // Passes loose constraints
    child: Text('Hello'), // Sizes itself, returns size to Center
  ),
)
```

**Optimization tips:**

```dart
// 1. Use const constructors
const Text('Hello')  // Same instance reused

// 2. Minimize rebuilds
Consumer<User>(      // Only rebuilds this subtree
  builder: (context, user, child) => Text(user.name),
)

// 3. Use RepaintBoundary for expensive paints
RepaintBoundary(
  child: ComplexAnimatedWidget(),
)

// 4. Cache expensive computations
@override
Widget build(BuildContext context) {
  // ❌ Computed every build
  final sorted = items.sorted();
  
  // ✅ Cache in state
  // final sorted = _sortedItems; // Computed once
}
```

---

### 20. What is the difference between hot reload and hot restart?

**Theory:**

Both are development features that speed up iteration, but they preserve different amounts of state.

```
┌─────────────────────────────────────────────────────────────────┐
│                 HOT RELOAD vs HOT RESTART                        │
├──────────────────────┬──────────────────────────────────────────┤
│     Hot Reload       │       Hot Restart                         │
├──────────────────────┼──────────────────────────────────────────┤
│ • Injects new code   │ • Restarts entire Dart VM                │
│ • Preserves state    │ • Loses all state                        │
│ • Sub-second         │ • Few seconds                            │
│ • build() re-runs    │ • main() re-runs                         │
│ • State unchanged    │ • initState() re-runs                    │
└──────────────────────┴──────────────────────────────────────────┘
```

**Hot Reload (r in terminal, ⚡ in IDE):**

```dart
class MyWidget extends StatefulWidget {
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  int _counter = 5;  // Value preserved!
  
  @override
  Widget build(BuildContext context) {
    return Text('Count: $_counter');  // Change color here
  }
}

// Hot reload:
// - build() called again
// - _counter stays 5
// - UI updates with new styling
```

**Hot Restart (R in terminal):**

```dart
// Hot restart:
// - App restarts from main()
// - All state reset to initial values
// - _counter becomes 0 again
```

**When hot reload doesn't work:**

```dart
// 1. Global variable changes
final globalConfig = loadConfig();  // Needs restart

// 2. initState changes
@override
void initState() {
  super.initState();
  _loadData();  // Needs restart to re-run
}

// 3. Enum changes
enum Status { idle, loading, done }  // Needs restart

// 4. Generic type changes
class Box<T> { }  // Needs restart

// 5. Native code changes (iOS/Android)
```

---

### 21. What are Keys in Flutter and when should you use them?

**Theory:**

Keys tell Flutter how to match widgets between rebuilds. Without keys, Flutter uses position (index) to match widgets.

```
┌─────────────────────────────────────────────────────────────────┐
│                      WHY KEYS MATTER                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Without keys (matching by position):                           │
│                                                                  │
│   Before:  [A] [B] [C]     After removing B:  [A] [C] [_]       │
│             0   1   2                          0   1   2         │
│                                                                  │
│   Flutter thinks:                                                │
│   - Position 0: A → A (same, keep)                              │
│   - Position 1: B → C (different! rebuild B to look like C)     │
│   - Position 2: C → _ (removed, dispose)                        │
│                                                                  │
│   With keys (matching by key):                                   │
│                                                                  │
│   Before:  [A:1] [B:2] [C:3]   After:  [A:1] [C:3]             │
│                                                                  │
│   Flutter thinks:                                                │
│   - Key 1: A → A (keep)                                         │
│   - Key 2: B → _ (removed, dispose)                             │
│   - Key 3: C → C (keep, just moved)                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Types of keys:**

```dart
// 1. ValueKey - based on value
ListTile(
  key: ValueKey(user.id),  // Use unique identifier
  title: Text(user.name),
)

// 2. ObjectKey - based on object identity
ListTile(
  key: ObjectKey(user),  // Use object reference
  title: Text(user.name),
)

// 3. UniqueKey - always unique (regenerated each build)
Container(
  key: UniqueKey(),  // Forces rebuild every time
)

// 4. GlobalKey - access state from anywhere
final _formKey = GlobalKey<FormState>();

Form(
  key: _formKey,
  child: ...
)

// Access: _formKey.currentState?.validate()
```

**When to use keys:**

```dart
// ✅ Use keys when:

// 1. List items can be reordered
ReorderableListView(
  children: items.map((item) => ListTile(
    key: ValueKey(item.id),  // Essential for reordering!
    title: Text(item.name),
  )).toList(),
)

// 2. Stateful widgets in lists that can change order
ListView.builder(
  itemBuilder: (context, index) {
    return StatefulItemWidget(
      key: ValueKey(items[index].id),  // Preserves state
      item: items[index],
    );
  },
)

// 3. Swapping widgets conditionally
isGridView 
  ? GridView(key: ValueKey('grid'), ...)
  : ListView(key: ValueKey('list'), ...)

// 4. Force widget recreation
RefreshIndicator(
  key: ValueKey(selectedCategory),  // Recreates on category change
  child: ProductList(),
)
```

**Common mistake - missing keys in dynamic lists:**

```dart
// ❌ Bad - no keys, stateful children will have wrong state
Column(
  children: items.map((item) => CheckboxTile(item: item)).toList(),
)

// ✅ Good - keys preserve checkbox state correctly
Column(
  children: items.map((item) => CheckboxTile(
    key: ValueKey(item.id),
    item: item,
  )).toList(),
)
```

---

### 22. Explain the difference between mainAxisAlignment and crossAxisAlignment

**Theory:**

These properties control how children are positioned in Row/Column/Flex widgets along their two axes.

```
┌─────────────────────────────────────────────────────────────────┐
│                    AXES IN ROW AND COLUMN                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ROW (horizontal):                                              │
│   ─────────────────                                              │
│                                                                  │
│   Main Axis →                                                    │
│   ┌────────────────────────────────────────┐                    │
│   │  [A]        [B]        [C]             │ ↑                  │
│   │                                         │ │ Cross Axis       │
│   │                                         │ ↓                  │
│   └────────────────────────────────────────┘                    │
│                                                                  │
│   COLUMN (vertical):                                             │
│   ──────────────────                                             │
│                                                                  │
│   Cross Axis →                                                   │
│   ┌──────────────┐                                              │
│   │     [A]      │                                              │
│   │              │ ↑                                             │
│   │     [B]      │ │ Main Axis                                  │
│   │              │ │                                             │
│   │     [C]      │ ↓                                             │
│   └──────────────┘                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**MainAxisAlignment options:**

```dart
Row(
  mainAxisAlignment: MainAxisAlignment.start,        // [A][B][C]______
  mainAxisAlignment: MainAxisAlignment.end,          // ______[A][B][C]
  mainAxisAlignment: MainAxisAlignment.center,       // ___[A][B][C]___
  mainAxisAlignment: MainAxisAlignment.spaceBetween, // [A]___[B]___[C]
  mainAxisAlignment: MainAxisAlignment.spaceAround,  // _[A]__[B]__[C]_
  mainAxisAlignment: MainAxisAlignment.spaceEvenly,  // __[A]__[B]__[C]__
  children: [A, B, C],
)
```

**CrossAxisAlignment options:**

```dart
Row(
  crossAxisAlignment: CrossAxisAlignment.start,    // Align to top
  crossAxisAlignment: CrossAxisAlignment.end,      // Align to bottom
  crossAxisAlignment: CrossAxisAlignment.center,   // Align to center (default)
  crossAxisAlignment: CrossAxisAlignment.stretch,  // Stretch to fill height
  crossAxisAlignment: CrossAxisAlignment.baseline, // Align text baselines
  textBaseline: TextBaseline.alphabetic,           // Required for baseline
  children: [...],
)
```

**Practical examples:**

```dart
// Navigation bar - space items evenly
Row(
  mainAxisAlignment: MainAxisAlignment.spaceEvenly,
  children: [
    IconButton(icon: Icon(Icons.home)),
    IconButton(icon: Icon(Icons.search)),
    IconButton(icon: Icon(Icons.person)),
  ],
)

// Form - stretch inputs to fill width
Column(
  crossAxisAlignment: CrossAxisAlignment.stretch,
  children: [
    TextField(decoration: InputDecoration(labelText: 'Email')),
    TextField(decoration: InputDecoration(labelText: 'Password')),
    ElevatedButton(onPressed: () {}, child: Text('Login')),
  ],
)

// Center content in available space
Column(
  mainAxisAlignment: MainAxisAlignment.center,
  crossAxisAlignment: CrossAxisAlignment.center,
  children: [
    Icon(Icons.check, size: 64),
    Text('Success!'),
  ],
)
```

---

### 23. What is the difference between Expanded and Flexible?

**Theory:**

Both distribute space in Row/Column/Flex, but `Expanded` forces the child to fill all available space, while `Flexible` lets the child be smaller if it wants.

```dart
// Expanded = Flexible with FlexFit.tight (must fill space)
// Flexible = FlexFit.loose by default (can be smaller)
```

**Expanded:**

```dart
Row(
  children: [
    Container(width: 50, color: Colors.red),
    Expanded(  // Takes ALL remaining space
      child: Container(color: Colors.green),
    ),
    Container(width: 50, color: Colors.blue),
  ],
)

// Result: [red:50px][green:fills rest][blue:50px]
```

**Flexible:**

```dart
Row(
  children: [
    Flexible(
      child: Text('Short'),  // Uses only what it needs
    ),
    Flexible(
      child: Text('Very long text that could wrap'),
    ),
  ],
)

// Children can be smaller than their flex allocation
```

**Flex factor:**

```dart
Row(
  children: [
    Expanded(
      flex: 1,  // 1 part
      child: Container(color: Colors.red),
    ),
    Expanded(
      flex: 2,  // 2 parts
      child: Container(color: Colors.green),
    ),
    Expanded(
      flex: 1,  // 1 part
      child: Container(color: Colors.blue),
    ),
  ],
)

// Total: 4 parts
// Red: 25%, Green: 50%, Blue: 25%
```

**Comparison:**

| Feature | Expanded | Flexible |
|---------|----------|----------|
| FlexFit | tight (must fill) | loose (can shrink) |
| Child fills space | Yes | Only if needed |
| Use case | Divide space evenly | Allow natural sizing |

---

### 24. Explain the Scaffold widget

**Theory:**

`Scaffold` provides the standard Material Design visual structure for an app screen, including app bar, body, floating action button, drawer, bottom navigation, and more.

```dart
Scaffold(
  // Top app bar
  appBar: AppBar(
    title: Text('My App'),
    leading: IconButton(icon: Icon(Icons.menu), onPressed: () {}),
    actions: [
      IconButton(icon: Icon(Icons.search), onPressed: () {}),
    ],
  ),
  
  // Main content
  body: Center(
    child: Text('Hello, World!'),
  ),
  
  // Floating action button
  floatingActionButton: FloatingActionButton(
    onPressed: () {},
    child: Icon(Icons.add),
  ),
  floatingActionButtonLocation: FloatingActionButtonLocation.centerDocked,
  
  // Side drawer
  drawer: Drawer(
    child: ListView(
      children: [
        DrawerHeader(child: Text('Menu')),
        ListTile(title: Text('Item 1')),
        ListTile(title: Text('Item 2')),
      ],
    ),
  ),
  
  // End drawer (right side)
  endDrawer: Drawer(...),
  
  // Bottom navigation
  bottomNavigationBar: BottomNavigationBar(
    items: [
      BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Home'),
      BottomNavigationBarItem(icon: Icon(Icons.settings), label: 'Settings'),
    ],
  ),
  
  // Bottom sheet (persistent)
  bottomSheet: Container(
    height: 50,
    child: Center(child: Text('Bottom Sheet')),
  ),
  
  // Snackbar area
  // ScaffoldMessenger.of(context).showSnackBar(...)
)
```

**Accessing Scaffold from descendants:**

```dart
// Show snackbar
ScaffoldMessenger.of(context).showSnackBar(
  SnackBar(content: Text('Hello!')),
);

// Open drawer
Scaffold.of(context).openDrawer();

// Show bottom sheet
Scaffold.of(context).showBottomSheet(
  (context) => Container(height: 200),
);

// Show modal bottom sheet (covers scaffold)
showModalBottomSheet(
  context: context,
  builder: (context) => Container(height: 200),
);
```

---

### 25. How do you handle navigation in Flutter?

**Theory:**

Flutter provides both imperative (Navigator 1.0) and declarative (Navigator 2.0/go_router) navigation approaches.

**Navigator 1.0 (Imperative):**

```dart
// Push new screen
Navigator.of(context).push(
  MaterialPageRoute(builder: (context) => DetailScreen()),
);

// Push with named route
Navigator.of(context).pushNamed('/detail');

// Push and remove all previous
Navigator.of(context).pushAndRemoveUntil(
  MaterialPageRoute(builder: (context) => HomeScreen()),
  (route) => false,
);

// Pop current screen
Navigator.of(context).pop();

// Pop with result
Navigator.of(context).pop('result data');

// Pop until specific route
Navigator.of(context).popUntil((route) => route.isFirst);

// Push replacement (replace current)
Navigator.of(context).pushReplacement(
  MaterialPageRoute(builder: (context) => NewScreen()),
);
```

**Passing data:**

```dart
// Via constructor (push)
Navigator.of(context).push(
  MaterialPageRoute(
    builder: (context) => DetailScreen(userId: '123'),
  ),
);

// Via named route arguments
Navigator.of(context).pushNamed(
  '/detail',
  arguments: {'userId': '123'},
);

// Receive arguments
final args = ModalRoute.of(context)!.settings.arguments as Map;
final userId = args['userId'];

// Return data on pop
// Screen A:
final result = await Navigator.of(context).push(
  MaterialPageRoute(builder: (context) => PickerScreen()),
);
print(result);  // Data from Screen B

// Screen B:
Navigator.of(context).pop('selected item');
```

**go_router (Declarative - recommended):**

```dart
final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => HomeScreen(),
      routes: [
        GoRoute(
          path: 'user/:id',
          builder: (context, state) {
            final id = state.pathParameters['id']!;
            return UserScreen(userId: id);
          },
        ),
      ],
    ),
  ],
);

// Navigate
context.go('/user/123');
context.push('/user/123');
context.pop();
```

---

# Widgets & UI

### 26. Explain the difference between Container and SizedBox

**Theory:**

Both can size children, but `Container` is a convenience widget combining many features, while `SizedBox` is lightweight and focused on sizing.

```dart
// Container - many features, heavier
Container(
  width: 100,
  height: 100,
  color: Colors.blue,
  padding: EdgeInsets.all(16),
  margin: EdgeInsets.all(8),
  decoration: BoxDecoration(
    borderRadius: BorderRadius.circular(8),
    gradient: LinearGradient(...),
    boxShadow: [...],
  ),
  alignment: Alignment.center,
  transform: Matrix4.rotationZ(0.1),
  child: Text('Hello'),
)

// SizedBox - just sizing, lighter
SizedBox(
  width: 100,
  height: 100,
  child: Text('Hello'),
)

// SizedBox.expand - fill parent
SizedBox.expand(child: Text('Fills parent'))

// SizedBox.shrink - 0x0 size
SizedBox.shrink()

// SizedBox for spacing (no child)
Column(
  children: [
    Text('Above'),
    SizedBox(height: 16),  // Spacer
    Text('Below'),
  ],
)
```

**When to use each:**

| Use Case | Widget |
|----------|--------|
| Just sizing/spacing | SizedBox |
| Background color only | ColoredBox |
| Decoration (borders, shadows) | Container (or DecoratedBox) |
| Padding only | Padding |
| Margin only | Container |
| Multiple features | Container |

---

### 27. What is SafeArea and when should you use it?

**Theory:**

`SafeArea` insets its child to avoid system UI intrusions like notches, status bars, and home indicators.

```dart
// Without SafeArea - content may be under notch
Scaffold(
  body: Column(
    children: [
      Text('This might be under the notch!'),
    ],
  ),
)

// With SafeArea - content respects system UI
Scaffold(
  body: SafeArea(
    child: Column(
      children: [
        Text('This is safely positioned'),
      ],
    ),
  ),
)

// Selective safe area
SafeArea(
  top: true,     // Avoid status bar
  bottom: true,  // Avoid home indicator
  left: false,   // Allow edge-to-edge
  right: false,
  child: ...,
)

// With minimum padding
SafeArea(
  minimum: EdgeInsets.all(16),  // At least 16px even if no intrusion
  child: ...,
)
```

**When to use:**

- ✅ Content that shouldn't be obscured (text, buttons)
- ✅ Lists that should scroll behind system UI but have safe content
- ❌ Background images that should extend under notch
- ❌ Bottom sheets (already handled)

---

### 28. How do you create responsive layouts in Flutter?

**Theory:**

Responsive layouts adapt to different screen sizes using queries, breakpoints, and flexible widgets.

**MediaQuery:**

```dart
@override
Widget build(BuildContext context) {
  final size = MediaQuery.of(context).size;
  final width = size.width;
  
  if (width < 600) {
    return MobileLayout();
  } else if (width < 900) {
    return TabletLayout();
  } else {
    return DesktopLayout();
  }
}
```

**LayoutBuilder:**

```dart
LayoutBuilder(
  builder: (context, constraints) {
    if (constraints.maxWidth < 600) {
      return ListView(...);  // Narrow: list
    } else {
      return GridView(...);  // Wide: grid
    }
  },
)
```

**Flexible widgets:**

```dart
// FractionallySizedBox - percentage of parent
FractionallySizedBox(
  widthFactor: 0.8,  // 80% of parent width
  child: Card(...),
)

// AspectRatio - maintain ratio
AspectRatio(
  aspectRatio: 16 / 9,
  child: VideoPlayer(),
)

// Flexible/Expanded - share space
Row(
  children: [
    Expanded(flex: 2, child: Content()),
    Expanded(flex: 1, child: Sidebar()),
  ],
)
```

**Responsive grid:**

```dart
GridView.builder(
  gridDelegate: SliverGridDelegateWithMaxCrossAxisExtent(
    maxCrossAxisExtent: 200,  // Max width per item
    childAspectRatio: 1,
    crossAxisSpacing: 16,
    mainAxisSpacing: 16,
  ),
  itemCount: items.length,
  itemBuilder: (context, index) => ItemCard(items[index]),
)
```

---

### 29. Explain ListView vs ListView.builder

**Theory:**

`ListView` creates all children immediately; `ListView.builder` creates them lazily, only when they scroll into view.

```dart
// ListView - all children created at once
// ❌ Bad for large lists
ListView(
  children: [
    ListTile(title: Text('Item 1')),
    ListTile(title: Text('Item 2')),
    ListTile(title: Text('Item 3')),
    // ... 1000 more items all created immediately!
  ],
)

// ListView.builder - creates on demand
// ✅ Good for large lists
ListView.builder(
  itemCount: 1000,
  itemBuilder: (context, index) {
    return ListTile(title: Text('Item $index'));
    // Only visible items + buffer are created
  },
)

// ListView.separated - with dividers
ListView.separated(
  itemCount: items.length,
  separatorBuilder: (context, index) => Divider(),
  itemBuilder: (context, index) => ListTile(title: Text('Item $index')),
)
```

**Performance comparison:**

| Feature | ListView | ListView.builder |
|---------|----------|------------------|
| Initial build | Slow (all items) | Fast (visible only) |
| Memory | High | Low |
| Use for | <20 items | Large lists |
| Scroll performance | May lag | Smooth |

---

### 30. What is CustomPainter and when would you use it?

**Theory:**

`CustomPainter` lets you draw custom graphics directly on a canvas when standard widgets aren't sufficient.

```dart
class CircleProgressPainter extends CustomPainter {
  final double progress;  // 0.0 to 1.0
  final Color color;
  
  CircleProgressPainter({required this.progress, required this.color});
  
  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = min(size.width, size.height) / 2;
    
    // Background circle
    final bgPaint = Paint()
      ..color = Colors.grey.shade300
      ..style = PaintingStyle.stroke
      ..strokeWidth = 10;
    
    canvas.drawCircle(center, radius, bgPaint);
    
    // Progress arc
    final progressPaint = Paint()
      ..color = color
      ..style = PaintingStyle.stroke
      ..strokeWidth = 10
      ..strokeCap = StrokeCap.round;
    
    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      -pi / 2,           // Start at top
      2 * pi * progress, // Sweep angle
      false,
      progressPaint,
    );
  }
  
  @override
  bool shouldRepaint(CircleProgressPainter oldDelegate) {
    return progress != oldDelegate.progress || color != oldDelegate.color;
  }
}

// Usage
CustomPaint(
  size: Size(100, 100),
  painter: CircleProgressPainter(progress: 0.7, color: Colors.blue),
)
```

**Use CustomPainter for:**

- Charts and graphs
- Custom progress indicators
- Drawing shapes
- Game graphics
- Signatures/sketches

---

### 31. How do you handle form validation?

**Theory:**

Flutter's `Form` widget manages validation state for a group of `TextFormField` widgets.

```dart
class LoginForm extends StatefulWidget {
  @override
  State<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends State<LoginForm> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  
  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }
  
  String? _validateEmail(String? value) {
    if (value == null || value.isEmpty) {
      return 'Email is required';
    }
    if (!value.contains('@')) {
      return 'Enter a valid email';
    }
    return null;  // Valid
  }
  
  String? _validatePassword(String? value) {
    if (value == null || value.isEmpty) {
      return 'Password is required';
    }
    if (value.length < 8) {
      return 'Password must be at least 8 characters';
    }
    return null;  // Valid
  }
  
  void _submit() {
    if (_formKey.currentState!.validate()) {
      // All fields valid
      print('Email: ${_emailController.text}');
      print('Password: ${_passwordController.text}');
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          TextFormField(
            controller: _emailController,
            decoration: InputDecoration(labelText: 'Email'),
            keyboardType: TextInputType.emailAddress,
            validator: _validateEmail,
            autovalidateMode: AutovalidateMode.onUserInteraction,
          ),
          SizedBox(height: 16),
          TextFormField(
            controller: _passwordController,
            decoration: InputDecoration(labelText: 'Password'),
            obscureText: true,
            validator: _validatePassword,
            autovalidateMode: AutovalidateMode.onUserInteraction,
          ),
          SizedBox(height: 24),
          ElevatedButton(
            onPressed: _submit,
            child: Text('Login'),
          ),
        ],
      ),
    );
  }
}
```

---

### 32. What is the difference between Visibility and Opacity?

**Theory:**

Both can hide widgets, but they have different effects on layout and performance.

```dart
// Opacity - renders but transparent
Opacity(
  opacity: 0.0,  // Invisible
  child: Text('Hello'),
)
// - Still takes space in layout
// - Still responds to hit tests
// - Still built and rendered (costly)

// Visibility - conditional rendering
Visibility(
  visible: false,
  child: Text('Hello'),
)
// - By default, doesn't take space
// - Not rendered (better performance)
// - Can configure behavior

// Visibility options
Visibility(
  visible: false,
  maintainSize: true,      // Keep space in layout
  maintainState: true,     // Keep state alive
  maintainAnimation: true, // Keep animations running
  maintainInteractivity: true, // Keep responding to input
  replacement: SizedBox.shrink(), // Widget shown when invisible
  child: ExpensiveWidget(),
)
```

**Comparison:**

| Feature | Opacity(0) | Visibility(false) |
|---------|------------|-------------------|
| Takes space | Yes | No (default) |
| Hit testing | Yes | No |
| Renders | Yes | No |
| Performance | Worse | Better |
| Keeps state | Yes | No (default) |

**Best practices:**

```dart
// ❌ Wasteful - still renders
Opacity(
  opacity: isVisible ? 1.0 : 0.0,
  child: ExpensiveWidget(),
)

// ✅ Better - doesn't render when hidden
if (isVisible) ExpensiveWidget()

// ✅ Or with visibility
Visibility(
  visible: isVisible,
  child: ExpensiveWidget(),
)

// ✅ Use Opacity for animations
AnimatedOpacity(
  opacity: isVisible ? 1.0 : 0.0,
  duration: Duration(milliseconds: 300),
  child: Widget(),
)
```

---

# State Management Basics

### 33. When should you use setState()?

**Theory:**

`setState()` tells Flutter that internal state has changed and triggers a rebuild of the widget.

```dart
class Counter extends StatefulWidget {
  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;
  
  void _increment() {
    setState(() {
      _count++;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_count'),
        ElevatedButton(
          onPressed: _increment,
          child: Text('Increment'),
        ),
      ],
    );
  }
}
```

**Rules for setState:**

```dart
// ✅ Correct - change state inside setState
setState(() {
  _count++;
});

// ⚠️ Works but not preferred
_count++;
setState(() {});  // Just triggers rebuild

// ❌ Don't call setState after dispose
@override
void dispose() {
  super.dispose();
}

Future<void> loadData() async {
  final data = await fetchData();
  if (mounted) {  // Check if still mounted!
    setState(() {
      _data = data;
    });
  }
}

// ❌ Don't call setState in build()
@override
Widget build(BuildContext context) {
  setState(() { });  // Bad! Infinite loop!
  return ...;
}
```

**When to use setState:**

- ✅ Simple local state
- ✅ Form inputs
- ✅ Animations
- ✅ Toggle switches, counters
- ❌ Complex state shared across screens (use state management)
- ❌ App-wide state (use Provider/Riverpod/BLoC)

---

### 34. What is InheritedWidget?

**Theory:**

`InheritedWidget` efficiently propagates data down the widget tree without explicitly passing it through every widget.

```dart
class ThemeProvider extends InheritedWidget {
  final ThemeData theme;
  
  const ThemeProvider({
    required this.theme,
    required Widget child,
    super.key,
  }) : super(child: child);
  
  // Static method for easy access
  static ThemeProvider of(BuildContext context) {
    final provider = context.dependOnInheritedWidgetOfExactType<ThemeProvider>();
    assert(provider != null, 'No ThemeProvider found in context');
    return provider!;
  }
  
  @override
  bool updateShouldNotify(ThemeProvider oldWidget) {
    return theme != oldWidget.theme;  // Rebuild dependents if theme changes
  }
}

// Usage
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ThemeProvider(
      theme: ThemeData.light(),
      child: MaterialApp(home: HomeScreen()),
    );
  }
}

class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Access theme anywhere below ThemeProvider
    final theme = ThemeProvider.of(context).theme;
    return Container(color: theme.primaryColor);
  }
}
```

**How Flutter uses InheritedWidget:**

- `Theme.of(context)` - Uses InheritedWidget
- `MediaQuery.of(context)` - Uses InheritedWidget
- `Navigator.of(context)` - Uses InheritedWidget
- `Scaffold.of(context)` - Uses InheritedWidget

**Note:** Provider is built on top of InheritedWidget but is easier to use.

---

### 35. Explain the concept of "lifting state up"

**Theory:**

When multiple widgets need to share state, move that state to their nearest common ancestor and pass it down.

```dart
// ❌ Before: Each toggle has its own state, can't coordinate
class ToggleA extends StatefulWidget { ... }  // Own state
class ToggleB extends StatefulWidget { ... }  // Own state
// Problem: They can't affect each other

// ✅ After: Parent owns state, passes down
class ToggleGroup extends StatefulWidget {
  @override
  State<ToggleGroup> createState() => _ToggleGroupState();
}

class _ToggleGroupState extends State<ToggleGroup> {
  // State lifted to parent
  bool _optionA = false;
  bool _optionB = false;
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Pass state and callbacks down
        ToggleItem(
          label: 'Option A',
          value: _optionA,
          onChanged: (value) => setState(() => _optionA = value),
        ),
        ToggleItem(
          label: 'Option B',
          value: _optionB,
          onChanged: (value) => setState(() => _optionB = value),
        ),
        Text('A: $_optionA, B: $_optionB'),
      ],
    );
  }
}

// Child is now stateless - receives everything from parent
class ToggleItem extends StatelessWidget {
  final String label;
  final bool value;
  final ValueChanged<bool> onChanged;
  
  const ToggleItem({
    required this.label,
    required this.value,
    required this.onChanged,
  });
  
  @override
  Widget build(BuildContext context) {
    return SwitchListTile(
      title: Text(label),
      value: value,
      onChanged: onChanged,
    );
  }
}
```

**When to lift state:**

- Two sibling widgets need the same state
- A child needs to change parent's state
- State needs to be shared across the app

---

### 36. What is Provider and why is it useful?

**Theory:**

Provider is a state management solution that wraps `InheritedWidget` to make it simpler to use, with additional features like lazy loading and automatic cleanup.

```dart
// 1. Create a ChangeNotifier (the state)
class CounterNotifier extends ChangeNotifier {
  int _count = 0;
  
  int get count => _count;
  
  void increment() {
    _count++;
    notifyListeners();  // Tells widgets to rebuild
  }
}

// 2. Provide it at the top of the tree
void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => CounterNotifier(),
      child: MyApp(),
    ),
  );
}

// 3. Consume it anywhere below
class CounterScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Watch for changes (rebuilds when notified)
    final counter = context.watch<CounterNotifier>();
    
    return Column(
      children: [
        Text('Count: ${counter.count}'),
        ElevatedButton(
          onPressed: () {
            // Read without watching (doesn't rebuild)
            context.read<CounterNotifier>().increment();
          },
          child: Text('Increment'),
        ),
      ],
    );
  }
}
```

**Provider types:**

```dart
// Provider - simple values
Provider(create: (_) => MyService())

// ChangeNotifierProvider - mutable state with notifications
ChangeNotifierProvider(create: (_) => CounterNotifier())

// FutureProvider - async value
FutureProvider(create: (_) async => await fetchData())

// StreamProvider - stream of values
StreamProvider(create: (_) => myStream)

// MultiProvider - multiple providers
MultiProvider(
  providers: [
    ChangeNotifierProvider(create: (_) => AuthNotifier()),
    ChangeNotifierProvider(create: (_) => CartNotifier()),
    Provider(create: (_) => ApiService()),
  ],
  child: MyApp(),
)
```

**read vs watch:**

```dart
// watch - subscribe to changes (use in build)
final counter = context.watch<CounterNotifier>();

// read - one-time read (use in callbacks)
context.read<CounterNotifier>().increment();

// select - watch specific property
final count = context.select<CounterNotifier, int>((c) => c.count);
```

---

### 37. What is the difference between Provider, Riverpod, and BLoC?

**Theory:**

All three are state management solutions, but with different philosophies and tradeoffs.

```
┌─────────────────────────────────────────────────────────────────┐
│                 STATE MANAGEMENT COMPARISON                      │
├─────────────┬─────────────┬─────────────┬───────────────────────┤
│   Aspect    │  Provider   │   Riverpod  │        BLoC           │
├─────────────┼─────────────┼─────────────┼───────────────────────┤
│ Learning    │   Easy      │   Medium    │   Steeper             │
│ Curve       │             │             │                       │
├─────────────┼─────────────┼─────────────┼───────────────────────┤
│ Boilerplate │   Low       │   Low       │   Higher              │
├─────────────┼─────────────┼─────────────┼───────────────────────┤
│ Type Safety │   Runtime   │   Compile   │   Compile time        │
│             │   errors    │   time      │                       │
├─────────────┼─────────────┼─────────────┼───────────────────────┤
│ Testing     │   Harder    │   Easy      │   Easy                │
├─────────────┼─────────────┼─────────────┼───────────────────────┤
│ Pattern     │   Flexible  │   Flexible  │   Strict (Events →    │
│             │             │             │   BLoC → States)      │
├─────────────┼─────────────┼─────────────┼───────────────────────┤
│ Best For    │   Simple    │   Most apps │   Large teams,        │
│             │   apps      │             │   complex apps        │
└─────────────┴─────────────┴─────────────┴───────────────────────┘
```

**Provider:**

```dart
// Simple, uses BuildContext, runtime checks
final counter = context.watch<CounterNotifier>();
```

**Riverpod:**

```dart
// No BuildContext needed, compile-time safe
@riverpod
class Counter extends _$Counter {
  @override
  int build() => 0;
  
  void increment() => state++;
}

// Usage
final count = ref.watch(counterProvider);
ref.read(counterProvider.notifier).increment();
```

**BLoC:**

```dart
// Events → BLoC → States pattern
class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(CounterInitial()) {
    on<Increment>((event, emit) {
      emit(CounterValue(state.count + 1));
    });
  }
}

// Usage
context.read<CounterBloc>().add(Increment());
BlocBuilder<CounterBloc, CounterState>(
  builder: (context, state) => Text('${state.count}'),
)
```

**Recommendation:**

- **Small apps/beginners**: Provider
- **Most apps**: Riverpod
- **Enterprise/large teams**: BLoC

---

## Summary

This guide covers Junior-level (0-2 years) Flutter interview questions:

| Section | Questions | Topics |
|---------|-----------|--------|
| Dart Fundamentals | 1-15 | Variables, null safety, collections, async, OOP |
| Flutter Basics | 16-25 | Widgets, lifecycle, navigation, rendering |
| Widgets & UI | 26-32 | Layout, responsive design, forms |
| State Management | 33-37 | setState, InheritedWidget, Provider basics |

---

**License**: MIT  
**Maintained by**: [debasmitasarkar](https://github.com/debasmitasarkar)