---
name: securing-flutter-apps
description: Secures Flutter applications following OWASP Mobile Top 10. Use when implementing secure storage, protecting API keys, adding certificate pinning, validating input, configuring platform security (iOS Keychain, Android Keystore), or auditing app security before release.
---

# Securing Flutter Apps

Security best practices for Flutter applications covering storage, network, authentication, and platform security.

## When to Use This Skill

- Storing sensitive data (tokens, credentials, user data)
- Protecting API keys and secrets
- Implementing certificate pinning
- Adding biometric authentication
- Validating and sanitizing user input
- Configuring iOS/Android platform security
- Pre-release security audit

## Secure Storage

### Never Store Secrets in Plain Text

```dart
// NEVER: SharedPreferences is NOT secure
await prefs.setString('password', 'user_password');
await prefs.setString('api_token', 'secret_token');

// ALWAYS: Use flutter_secure_storage
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

final storage = FlutterSecureStorage();

await storage.write(key: 'auth_token', value: token);
final token = await storage.read(key: 'auth_token');
await storage.delete(key: 'auth_token');
await storage.deleteAll();
```

### Platform-Specific Configuration

```dart
const secureStorage = FlutterSecureStorage(
  aOptions: AndroidOptions(
    encryptedSharedPreferences: true,
  ),
  iOptions: IOSOptions(
    // first_unlock_this_device: doesn't transfer to new devices
    // first_unlock: can migrate via backup
    accessibility: KeychainAccessibility.first_unlock_this_device,
  ),
);
```

- **iOS**: Uses Keychain (hardware-backed on devices with Secure Enclave)
- **Android**: Uses EncryptedSharedPreferences (Android Keystore-backed AES)

## API Key Protection

### Use Compile-Time Environment Variables

```bash
# Create env.json (ADD TO .gitignore!)
# { "API_KEY": "your_key", "API_URL": "https://api.example.com" }

flutter run --dart-define-from-file=env.json
flutter build apk --dart-define-from-file=env.json
```

```dart
class EnvConfig {
  static const apiKey = String.fromEnvironment('API_KEY');
  static const apiUrl = String.fromEnvironment(
    'API_URL',
    defaultValue: 'https://api.example.com',
  );
}
```

### Never Hardcode Secrets

```dart
// NEVER
const apiKey = 'abc123secret';

// NEVER commit to git
// env.json
// env.*.json
// .env
```

## Network Security

### HTTPS Only

```dart
// NEVER use HTTP in production
// ALWAYS use HTTPS

final dio = Dio(BaseOptions(
  baseUrl: 'https://api.example.com',
));
```

### Certificate Pinning

```dart
import 'package:dio/dio.dart';
import 'package:dio/io.dart';
import 'dart:io';

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

  AuthService(this._storage, this._dio) {
    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) async {
        final token = await _storage.read(key: 'auth_token');
        if (token != null) {
          options.headers['Authorization'] = 'Bearer $token';
        }
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

  Future<bool> _refreshToken() async {
    try {
      final refreshToken = await _storage.read(key: 'refresh_token');
      if (refreshToken == null) return false;

      final response = await _dio.post('/auth/refresh', data: {
        'refresh_token': refreshToken,
      });

      await _storage.write(
        key: 'auth_token',
        value: response.data['access_token'],
      );
      await _storage.write(
        key: 'refresh_token',
        value: response.data['refresh_token'],
      );
      return true;
    } catch (e) {
      await logout();
      return false;
    }
  }

  Future<void> logout() async {
    await _storage.deleteAll();
  }
}
```

### Biometric Authentication

```dart
import 'package:local_auth/local_auth.dart';

class BiometricAuth {
  final LocalAuthentication _auth = LocalAuthentication();

  Future<bool> canCheckBiometrics() async {
    try {
      return await _auth.canCheckBiometrics ||
             await _auth.isDeviceSupported();
    } catch (e) {
      return false;
    }
  }

  Future<bool> authenticate({required String reason}) async {
    try {
      return await _auth.authenticate(
        localizedReason: reason,
        options: const AuthenticationOptions(
          stickyAuth: true,
          biometricOnly: false,
          sensitiveTransaction: true,
        ),
      );
    } catch (e) {
      return false;
    }
  }
}
```

