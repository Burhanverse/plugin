# Repository Agent Guidelines & Workflow Rules

This document outlines mandatory operational rules and conventions for developing, building, and maintaining CloudStream providers in this repository.

---

## 1. Version Bumping Rule (MANDATORY)

Whenever any code change, bug fix, feature addition, or rebuild is made to a provider:

1. **Always Increment the Version Integer** (`version = N + 1`):
   - In `<Provider>/build.gradle.kts`: `version = N`
   - In `<Provider>/src/main/resources/manifest.json`: `"version": N`
   - In root `plugins.json`: `"version": N`
2. **Never deploy or commit code changes without bumping the version.** CloudStream uses this integer to notify users of updates and download updated plugins.

---

## 2. Build & Artifact Workflow

Whenever building or releasing a provider:

1. **Clean Build**:
   ```bash
   cd <ProviderSubmodule>
   ./gradlew clean make
   ```
2. **Copy Artifact to Repository Root**:
   ```bash
   cp <ProviderSubmodule>/build/<ProviderName>.cs3 ./<ProviderName>.cs3
   ```
3. **Compute Metrics**:
   - Compute the exact file size in bytes (`os.path.getsize(...)`).
   - Compute the SHA-256 hash formatted as `sha256-<64_hex_digits>`.
4. **Update `plugins.json`**:
   - Update `version`, `fileSize`, and `fileHash` for that provider entry.
5. **Stage and Commit**:
   - Ensure the updated `<Provider>.cs3`, `plugins.json`, and submodule gitlink are properly staged.

---

## 3. Kotlin & Jackson Deserialization Standards

To prevent runtime `ClassCastException` (`java.util.LinkedHashMap cannot be cast to ...`):

1. **Reified Array Deserialization for Collections**:
   - **Never** use `parsedSafe<List<T>>()` or `tryParseJson<List<T>>()`. Due to JVM generic type erasure on root types, Jackson deserializes elements as `LinkedHashMap`.
   - **Always** use `parsedSafe<Array<T>>()?.toList()` or `tryParseJson<Array<T>>()?.toList()`. Array types preserve their component type in JVM bytecode (`[Lcom.xie...`), guaranteeing strongly typed instances.
2. **Jackson Annotations**:
   - Use `@param:JsonProperty("field_name")` on constructor parameters of data classes to avoid Kotlin compiler annotation-target warnings and ensure exact field mapping.
3. **Defensive Converters**:
   - For link and episode payloads, provide defensive conversion extensions (`Any?.toTargetObject()`) using `.toJson()` fallback to gracefully absorb untyped Maps if passed from previous runtime states.

---

## 4. Code Writing Standards

Keep the codebase clean and organized. Remove comments and stop using comments while writing codes.
