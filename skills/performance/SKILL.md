---
name: performance-optimization
description: Optimize Flutter app performance - build optimization, rendering performance, memory management, image optimization, network efficiency, battery optimization and APK/IPA size reduction for lightning-fast apps.
---

# Performance Optimization Masterclass

## Quick Start - Top 10 Optimizations

```dart
// 1. Use const constructors
const SizedBox(height: 16);  // ✅ Better
SizedBox(height: 16);        // ❌ Rebuilds unnecessarily

// 2. Use ListView.builder instead of ListView
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) => ItemWidget(items[index]),
)

// 3. Optimize images
Image.asset(
  'assets/image.png',
  cacheHeight: 500,
  cacheWidth: 500,
)

// 4. Use Repaint Boundaries for expensive widgets
RepaintBoundary(
  child: ExpensiveWidget(),
)

// 5. Lazy load heavy content
FutureBuilder(
  future: loadExpensiveData(),
  builder: (context, snapshot) {
    // Only build when data is ready
  },
)

// 6. Use FractionallySizedBox for responsive layouts
FractionallySizedBox(
  widthFactor: 0.8,
  child: MyWidget(),
)

// 7. Profile with DevTools
flutter pub global activate devtools
devtools

// 8. Use const Widget types
class MyWidget extends StatelessWidget {
  const MyWidget();  // ✅ Const

  @override
  Widget build(context) => Container();
}

// 9. Batch state updates
setState(() {
  _name = 'John';
  _email = 'john@example.com';
  _age = 25;
});

// 10. Use Isolates for heavy computation
Future<int> computeExpensive() {
  return compute(heavyComputation, data);
}
```

## Build Performance

```dart
// Run production build
flutter build apk --release
flutter build ios --release

// Profile build time
flutter build apk --profile --analyze-size

// Check app size
flutter build apk --release --target-platform=android-arm64
```

## Rendering Performance - 60+ FPS

```dart
// Identify jank with DevTools Timeline
// Target: <16.67ms per frame (60 FPS)
// Warning: 16-33ms (30-60 FPS)
// Danger: >33ms (<30 FPS)

// Bad: Rebuilds entire tree
class Counter extends StatefulWidget {
  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;

  @override
  Widget build(context) {
    return Column(
      children: [
        Text('Count: $_count'),  // Rebuilds with counter
        ExpensiveWidget(),        // Rebuilds unnecessarily!
      ],
    );
  }
}

// Good: Extracts widgets
class Counter extends StatefulWidget {
  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;

  @override
  Widget build(context) {
    return Column(
      children: [
        CounterDisplay(count: _count),
        const ExpensiveWidget(),  // Const - no rebuild
      ],
    );
  }
}

class CounterDisplay extends StatelessWidget {
  final int count;
  const CounterDisplay({required this.count});

  @override
  Widget build(context) => Text('Count: $count');
}
```

## Memory Management

```dart
// Profile memory with DevTools
// Monitor: Heap size, memory allocation, garbage collection

// Memory leak: Event listener not disposed
class MyWidget extends StatefulWidget {
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  late StreamSubscription _subscription;

  @override
  void initState() {
    super.initState();
    _subscription = myStream.listen((data) {
      setState(() => _data = data);
    });
  }

  @override
  void dispose() {
    _subscription.cancel();  // ✅ Clean up!
    super.dispose();
  }

  @override
  Widget build(context) => Text(_data);
}

// Avoid keeping references to context
class MyService {
  late BuildContext _context;  // ❌ Memory leak!

  void init(BuildContext context) {
    _context = context;  // Don't keep this!
  }
}

// Good: Use mounted to check if widget still exists
void updateUI() {
  if (mounted) {
    setState(() {});
  }
}
```

## Image Optimization

```dart
import 'package:cached_network_image/cached_network_image.dart';

// Network images with caching
CachedNetworkImage(
  imageUrl: 'https://example.com/image.jpg',
  placeholder: (context, url) => CircularProgressIndicator(),
  errorWidget: (context, url, error) => Icon(Icons.error),
  fit: BoxFit.cover,
  memCacheHeight: 500,
  memCacheWidth: 500,
)

// Lazy load images in lists
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    return CachedNetworkImage(
      imageUrl: items[index].imageUrl,
      placeholder: (context, url) => Container(color: Colors.grey),
    );
  },
)

// Use WebP or HEIC for smaller file sizes
// Asset image with appropriate quality
Image.asset(
  'assets/logo.png',
  fit: BoxFit.contain,
)
```

