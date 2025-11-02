# Flutter Agent Implementation Guide

## Quick Start: Implementing the Agent Architecture

This guide provides practical steps for implementing the Flutter agent architecture defined in `FLUTTER_AGENT_ARCHITECTURE_PLAN.md`.

---

## Implementation Approach

### Option 1: Create as Claude Code Sub-Agents

Each agent should be implemented as a specialized Claude Code sub-agent with focused expertise. Sub-agents are ideal for workflow-based tasks that require orchestration.

**When to use Sub-Agents:**
- Multi-step workflows (e.g., Design Iteration Coordinator)
- Tasks requiring tool usage (e.g., Device Orchestrator)
- Context-heavy operations (e.g., Platform Integration Specialists)
- Orchestration between multiple operations

### Option 2: Create as Claude Code Skills

Skills are better for on-demand knowledge and pattern libraries that augment general development.

**When to use Skills:**
- Reusable code patterns and templates
- Best practices and guidelines
- Reference implementations
- Knowledge that enhances existing agents

---

## Phase 1: Design-to-Implementation Pipeline (Priority 1)

This is the highest priority as it addresses the core workflow requirement.

### Agent 1.1: Flutter UI Designer

**Implementation Type**: Sub-agent

**Core Capabilities Needed:**
```markdown
# Agent Prompt Template

You are a Flutter UI Design Analyst specializing in converting visual designs
into Flutter widget hierarchies.

## Your Expertise:
- 300+ Flutter widgets (Material, Cupertino, base widgets)
- Material 3 design system
- Layout constraint system
- Responsive design patterns

## Your Process:
1. Analyze design input (Figma export, screenshot, mockup)
2. Identify all UI components and their relationships
3. Map components to appropriate Flutter widgets
4. Design widget composition hierarchy
5. Specify layout approach (Row, Column, Stack, etc.)
6. Recommend styling strategy
7. Output detailed implementation plan

## Tools You Use:
- Image analysis for design inspection
- Widget catalog reference
- Material Design guidelines

## Output Format:
- Widget hierarchy tree
- Layout approach per section
- Styling specifications
- Implementation complexity assessment
```

**Key Tools Required:**
- Read (for viewing design images)
- WebFetch (for referencing Flutter docs)
- Write (for creating implementation plans)

---

### Agent 1.2: Flutter UI Implementer

**Implementation Type**: Sub-agent

**Core Capabilities Needed:**
```markdown
# Agent Prompt Template

You are a Flutter UI Implementation Expert specializing in generating
pixel-perfect Flutter code from design specifications.

## Your Expertise:
- Flutter widget building and composition
- Material 3 theming and styling
- Responsive layouts with MediaQuery and LayoutBuilder
- Animation implementation
- Accessibility compliance

## Your Process:
1. Review implementation plan from UI Designer
2. Generate complete Flutter widget code
3. Implement precise styling (colors, fonts, spacing)
4. Add responsive behavior for different screen sizes
5. Implement animations and transitions
6. Ensure accessibility features
7. Optimize code for performance (const, keys)

## Code Quality Standards:
- Use const constructors wherever possible
- Proper widget keys for stateful widgets
- Extract complex widgets to separate methods/classes
- Follow Flutter style guide
- Add comments for complex logic

## Output:
- Complete, runnable Flutter widget code
- README with implementation notes
- List of dependencies if any new packages needed
```

**Key Tools Required:**
- Read (for reading implementation plans)
- Write (for generating Flutter code)
- Edit (for modifying existing code)
- Bash (for running flutter commands)

---

### Agent 1.3: Flutter Device Orchestrator

**Implementation Type**: Sub-agent

**Core Capabilities Needed:**
```markdown
# Agent Prompt Template

You are a Flutter Device Management Expert specializing in iOS simulator
and Android emulator orchestration.

## Your Expertise:
- iOS simulator management (simctl, xcodebuild)
- Android emulator management (avdmanager, adb)
- Device configuration and setup
- Flutter app installation and hot reload
- Screenshot capture

## Your Process:
1. List available simulators/emulators
2. Launch requested device configuration
3. Install Flutter app on device
4. Enable hot reload connection
5. Capture screenshots as needed
6. Manage multiple devices simultaneously

## Commands You Master:
iOS:
- xcrun simctl list devices
- xcrun simctl boot <device-id>
- xcrun simctl install <device-id> <app-path>
- xcrun simctl io <device-id> screenshot <path>

Android:
- emulator -list-avds
- emulator -avd <name>
- adb devices
- adb install <apk-path>
- adb shell screencap <path>

Flutter:
- flutter devices
- flutter run -d <device-id>
- flutter install -d <device-id>
```

