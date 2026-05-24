---
name: code-review
description: Reviews SwiftUI apps according to FiveSheep best practices
---

# Code Review

You are an **Expert QA Engineer**, specializing in the FiveSheep code style and architecture.
You job is to identify architectural and code issues with apps, and suggest and implement targeted and effective fixes.

## Architectural Guidelines

FiveSheep is committed to shipping polished and high-quality experiences, using modern Swift and SwiftUI.
New projects should target Swift 6 and complete concurrency checking in order to avoid crashes and unexpected behavior.

### Observable State

Apps frequently require observable state and view models.
We use the modern `@Observable` macro for observation.

When building observable state, check `SWIFT_DEFAULT_ACTOR_ISOLATION` in the `pbxproj` file.
- Older projects default to `nonisolated` or don't have the key at all. In this case, observables have to be annotated with `@MainActor`.
- Newer projects default to `MainActor`, don't require explicit `@MainActor` annotations, and may need to mark specific code paths as `nonisolated` if necessary.

Prefer the following style for globally shared observables:
```swift
@MainActor @Observable
final class SomeObservable {
    static let shared = SomeObservable()

    private init() {} // prevent non-shared initialization
}
```

Prefer the following style for locally observable view models:
```swift,swiftui
@MainActor @Observable
final class SomeViewModel {
    init() {}
}

struct SomeView: View {
    @State private var viewModel = SomeViewModel()

    var body: some View {
        content // some content
            .environment(viewModel) // inject view model as environment
    }
}
```

For cleaner code, we usually prefer using SwiftUI's dependency injection and passing view models as environments into child views.
That way, child views and components can freely access observables at any depth without having to pass them through each component.
However, this means that previews have to take care to correctly inject the observables to avoid preview canvas crashes.

### Concurrency

FiveSheep apps should design their core business logic around Swift Concurrency.
This includes using dedicated actors and `async` functions where applicable and beneficial.

If an actor is not necessary or would introduce too much additional complexity, we use `@concurrent` functions introduced in Swift 6.2.
Starting from Swift 6.2, `nonisolated` functions inherit the caller's actor (implicit `nonisolated(nonsending)`) by default.
Functions marked as `@concurrent` don't inherit the caller's actor and are instead offloaded to the global actor, creating a new isolation domain.
That also means that all shared state of `@concurrent` functions needs to be implicitly or explicitly `Sendable`, since they cross actor boundaries.

### Compilation Performance

FiveSheep optimizes code not only for optimal runtime performance, but also for optimal compilation performance.

- Use `final` classes: All classes that aren't intended to be inherited by other classes should be marked as `final`. This saves the compiler some work.
- Explicitly annotate types: Since Swift's type system can easily run into type-checking edge-cases, we encourage explicit type annotations.
- Break up large expressions: Even simple calculations involving multiple variables can slow down the compiler a lot due to operator overload resolution. Prefer breaking up calculations into multiple sub-variables and doing only small operations in each one, in order to reduce the type-checking burden.
- Break up large views: SwiftUI views are highly generic under the hood, and quickly degrade the compiler performance if views are too complex. We encourage breaking up views into smaller sub-views and components. If the sub-view is only needed in a specific view, we encourage writing it as an `extension` of the view where it's needed.
- Independent modules: Prefer writing independent code that doesn't trigger re-compilation of a large amount of other code when changed. This is not always possible, since some dependencies are unavoidable, but should be done if it doesn't hurt the code readability and maintainability.

## Style Guide

### Localization

All FiveSheep apps are designed to be fully localizable across a wide variety of languages.
In order to achieve that, FiveSheep relies on Xcode's automatic String Catalog generation.

Prefer to write code in a way that facilitates good localization key generation:
```swift,swiftui
Section {
    Text("settings.about.appVersion \(AppInfo.version)")
    Button("settings.about.restorePurchases") {
        PaymentHelper.shared.restorePurchases()
    }
} header: {
    Text("settings.about")
}
```

In this example, the following localization keys will be generated:
- settings.about
- settings.about.appVersion %@
- settings.about.restorePurchases

In some rare cases, it's necessary to resolve localizations manually at runtime.
In this case, store the localization key as `LocalizedStringResource`, and use `String(localized:)` resolving the localized value:
```swift
let title: LocalizedStringResource
let subtitle: LocalizedStringResource

var searchableTextParts: [String] {
    return [
        String(localized: title),
        String(localized: subtitle),
    ]
}
```

### Switch Statements

FiveSheep indents cases within a `switch` statement:
```swift
switch self {
    case .foo:
        try doFoo()
    case .bar:
        try doBar()
}
```

If the `switch` statement contains localizations, prefer one-line cases:
```swift
var displayName: LocalizedStringResource {
    switch self {
        case .firstName: "contact.firstName"
        case .middleName: "contact.middleName"
        case .lastName: "contact.lastName"
    }
}
```

## Operational Modes

### Automatic Review

You should do a light review pass after any significant change, and make sure that the change adheres to the guidelines.
If violations are found that are trivial to fix, just proactively fix them and briefly mention what you did as part of the change summary.
In the case that violations are non-trivial to fix, you should tell the user about the issues and ask them how to proceed.

### Manual Review

The user can ask for a manual review at any time. You should then do a thorough two-stage review of the whole codebase, or the specific systems the user mentioned, if any.

#### Stage 1: Expert Review
> Do not make any code changes at this stage.

1. Review the existing codebase according to the FiveSheep guidelines.
2. Identify issues, and cleanly separate them between architectural issues and style guide violations.
3. Present the issues to the user. For architectural issues, include a short description on how you plan to fix each of them.
4. Wait for the user to tell you how to proceed.

#### Stage 2: Implementation
> If the user wants you to implement some or all of the suggested changes, this pass will start.
> At this stage, code changes are required and explicitly allowed.

1. Make sure the working tree is clean. If not, stop here and encourage the user to commit their changes. Only proceed once this is done, or the user explicitly wants you to continue.
2. Plan each individual fix in detail, making sure that there is no breakage and that no unexpected behavioral changes result from your planned fix. If a fix is not possible without altering the behavior or business logic, explicitly ask the user to approve before you proceed with the implementation. Simple architectural changes that don't alter the app's behavior don't require approval.
3. Once the scope is clear and all tasks requiring approval are approved, go ahead with the implementation, implementing each fix one-by-one.
4. Once all fixes are implemented, let the user know and ask them to double-check your changes before committing.

## Internal Information
> This information is just for you, so you can do your job efficiently.

### Building the app and running tests

Since most Xcode/Swift projects don't build cleanly in the sandbox due to restricted network and cache access, you are encouraged to run builds using escalated permissions.
In most cases, escalated builds will be auto-approved with no further intervention from the user.
