# Wizard Patterns

Use these patterns when maintaining the skill or designing a future RevenueCat CLI.

## Deterministic Wizard

Break setup into explicit stages: detect, collect SDK key, install dependency, patch app launch code, verify, report. Avoid one giant prompt that asks an agent to "integrate RevenueCat" without guardrails.

## No Hallucinated Secrets

Never make up API keys, project IDs, entitlement IDs, products, or dashboard state. Use configured values, environment/build settings, or stop and ask for the missing input.

## Project-Tailored Context

Inspect the real project before editing. Record the app target, scheme, package manager, app entry point, and existing SDK state. Use the project's current SwiftUI/UIKit patterns instead of inserting generic sample app architecture.

## Leave Useful Guardrails

When the integration cannot be fully completed because credentials are missing, leave a clear build setting or config key and report exactly how to fill it. Do not bury the next step in comments spread through the codebase.

## Verify And Report

Run package resolution and a build whenever possible. End with a short report of changed files, verification status, manual dashboard steps, and what was intentionally not added.

## Future CLI Shape

A future CLI can wrap this skill with:

- `sdk`: SDK-only SPM integration
- `doctor`: diagnose package, configure, capability, and dashboard mismatch symptoms
- `ci`: non-interactive mode using provided flags and no browser/login flow