**Key Tools Required:**
- Bash (for all device commands)
- Read (for configuration files)
- Write (for saving screenshots and reports)

---

### Agent 1.4: UI Comparison & Validation Specialist

**Implementation Type**: Sub-agent

**Core Capabilities Needed:**
```markdown
# Agent Prompt Template

You are a UI Validation Expert specializing in comparing implemented UIs
with original designs to ensure pixel-perfect accuracy.

## Your Expertise:
- Image comparison and analysis
- Design fidelity metrics
- Layout inspection techniques
- Color, spacing, and typography validation
- Visual regression detection

## Your Process:
1. Receive original design and implementation screenshot
2. Perform detailed visual comparison
3. Identify discrepancies:
   - Spacing differences
   - Color mismatches
   - Font size/weight differences
   - Alignment issues
   - Missing elements
   - Extra elements
4. Quantify differences (pixel measurements)
5. Generate prioritized fix list
6. Suggest specific code changes
7. Calculate fidelity score

## Validation Criteria:
- Color accuracy (hex value matching)
- Spacing accuracy (±2px tolerance)
- Font matching (family, size, weight)
- Element positioning
- Responsive behavior
- Animation smoothness

## Output:
- Detailed difference report
- Annotated screenshots showing issues
- Prioritized fix list with code suggestions
- Fidelity score (0-100%)
```

**Key Tools Required:**
- Read (for reading screenshots)
- Image comparison capabilities
- Write (for generating reports)

---

### Agent 1.5: Design Iteration Coordinator

**Implementation Type**: Sub-agent (Orchestrator)

**Core Capabilities Needed:**
```markdown
# Agent Prompt Template

You are a Design-to-Implementation Workflow Coordinator managing the entire
process of converting designs to pixel-perfect Flutter implementations.

## Your Role:
Orchestrate the complete workflow between:
- UI Designer (1.1)
- UI Implementer (1.2)
- Device Orchestrator (1.3)
- UI Comparison Specialist (1.4)

## Your Process:
1. Receive design input from user
2. Invoke UI Designer to create implementation plan
3. Invoke UI Implementer to generate code
4. Invoke Device Orchestrator to launch and run app
5. Invoke UI Comparison Specialist to validate
6. If discrepancies found:
   - Prioritize fixes
   - Invoke UI Implementer with specific changes
   - Repeat steps 4-5
7. Declare completion when fidelity score >95%

## Success Criteria:
- Fidelity score >95%
- All major discrepancies resolved
- Responsive behavior validated
- User approval obtained

## Iteration Management:
- Track iteration count
- Monitor progress per iteration
- Prevent infinite loops (max 10 iterations)
- Generate progress reports
- Document final state
```

**Key Tools Required:**
- Task (for invoking other agents)
- TodoWrite (for tracking progress)
- Write (for reports)

---

## Phase 2: Architecture & Code Quality (Priority 2)

### Agent 2.1: Flutter Architect

**Implementation Type**: Sub-agent

**Key Documentation Sources:**
- /resources/architectural-overview
- /data-and-backend/state-mgmt/intro
- /get-started/fundamentals

**Core Focus:**
- Clean Architecture implementation
- Feature-based folder structure
- Dependency injection setup
- Navigation architecture

**Sample Folder Structure to Generate:**
```
lib/
  core/
    constants/
    utils/
    errors/
    network/
  features/
    authentication/
      data/
        models/
        repositories/
        data_sources/
      domain/
        entities/
        repositories/
        use_cases/
      presentation/
        pages/
        widgets/
        bloc/ (or provider, riverpod)
    home/
      ... (same structure)
  shared/
    widgets/
    themes/
```

---

### Agent 2.2: Flutter State Management Specialist

**Implementation Type**: Sub-agent

**Key Documentation Sources:**
- /data-and-backend/state-mgmt/intro
- /data-and-backend/state-mgmt/options

**Decision Matrix to Implement:**

| App Complexity | Recommended Solution | Alternative |
|---------------|---------------------|-------------|
| Simple (1-5 screens) | setState + InheritedWidget | Provider |
| Medium (5-20 screens) | Provider or Riverpod | BLoC |
| Large (20+ screens) | BLoC or Riverpod | Redux |
| Real-time heavy | BLoC with streams | Riverpod |

---

## Phase 3: Platform Integration (Priority 3)

### Agent 3.1: iOS Integration Specialist
### Agent 3.2: Android Integration Specialist

**Implementation Type**: Sub-agents

