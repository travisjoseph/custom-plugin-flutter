---
name: database-storage
description: Master local storage solutions (SharedPreferences, Hive, SQLite, ObjectBox), Firebase databases, schema design, CRUD operations, query optimization, encryption, offline-first patterns and data synchronization.
---

# Database & Storage Solutions

## Quick Start

Use Hive for fast local storage:

```dart
import 'package:hive/hive.dart';

// Initialize
await Hive.initFlutter();
Hive.registerAdapter(UserAdapter());

// Open box
final box = await Hive.openBox<User>('users');

// CRUD
await box.put('user_1', User(name: 'John', email: 'john@example.com'));
final user = box.get('user_1');
await user.delete();

// Listening to changes
box.listenable().addListener(() {
  print('Box changed!');
});
```

## Storage Solutions Comparison

### SharedPreferences (Simple Key-Value)
**Best for**: App settings, user preferences, small data (<1MB)

```dart
final prefs = await SharedPreferences.getInstance();

// Store
await prefs.setString('user_theme', 'dark');
await prefs.setInt('login_count', 5);
await prefs.setBool('notifications_enabled', true);

// Retrieve
final theme = prefs.getString('user_theme');
final count = prefs.getInt('login_count');
final enabled = prefs.getBool('notifications_enabled');
```

### Hive (Flexible Local Storage)
**Best for**: Most projects, flexible schema-less data, fast performance

```dart
import 'package:hive/hive.dart';
import 'package:hive_flutter/hive_flutter.dart';

part 'user.g.dart';

@HiveType(typeId: 0)
class User {
  @HiveField(0)
  final String id;

  @HiveField(1)
  final String name;

  @HiveField(2)
  final String email;

  User({required this.id, required this.name, required this.email});
}

// Usage
final box = await Hive.openBox<User>('users');
await box.put('u1', User(id: '1', name: 'John', email: 'john@example.com'));

// Query
final users = box.values.where((u) => u.name.contains('John')).toList();

// Update
final user = box.get('u1');
await user?.save(); // After modifications
```

### SQLite (Relational Database)
**Best for**: Complex relational data, large datasets, complex queries

```dart
import 'package:sqflite/sqflite.dart';

class UserDatabase {
  late Database db;

  Future<void> init() async {
    db = await openDatabase(
      'app.db',
      version: 1,
      onCreate: (db, version) async {
        await db.execute('''
          CREATE TABLE users (
            id TEXT PRIMARY KEY,
            name TEXT NOT NULL,
            email TEXT NOT NULL,
            created_at TEXT NOT NULL
          )
        ''');
      },
    );
  }

  // CRUD
  Future<void> insertUser(User user) async {
    await db.insert('users', {
      'id': user.id,
      'name': user.name,
      'email': user.email,
      'created_at': user.createdAt.toIso8601String(),
    });
  }

  Future<User?> getUser(String id) async {
    final result = await db.query(
      'users',
      where: 'id = ?',
      whereArgs: [id],
    );

    if (result.isEmpty) return null;
    return User.fromJson(result.first);
  }

  Future<List<User>> getAllUsers() async {
    final result = await db.query('users');
    return result.map((json) => User.fromJson(json)).toList();
  }

  Future<void> updateUser(User user) async {
    await db.update(
      'users',
      user.toJson(),
      where: 'id = ?',
      whereArgs: [user.id],
    );
  }

  Future<void> deleteUser(String id) async {
    await db.delete('users', where: 'id = ?', whereArgs: [id]);
  }
}
```

### ObjectBox (High-Performance)
**Best for**: High-performance apps, real-time features, complex object graphs

```dart
import 'package:objectbox/objectbox.dart';

@Entity()
class User {
  @Id()
  int id = 0;

  late String name;
  late String email;

  @Backlink()
  final posts = <Post>[];
}

@Entity()
class Post {
  @Id()
  int id = 0;

  late String title;
  final user = ToOne<User>();
}

// Usage
final store = await openStore();
final userBox = store.box<User>();

// Create
final user = User()..name = 'John'..email = 'john@example.com';
userBox.put(user);

// Query
final users = userBox.query(User_.name.contains('John')).build().find();

// Listening
userBox.query().watch().listen((query) {
  print('Users changed!');
});
```

## Firebase Firestore (Cloud Database)

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

class FirestoreUserRepository {
  final firestore = FirebaseFirestore.instance;

  // Create
  Future<void> createUser(User user) async {
    await firestore.collection('users').doc(user.id).set(
      user.toJson(),
    );
  }

  // Read
  Future<User?> getUser(String id) async {
    final doc = await firestore.collection('users').doc(id).get();
    if (!doc.exists) return null;
    return User.fromJson(doc.data()!);
  }

