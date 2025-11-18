---
name: backend-integration
description: Master HTTP clients (http, Dio, Chopper), REST APIs, GraphQL, WebSockets, JWT authentication, Firebase integration, error handling, serialization and real-time data synchronization for robust Flutter backend integration.
---

# Backend Integration & API Communication

## Quick Start

Build production-ready API clients with Dio:

```dart
import 'package:dio/dio.dart';

final dio = Dio(BaseOptions(
  baseUrl: 'https://api.example.com',
  connectTimeout: const Duration(seconds: 10),
  receiveTimeout: const Duration(seconds: 10),
));

// Add interceptors for logging and error handling
dio.interceptors.add(
  InterceptorsWrapper(
    onRequest: (options, handler) {
      print('REQUEST[${options.method}] => PATH: ${options.path}');
      return handler.next(options);
    },
    onError: (err, handler) {
      print('ERROR[${err.response?.statusCode}] => PATH: ${err.requestOptions.path}');
      return handler.next(err);
    },
  ),
);

// Make requests
final response = await dio.get('/users');
```

## HTTP Clients Comparison

### http Package (Simplicity-First)
**Best for**: Simple MVPs, learning, minimal dependencies
```dart
import 'package:http/http.dart' as http;

final response = await http.get(Uri.parse('https://api.example.com/users'));
if (response.statusCode == 200) {
  print(response.body);
}
```

### Dio (Enterprise-Grade)
**Best for**: Production apps, complex scenarios, enterprise features
- ✅ Interceptors for logging, auth, error handling
- ✅ Request/response transformation
- ✅ Retry logic and timeout management
- ✅ File upload/download
- ✅ Request cancellation
- ✅ Mock adapter for testing

### Chopper (Type-Safe REST)
**Best for**: Type safety, code generation, REST clarity
```dart
@ChopperApi(baseUrl: '/users')
abstract class UserService extends ChopperService {
  @Get()
  Future<Response<List<UserModel>>> getUsers();

  @Post()
  Future<Response<UserModel>> createUser(@Body() UserModel user);

  static UserService create([ChopperClient? client]) =>
    _$UserService(client);
}
```

## REST API Patterns

### Repository Pattern (Clean Architecture)
```dart
abstract class UserRepository {
  Future<List<User>> getUsers();
  Future<User> createUser(User user);
  Future<void> deleteUser(String id);
}

class UserRepositoryImpl implements UserRepository {
  final Dio dio;

  const UserRepositoryImpl(this.dio);

  @override
  Future<List<User>> getUsers() async {
    try {
      final response = await dio.get('/users');
      final users = (response.data as List)
          .map((u) => User.fromJson(u))
          .toList();
      return users;
    } on DioException catch (e) {
      throw ApiException(e.message ?? 'Unknown error');
    }
  }
}
```

### Pagination
```dart
class PaginatedUsers {
  final List<User> data;
  final int total;
  final int page;
  final int pageSize;

  bool get hasNextPage => (page * pageSize) < total;

  PaginatedUsers({
    required this.data,
    required this.total,
    required this.page,
    required this.pageSize,
  });
}

// Usage
Future<PaginatedUsers> getUsers({int page = 1, int pageSize = 20}) async {
  final response = await dio.get(
    '/users',
    queryParameters: {'page': page, 'pageSize': pageSize},
  );
  return PaginatedUsers.fromJson(response.data);
}
```

## GraphQL Integration

### Apollo Setup
```dart
import 'package:graphql/client.dart';

final httpLink = HttpLink('https://api.example.com/graphql');

final authLink = AuthLink(
  getToken: () async => 'Bearer YOUR_JWT_TOKEN',
);

final link = authLink.concat(httpLink);

final graphQLClient = GraphQLClient(
  cache: GraphQLCache(),
  link: link,
);

// Query
final options = QueryOptions(
  document: gql(getUserQuery),
  variables: {'id': '123'},
);

final result = await graphQLClient.query(options);
```

## WebSocket Real-Time Communication

```dart
import 'package:web_socket_channel/web_socket_channel.dart';
import 'package:web_socket_channel/status.dart' as status;

class ChatService {
  late final WebSocketChannel _channel;
  final StreamController<Message> _messageController =
    StreamController<Message>.broadcast();

  Future<void> connect() async {
    _channel = WebSocketChannel.connect(
      Uri.parse('wss://api.example.com/chat'),
    );

    _channel.stream.listen(
      (data) {
        final message = Message.fromJson(jsonDecode(data));
        _messageController.add(message);
      },
      onError: (error) => print('WebSocket error: $error'),
      onDone: () => print('WebSocket closed'),
    );
  }

  void sendMessage(Message message) {
    _channel.sink.add(jsonEncode(message.toJson()));
  }

  Stream<Message> get messages => _messageController.stream;

  void disconnect() {
    _channel.sink.close(status.goingAway);
    _messageController.close();
  }
}
```

## JWT Authentication