## Network Optimization

```dart
// Connection pooling with Dio
final dio = Dio(BaseOptions(
  baseUrl: 'https://api.example.com',
  connectTimeout: const Duration(seconds: 10),
  receiveTimeout: const Duration(seconds: 10),
  extra: {'maxRedirects': 5},
));

// Response caching
dio.interceptors.add(
  CacheInterceptor(
    options: CacheOptions(
      store: MemoryCacheStore(),  // Or PersistentCacheStore
      policy: CachePolicy.request,
      maxStale: const Duration(days: 7),
    ),
  ),
);

// Request batching
Future<(UserList, PostList)> fetchUserAndPosts(String userId) async {
  final futures = await Future.wait([
    dio.get('/users/$userId'),
    dio.get('/posts?userId=$userId'),
  ]);

  return (
    UserList.fromJson(futures[0].data),
    PostList.fromJson(futures[1].data),
  );
}

// Compression
dio.interceptors.add(
  InterceptorsWrapper(
    onRequest: (options, handler) {
      options.headers['Content-Encoding'] = 'gzip';
      return handler.next(options);
    },
  ),
);
```

## CPU & Battery Efficiency

```dart
import 'dart:isolate';

// Heavy computation in Isolate (doesn't block UI)
Future<int> fibonacci(int n) {
  return compute(_fibonacci, n);
}

int _fibonacci(int n) {
  if (n <= 1) return n;
  return _fibonacci(n - 1) + _fibonacci(n - 2);
}

// Usage
final result = await fibonacci(40);  // Non-blocking

// Battery monitoring
import 'package:battery_plus/battery_plus.dart';

class BatteryMonitor {
  final battery = Battery();

  Future<void> startMonitoring() async {
    battery.onBatteryStateChanged.listen((state) {
      if (state == BatteryState.discharging) {
        _disablePowerHungryFeatures();
      }
    });
  }

  void _disablePowerHungryFeatures() {
    // Disable location tracking, reduce animation frame rate, etc.
  }
}
```

## APK/IPA Size Reduction

```yaml
# pubspec.yaml - Avoid unnecessary packages
dependencies:
  flutter:
    sdk: flutter
  # Only include essential packages

# android/app/build.gradle
android {
  buildTypes {
    release {
      minifyEnabled true
      shrinkResources true
      proguardFiles getDefaultProguardFile('proguard-android.txt')
    }
  }
}

# split APKs by ABI (much smaller)
flutter build apk --split-per-abi

# Results: arm64-v8a (~30MB), armeabi-v7a (~25MB)
```

## DevTools Profiling

```bash
# Start your app
flutter run

# Open DevTools
flutter pub global activate devtools
flutter pub global run devtools

# Or directly
dart devtools

# Key tabs:
# 1. Performance - Frame timeline, jank detection
# 2. Memory - Heap snapshot, allocation tracking
# 3. Network - API calls, response times
# 4. Logging - Debug output, errors
```

## Performance Targets

```
Metric                  | Target      | Optimization
─────────────────────────┼─────────────┼──────────────────
App Startup            | <2 seconds  | Lazy load, AOT, tree-shake
Frame Rate             | 60+ FPS     | const, Repaint, efficient rebuilds
Memory Avg             | <100 MB     | Dispose listeners, cleanup
Memory Peak            | <150 MB     | Stream unbuild, lazy load
APK Size               | <50 MB      | Minify, R8, split APKs
IPA Size               | <50 MB      | Bitcode, asset optimization
Network Latency p95    | <500ms      | Cache, batch, compression
Battery Drain          | 5-7%/hour   | Isolates, debounce, optimize
Build Time             | <2 min      | Incremental, tree-shake
Release Build Time     | <5 min      | Parallel build, cache
```

## Best Practices Checklist

- ✅ Use const constructors everywhere
- ✅ Profile before optimizing
- ✅ Lazy load non-critical content
- ✅ Dispose listeners and streams
- ✅ Cache images and API responses
- ✅ Use Repaint Boundary for expensive widgets
- ✅ Implement proper error boundaries
- ✅ Test on low-end devices regularly
- ✅ Monitor production with Crashlytics
- ✅ Update dependencies regularly

---

**Build lightning-fast Flutter apps that users love!**