**Key Documentation Sources:**
- /platform-integration/ios/setup
- /platform-integration/android/setup
- /platform-integration/platform-channels

**Common Tasks:**
1. Configure platform-specific settings
2. Implement platform channels
3. Add native dependencies
4. Handle permissions
5. Integrate native SDKs

---

### Agent 3.3: Platform Channel Architect

**Implementation Type**: Sub-agent

**Key Documentation Sources:**
- /platform-integration/platform-channels (comprehensive guide)

**Template for Platform Channel Implementation:**

```dart
// Dart side
class NativeService {
  static const platform = MethodChannel('com.example.app/native');

  Future<String> getNativeData() async {
    try {
      final result = await platform.invokeMethod('getData');
      return result;
    } on PlatformException catch (e) {
      print("Failed to get data: '${e.message}'.");
      return null;
    }
  }
}
```

```swift
// iOS side (Swift)
let channel = FlutterMethodChannel(
    name: "com.example.app/native",
    binaryMessenger: controller.binaryMessenger
)

channel.setMethodCallHandler { (call: FlutterMethodCall, result: @escaping FlutterResult) in
  if call.method == "getData" {
    result("Native iOS data")
  } else {
    result(FlutterMethodNotImplemented)
  }
}
```

```kotlin
// Android side (Kotlin)
val channel = MethodChannel(flutterEngine.dartExecutor.binaryMessenger,
    "com.example.app/native")

channel.setMethodCallHandler { call, result ->
  when (call.method) {
    "getData" -> result.success("Native Android data")
    else -> result.notImplemented()
  }
}
```

---

## Phase 4: API & Backend Integration (Priority 4)

### Agent 5.1: Flutter REST API Specialist

**Implementation Type**: Sub-agent

**Key Documentation Sources:**
- /data-and-backend/networking
- /cookbook/networking/fetch-data

**Standard API Service Template:**

```dart
import 'package:dio/dio.dart';

class ApiService {
  final Dio _dio;

  ApiService() : _dio = Dio(BaseOptions(
    baseUrl: 'https://api.example.com',
    connectTimeout: Duration(seconds: 5),
    receiveTimeout: Duration(seconds: 3),
  )) {
    _dio.interceptors.add(LogInterceptor());
    _dio.interceptors.add(AuthInterceptor());
  }

  Future<T> get<T>(String path, {Map<String, dynamic>? queryParameters}) async {
    try {
      final response = await _dio.get(path, queryParameters: queryParameters);
      return response.data;
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }

  // ... post, put, delete methods
}
```

---

### Agent 5.2: Flutter Firebase Integration Expert

**Implementation Type**: Sub-agent

**Key Documentation Sources:**
- /data-and-backend/firebase
- https://firebase.google.com/docs/flutter/setup

**Standard Firebase Setup Template:**

```dart
// firebase_options.dart (generated with flutterfire configure)
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  runApp(MyApp());
}

// Authentication Service
class AuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;

  Stream<User?> get authStateChanges => _auth.authStateChanges();

  Future<UserCredential> signInWithEmail(String email, String password) async {
    return await _auth.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
  }

  Future<void> signOut() async {
    await _auth.signOut();
  }
}

// Firestore Service
class FirestoreService {
  final FirebaseFirestore _db = FirebaseFirestore.instance;

  Stream<List<T>> streamCollection<T>(
    String path,
    T Function(Map<String, dynamic>) fromJson,
  ) {
    return _db.collection(path).snapshots().map((snapshot) =>
      snapshot.docs.map((doc) => fromJson(doc.data())).toList()
    );
  }
}
```

---

### Agent 5.3: Flutter AWS Integration Expert

**Implementation Type**: Sub-agent

**Standard Amplify Setup:**

```dart
import 'package:amplify_flutter/amplify_flutter.dart';
import 'package:amplify_auth_cognito/amplify_auth_cognito.dart';
import 'package:amplify_api/amplify_api.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await _configureAmplify();
  runApp(MyApp());
}

Future<void> _configureAmplify() async {
  try {
    await Amplify.addPlugins([
      AmplifyAuthCognito(),
      AmplifyAPI(),
    ]);
    await Amplify.configure(amplifyconfig);
  } on AmplifyAlreadyConfiguredException {
    print('Amplify already configured');
  }
}
```

---

### Agent 5.4: Flutter GraphQL Integration Expert

**Implementation Type**: Sub-agent

**Standard GraphQL Client Setup:**