```dart
class AuthService {
  final Dio dio;
  String? _accessToken;
  String? _refreshToken;

  Future<void> login(String email, String password) async {
    final response = await dio.post('/auth/login', data: {
      'email': email,
      'password': password,
    });

    _accessToken = response.data['accessToken'];
    _refreshToken = response.data['refreshToken'];

    // Store securely
    await _secureStorage.write(key: 'accessToken', value: _accessToken);
    await _secureStorage.write(key: 'refreshToken', value: _refreshToken);
  }

  Future<String> getValidToken() async {
    if (_isTokenExpired(_accessToken)) {
      await _refreshAccessToken();
    }
    return _accessToken!;
  }

  Future<void> _refreshAccessToken() async {
    try {
      final response = await dio.post('/auth/refresh', data: {
        'refreshToken': _refreshToken,
      });

      _accessToken = response.data['accessToken'];
      await _secureStorage.write(key: 'accessToken', value: _accessToken);
    } catch (e) {
      // Refresh failed, redirect to login
      await logout();
      rethrow;
    }
  }

  Future<void> logout() async {
    _accessToken = null;
    _refreshToken = null;
    await _secureStorage.deleteAll();
  }

  bool _isTokenExpired(String? token) {
    if (token == null) return true;
    final decoded = jwt.decode(token);
    final expiration = DateTime.fromMillisecondsSinceEpoch(
      decoded['exp'] * 1000,
    );
    return DateTime.now().isAfter(expiration);
  }
}

// Interceptor for auto-adding token
dio.interceptors.add(
  InterceptorsWrapper(
    onRequest: (options, handler) async {
      final token = await authService.getValidToken();
      options.headers['Authorization'] = 'Bearer $token';
      return handler.next(options);
    },
  ),
);
```

## Error Handling & Retries

```dart
class ApiException implements Exception {
  final String message;
  final int? statusCode;
  final dynamic originalError;

  ApiException({
    required this.message,
    this.statusCode,
    this.originalError,
  });

  @override
  String toString() => 'ApiException: $message (Status: $statusCode)';
}

Future<T> withRetry<T>(
  Future<T> Function() operation, {
  int maxRetries = 3,
  Duration initialDelay = const Duration(milliseconds: 100),
}) async {
  int retries = 0;
  Duration delay = initialDelay;

  while (true) {
    try {
      return await operation();
    } catch (e) {
      if (retries >= maxRetries || !_isRetryable(e)) {
        rethrow;
      }

      await Future.delayed(delay);
      delay *= 2; // Exponential backoff
      retries++;
    }
  }
}

bool _isRetryable(dynamic error) {
  if (error is! DioException) return false;

  // Retry on timeout or connection errors
  if (error.type == DioExceptionType.connectionTimeout ||
      error.type == DioExceptionType.receiveTimeout) {
    return true;
  }

  // Retry on 5xx status codes
  if (error.response?.statusCode != null &&
      error.response!.statusCode! >= 500) {
    return true;
  }

  return false;
}

// Usage
final users = await withRetry(
  () => userRepository.getUsers(),
  maxRetries: 3,
);
```

## Firebase Integration

```dart
import 'package:firebase_core/firebase_core.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';

// Initialize Firebase
await Firebase.initializeApp();

final firestore = FirebaseFirestore.instance;
final auth = FirebaseAuth.instance;

// CRUD with Firestore
class UserRepository {
  Future<User> getUser(String id) async {
    final doc = await firestore.collection('users').doc(id).get();
    return User.fromJson(doc.data()!);
  }

  Future<void> createUser(User user) async {
    await firestore.collection('users').doc(user.id).set(
      user.toJson(),
    );
  }

  Future<void> updateUser(User user) async {
    await firestore.collection('users').doc(user.id).update(
      user.toJson(),
    );
  }

  Future<void> deleteUser(String id) async {
    await firestore.collection('users').doc(id).delete();
  }

  // Real-time listening
  Stream<List<User>> watchUsers() {
    return firestore
        .collection('users')
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => User.fromJson(doc.data()))
            .toList());
  }
}
```

## Serialization (json_serializable + Freezed)

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required String email,
    @JsonKey(name: 'created_at')
    required DateTime createdAt,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) =>
    _$UserFromJson(json);
}

// Usage
final user = User(
  id: '123',
  name: 'John Doe',
  email: 'john@example.com',
  createdAt: DateTime.now(),
);

final json = user.toJson();
final reconstructed = User.fromJson(json);
```

## Best Practices

1. **Use Dio** for production apps with interceptors
2. **Implement Repository Pattern** for clean architecture
3. **Handle errors explicitly** with custom exception types
4. **Add request/response logging** for debugging
5. **Implement JWT token refresh** automatically
6. **Use code generation** (Freezed, json_serializable) for serialization
7. **Test API clients** with Mockito or MockClient
8. **Implement timeout handling** for slow networks
9. **Cache responses intelligently** to reduce API calls
10. **Monitor API performance** with analytics

---

**Master backend integration and build reliable, responsive Flutter apps!**
