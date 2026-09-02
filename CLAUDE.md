# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Graylog plugin that sends alert notifications in IDMEFv2 (Intrusion Detection Message Exchange Format v2, draft 08-Dev) format to configurable HTTP(S) endpoints. Requires Graylog 7.0+, Java 21+, Maven 3.

## Build Commands

```bash
# Full build (Java + React web interface)
mvn package

# Java only (skip web build — much faster for backend-only changes)
mvn package -Dskip.web.build=true

# Same as above (used in build.sh)
mvn compile -Dskip.web.build=true

# Package formats
mvn jdeb:jdeb     # .deb package
mvn rpm:rpm       # .rpm package

# Release
mvn release:prepare
mvn release:perform
```

## Critical Build Requirement: Parent POM

The `pom.xml` declares a local `relativePath` for the parent POM:

```xml
<relativePath>../graylog2-server/graylog-plugin-parent/graylog-plugin-web-parent</relativePath>
```

This requires a local clone of `graylog2-server` at the **sibling path** `../graylog2-server/` (i.e., both repos checked out side-by-side under the same parent directory). Without it, Maven falls back to downloading `org.graylog.plugins:graylog-plugin-web-parent:7.2.0-SNAPSHOT` from the snapshot repository — which may fail if the snapshot isn't published there.

If you hit a "Non-resolvable parent POM" error, clear Maven's failure cache and force an update:
```bash
mvn package -U
```

Or clone graylog2-server at the expected path:
```bash
cd ..
git clone https://github.com/Graylog2/graylog2-server.git
```

## Architecture

This is a Graylog **AlarmCallback** plugin. The flow when a Graylog alert fires:

1. `Idmefv2Alert.call()` is invoked by the Graylog alerting framework
2. Extracts stream name, alert condition title, and matching message count from the callback context
3. Builds an `Idmefv2Message` POJO (with UUID, ISO timestamps, org info, analyzer metadata)
4. Serializes to JSON via Jackson and HTTP POSTs to the configured URL
5. No retry logic — throws `AlarmCallbackException` if the HTTP response is not 2xx

### Key source files

| File | Role |
|------|------|
| `src/main/java/org/idmefv2/Idmefv2Alert.java` | Core callback — config fields, `call()`, `buildIdmefv2Message()`, `sendToHttp()` |
| `src/main/java/org/idmefv2/Idmefv2Message.java` | IDMEFv2 data model with Jackson `@JsonProperty` annotations; inner `Analyzer` class |
| `src/main/java/org/idmefv2/Idmefv2AlertModule.java` | Guice module — registers `Idmefv2Alert` as an `AlarmCallback` |
| `src/main/java/org/idmefv2/Idmefv2AlertPlugin.java` | Plugin entry point |
| `src/web/index.jsx` | React plugin registration (minimal — no routes defined) |

### Plugin configuration fields (set via Graylog UI)

- `CK_URL` — HTTP(S) endpoint (required, validated as a URL)
- `CK_ORGANIZATION_NAME` — defaults to `"Graylog"`
- `CK_ORGANIZATION_ID` — defaults to `"graylog"`

### Graylog plugin wiring

- `META-INF/services/org.graylog2.plugin.Plugin` → `Idmefv2AlertPlugin`
- The shade plugin bundles all non-`provided` dependencies into the output JAR
- The JAR manifest entry `Graylog-Plugin-Properties-Path` points to the properties file for version discovery

## Web Interface

The React entry point (`src/web/index.jsx`) is a near-empty template — it registers the plugin manifest but has no routes or UI components. Webpack output lands in `target/web/`. The `graylog-web-plugin` dev dependency is resolved from the local `../graylog2-server/` clone.

## No Tests

There are currently no Java or JavaScript tests in this repository.
