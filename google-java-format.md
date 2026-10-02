# GOOGLE JAVA FORMAT

## Official Google Java Format Reference

This document defines the operational reference for using **google-java-format** as the mechanical Java formatting baseline for the `JAVA FORMAT ULTIMATE` agent.

The official project describes `google-java-format` as a program that reformats Java source code to comply with **Google Java Style**. The formatter deliberately does not expose configurability for its formatting algorithm, with the goal of maintaining a single unified format. citeturn0view0

Official repository:
https://github.com/google/google-java-format

---

## 1. IDENTITY

**Tool:** `google-java-format`

**Purpose:** Format Java source code according to Google Java Style.

**Role in JAVA FORMAT ULTIMATE:**

```text
GOOGLE JAVA FORMAT
        ↓
Mechanical / structural formatting baseline
        ↓
JAVA PROFESSIONAL SEMANTIC FORMATTER
        ↓
Semantic grouping + minimal breaking + preservation rules
```

Google Java Format is the **base formatting reference**, not the complete semantic policy of `JAVA FORMAT ULTIMATE`.

---

## 2. CORE PRINCIPLE

Use Google Java Format for the mechanical formatting conventions it already defines.

Do not attempt to reproduce or override its internal formatting algorithm with arbitrary IntelliJ or `.editorconfig` properties.

The official project explicitly states that its formatter algorithm is not configurable. citeturn0view0

---

## 3. INDENTATION

The standard Google Java Format uses:

```text
2 spaces
```

The official project distinguishes this from its AOSP formatter variant, which uses 4 spaces. citeturn0view0

For `JAVA FORMAT ULTIMATE`, use the standard Google Java Format convention:

```text
indent = 2 spaces
tabs = prohibited
```

---

## 4. GOOGLE JAVA STYLE

The formatter's purpose is to make Java source comply with Google Java Style.

Use Google Java Format as the authoritative mechanical reference for:

- indentation;
- wrapping;
- braces;
- continuation;
- expressions;
- method calls;
- parameters;
- streams;
- lambdas;
- comments/Javadoc;
- imports;
- other Java formatting structures.

The semantic agent may add additional presentation rules, but must not change Java code semantics.

---

## 5. NO CUSTOM FORMATTER ALGORITHM

Do not invent a replacement algorithm and call it Google Java Format.

The distinction is:

```text
Google Java Format
    = Google-defined formatter

JAVA FORMAT ULTIMATE
    = Google Java Format baseline
      + semantic formatting policy
      + preservation policy
      + minimal-breaking policy
```

---

## 6. COMMAND-LINE REFERENCE

The official formatter can be invoked with:

```bash
java -jar /path/to/google-java-format-VERSION-all-deps.jar <options> [files...]
```

The formatter can operate on:

- whole files;
- selected lines;
- specific offsets;
- standard output;
- files in-place.

Relevant official options include:

```text
--lines
--offset
--replace
--aosp
--fix-imports-only
--skip-sorting-imports
--skip-removing-unused-import
--skip-reflowing-long-strings
--skip-javadoc-formatting
--dry-run
--set-exit-if-changed
```

The formatter also supports an `@<filename>` argument file. citeturn0view0

---

## 7. FORMATTER VERSION

Do not hard-code a formatter version into the semantic rules unless the implementation requires reproducibility.

When reproducibility is required:

```text
Pin the google-java-format version explicitly.
```

The exact installed version becomes part of the formatter toolchain, not part of the semantic formatting rules.

---

## 8. JAVA RUNTIME

The official project states that the command-line formatter uses the `jdk.compiler` module to parse Java source and therefore must run using a JDK whose version is equal to or newer than the Java language version of the source being formatted.

The project's current minimum Java version is documented as Java 21. citeturn0view0

Do not interpret this as a restriction on the Java version of source code that can conceptually be formatted; it is a runtime/tooling requirement.

---

## 9. INTELLIJ IDEA

An official `google-java-format` IntelliJ plugin is available through the JetBrains plugin repository.

Installation:

```text
IntelliJ IDEA
    ↓
Settings
    ↓
Plugins
    ↓
Marketplace
    ↓
google-java-format
    ↓
Install
```

The plugin is disabled by default.

Enable it through:

```text
Project Settings
    ↓
google-java-format Settings
    ↓
Enable google-java-format
```

When enabled, the plugin replaces the normal:

```text
Reformat Code
Optimize Imports
```

actions. citeturn0view0

---

## 10. INTELLIJ JRE CONFIGURATION

The official plugin documentation states that additional JVM exports may be required for the IDE runtime.

The documented configuration is:

```text
--add-exports=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED
--add-exports=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED
```

The official documentation places these in the IntelliJ custom VM options and requires restarting the IDE afterward. citeturn0view0

---

## 11. ECLIPSE

The official project also provides an Eclipse plugin.

It exposes:

```text
google-java-format
    = 2 spaces

aosp-java-format
    = 4 spaces
```

The standard Google formatter uses 2 spaces. citeturn0view0

---

## 12. LIBRARY USAGE

`google-java-format` can also be embedded into software that generates Java source.

The official Maven dependency uses:

```xml
<dependency>
  <groupId>com.google.googlejavaformat</groupId>
  <artifactId>google-java-format</artifactId>
  <version>${google-java-format.version}</version>
</dependency>
```

Gradle:

```gradle
dependencies {
  implementation 'com.google.googlejavaformat:google-java-format:$googleJavaFormatVersion'
}
```

The primary formatter API is:

```java
com.google.googlejavaformat.java.Formatter
```

For example:

```java
String formattedSource = new Formatter().formatSource(sourceString);
```

These APIs are documented by the official project. citeturn0view0

---

## 13. FORMATTER API ROLE

