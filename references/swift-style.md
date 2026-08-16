# Swift and repository style

Match existing files before applying these defaults. Preserve user-authored formatting in unrelated code.

## File form

Keep the Xcode-style header used by the repository:

```swift
//
//  CreateItemUseCase.swift
//  <package-name>
//
//  Created by <author> on <date>.
//
```

Use the current date and repository author convention for new files when known. Do not rewrite historical headers.

Use conditional Foundation imports in production files that need Foundation values:

```swift
#if canImport(FoundationEssentials)
import FoundationEssentials
#else
import Foundation
#endif
```

Order imports alphabetically by module name after any conditional Foundation block. Put one blank line between a conditional import block and other imports. Do not retain unused imports.

## Layout

- Indent with four spaces; never tabs in Swift.
- Put one declaration per file, except tiny, tightly related conversion extensions.
- Use trailing commas in multiline arrays and manifest dependency lists where the surrounding manifest does.
- Break multiline initializers one argument per line.
- Place the opening brace on the declaration line.
- Keep a type declaration and its qualified protocol conformances on one line; do not split the protocol name or put the opening brace on a separate line.
- Use a single blank line between logical blocks and declarations.
- Avoid trailing whitespace and whitespace-only lines.
- Keep lines readable, but do not mechanically wrap a generic `where` clause if the established code keeps the signature on one line.
- Prefer explicit, descriptive local names: `database`, `postgresClient`, `itemService`, `serverConfig`.

Use generic constraints in this form:

```swift
func withConnection<T: Sendable>(
    _ operation: @Sendable (Context) async throws -> T
) async throws -> T
```

Do not use `T : Sendable` or move the constraint to a trailing `where` unless required by the compiler.

## Access and concurrency

- Use `package` for declarations shared across targets in one service package.
- Use `private` for stored dependencies and implementation details.
- Use `public` only for packages consumed externally or established monolith module APIs.
- Make protocols and values crossing concurrency boundaries `Sendable`.
- Use actors for mutable in-memory repositories.
- Mark database operation closures `@Sendable`.
- Prefer immutable `let` properties.

## APIs and errors

- Use `callAsFunction` for use cases.
- Keep the input label when established: `useCase(input: input)` in the extracted service.
- Use typed throws for Core use-case protocols and implementations.
- Catch named enum cases directly: `catch ItemRepositoryError.duplicateIdentifier`.
- End with a deliberate catch-all mapping when the public typed error includes `.unknown`.
- Do not log in Core use cases unless logging is itself an explicit application port; log at executable, transport, or infrastructure boundaries.

## Formatting verification

Use this `.swift-format` baseline when the repository does not already provide one:

```json
{
    "version": 1,
    "indentConditionalCompilationBlocks": false,
    "lineLength": 400,
    "indentation": {
        "spaces": 4
    }
}
```

`indentConditionalCompilationBlocks` must remain `false` so conditional
`FoundationEssentials` and `Foundation` imports stay flush-left.

Use the repository's formatter if it has configuration or a formatting command. Otherwise, inspect changed Swift files and use `swift format lint --strict` only if the installed Swift toolchain and existing project support it. Do not introduce a new formatting tool or reformat unrelated files during extraction.
