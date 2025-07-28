# Dart Workspaces Bug Reproduction

This repository demonstrates a critical bug in Dart workspaces where dependencies from one package incorrectly leak into apps that don't depend on it, causing unintended permissions and native code to be included in final APK builds.

## 🐛 Bug Description

When using Dart workspaces in a monorepo with multiple apps and packages, transitive dependencies from packages get incorrectly included in **all** apps within the workspace, regardless of whether those apps actually depend on the packages.

### Specific Issue

In this reproduction case:
- **Package One** depends on the Flutter `camera` plugin (requires camera permissions)
- **App One** depends on Package One (should include camera permissions)
- **App Two** does NOT depend on Package One (should NOT include camera permissions)

**Expected behavior**: Only App One should have camera permissions in its final APK.

**Actual bug**: Both apps end up with camera permissions due to Dart workspaces incorrectly including all workspace dependencies.

## 📁 Project Structure

```
dart_workspaces_example/
├── pubspec.yaml              # Workspace root configuration
├── apps/
│   ├── app_one/              # App that depends on package_one
│   │   ├── pubspec.yaml      # ✅ Declares dependency on package_one
│   │   └── lib/main.dart     # Uses Calculator from package_one
│   └── app_two/              # App that does NOT depend on package_one
│       ├── pubspec.yaml      # ❌ No dependency on package_one
│       └── lib/main.dart     # Independent implementation
└── packages/
    └── package_one/          # Package with camera dependency
        ├── pubspec.yaml      # 📷 Depends on camera: ^0.11.2
        └── lib/package_one.dart
```

## 🔬 Reproduction Steps

### Prerequisites

- Flutter 3.32.8+ with Dart workspaces support
- Android SDK for APK building and analysis
- Android Studio with APK Analyzer

### Step 1: Build APKs from Main Branch (No Workspaces)

```bash
# Clone the repository
git clone <this-repo-url>
cd dart_workspaces_example

# Switch to main branch (baseline without workspaces)
git checkout main

# Build App One
cd apps/app_one
flutter pub get
flutter build apk --debug
cp build/app/outputs/flutter-apk/app-debug.apk ../../app_one_main.apk

# Build App Two  
cd ../app_two
flutter pub get
flutter build apk --debug
cp build/app/outputs/flutter-apk/app-debug.apk ../../app_two_main.apk

cd ../..
```

### Step 2: Build APKs from Dart Workspaces Branch

```bash
# Switch to dart_workspaces branch (with bug)
git checkout dart_workspaces

# Build App One (workspace mode)
cd apps/app_one
flutter build apk --debug
cp build/app/outputs/flutter-apk/app-debug.apk ../../app_one_workspace.apk

# Build App Two (workspace mode)
cd ../app_two
flutter build apk --debug
cp build/app/outputs/flutter-apk/app-debug.apk ../../app_two_workspace.apk

cd ../..
```

### Step 3: Compare APKs

Using Android Studio APK Analyzer:

1. Open Android Studio
2. Go to **Build > Analyze APK**
3. Compare the AndroidManifest.xml files:

   **Main Branch APKs (Expected behavior):**
   - `app_one_main.apk` - Has camera permissions ✅
   - `app_two_main.apk` - No camera permissions ✅

   **Workspace Branch APKs (Bug demonstration):**
   - `app_one_workspace.apk` - Has camera permissions ✅  
   - `app_two_workspace.apk` - **BUG**: Also has camera permissions ❌


## 🔍 Expected vs Actual Results

| Branch | App One | App Two |
|--------|---------|---------|
| **main** (no workspaces) | ✅ Has camera permissions | ✅ No camera permissions |
| **dart_workspaces** | ✅ Has camera permissions | ❌ **BUG**: Has camera permissions |

## 🚨 Impact

This bug causes:

1. **Security concerns**: Apps get permissions they shouldn't have
2. **App store rejection**: Unnecessary permissions can cause app store rejections
3. **Larger APK sizes**: Unused native code gets bundled
4. **Privacy issues**: Apps appear to request permissions they don't actually use

## 🔧 Technical Details

- **Flutter Version**: 3.32.8
- **Dart SDK**: >=3.8.0 <4.0.0
- **Camera Plugin**: ^0.11.2
- **Workspace Resolution**: Used in all package pubspec.yaml files

## 📚 Related Issues

The issue likely extends beyond just permissions to include any native code, custom platform channels, and potentially Dart code tree-shaking problems.