  // Update
  Future<void> updateUser(String id, Map<String, dynamic> data) async {
    await firestore.collection('users').doc(id).update(data);
  }

  // Delete
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

  // Paginated queries
  Future<(List<User>, DocumentSnapshot?)> getUsers({
    int pageSize = 10,
    DocumentSnapshot? startAfter,
  }) async {
    var query = firestore
        .collection('users')
        .orderBy('created_at')
        .limit(pageSize + 1);

    if (startAfter != null) {
      query = query.startAfterDocument(startAfter);
    }

    final snapshot = await query.get();
    final users = snapshot.docs
        .take(pageSize)
        .map((doc) => User.fromJson(doc.data()))
        .toList();

    final nextStartAfter = snapshot.docs.length > pageSize
        ? snapshot.docs[pageSize - 1]
        : null;

    return (users, nextStartAfter);
  }
}
```

## Offline-First Architecture

```dart
class OfflineFirstUserRepository {
  final localBox = await Hive.openBox<User>('users');
  final firestore = FirebaseFirestore.instance;
  final _syncQueue = <SyncOperation>[];

  // Get data (prefer local, sync in background)
  Future<List<User>> getUsers() async {
    final local = localBox.values.toList();

    // Sync in background
    _syncUsersInBackground();

    return local;
  }

  // Create user (optimistic update)
  Future<void> createUser(User user) async {
    // Update local immediately
    await localBox.put(user.id, user);

    // Queue for sync
    _syncQueue.add(SyncOperation(
      type: OperationType.create,
      entity: user,
    ));

    // Sync to server
    _syncToServer();
  }

  // Sync with conflict resolution
  Future<void> _syncToServer() async {
    for (final operation in _syncQueue) {
      try {
        switch (operation.type) {
          case OperationType.create:
            await firestore
                .collection('users')
                .doc(operation.entity.id)
                .set(operation.entity.toJson());
            break;
          case OperationType.update:
            await firestore
                .collection('users')
                .doc(operation.entity.id)
                .update(operation.entity.toJson());
            break;
          case OperationType.delete:
            await firestore
                .collection('users')
                .doc(operation.entity.id)
                .delete();
            break;
        }

        _syncQueue.removeWhere(
          (op) => op.entity.id == operation.entity.id,
        );
      } on FirebaseException catch (e) {
        print('Sync failed: ${e.message}');
        // Queue will retry on next sync
      }
    }
  }

  Future<void> _syncUsersInBackground() async {
    try {
      final remote = await firestore.collection('users').get();
      final remoteUsers = remote.docs
          .map((doc) => User.fromJson(doc.data()))
          .toList();

      // Merge: newer version wins
      for (final remoteUser in remoteUsers) {
        final local = localBox.get(remoteUser.id);

        if (local == null ||
            remoteUser.updatedAt.isAfter(local.updatedAt)) {
          await localBox.put(remoteUser.id, remoteUser);
        }
      }
    } catch (e) {
      print('Background sync failed: $e');
    }
  }
}

enum OperationType { create, update, delete }

class SyncOperation {
  final OperationType type;
  final User entity;

  SyncOperation({required this.type, required this.entity});
}
```

## Encryption at Rest

```dart
import 'package:encrypt/encrypt.dart' as encrypt;

class SecureStorage {
  late final encrypt.Key _key;
  late final encrypt.IV _iv;

  Future<void> init() async {
    // Generate or retrieve key
    _key = encrypt.Key.fromSecureRandom(32);
    _iv = encrypt.IV.fromSecureRandom(16);
  }

  String encryptData(String data) {
    final encrypter = encrypt.Encrypter(encrypt.AES(_key));
    final encrypted = encrypter.encrypt(data, iv: _iv);
    return encrypted.base64;
  }

  String decryptData(String encryptedData) {
    final encrypter = encrypt.Encrypter(encrypt.AES(_key));
    final decrypted = encrypter.decrypt64(encryptedData, iv: _iv);
    return decrypted;
  }
}
```

## Best Practices

1. **Start with Hive** for most new projects
2. **Use SQLite** for complex relational data
3. **Implement Firestore** for cloud-sync features
4. **Always encrypt sensitive data** (passwords, tokens)
5. **Use offline-first patterns** for reliability
6. **Implement proper migrations** when schema changes
7. **Test database code thoroughly** with unit tests
8. **Monitor database performance** and optimize queries
9. **Backup critical data** regularly
10. **Implement conflict resolution** for offline sync

---

**Master data storage and build robust, offline-first Flutter apps!**
