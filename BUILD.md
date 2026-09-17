# Building Pipali-FreeSearch From Source

This document describes how to build Pipali-FreeSearch from a clean source checkout, with an emphasis on the Tauri desktop application and Linux AppImage.

Pipali-FreeSearch uses **Bun** for the application/server build and **Tauri 2** for desktop packaging. The repository contains a dedicated Tauri build script because the desktop build is more than a plain `tauri build`: it prepares the bundled Bun/UV runtimes, builds the server resources, applies Linux/AppImage compatibility workarounds, and repacks the Linux AppImage with pristine sidecar binaries.

## 1. Prerequisites

### Required

- Git
- Bun 1.2.19 or newer
- Rust/Cargo and the Tauri 2 Linux prerequisites when building the desktop application
- A working C/C++ build environment appropriate for your platform

For Tauri's platform-specific system requirements, see the [Tauri prerequisites](https://v2.tauri.app/start/prerequisites/).

### Linux build tools

The Pipali-FreeSearch Tauri build script downloads the Bun and UV runtime archives itself, but it expects common command-line tools to be available, including:

- `curl`
- `unzip`
- `tar`
- GNU `strip`

The Linux AppImage build also uses the Tauri/AppImage tooling prepared by the build script.

On an Arch/EndeavourOS system, make sure your normal development toolchain and Tauri/WebKit dependencies are installed before starting the build. If Tauri reports a missing system library, install the package corresponding to the library named in the error.

## 2. Clone the repository

Clone Pipali-FreeSearch and enter the checkout:

```bash
git clone https://github.com/michieal/pipali-freesearch.git
cd pipali-freesearch
```

If you are working from a fork or a development branch, check out the desired branch before building.

> **Bun and symbolic links:** If Bun reports resolver errors that appear to be related to paths or symlinks, move the checkout to a normal local directory. A non-symlinked path avoids a known class of Bun resolver problems.

## 3. Install dependencies

Install the JavaScript/TypeScript dependencies with Bun:

```bash
bun install
```

The repository's `package.json` contains the development dependencies, Tauri CLI, frontend dependencies, and server dependencies needed by the build.

## 4. Configure the environment (optional)

For normal hosted Pipali usage, no provider API key is required. If you need local development configuration or direct model-provider access, copy the example environment file:

```bash
cp .env.example .env
```

Then edit `.env` as needed.

For example, a locally hosted OpenAI-compatible model can be configured with:

```bash
OPENAI_BASE_URL=http://127.0.0.1:11434/v1
OPENAI_API_KEY=ollama
```

See `.env.example` for the available application, database, sandbox, search-provider, and model-provider settings.

## 5. Build the Linux AppImage

### Recommended Linux build

For a standard x86-64 Linux build, use the repository's Tauri build script:

```bash
bun run tauri:build:linux-x64 --no-sign
```

This is the **recommended command for building a distributable Linux AppImage**.

The build script performs the complete desktop build pipeline:

1. Detects/prepares the target platform.
2. Downloads the matching Bun runtime.
3. Downloads the bundled UV/UVX runtime.
4. Copies those runtimes into `src-tauri/binaries/` using Tauri's sidecar naming convention.
5. Builds the frontend.
6. Bundles the Bun/Hono server into Tauri resources.
7. Copies migrations, builtin skills, frontend assets, and other required resources.
8. Installs the external runtime dependencies that cannot be bundled into the server bundle.
9. Applies the Pipali-FreeSearch Linux/AppImage compatibility fixes.
10. Runs the Tauri AppImage bundler.
11. Re-extracts and repacks the resulting AppImage with the pristine Bun/UV/UVX sidecars.
12. Verifies the bundled Bun runtime before completing the build.

The resulting AppImage is placed in:

```text
src-tauri/target/release/bundle/appimage/
```

### Why not just run `bunx tauri build`?

You can invoke Tauri directly, but **do not use that as the normal Pipali-FreeSearch distribution build**.

```bash
bunx tauri build
```

This bypasses `scripts/build-tauri.ts`, which is responsible for preparing Pipali's bundled runtimes and desktop resources and for applying the Linux/AppImage post-processing. A directly invoked Tauri build can therefore produce an AppImage that launches but cannot start the embedded Pipali server correctly.

The repository's intended entry point is:

```bash
bun run tauri:build
```

or, for an explicit Linux x86-64 target:

```bash
bun run tauri:build:linux-x64 --no-sign
```

## 6. AppImage signing

For Linux AppImages, use `--no-sign` unless you have deliberately configured your own Tauri updater signing key.

```bash
bun run tauri:build:linux-x64 --no-sign
```

The build script accepts `--no-sign` as an alias for disabling updater artifacts/signing. This is appropriate for Pipali-FreeSearch distributions because the fork does not possess the upstream project's private signing credentials.

If you intentionally configure your own signing key, the build script can preserve/re-sign the AppImage during its Linux repackaging step. Do **not** use upstream signing credentials.

## 7. Debug desktop build

To create a debug Tauri build:

```bash
bun run tauri:build:debug
```

For Linux, the script detects the current platform and produces the corresponding debug AppImage under:

```text
src-tauri/target/debug/bundle/appimage/
```

You can also invoke the underlying script directly with `--debug`:

```bash
bun run scripts/build-tauri.ts --debug
```

## 8. Development mode

For normal server/web development:

```bash
bun run dev
```

Then open:

```text
http://localhost:6464
```

For the Tauri development application:

```bash
bun run tauri:dev
```

The development Tauri command is different from the release packaging command. `tauri:dev` is intended for an interactive development cycle; `tauri:build` is the distribution build.

## 9. Other platforms

The Tauri build script contains platform support for:

- `darwin-arm64`
- `darwin-x64`
- `linux-x64`
- `linux-arm64`
- `windows-x64`

The generic Tauri build command automatically detects the current platform:

```bash
bun run tauri:build
```

The platform-specific script currently exposed in `package.json` for Linux is:

```bash
bun run tauri:build:linux-x64 --no-sign
```

For targets not exposed as a package script, invoke the build script directly. For example, on Linux ARM64:

```bash
bun run scripts/build-tauri.ts --platform=linux-arm64 --no-sign
```

Cross-platform builds may require the appropriate Rust target and platform SDK/toolchain in addition to the normal Tauri prerequisites.

## 10. Build outputs

### Standalone Bun executable

The plain application build command:

```bash
bun run build
```

runs `scripts/build.ts`. It produces a standalone Bun executable at:

```text
dist/pipali
```

This is **not** the Tauri desktop/AppImage build.

### Tauri desktop build

The Tauri build produces platform-specific artifacts under:

```text
src-tauri/target/release/bundle/
```

On Linux, the distributable AppImage is under:

```text
src-tauri/target/release/bundle/appimage/
```

## 11. Testing the AppImage

After a successful Linux build, list the generated artifacts:

```bash
ls -lh src-tauri/target/release/bundle/appimage/
```

Make the AppImage executable if necessary:

```bash
chmod +x src-tauri/target/release/bundle/appimage/*.AppImage
```

Then run it:

```bash
src-tauri/target/release/bundle/appimage/*.AppImage
```

A successful launch should proceed past the initial loading/splash screen and start the embedded Pipali server.

If the application remains indefinitely on **Loading...**, run the AppImage from a terminal so that startup errors are visible. The Tauri desktop application starts Pipali as a bundled sidecar and waits for its health endpoint before displaying the main window; a sidecar startup failure therefore commonly appears as a window that opens but never finishes loading.

## 12. Database

Pipali uses PGlite with Drizzle ORM. The local development database is normally stored in:

```text
./pipali.db/
```

Database commands:

```bash
bun run db:generate
bun run db:migrate
```

The desktop distribution build copies the required Drizzle migrations into its bundled server resources.

## 13. End-to-end tests

Run the Playwright test suite with:

```bash
bun run test:e2e
```

Useful variants are:

```bash
bun run test:e2e:ui
bun run test:e2e:headed
bun run test:e2e:debug
```

## 14. Troubleshooting

### `bun run build` only creates `dist/pipali`

This is expected. `bun run build` invokes `scripts/build.ts`, which creates the standalone Bun executable. It does not package the Tauri desktop application.

Use:

```bash
bun run tauri:build
```

or:

```bash
bun run tauri:build:linux-x64 --no-sign
```

### The AppImage opens but remains on `Loading...`

First verify that the AppImage was built using the Pipali-FreeSearch Tauri build script rather than a direct `bunx tauri build` invocation.

Then launch the AppImage from a terminal and inspect the startup output. The desktop frontend waits for the bundled server sidecar to become healthy before revealing the main application window.

### Bun resolver errors

Move the repository to a normal, non-symlinked local path and run:

```bash
bun install
bun run tauri:build:linux-x64 --no-sign
```

### Tauri reports a missing Linux dependency

Install the system package corresponding to the library named by the error, then rerun the build. Refer to the Tauri prerequisites for the current requirements for your distribution.

### AppImage build fails in `linuxdeploy`

The Linux build script contains Pipali-FreeSearch-specific compatibility handling for known AppImage/linuxdeploy issues. Make sure you are using:

```bash
bun run tauri:build:linux-x64 --no-sign
```

rather than bypassing the wrapper with `bunx tauri build`.

If the failure persists, keep the complete build output when reporting the problem; the exact linuxdeploy error is important for determining which stage failed.

## 15. Quick reference

### Install

```bash
bun install
```

### Run server in development

```bash
bun run dev
```

### Run Tauri in development

```bash
bun run tauri:dev
```

### Build standalone executable

```bash
bun run build
```

Output:

```text
dist/pipali
```

### Build Linux AppImage

```bash
bun run tauri:build:linux-x64 --no-sign
```

Output:

```text
src-tauri/target/release/bundle/appimage/
```

### Build Linux AppImage in debug mode

```bash
bun run tauri:build:debug
```

### Run tests

```bash
bun run test:e2e
```