**Platform configuration required:**

```xml
<!-- iOS: ios/Runner/Info.plist -->
<key>NSFaceIDUsageDescription</key>
<string>We use Face ID to securely authenticate you</string>

<!-- Android: android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.USE_BIOMETRIC"/>
```

## Data Encryption

```dart
import 'package:encrypt/encrypt.dart';

class DataEncryption {
  final Encrypter _encrypter;

  DataEncryption._(Key key) : _encrypter = Encrypter(AES(key));

  static Future<DataEncryption> create() async {
    final keyString = await KeyManager.getOrCreateKey();
    return DataEncryption._(Key.fromBase64(keyString));
  }

  String encrypt(String plainText) {
    final iv = IV.fromSecureRandom(16); // Fresh IV per encryption
    final encrypted = _encrypter.encrypt(plainText, iv: iv);
    return '${iv.base64}:${encrypted.base64}';
  }

  String decrypt(String encryptedText) {
    final parts = encryptedText.split(':');
    final iv = IV.fromBase64(parts[0]);
    return _encrypter.decrypt(Encrypted.fromBase64(parts[1]), iv: iv);
  }
}

class KeyManager {
  static const _storage = FlutterSecureStorage();

  static Future<String> getOrCreateKey() async {
    var key = await _storage.read(key: 'encryption_key');
    if (key == null) {
      key = base64.encode(
        List<int>.generate(32, (_) => Random.secure().nextInt(256)),
      );
      await _storage.write(key: 'encryption_key', value: key);
    }
    return key;
  }
}
```

## Input Validation

```dart
class InputValidator {
  static bool isValidEmail(String email) {
    return RegExp(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
        .hasMatch(email);
  }

  static bool isStrongPassword(String password) {
    if (password.length < 8) return false;
    return password.contains(RegExp(r'[A-Z]')) &&
           password.contains(RegExp(r'[a-z]')) &&
           password.contains(RegExp(r'[0-9]')) &&
           password.contains(RegExp(r'[!@#$%^&*(),.?":{}|<>]'));
  }

  static String sanitizeHtml(String input) {
    return input
        .replaceAll('<', '&lt;')
        .replaceAll('>', '&gt;')
        .replaceAll('"', '&quot;')
        .replaceAll("'", '&#x27;');
  }

  static bool isValidUrl(String url) {
    try {
      final uri = Uri.parse(url);
      return uri.hasScheme && (uri.scheme == 'http' || uri.scheme == 'https');
    } catch (e) {
      return false;
    }
  }
}
```

## Platform Security

### Android

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<!-- Network security config: block cleartext -->
<application
  android:networkSecurityConfig="@xml/network_security_config">
</application>
```

```xml
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
  <domain-config cleartextTrafficPermitted="false">
    <domain includeSubdomains="true">yourdomain.com</domain>
  </domain-config>
</network-security-config>
```

### iOS

```xml
<!-- ios/Runner/Info.plist -->
<!-- App Transport Security: block HTTP -->
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <false/>
</dict>
```

### iOS Privacy Manifests (Required since iOS 17)

```xml
<!-- ios/Runner/PrivacyInfo.xcprivacy -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>NSPrivacyTracking</key>
  <false/>
  <key>NSPrivacyTrackingDomains</key>
  <array/>
  <key>NSPrivacyCollectedDataTypes</key>
  <array/>
  <key>NSPrivacyAccessedAPITypes</key>
  <array>
    <dict>
      <key>NSPrivacyAccessedAPIType</key>
      <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
      <key>NSPrivacyAccessedAPITypeReasons</key>
      <array><string>CA92.1</string></array>
    </dict>
  </array>
