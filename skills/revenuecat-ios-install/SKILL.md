---
name: tortuga:revenuecat-ios-install
description: Interactive wizard-style RevenueCat SDK install for native iOS Swift and SwiftUI apps. Use when guiding a user through choices to install only the RevenueCat Purchases SDK with Swift Package Manager, configure Purchases at app launch, enable In-App Purchase capability, and verify the build. This skill does not use bundled scripts and does not add RevenueCatUI or paywalls.
license: MIT
metadata:
  author: tortuga-ai
  version: "1.0.0"
  organization: Tortuga AI
  public: true
  repo: https://github.com/tortuga-ai/skills
  date: May 2026
  abstract: Interactive wizard for installing the core RevenueCat Purchases SDK in native iOS Swift and SwiftUI projects with SPM, app launch configuration, In-App Purchase capability setup, and build verification.
---

# RevenueCat iOS Install Wizard

Use this skill for one job only: guide an interactive wizard that installs and configures the core RevenueCat Purchases SDK in an iOS Xcode project. Do not add prebuilt UI products, product listing UI, entitlement-gated app flows, or helper scripts.

## Step 1: Detect Project

Inspect the project directly with normal shell reads/searches. Confirm:

- one usable `.xcodeproj`
- one iOS app target and scheme, or a clear target choice
- a SwiftUI `@main App` entry point or UIKit app delegate
- whether RevenueCat SDK package/products, `Purchases.configure`, and In-App Purchase capability already exist

Stop if no iOS app target or app launch entry point exists. If prebuilt RevenueCat UI products are already installed, report that the project is outside this SDK-only wizard.

Before editing, show the user the detected facts and ask for any missing choice. Use this format:

```text
RevenueCat SDK Install Wizard

Step 1 of 6: Project detected
- Project: <project>
- Target: <target>
- Scheme: <scheme>
- App entry: <file>
- Existing RevenueCat SDK: yes/no

Next choice:
1. Continue with this target
2. Pick a different target
3. Stop
```

## Step 2: Choose SDK Key Handling

Require a real RevenueCat public Apple SDK key for production. If the user asks to test with a placeholder, use a visibly fake value such as `appl_placeholder_revenuecat_public_sdk_key`.

Never invent SDK keys, product IDs, entitlement IDs, offering IDs, App Store Connect state, or dashboard state. Default to anonymous RevenueCat app user IDs unless the app already has auth and the user asks for identity wiring.

Ask:

```text
Step 2 of 6: SDK key
Choose how to configure the RevenueCat public Apple SDK key:
1. I will paste a real public Apple SDK key now
2. Use a placeholder key for a local install test
3. Create a build setting placeholder that I fill in later
```

## Step 3: Install SDK With SPM

Use Swift Package Manager:

- repository: `https://github.com/RevenueCat/purchases-ios-spm.git`
- version rule: up to next major from `5.0.0`
- product: `RevenueCat`

Do not add any optional UI product. If the project already uses another package manager for RevenueCat, stop and ask before changing package managers.

Ask for confirmation before changing the Xcode project:

```text
Step 3 of 6: Swift Package Manager
This will add:
- Package: https://github.com/RevenueCat/purchases-ios-spm.git
- Version: 5.0.0 ..< 6.0.0
- Product: RevenueCat

Choices:
1. Install SDK with SPM
2. Stop
```

## Step 4: Configure App Launch

Configure `Purchases` exactly once, early in app launch:

- SwiftUI: app struct `init`
- UIKit: `application(_:didFinishLaunchingWithOptions:)`
- `Purchases.logLevel = .debug` only for Debug builds
- `Purchases.configure(withAPIKey:)` with a hard-coded string when the user pasted a real key or chose the placeholder key
- use a build-setting-backed value only when the user explicitly chose the build setting placeholder option
- guard with `Purchases.isConfigured`
- enable In-App Purchase capability on the app target

If the user chooses option 1 and pastes a real public Apple SDK key, hard-code that exact key string in the app launch configuration. If the user chooses option 2, hard-code `appl_placeholder_revenuecat_public_sdk_key`. If the user chooses option 3, create a build setting placeholder and read it from the generated Info.plist.

Ask:

```text
Step 4 of 6: App launch configuration
Choices:
1. Configure in the SwiftUI App init
2. Configure in the UIKit app delegate
3. Stop and show me what would change
```

## Step 5: Enable Capability

Enable In-App Purchase capability on the app target. Do not add StoreKit files, product IDs, paywall screens, or purchase buttons.

Ask:

```text
Step 5 of 6: In-App Purchase capability
Choices:
1. Enable capability on the detected app target
2. Skip for now and report manual Xcode steps
```

## Step 6: Verify

Run:

```bash
xcodebuild -resolvePackageDependencies -project <App>.xcodeproj -scheme <Scheme>
xcodebuild -project <App>.xcodeproj -scheme <Scheme> -destination 'generic/platform=iOS Simulator' build
```

Then inspect:

- `RevenueCat` product is linked
- optional UI products/imports are absent
- exactly one `Purchases.configure(...)` exists
- In-App Purchase capability is enabled

If verification is skipped or fails, report the exact reason and the next command to run.

## Step 7: Report

Report changed files, package/product added, SDK key handling, verification results, and manual RevenueCat dashboard/App Store Connect steps still required.

## References

Read `references/revenuecat-ios-sdk.md` before editing installation, launch configuration, CustomerInfo, restore purchases, or troubleshooting.

Read `references/wizard-patterns.md` when improving the wizard or designing a future standalone CLI.
