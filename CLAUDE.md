# CLAUDE.md

This file provides guidance to Claude Code when working with this Tauri fork.

## Fork Purpose

This fork adds **webview z-order control** and **hit-test passthrough** for the eidolon browser project. It integrates with a companion WRY fork at `/Volumes/Vault/Workspace/wry`.

## What Was Changed

### 1. `tauri-runtime` crate (`crates/tauri-runtime/src/lib.rs`)

Added types:
```rust
pub enum HitTestMode { Normal, RegionBased, PassThrough }
pub struct HitRegionId(pub u64);
```

Added to `WebviewDispatch` trait:
```rust
// Z-order control
fn bring_to_front(&self) -> Result<()>;
fn send_to_back(&self) -> Result<()>;

// Hit-test passthrough
fn set_hit_test_mode(&self, mode: HitTestMode) -> Result<()>;
fn hit_test_mode(&self) -> HitTestMode;
fn set_hit_regions(&self, regions: Vec<Rect>) -> Result<()>;
fn add_hit_region(&self, bounds: Rect) -> Result<HitRegionId>;
fn remove_hit_region(&self, id: HitRegionId) -> Result<()>;
fn clear_hit_regions(&self) -> Result<()>;
```

### 2. `tauri-runtime-wry` crate (`crates/tauri-runtime-wry/src/lib.rs`)
- Added `BringToFront`, `SendToBack` message variants
- Added `SetHitTestMode`, `HitTestMode`, `SetHitRegions`, `AddHitRegion`, `RemoveHitRegion`, `ClearHitRegions` message variants
- Implemented dispatcher methods that send these messages
- Added message handlers that call WRY's APIs
- **Changed `wry` dependency to local path**: `path = "../../../wry"`

### 3. `tauri` crate (`crates/tauri/src/webview/mod.rs`)
Added methods to `Webview`:
```rust
// Z-order
pub fn bring_to_front(&self) -> crate::Result<()>;
pub fn send_to_back(&self) -> crate::Result<()>;

// Hit-test passthrough
pub fn set_hit_test_mode(&self, mode: HitTestMode) -> crate::Result<()>;
pub fn hit_test_mode(&self) -> HitTestMode;
pub fn set_hit_regions(&self, regions: Vec<tauri_runtime::dpi::Rect>) -> crate::Result<()>;
pub fn add_hit_region(&self, bounds: tauri_runtime::dpi::Rect) -> crate::Result<HitRegionId>;
pub fn remove_hit_region(&self, id: HitRegionId) -> crate::Result<()>;
pub fn clear_hit_regions(&self) -> crate::Result<()>;
```

Also added to `WebviewWindow` (`crates/tauri/src/webview/webview_window.rs`).

Re-exported types: `pub use tauri_runtime::{HitRegionId, HitTestMode};`

## Usage in eidolon

In your Tauri app's Rust backend:

```rust
use tauri::webview::HitTestMode;

// Z-order control
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

// Hit-test passthrough (for overlay UIs)
#[tauri::command]
fn set_overlay_passthrough(app: tauri::AppHandle, label: String, enabled: bool) -> Result<(), String> {
    if let Some(webview) = app.get_webview(&label) {
        let mode = if enabled { HitTestMode::PassThrough } else { HitTestMode::Normal };
        webview.set_hit_test_mode(mode).map_err(|e| e.to_string())
    } else {
        Err(format!("Webview '{}' not found", label))
    }
}
```

In your React/TypeScript frontend:

```typescript
import { invoke } from '@tauri-apps/api/core';

// Z-order control
await invoke('bring_webview_to_front', { label: 'content-tab-1' });
await invoke('send_webview_to_back', { label: 'content-tab-1' });

// Hit-test passthrough
await invoke('set_overlay_passthrough', { label: 'hud', enabled: true });
```

## Hit-Test Modes Explained

- **Normal**: Default. All mouse events are captured by the webview.
- **RegionBased**: Only capture events in explicitly defined regions. Clicks outside regions pass through to views below. Useful for UI with specific interactive areas.
- **PassThrough**: All mouse events pass through to views below. Useful for purely visual overlays.

## Build Commands

```bash
# Build the runtime crate (fastest way to test changes)
cargo build -p tauri-runtime-wry

# Build full tauri crate
cargo build -p tauri

# Build all relevant crates
cargo build -p tauri-runtime -p tauri-runtime-wry -p tauri
```

## Platform Support

| Platform | Z-Order | Hit-Test |
|----------|---------|----------|
| **macOS** | ✅ Working | ✅ Working |
| Windows | Stub (no-op) | Stub (no-op) |
| Linux | Stub (no-op) | Stub (no-op) |
| iOS | Stub (no-op) | Stub (no-op) |
| Android | Not supported | Not supported |

## Related Repositories

- **WRY fork**: `/Volumes/Vault/Workspace/wry` - Contains the actual implementations
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