</dict>
</plist>
```

## Screenshot Prevention

```kotlin
// android/app/src/main/kotlin/.../MainActivity.kt
import android.view.WindowManager

class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        // Prevents screenshots and screen recording on sensitive apps
        window.setFlags(
            WindowManager.LayoutParams.FLAG_SECURE,
            WindowManager.LayoutParams.FLAG_SECURE
        )
    }
}
```

**iOS:** No direct equivalent. Use `UIScreen.capturedDidChangeNotification` to detect screen recording and blur/hide sensitive content.

## Root/Jailbreak Detection

```dart
// dependencies: flutter_jailbreak_detection: ^1.10.0
import 'package:flutter_jailbreak_detection/flutter_jailbreak_detection.dart';

Future<bool> isDeviceCompromised() async {
  try {
    return await FlutterJailbreakDetection.jailbroken;
  } catch (e) {
    return false; // Fail open — don't lock users out
  }
}
```

**Important:** Root/jailbreak detection is bypassable — use as defense-in-depth alongside secure storage and certificate pinning, not as the sole security gate.

## Code Obfuscation

```bash
# Build with obfuscation (always for release)
flutter build apk --obfuscate --split-debug-info=./debug-info
flutter build ios --obfuscate --split-debug-info=./debug-info
```

## Secure Deep Links

```dart
class DeepLinkHandler {
  static Future<void> handleDeepLink(Uri uri) async {
    // Validate scheme
    if (uri.scheme != 'https' && uri.scheme != 'yourapp') {
      throw SecurityException('Invalid scheme');
    }
    // Validate host
    if (uri.host != 'yourdomain.com') {
      throw SecurityException('Invalid host');
    }
    // Sanitize all parameters
    final params = uri.queryParameters.map(
      (key, value) => MapEntry(key, InputValidator.sanitizeHtml(value)),
    );
    // Route with validated params only
  }
}
```

## Logging Security

```dart
// NEVER log sensitive data
logger.d('User password: $password');   // NEVER!
logger.d('API token: $token');          // NEVER!
logger.d('Credit card: $ccNumber');     // NEVER!

// GOOD: Log actions, not data
logger.i('User login attempt');
logger.i('Payment processed successfully');

// Remove debug logs in production
// Use Logger with ProductionFilter or kReleaseMode checks
```

## OWASP Mobile Top 10 Checklist

| # | Risk | Flutter Mitigation |
|---|------|-------------------|
| M1 | Improper Credential Usage | flutter_secure_storage, no hardcoded keys |
| M2 | Inadequate Supply Chain | Lock dependency versions, audit packages |
| M3 | Insecure Auth/Authorization | Token refresh, biometrics, session timeout |
| M4 | Insufficient Input/Output Validation | InputValidator, sanitize HTML/SQL |
| M5 | Insecure Communication | HTTPS only, certificate pinning |
| M6 | Inadequate Privacy Controls | Privacy manifests, minimum permissions |
| M7 | Insufficient Binary Protections | Code obfuscation, ProGuard/R8 |
| M8 | Security Misconfiguration | Network security config, ATS |
| M9 | Insecure Data Storage | Encrypt at rest, secure storage |
| M10 | Insufficient Cryptography | AES-256, random IVs, secure key storage |

## Pre-Release Security Audit

- [ ] All secrets in secure storage (not SharedPreferences)
- [ ] No hardcoded API keys in source code
- [ ] HTTPS only for all network requests
- [ ] Certificate pinning for critical endpoints
- [ ] Token refresh implemented
- [ ] Biometric auth for sensitive operations
- [ ] Input validation on all user inputs
- [ ] Code obfuscation enabled for release builds
- [ ] Debug logs removed from production
- [ ] Sensitive data never logged
- [ ] Deep links validated and sanitized
- [ ] Minimum permissions requested
- [ ] iOS Privacy Manifest configured
- [ ] Android network security config set
- [ ] Dependencies audited for vulnerabilities
- [ ] env.json / .env files in .gitignore
