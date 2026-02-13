# Flutter Backend Integration Reference

Consolidated reference for integrating Flutter apps with REST APIs, Firebase, AWS Amplify, and GraphQL backends.

## Table of Contents

- [Backend Selection Guide](#backend-selection-guide)
- [1. REST API with Dio](#1-rest-api-with-dio)
- [2. Firebase (FlutterFire)](#2-firebase-flutterfire)
- [3. AWS Amplify](#3-aws-amplify)
- [4. GraphQL with graphql_flutter](#4-graphql-with-graphql_flutter)

---

## Backend Selection Guide

| Criteria | REST + Dio | Firebase | AWS Amplify | GraphQL |
|---|---|---|---|---|
| **Best for** | Custom servers, existing APIs | Rapid prototyping, real-time apps | Enterprise apps, AWS ecosystem | Complex data graphs, real-time |
| **Auth** | Manual JWT/OAuth | Built-in (email, Google, Apple) | Cognito (email, social, MFA) | Bring your own |
| **Real-time** | Polling or WebSocket (separate) | Firestore snapshots (built-in) | AppSync subscriptions | WebSocket subscriptions |
| **Offline** | Manual caching | Firestore persistence (built-in) | DataStore (built-in) | Cache policies |
| **Setup effort** | Low | Medium (CLI + console) | High (CLI + AWS console) | Medium |
| **Vendor lock-in** | None | Google | Amazon | None |
| **Cost model** | Your server costs | Pay-per-use (generous free tier) | Pay-per-use | Your server costs |

---

## 1. REST API with Dio

### Setup

```yaml
# pubspec.yaml
dependencies:
  dio: ^5.0.0
  json_annotation: ^4.8.0
  freezed_annotation: ^2.4.0
dev_dependencies:
  build_runner: ^2.4.0
  json_serializable: ^6.7.0
  freezed: ^2.4.0
```

### Dio Client with Interceptors

```dart
import 'package:dio/dio.dart';

class ApiClient {
  late final Dio _dio;

  ApiClient({String? baseUrl}) {
    _dio = Dio(BaseOptions(
      baseUrl: baseUrl ?? 'https://api.example.com',
      connectTimeout: const Duration(seconds: 5),
      receiveTimeout: const Duration(seconds: 3),
      headers: {'Content-Type': 'application/json', 'Accept': 'application/json'},
    ));
    _dio.interceptors.addAll([
      AuthInterceptor(_tokenStorage),
      RetryInterceptor(),
      ErrorInterceptor(),
      LogInterceptor(requestBody: true, responseBody: true),
    ]);
  }

  Dio get dio => _dio;
}
```

### Auth Interceptor with Token Refresh

```dart
class AuthInterceptor extends Interceptor {
  final TokenStorage _tokenStorage;
  AuthInterceptor(this._tokenStorage);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final token = await _tokenStorage.getAccessToken();
    if (token != null) options.headers['Authorization'] = 'Bearer $token';
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      try {
        final refreshToken = await _tokenStorage.getRefreshToken();
        final response = await Dio().post(
          'https://api.example.com/auth/refresh',
          data: {'refresh_token': refreshToken},
        );
        final newToken = response.data['access_token'];
        await _tokenStorage.saveAccessToken(newToken);
        final opts = err.requestOptions..headers['Authorization'] = 'Bearer $newToken';
        return handler.resolve(await Dio().fetch(opts));
      } catch (e) {
        await _tokenStorage.clear();
        return handler.reject(err);
      }
    }
    handler.next(err);
  }
}
```

### Error Interceptor and Retry Logic

```dart
class ErrorInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // Map err.type (connectionTimeout, sendTimeout, receiveTimeout, badResponse, cancel)
    // and err.response?.statusCode (400, 401, 403, 404, 500) to typed failures
    handler.next(err);
  }
}

class RetryInterceptor extends Interceptor {
  final int maxRetries;
  RetryInterceptor({this.maxRetries = 3});

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    final shouldRetry = err.type == DioExceptionType.connectionTimeout ||
        err.type == DioExceptionType.receiveTimeout ||
        (err.response?.statusCode ?? 0) >= 500;
    final retryCount = err.requestOptions.extra['retry_count'] as int? ?? 0;

    if (shouldRetry && retryCount < maxRetries) {
      err.requestOptions.extra['retry_count'] = retryCount + 1;
      await Future.delayed(Duration(seconds: retryCount + 1)); // exponential backoff
      try { return handler.resolve(await Dio().fetch(err.requestOptions)); } catch (_) {}
    }
    handler.next(err);
  }
}
```

### Repository Pattern (CRUD)

```dart
class ProductRepositoryImpl implements ProductRepository {
  final ApiClient _apiClient;
  ProductRepositoryImpl(this._apiClient);

  Future<Either<Failure, List<Product>>> getProducts({int page = 1}) async {
    try {
      final response = await _apiClient.dio.get('/products',
          queryParameters: {'page': page, 'limit': 20});
      final products = (response.data['products'] as List)
          .map((json) => ProductModel.fromJson(json)).toList();
      return Right(products);
    } on DioException catch (e) { return Left(_mapException(e)); }
  }

  Future<Either<Failure, Product>> createProduct(Product product) async {
    try {
      final response = await _apiClient.dio.post('/products',
          data: (product as ProductModel).toJson());
      return Right(ProductModel.fromJson(response.data));
    } on DioException catch (e) { return Left(_mapException(e)); }
  }

  Failure _mapException(DioException e) {
    if (e.type == DioExceptionType.connectionTimeout) return NetworkFailure('Timeout');
    if (e.response?.statusCode == 401) return AuthFailure('Unauthorized');
    if (e.response?.statusCode == 404) return NotFoundFailure('Not found');
    return ServerFailure('Server error: ${e.response?.statusCode}');
  }
}
```

---

## 2. Firebase (FlutterFire)

### CLI Setup and Initial Configuration

This is the trickiest part. Follow these steps exactly:

```bash
# 1. Install the FlutterFire CLI
dart pub global activate flutterfire_cli

# 2. Run configure from your Flutter project root
#    Auto-creates firebase_options.dart, downloads
#    google-services.json (Android) and GoogleService-Info.plist (iOS)
flutterfire configure
```

### Dependencies

```yaml
dependencies:
  firebase_core: ^2.24.0
  firebase_auth: ^4.15.0
  cloud_firestore: ^4.13.0
  firebase_storage: ^11.5.0
  firebase_messaging: ^14.7.0
  google_sign_in: ^6.1.5
  sign_in_with_apple: ^5.0.0
```

### Initialize Firebase

```dart
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  runApp(MyApp());
}
```

### Authentication (Email, Google, Apple)

```dart
class FirebaseAuthDataSource {
  final FirebaseAuth _auth;
  final GoogleSignIn _googleSignIn;

  FirebaseAuthDataSource({FirebaseAuth? auth, GoogleSignIn? googleSignIn})
      : _auth = auth ?? FirebaseAuth.instance,
        _googleSignIn = googleSignIn ?? GoogleSignIn();

  Stream<User?> get authStateChanges => _auth.authStateChanges();

  Future<UserCredential> signUpWithEmail({
    required String email, required String password,
  }) => _auth.createUserWithEmailAndPassword(email: email, password: password);

  Future<UserCredential> signInWithEmail({
    required String email, required String password,
  }) => _auth.signInWithEmailAndPassword(email: email, password: password);

  Future<UserCredential> signInWithGoogle() async {
    final googleUser = await _googleSignIn.signIn();
    if (googleUser == null) throw Exception('Google sign in aborted');
    final googleAuth = await googleUser.authentication;
    final credential = GoogleAuthProvider.credential(
      accessToken: googleAuth.accessToken, idToken: googleAuth.idToken,
    );
    return await _auth.signInWithCredential(credential);
  }

  Future<UserCredential> signInWithApple() =>
      _auth.signInWithProvider(AppleAuthProvider());

  Future<void> signOut() => Future.wait([_auth.signOut(), _googleSignIn.signOut()]);
}
```

### Cloud Firestore (CRUD, Queries, Real-time)

```dart
class FirestoreDataSource {
  final FirebaseFirestore _firestore;
  FirestoreDataSource({FirebaseFirestore? firestore})
      : _firestore = firestore ?? FirebaseFirestore.instance;

  // Create
  Future<DocumentReference> addDocument(String collection, Map<String, dynamic> data) =>
      _firestore.collection(collection).add(data);

  // Read
  Future<DocumentSnapshot> getDocument(String collection, String docId) =>
      _firestore.collection(collection).doc(docId).get();

  // Update
  Future<void> updateDocument(String collection, String docId, Map<String, dynamic> data) =>
      _firestore.collection(collection).doc(docId).update(data);

  // Delete
  Future<void> deleteDocument(String collection, String docId) =>
      _firestore.collection(collection).doc(docId).delete();

  // Filtered query
  Future<QuerySnapshot> queryDocuments(String collection, {
    List<QueryFilter>? filters, List<QueryOrder>? orderBy, int? limit,
  }) async {
    Query<Map<String, dynamic>> query = _firestore.collection(collection);
    for (final f in filters ?? []) {
      query = query.where(f.field, isEqualTo: f.isEqualTo,
          isGreaterThan: f.isGreaterThan, isLessThan: f.isLessThan);
    }
    for (final o in orderBy ?? []) {
      query = query.orderBy(o.field, descending: o.descending);
    }
    if (limit != null) query = query.limit(limit);
    return await query.get();
  }

  // Real-time streams
  Stream<QuerySnapshot> streamCollection(String collection) =>
      _firestore.collection(collection).snapshots();

  Stream<DocumentSnapshot> streamDocument(String collection, String docId) =>
      _firestore.collection(collection).doc(docId).snapshots();
}
```

### Cloud Storage

```dart
class FirebaseStorageDataSource {
  final FirebaseStorage _storage;
  FirebaseStorageDataSource({FirebaseStorage? storage})
      : _storage = storage ?? FirebaseStorage.instance;

  Future<String> uploadFile({required File file, required String path}) async {
    final ref = _storage.ref().child(path);
    await ref.putFile(file);
    return await ref.getDownloadURL();
  }

  Future<void> deleteFile(String path) => _storage.ref().child(path).delete();
}
```

### Cloud Messaging (Push Notifications)

```dart
@pragma('vm:entry-point')
Future<void> _bgHandler(RemoteMessage message) async { await Firebase.initializeApp(); }

class FCMDataSource {
  final FirebaseMessaging _messaging = FirebaseMessaging.instance;

  Future<void> initialize() async {
    await _messaging.requestPermission(alert: true, badge: true, sound: true);
    FirebaseMessaging.onBackgroundMessage(_bgHandler);
    FirebaseMessaging.onMessage.listen((msg) { /* show local notification */ });
    FirebaseMessaging.onMessageOpenedApp.listen((msg) { /* navigate */ });
  }

  Future<String?> getToken() => _messaging.getToken();
  Future<void> subscribeToTopic(String topic) => _messaging.subscribeToTopic(topic);
}
```

Platform config: add `com.google.firebase.messaging.default_notification_channel_id` meta-data to Android manifest; add `fetch` and `remote-notification` to `UIBackgroundModes` in iOS `Info.plist`.

### Security Rules

```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isAuth() { return request.auth != null; }
    function isOwner(uid) { return isAuth() && request.auth.uid == uid; }

    match /products/{id} {
      allow read: if true;
      allow create: if isAuth();
      allow update, delete: if isOwner(resource.data.userId);
    }
    match /users/{uid} { allow read, write: if isOwner(uid); }
  }
}
// storage.rules -- allow user-scoped writes, 5MB max, public reads for products
```

---

## 3. AWS Amplify

### CLI Setup and Initial Configuration

This is the most involved setup. Follow in order:

```bash
# 1. Install Amplify CLI
npm install -g @aws-amplify/cli

# 2. Configure AWS credentials (opens browser to create IAM user)
amplify configure

# 3. Initialize in your Flutter project root
amplify init
# Prompts: project name, environment (dev), app type (flutter)
# Creates: amplify/ directory, amplifyconfiguration.dart

# 4. Add authentication (Cognito)
amplify add auth    # Select: Default config with email sign-in

# 5. Add API (REST or GraphQL)
amplify add api     # REST: API Gateway + Lambda | GraphQL: AppSync

# 6. Add storage (S3)
amplify add storage # Select: Content (images, audio, video)

# 7. Push all resources to AWS
amplify push
```

### Dependencies

```yaml
dependencies:
  amplify_flutter: ^1.5.0
  amplify_auth_cognito: ^1.5.0
  amplify_api: ^1.5.0
  amplify_storage_s3: ^1.5.0
```

### Initialize Amplify

```dart
import 'package:amplify_flutter/amplify_flutter.dart';
import 'package:amplify_auth_cognito/amplify_auth_cognito.dart';
import 'package:amplify_api/amplify_api.dart';
import 'package:amplify_storage_s3/amplify_storage_s3.dart';
import 'amplifyconfiguration.dart';

Future<void> _configureAmplify() async {
  try {
    await Amplify.addPlugins([AmplifyAuthCognito(), AmplifyAPI(), AmplifyStorageS3()]);
    await Amplify.configure(amplifyconfig);
  } on AmplifyAlreadyConfiguredException {
    safePrint('Amplify already configured');
  }
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await _configureAmplify();
  runApp(MyApp());
}
```

### Cognito Auth

```dart
class AmplifyAuthDataSource {
  Future<SignUpResult> signUp({
    required String username, required String password, required String email,
  }) => Amplify.Auth.signUp(username: username, password: password,
    options: SignUpOptions(userAttributes: {AuthUserAttributeKey.email: email}));

  Future<SignUpResult> confirmSignUp({
    required String username, required String confirmationCode,
  }) => Amplify.Auth.confirmSignUp(username: username, confirmationCode: confirmationCode);

  Future<SignInResult> signIn({required String username, required String password}) =>
      Amplify.Auth.signIn(username: username, password: password);

  Future<SignInResult> signInWithWebUI({AuthProvider? provider}) =>
      Amplify.Auth.signInWithWebUI(provider: provider);

  Future<void> signOut() => Amplify.Auth.signOut();
  Future<AuthUser> getCurrentUser() => Amplify.Auth.getCurrentUser();
}
```

### AppSync GraphQL

Define a schema, then `amplify push` generates Dart models automatically:

```graphql
# amplify/backend/api/myapi/schema.graphql
type Product @model @auth(rules: [{ allow: public }]) {
  id: ID!
  name: String!
  description: String
  price: Float!
  category: String
}
```

```dart
class AppSyncDataSource {
  Future<List<Product>> queryProducts() async {
    final request = ModelQueries.list(Product.classType);
    final response = await Amplify.API.query(request: request).response;
    return response.data!.items.whereType<Product>().toList();
  }

  Future<Product> createProduct(Product product) async {
    final response = await Amplify.API
        .mutate(request: ModelMutations.create(product)).response;
    return response.data!;
  }

  Future<void> deleteProduct(Product product) async {
    await Amplify.API.mutate(request: ModelMutations.delete(product)).response;
  }

  Stream<GraphQLResponse<Product>> onCreateProduct() =>
      Amplify.API.subscribe(
        request: ModelSubscriptions.onCreate(Product.classType),
      );
}
```

### S3 Storage

```dart
class AmplifyStorageDataSource {
  Future<String> uploadFile({required String key, required File localFile}) async {
    await Amplify.Storage.uploadFile(
      localFile: AWSFile.fromPath(localFile.path), key: key,
    ).result;
    return (await Amplify.Storage.getUrl(key: key).result).url.toString();
  }

  Future<void> deleteFile(String key) async {
    await Amplify.Storage.remove(key: key).result;
  }
}
```

### Lambda Integration

Lambda functions are created automatically when you `amplify add api` with REST. The generated function handles CRUD against DynamoDB. Custom logic goes in `amplify/backend/function/<name>/src/index.js`.

---

## 4. GraphQL with graphql_flutter

### Setup

```yaml
dependencies:
  graphql_flutter: ^5.1.0
```

### Client Configuration

```dart
import 'package:graphql_flutter/graphql_flutter.dart';

class GraphQLConfig {
  static final _httpLink = HttpLink('https://api.example.com/graphql');
  static final _authLink = AuthLink(getToken: () async {
    final token = await getAuthToken();
    return token != null ? 'Bearer $token' : null;
  });
  static final _wsLink = WebSocketLink('wss://api.example.com/graphql',
    config: SocketClientConfig(autoReconnect: true));

  // Route subscriptions over WebSocket, everything else over HTTP
  static final _link = Link.split(
    (request) => request.isSubscription, _wsLink, _authLink.concat(_httpLink));

  static ValueNotifier<GraphQLClient> initializeClient() => ValueNotifier(
    GraphQLClient(link: _link, cache: GraphQLCache(store: InMemoryStore()),
      defaultPolicies: DefaultPolicies(
        query: Policies(fetch: FetchPolicy.cacheAndNetwork),
        mutate: Policies(fetch: FetchPolicy.networkOnly))));
}

// In main.dart:
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initHiveForFlutter();
  runApp(GraphQLProvider(client: GraphQLConfig.initializeClient(), child: MyApp()));
}
```

### Queries with Variables

```dart
const getProductsQuery = r'''
  query GetProducts {
    products { id name description price imageUrl }
  }
''';

const getProductQuery = r'''
  query GetProduct($id: ID!) {
    product(id: $id) { id name description price imageUrl category { id name } }
  }
''';

class ProductsRepository {
  final GraphQLClient client;
  ProductsRepository(this.client);

  Future<Either<Failure, List<Product>>> getProducts() async {
    final result = await client.query(QueryOptions(
      document: gql(getProductsQuery), fetchPolicy: FetchPolicy.cacheAndNetwork,
    ));
    if (result.hasException) return Left(ServerFailure(result.exception.toString()));
    return Right((result.data!['products'] as List)
        .map((json) => Product.fromJson(json)).toList());
  }

  Future<Either<Failure, Product>> getProduct(String id) async {
    final result = await client.query(QueryOptions(
      document: gql(getProductQuery), variables: {'id': id},
    ));
    if (result.hasException) return Left(ServerFailure(result.exception.toString()));
    return Right(Product.fromJson(result.data!['product']));
  }
}
```

### Mutations with Optimistic Updates

```dart
const createProductMutation = r'''
  mutation CreateProduct($input: CreateProductInput!) {
    createProduct(input: $input) { id name description price imageUrl }
  }
''';

// Imperative mutation in repository
Future<Either<Failure, Product>> createProduct({
  required String name, required double price,
}) async {
  final result = await client.mutate(MutationOptions(
    document: gql(createProductMutation),
    variables: {'input': {'name': name, 'price': price}},
  ));
  if (result.hasException) return Left(ServerFailure(result.exception.toString()));
  return Right(Product.fromJson(result.data!['createProduct']));
}

// Widget mutation with optimistic UI update
Mutation(
  options: MutationOptions(
    document: gql(createProductMutation),
    optimisticResult: {
      'createProduct': {'__typename': 'Product', 'id': 'temp-id',
        'name': nameController.text, 'price': double.parse(priceController.text)},
    },
    update: (cache, result) {
      if (result?.data == null) return;
      final existing = cache.readQuery(
        Request(operation: Operation(document: gql(getProductsQuery))));
      if (existing != null) {
        cache.writeQuery(
          Request(operation: Operation(document: gql(getProductsQuery))),
          data: {'products': [...List.from(existing['products']),
            result!.data!['createProduct']]});
      }
    },
  ),
  builder: (runMutation, result) { /* build UI, call runMutation({...}) */ },
);
```

### Subscriptions

```dart
const onProductCreatedSub = r'''
  subscription OnProductCreated {
    productCreated { id name description price imageUrl }
  }
''';

// Stream-based (in repository)
Stream<Product> watchProductCreated() => client
    .subscribe(SubscriptionOptions(document: gql(onProductCreatedSub)))
    .map((result) {
      if (result.hasException) throw result.exception!;
      return Product.fromJson(result.data!['productCreated']);
    });

// Widget-based
Subscription(
  options: SubscriptionOptions(document: gql(onProductCreatedSub)),
  builder: (result) {
    if (result.isLoading) return Text('Listening...');
    if (result.hasException) return Text('Error: ${result.exception}');
    if (result.data != null) {
      final product = Product.fromJson(result.data!['productCreated']);
      return ListTile(title: Text('New: ${product.name}'));
    }
    return Text('Waiting for updates...');
  },
);
```

### Caching with Fetch Policies

| Policy | Behavior |
|---|---|
| `cacheFirst` | Use cache; fallback to network (default) |
| `cacheAndNetwork` | Show cache immediately, then update from network |
| `networkOnly` | Always fetch from network, update cache |
| `cacheOnly` | Offline mode -- never hit network |
| `noCache` | Fetch from network, do not store result |

Manual cache operations:

```dart
// Read
final data = client.readQuery(
  Request(operation: Operation(document: gql(getProductsQuery))));

// Write
client.writeQuery(
  Request(operation: Operation(document: gql(getProductsQuery))),
  data: {'products': [...]});

// Update a fragment
client.writeFragment(
  Fragment(document: gql(r'fragment ProductPrice on Product { price }')),
  idFields: {'__typename': 'Product', 'id': '1'},
  data: {'price': 19.99});
```
