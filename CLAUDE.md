# CLAUDE.md

This file provides guidance to Claude Code when working with this Tauri fork.

## Fork Purpose

This fork adds **webview z-order control** for the eidolon browser project. It integrates with a companion WRY fork at `/Volumes/Vault/Workspace/wry`.

## What Was Changed

### 1. `tauri-runtime` crate
Added to `WebviewDispatch` trait (`crates/tauri-runtime/src/lib.rs`):
```rust
fn bring_to_front(&self) -> Result<()>;
fn send_to_back(&self) -> Result<()>;
```

### 2. `tauri-runtime-wry` crate
- Added `BringToFront` and `SendToBack` variants to `WebviewMessage` enum
- Implemented dispatcher methods that send these messages
- Added message handlers that call WRY's `bring_to_front()` / `send_to_back()`
- **Changed `wry` dependency to local path**: `path = "../../../wry"`

### 3. `tauri` crate
Added methods to `Webview` (`crates/tauri/src/webview/mod.rs`):
```rust
pub fn bring_to_front(&self) -> crate::Result<()>;
pub fn send_to_back(&self) -> crate::Result<()>;
```

Also added to `WebviewWindow` (`crates/tauri/src/webview/webview_window.rs`).

## Usage in eidolon

In your Tauri app's Rust backend:

```rust
#[tauri::command]
fn bring_webview_to_front(app: tauri::AppHandle, label: String) -> Result<(), String> {
    if let Some(webview) = app.get_webview(&label) {
        webview.bring_to_front().map_err(|e| e.to_string())
    } else {
        Err(format!("Webview '{}' not found", label))
    }
}

#[tauri::command]
fn send_webview_to_back(app: tauri::AppHandle, label: String) -> Result<(), String> {
    if let Some(webview) = app.get_webview(&label) {
        webview.send_to_back().map_err(|e| e.to_string())
    } else {
        Err(format!("Webview '{}' not found", label))
    }
}
```

In your React/TypeScript frontend:

```typescript
import { invoke } from '@tauri-apps/api/core';

// Bring a content webview to front
await invoke('bring_webview_to_front', { label: 'content-tab-1' });

// Send it back behind UI
await invoke('send_webview_to_back', { label: 'content-tab-1' });
```

## Build Commands

```bash
# Build the runtime crate (fastest way to test changes)
cargo build -p tauri-runtime-wry

# Build full tauri crate
cargo build -p tauri
```

## Platform Support

| Platform | Status |
|----------|--------|
| **macOS** | ✅ Working |
| Windows | Stub (no-op) |
| Linux | Stub (no-op) |
| iOS | Stub (no-op) |
| Android | Not supported |

## Related Repositories

- **WRY fork**: `/Volumes/Vault/Workspace/wry` - Contains the actual z-order implementation
- **eidolon**: `/Volumes/Vault/Workspace/eidolon` - Target application

## To Use This Fork in eidolon

In eidolon's `src-tauri/Cargo.toml`, override the tauri dependency:

```toml
[dependencies]
tauri = { path = "../tauri/crates/tauri", features = [...] }

# Or use patch section:
[patch.crates-io]
tauri = { path = "../tauri/crates/tauri" }
tauri-runtime = { path = "../tauri/crates/tauri-runtime" }
tauri-runtime-wry = { path = "../tauri/crates/tauri-runtime-wry" }
```
