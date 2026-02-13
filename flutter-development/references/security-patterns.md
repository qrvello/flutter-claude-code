# Flutter Security Patterns

Production-ready security patterns to protect user data and prevent vulnerabilities.

## Table of Contents

- [Secure Storage](#secure-storage)
- [API Security](#api-security)
- [Authentication Security](#authentication-security)
- [Data Encryption](#data-encryption)
- [Input Validation](#input-validation)
- [Platform Security](#platform-security)
- [Code Obfuscation](#code-obfuscation)
- [Security Checklist](#security-checklist)

## Secure Storage

### Never Store Sensitive Data in Plain Text

```dart
// NEVER: await prefs.setString('password', 'user_password');

// Use flutter_secure_storage
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

final storage = FlutterSecureStorage();
await storage.write(key: 'auth_token', value: token);
final token = await storage.read(key: 'auth_token');
await storage.delete(key: 'auth_token');
```

### Secure Storage Configuration

```dart
const secureStorage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
  iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
);
```

## API Security

### Certificate Pinning

```dart
class SecureApiClient {
  final Dio dio;
  SecureApiClient() : dio = Dio() {
    dio.httpClientAdapter = IOHttpClientAdapter(
      createHttpClient: () {
        final client = HttpClient();
        client.badCertificateCallback = (cert, host, port) {
          final certSha256 = sha256.convert(cert.der).toString();
          const expectedHash = 'YOUR_CERTIFICATE_SHA256_HASH';
          return certSha256 == expectedHash;
        };
        return client;
      },
    );
  }
}
```

## Authentication Security

### Secure Token Management

```dart
class AuthService {
  final FlutterSecureStorage _storage;
  final Dio _dio;

  void _setupInterceptors() {
    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) async {
        final token = await _storage.read(key: 'auth_token');
        if (token != null) options.headers['Authorization'] = 'Bearer $token';
        handler.next(options);
      },
      onError: (error, handler) async {
        if (error.response?.statusCode == 401) {
          final refreshed = await _refreshToken();
          if (refreshed) {
            final options = error.requestOptions;
            final token = await _storage.read(key: 'auth_token');
            options.headers['Authorization'] = 'Bearer $token';
            final response = await _dio.fetch(options);
            return handler.resolve(response);
          }
        }
        handler.next(error);
      },
    ));
  }
}
```

### Biometric Authentication

```dart
import 'package:local_auth/local_auth.dart';

class BiometricAuth {
  final LocalAuthentication _auth = LocalAuthentication();

  Future<bool> authenticate({required String localizedReason}) async {
    try {
      return await _auth.authenticate(
        localizedReason: localizedReason,
        options: const AuthenticationOptions(stickyAuth: true, biometricOnly: true),
      );
    } catch (e) { return false; }
  }
}
```

## Data Encryption

```dart
import 'package:encrypt/encrypt.dart';

class DataEncryption {
  static final _key = Key.fromSecureRandom(32);
  static final _iv = IV.fromSecureRandom(16);
  static final _encrypter = Encrypter(AES(_key));

  static String encrypt(String plainText) => _encrypter.encrypt(plainText, iv: _iv).base64;
  static String decrypt(String encryptedText) => _encrypter.decrypt(Encrypted.fromBase64(encryptedText), iv: _iv);
}
```

## Input Validation

```dart
class InputValidator {
  static bool isValidEmail(String email) =>
    RegExp(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$').hasMatch(email);

  static bool isStrongPassword(String password) {
    if (password.length < 8) return false;
    return password.contains(RegExp(r'[A-Z]')) && password.contains(RegExp(r'[a-z]'))
        && password.contains(RegExp(r'[0-9]')) && password.contains(RegExp(r'[!@#$%^&*(),.?":{}|<>]'));
  }

  static String sanitizeHtml(String input) => input
      .replaceAll('<', '&lt;').replaceAll('>', '&gt;')
      .replaceAll('"', '&quot;').replaceAll("'", '&#x27;');
}
```

## Platform Security

### Android

```xml
<!-- Network security config: android/app/src/main/res/xml/network_security_config.xml -->
<network-security-config>
  <domain-config cleartextTrafficPermitted="false">
    <domain includeSubdomains="true">yourdomain.com</domain>
  </domain-config>
</network-security-config>
```

### iOS

```xml
<!-- ios/Runner/Info.plist -->
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key><false/>
</dict>
```

## Code Obfuscation

```bash
flutter build apk --obfuscate --split-debug-info=./debug-info
flutter build ios --obfuscate --split-debug-info=./debug-info
```

## Environment Variables

```dart
// Use flutter_dotenv - ADD .env TO .gitignore!
import 'package:flutter_dotenv/flutter_dotenv.dart';
await dotenv.load(fileName: ".env");
final apiKey = dotenv.env['API_KEY'];
```

## Security Checklist

- [ ] All secrets in secure storage (not SharedPreferences)
- [ ] HTTPS only for all network requests
- [ ] Certificate pinning implemented
- [ ] API tokens refreshed automatically
- [ ] Data encrypted at rest
- [ ] Input validation on all user inputs
- [ ] Code obfuscation enabled
- [ ] Debug logs removed from production
- [ ] Sensitive data not logged
- [ ] No hardcoded API keys in code
- [ ] Environment variables for secrets (.env in .gitignore)
- [ ] Minimum permissions requested
- [ ] Runtime permissions handled gracefully

## Common Vulnerabilities to Avoid

1. Hardcoded secrets in code
2. Plain text storage of passwords/tokens
3. HTTP in production (always HTTPS)
4. Unvalidated user input
5. Logging sensitive data (passwords, tokens, credit cards)
6. Weak encryption (use AES, not MD5 for passwords)
7. Unprotected routes without auth checks