When `JAVA FORMAT ULTIMATE` is implemented programmatically, Google Java Format may be used as the mechanical formatting engine.

Conceptually:

```text
Java Source
    ↓
Google Java Format
    ↓
Google-formatted Java
    ↓
Semantic post-processing / policy validation
    ↓
JAVA FORMAT ULTIMATE output
```

However, any post-processing must obey the preservation contract.

---

## 14. PRESERVATION REQUIREMENT

Google Java Format is a formatter, not a refactoring engine.

For `JAVA FORMAT ULTIMATE`, the preservation contract remains absolute:

```text
NO LOGIC CHANGE
NO BEHAVIOR CHANGE
NO STRUCTURAL REFACTORING
NO RENAMING
NO NEW CODE
NO REMOVED CODE
```

Formatting is presentation only.

---

## 15. IMPORTS

Google Java Format has import-related behavior and official command-line options such as:

```text
--fix-imports-only
--skip-sorting-imports
--skip-removing-unused-import
```

For `JAVA FORMAT ULTIMATE`, the agent's own preservation policy takes precedence if the explicit requirement is to preserve imports exactly.

Do not silently introduce import changes when the semantic formatter contract says imports must remain untouched. citeturn0view0

---

## 16. COMMENTS AND JAVADOC

Google Java Format provides formatting behavior for comments and Javadoc and exposes:

```text
--skip-javadoc-formatting
```

For `JAVA FORMAT ULTIMATE`, comments and Javadoc must remain semantically and textually preserved unless the selected implementation explicitly defines formatting-only whitespace changes as permitted. citeturn0view0

---

## 17. LINE-BASED FORMATTING

Google Java Format supports limiting formatting to specific lines through:

```text
--lines
```

and to specific character offsets through:

```text
--offset
```

This can be useful for editor integrations and incremental formatting workflows. citeturn0view0

---

## 18. DIFF / PATCH FORMATTING

The official project provides:

```text
google-java-format-diff.py
```

for reformatting changed lines in a specific patch.

This is useful for workflows where only modified regions should be considered. citeturn0view0

---

## 19. VALIDATION

The official formatter provides:

```text
--dry-run
--set-exit-if-changed
```

These can be used in CI to detect files that differ from the expected Google Java Format output. citeturn0view0

For `JAVA FORMAT ULTIMATE`, additionally validate:

```text
AST equivalence
+
semantic preservation
+
idempotence
```

---

## 20. IDE ACTION MODEL

With the official IntelliJ plugin enabled:

```text
Ctrl + Alt + L
        ↓
Reformat Code
        ↓
google-java-format
```

The plugin replaces IntelliJ's normal formatting action with Google Java Format when enabled. citeturn0view0

---

## 21. AOSP VARIANT

Google Java Format also has an AOSP mode:

```text
--aosp
```

The official documentation identifies the AOSP formatter as using 4-space indentation.

For `JAVA FORMAT ULTIMATE`, **do not use AOSP mode** unless explicitly requested.

Default:

```text
Google Java Format
2 spaces
```

---

## 22. GOOGLE JAVA FORMAT VS INTELLIJ FORMATTER

Do not treat these as the same formatter.

```text
IntelliJ default formatter
    ≠
Google Java Format
```

When the Google Java Format IntelliJ plugin is enabled, the Google formatter replaces the normal `Reformat Code` behavior. citeturn0view0

---

## 23. GOOGLE JAVA FORMAT VS EDITORCONFIG

Do not create a fictional Google `.editorconfig`.

Google Java Format's formatting algorithm is not controlled by an official Google `.editorconfig`.

Use `.editorconfig` only for generic editor/file settings when needed.

The actual Java formatting behavior comes from `google-java-format`.

---

## 24. GOOGLE JAVA FORMAT VS JAVA FORMAT ULTIMATE

### Google Java Format

```text
Mechanical formatting standard
```

### JAVA FORMAT ULTIMATE

```text
Google Java Format
        +
Semantic formatting rules
        +
Minimal breaking
        +
Semantic blank lines
        +
Semantic grouping
        +
Preservation validation
```

---

# OFFICIAL BASELINE FOR JAVA FORMAT ULTIMATE

Use these Google Java Format characteristics as the mechanical baseline:

```text
Formatter:
    google-java-format

Style:
    Google Java Style

Standard indentation:
    2 spaces

AOSP:
    4 spaces — only when explicitly requested

Tabs:
    Do not use as indentation

Algorithm:
    Not user-configurable

IDE:
    IntelliJ / Android Studio plugin available

Primary action:
    Reformat Code

CLI:
    google-java-format

Validation:
    --dry-run
    --set-exit-if-changed

Incremental:
    --lines
    --offset
    google-java-format-diff.py
```

---

# INTEGRATION CONTRACT

The `JAVA FORMAT ULTIMATE` agent must treat this document as its **Google Java Format baseline**.

Priority:

```text
JAVA FORMAT ULTIMATE preservation rules
            ↓
JAVA FORMAT ULTIMATE semantic rules
            ↓
Google Java Format mechanical conventions
            ↓
Generic editor settings
```

The Google Java Format algorithm must not be falsely represented as configurable.

---

# OFFICIAL SOURCE

Primary source:

https://github.com/google/google-java-format

The official repository states that `google-java-format` reformats Java source to comply with Google Java Style, documents its CLI, IntelliJ plugin, Eclipse plugin, library API, and explicitly states that the formatting algorithm is intentionally not configurable. citeturn0view0

# FINAL RULE

```text
GOOGLE JAVA FORMAT = MECHANICAL BASELINE

JAVA FORMAT ULTIMATE = GOOGLE BASELINE
                     + SEMANTIC FORMAT
                     + MINIMAL BREAKING
                     + PRESERVATION
```
