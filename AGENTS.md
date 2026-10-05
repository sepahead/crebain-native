# crebain-native agent contract

crebain-native is the Bevy-based native CREBAIN application.
It provides 3D tactical visualization, ML drone detection, sensor fusion, and Zenoh transport.
This file is the operating contract for maintainers and coding agents.
[README.md](README.md) owns features and installation.
[CONTRIBUTING.md](CONTRIBUTING.md) owns the external contributor workflow and the commit-message format.

## Authority and workflow

- The owner authorizes agents to commit, push, and merge to `main`.
- `main` has no branch protection. Run the CI commands in [Gates](#gates) before you push a code change.
- Sign every commit. The Git configuration signs with the owner's SSH key.
- Use [Conventional Commits](https://www.conventionalcommits.org/).
- Do not add AI attribution or co-author trailers.
- Releases, tags, and repository settings remain owner actions.
- Preserve unrelated work and another contributor's active scope.

## Gates

CI runs these commands on every push. Run all three before you push a change to code, manifests, or `native/`:

```bash
cargo check --workspace
cargo clippy --workspace -- -D warnings
cargo test --workspace
```

On Linux, CI first installs the Bevy build headers (ALSA, udev, X11, xkbcommon, Wayland),
D-Bus, OpenSSL, and `libappindicator3-dev`. See `.github/workflows/ci.yml` for the exact list.

## Build commands

```bash
# Development
cargo run                        # Run the Bevy app
cargo run --release              # Release build and run

# Type checking
cargo check --workspace          # Type check all crates
cargo check -p crebain-core      # Type check core only
cargo check -p crebain-app       # Type check app only

# Linting
cargo clippy --workspace         # Lint all crates
cargo clippy -- -D warnings      # Lint with warnings as errors

# Testing
cargo test --workspace           # Run all tests
cargo test -p crebain-core       # Run core tests only
cargo test -- --nocapture         # Run tests with stdout

# Build
cargo build --workspace          # Debug build
cargo build --release --workspace # Release build
```

## Code style

- Run `cargo clippy` before committing.
- Use `log::info/warn/error` instead of `println!`.
- Validate all external inputs (paths, user data).
- Use `spawn_blocking` for CPU-intensive operations in async contexts.
- Structure behavior as Bevy ECS systems, resources, and events.
- Use `ResMut` only when mutation is needed. Prefer `Res` for read-only access.
- Derive `Resource` for app state and `Component` for entity data.

## Architecture notes

### Core (`crates/crebain-core/`)

- `common/` - Detection types, NMS, YOLO helpers, COCO labels, error types, path validation
- `inference/` - ML abstraction layer (CoreML, ONNX, CUDA, TensorRT, MLX)
- `coreml.rs`, `onnx_detector.rs` - CoreML and ONNX detector backends
- `sensor_fusion.rs` - Kalman/EKF/UKF/Particle/IMM filters
- `transport/` - Zenoh low-latency transport and broadcast channels

### App (`crates/crebain-app/`)

- `app_state/` - CrebainConfig, AppState, RenderQuality
- `camera/` - Tactical camera (WASD+QE controls, zoom)
- `detection/` - DetectionPlugin, DetectionState, detection loop
- `transport/` - TransportPlugin bridging Zenoh to Bevy events
- `ui/hud/` - Status bar, performance panel, sensor fusion panel
- `ui/top_menu/` - Menu bar (File/View/Detection/Help)
- `viewer/` - Tactical grid, terrain, drones, detection overlay

### Native (`native/`)

- `coreml-ffi/` - Swift/CoreML FFI bridge (macOS)

## Performance guidelines

- Use bounded containers for high-frequency position data.
- Prefer squared distance comparisons. Avoid `sqrt()` where a comparison is enough.
- Memoize derived state to prevent unnecessary recomputations.
- Keep camera feed updates at about 12 FPS (83 ms interval) during processing.
- Use Bevy change detection (`is_changed()`) to avoid redundant work.

## Testing

Tests use Rust's built-in test framework. Place tests in `#[cfg(test)]` modules.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn test_example() { ... }
}
```