```dart
import 'package:graphql_flutter/graphql_flutter.dart';

final HttpLink httpLink = HttpLink('https://api.example.com/graphql');

final AuthLink authLink = AuthLink(
  getToken: () async => 'Bearer $token',
);

final Link link = authLink.concat(httpLink);

final client = GraphQLClient(
  cache: GraphQLCache(store: InMemoryStore()),
  link: link,
);

// Query example
const String getUsers = '''
  query GetUsers {
    users {
      id
      name
      email
    }
  }
''';

final result = await client.query(
  QueryOptions(document: gql(getUsers)),
);
```

---

## Phase 5: Performance & Deployment (Priority 5)

### Agent 4.1: Flutter Performance Analyzer

**Implementation Type**: Sub-agent

**Key Documentation Sources:**
- /perf
- /perf/rendering-performance
- /tools/devtools/performance

**Performance Checklist:**
- [ ] Frame rendering <16ms (60fps)
- [ ] No jank in animations
- [ ] Memory usage stable (no leaks)
- [ ] App startup time <3 seconds
- [ ] Build time optimized
- [ ] Image sizes optimized
- [ ] Expensive operations on isolates

---

### Agent 4.2: Flutter Performance Optimizer

**Implementation Type**: Sub-agent

**Optimization Patterns to Apply:**

```dart
// 1. Use const constructors
const Text('Hello'); // Good
Text('Hello'); // Bad for static text

// 2. Use keys for stateful widgets in lists
ListView.builder(
  itemBuilder: (context, index) => MyWidget(key: ValueKey(items[index].id)),
);

// 3. Use RepaintBoundary for complex widgets
RepaintBoundary(
  child: ComplexWidget(),
);

// 4. Use ListView.builder for long lists
ListView.builder( // Good - lazy loading
  itemCount: 10000,
  itemBuilder: (context, index) => ListTile(...),
);

// 5. Optimize images
Image.network(
  url,
  cacheHeight: 300, // Resize in memory
  cacheWidth: 300,
);
```

---

### Agent 6.1: Flutter Testing Expert

**Implementation Type**: Sub-agent

**Key Documentation Sources:**
- /testing/overview
- /cookbook/testing

**Test Template Library:**

```dart
// Unit Test
void main() {
  group('Calculator', () {
    test('adds two numbers', () {
      expect(Calculator.add(2, 3), 5);
    });
  });
}

// Widget Test
void main() {
  testWidgets('MyWidget has a title and message', (WidgetTester tester) async {
    await tester.pumpWidget(MyWidget(title: 'T', message: 'M'));

    expect(find.text('T'), findsOneWidget);
    expect(find.text('M'), findsOneWidget);
  });
}

// Integration Test
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('complete login flow', (tester) async {
    await tester.pumpWidget(MyApp());

    await tester.enterText(find.byKey(Key('email')), 'test@test.com');
    await tester.enterText(find.byKey(Key('password')), 'password');
    await tester.tap(find.byKey(Key('login')));
    await tester.pumpAndSettle();

    expect(find.text('Welcome'), findsOneWidget);
  });
}
```

---

### Agent 6.2: Flutter Deployment Specialist (iOS)

**Implementation Type**: Sub-agent

**Key Documentation Sources:**
- /deployment/ios

**Deployment Checklist:**
- [ ] Bundle ID configured
- [ ] Version and build number updated
- [ ] Code signing configured
- [ ] Release build tested
- [ ] App icons added
- [ ] Launch screen configured
- [ ] App Store Connect app created
- [ ] Privacy manifest added (iOS 17+)

**Build Commands:**
```bash
# Release build
flutter build ipa --release

# With obfuscation
flutter build ipa --obfuscate --split-debug-info=build/debug-info

# Upload to App Store
xcrun altool --upload-app --file build/ios/ipa/*.ipa --username <email>
```

---

### Agent 6.3: Flutter Deployment Specialist (Android)

**Implementation Type**: Sub-agent

**Key Documentation Sources:**
- /deployment/android

**Deployment Checklist:**
- [ ] Application ID configured
- [ ] Version code and name updated
- [ ] Signing key created and configured
- [ ] Release build tested
- [ ] App icons added (all densities)
- [ ] ProGuard rules configured
- [ ] Play Console app created
- [ ] Play Store listing prepared

**Build Commands:**
```bash
# App Bundle (recommended)
flutter build appbundle --release

# APKs (architecture-specific)
flutter build apk --split-per-abi --release

# This creates:
# - app-armeabi-v7a-release.apk
# - app-arm64-v8a-release.apk
# - app-x86_64-release.apk
```

---

## Skill Recommendations

Create these as Claude Code Skills to complement the sub-agents:

### 1. flutter-widget-patterns
**Content**: Common widget composition patterns and best practices
- Common layouts (app bar + body + bottom nav)
- Form patterns
- List patterns
- Card layouts
- Bottom sheets and dialogs

### 2. flutter-animation-cookbook
**Content**: Reusable animation implementations
- Page transitions
- Staggered animations
- Hero animations
- Loading animations
- Gesture-based animations

### 3. flutter-testing-recipes
**Content**: Test templates and patterns
- Widget test templates
- Mock generation patterns
- Golden test setup
- Integration test patterns

### 4. flutter-performance-checklist
**Content**: Performance optimization checklist
- Pre-deployment performance checks
- Frame rendering optimization
- Memory optimization
- Build size optimization

### 5. flutter-security-guide
**Content**: Security best practices
- Secure storage patterns
- API key handling
- Certificate pinning
- Code obfuscation
- Sensitive data handling

---

## Next Steps

### Step 1: Implement Phase 1 Agents (Week 1-2)
1. Create Agent 1.1 (Flutter UI Designer)
2. Create Agent 1.2 (Flutter UI Implementer)
3. Create Agent 1.3 (Flutter Device Orchestrator)
4. Create Agent 1.4 (UI Comparison Specialist)
5. Create Agent 1.5 (Design Iteration Coordinator)
6. Test complete design-to-implementation workflow

### Step 2: Implement Phase 2 Agents (Week 3)
1. Create Agent 2.1 (Flutter Architect)
2. Create Agent 2.2 (State Management Specialist)
3. Create Agent 6.1 (Testing Expert)
4. Test architecture setup workflow

### Step 3: Implement Phase 3 Agents (Week 4)
1. Create Agent 3.1 (iOS Integration Specialist)
2. Create Agent 3.2 (Android Integration Specialist)
3. Create Agent 3.3 (Platform Channel Architect)
4. Test platform integration workflow

### Step 4: Implement Phase 4 Agents (Week 5)
1. Create Agent 5.1 (REST API Specialist)
2. Create Agent 5.2 (Firebase Expert)
3. Create Agent 5.3 (AWS Expert) - if needed
4. Create Agent 5.4 (GraphQL Expert) - if needed
5. Test API integration workflows

### Step 5: Implement Phase 5 Agents (Week 6)
1. Create Agent 4.1 (Performance Analyzer)
2. Create Agent 4.2 (Performance Optimizer)
3. Create Agent 6.2 (iOS Deployment)
4. Create Agent 6.3 (Android Deployment)
5. Test complete deployment workflow

### Step 6: Create Complementary Skills (Week 7)
1. Create flutter-widget-patterns skill
2. Create flutter-animation-cookbook skill
3. Create flutter-testing-recipes skill
4. Create flutter-performance-checklist skill
5. Create flutter-security-guide skill

### Step 7: Integration Testing (Week 8)
1. Test complete lifecycle: Design → Implementation → Testing → Deployment
2. Identify gaps and create additional agents/skills as needed
3. Document workflows and examples
4. Create user guides

---

## Tool Requirements Summary

Each agent will need access to different tools:

**All Agents:**
- Read, Write, Edit (file operations)
- Bash (command execution)
- WebFetch (documentation reference)

**Device Orchestrator (1.3):**
- Bash with extensive iOS/Android device commands
- Screenshot capture capabilities

**UI Comparison (1.4):**
- Image reading and comparison
- Visual analysis capabilities

**Design Iteration Coordinator (1.5):**
- Task tool (for invoking other agents)
- TodoWrite (for tracking iterations)

**Platform Integration Agents (3.1, 3.2, 3.3):**
- Bash (for native tooling)
- File editing for platform-specific configs

**Deployment Agents (6.2, 6.3):**
- Bash (for build and upload commands)
- Secure credential handling

---

## Success Metrics

Track these metrics to validate agent effectiveness:

1. **Design-to-Implementation**: Time from design input to pixel-perfect implementation
2. **Iteration Count**: Average iterations needed to achieve >95% fidelity
3. **Code Quality**: Maintainability score, test coverage
4. **Platform Parity**: iOS/Android feature parity achievement rate
5. **Performance**: Frame rendering, memory usage, app size
6. **Deployment Success**: App store approval rate, deployment time
7. **Developer Satisfaction**: User feedback on agent helpfulness

---

## Conclusion

This implementation guide provides concrete templates, code samples, and step-by-step instructions for building the complete Flutter agent architecture. Start with Phase 1 (design-to-implementation) and progressively build out the ecosystem based on priority and need.
