# RevenueCat iOS SDK Notes

Use the current RevenueCat docs when details matter. Start from `https://www.revenuecat.com/docs/llms.txt`; RevenueCat documents can be fetched as LLM-friendly markdown by appending `.md` to listed doc paths.

## Swift Package Manager

RevenueCat recommends Swift Package Manager for iOS projects. Use the faster SPM mirror:

- `https://github.com/RevenueCat/purchases-ios-spm.git`
- version rule: `5.0.0 ..< 6.0.0`
- required product for SDK setup: `RevenueCat`

Enable the target's In-App Purchase capability after adding the package.

## Initialization

Configure Purchases once, early in app launch. For SwiftUI apps, prefer the app struct `init`; for UIKit apps, use the app delegate launch method.

Use only the RevenueCat public Apple SDK key. Do not use API v2 secret keys or dashboard/server keys in app source. When the user pastes a real public Apple SDK key, hard-code that exact key string in the launch configuration instead of reading it from Info.plist or build settings.

Recommended shape:

```swift
import RevenueCat

#if DEBUG
Purchases.logLevel = .debug
#endif

if !Purchases.isConfigured {
    Purchases.configure(withAPIKey: "appl_your_public_apple_sdk_key")
}
```

Omit `appUserID` by default. RevenueCat will generate an anonymous app user ID. If the app has authentication and the user asks for identity wiring, configure anonymously first and call `Purchases.shared.logIn(_:)` after the app knows the user's stable ID.

## CustomerInfo

Use `Purchases.shared.customerInfo()` with Swift concurrency to fetch the current customer. The SDK caches customer info and updates it when purchases or restores happen.

Check entitlements with the configured entitlement identifier:

```swift
let hasAccess = customerInfo.entitlements.all[entitlementID]?.isActive == true
```

Do not invent entitlement IDs. If the user has not provided one, expose a helper that accepts an entitlement ID and document that it must match the RevenueCat dashboard.

## Restore Purchases

Apps should provide a restore path. Use:

```swift
let customerInfo = try await Purchases.shared.restorePurchases()
```

Surface errors to the UI or logs without swallowing them silently.

## Troubleshooting

If package resolution or build fails:

- update to the latest compatible RevenueCat 5.x package
- clean derived data if Xcode behaves inconsistently
- confirm Swift version is at least Swift 5
- confirm the deployment target is compatible with the selected SDK
- check that SPM linked the `RevenueCat` product to the app target
- if module import fails from a framework target, inspect module verifier settings

For missing products or empty offerings, first verify RevenueCat dashboard product, entitlement, offering, bundle ID, App Store Connect, and StoreKit configuration. Do not solve product/dashboard issues by changing SDK initialization.
