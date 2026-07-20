# opaque-types

`opaque-types` generates Rust structs with the size and alignment of types from
another crate. It is intended for build scripts that need layout-compatible
opaque storage—for example, when exposing a Rust value by value through an FFI
generator.

The layout must be measured for the compilation target, not the machine running
`build.rs`. `opaque-types` creates a probe crate, builds it for Cargo's `TARGET`,
and reads exported size and alignment constants from the resulting object file.
Target code is never executed.

For each mapping, the crate emits a definition like this:

```rust
#[repr(C, align(8))]
pub struct message_t {
    pub _0: [u8; 32],
}
```

The generated struct reproduces layout only. `opaque-types` does not define
conversions or make transmutation safe. The consumer must establish all
representation invariants and should add compile-time size and alignment
assertions wherever the real and opaque types are converted.

## Usage

Add `opaque-types` as a build dependency:

```toml
[build-dependencies]
opaque-types = "0.1"
```

Call it from `build.rs`:

```rust,no_run
let generated = opaque_types::OpaqueTypes::new("../model")
    .features(["shared-memory", "unstable"])
    .default_features(false)
    .add("model::Message", "message_t")
    .add("model::Header", "header_t")
    .generate()
    .expect("generate opaque types");

// `generate` also writes `$OUT_DIR/opaque_probe/opaque_types.rs`.
println!("generated {} bytes", generated.len());
```

## Parameters

- `OpaqueTypes::new(source_manifest_dir)` takes the directory containing the
  source package's `Cargo.toml`. Relative paths are resolved from the process's
  current directory. The generated probe depends on that package by path.
- `features(iterable)` takes Cargo feature names exactly as written under the
  source package's `[features]` table, such as `"shared-memory"`. Pass separate
  items; package prefixes and whitespace-separated feature strings are not
  accepted.
- `default_features(bool)` controls the probe dependency's Cargo
  `default-features` setting. It is independent of `features` and defaults to
  `true`.
- `add(rust_type, opaque_name)` maps a Rust type expression resolvable from the
  probe crate, such as `"model::Message"` or `"model::Container<u32>"`, to one
  valid Rust identifier used for the generated struct, such as `"message_t"`.
- `cargo_lock(path)` selects the `Cargo.lock` copied into the probe. By default,
  the lockfile comes from the workspace consuming this external crate through
  [`get-cargo-lock`](https://crates.io/crates/get-cargo-lock).
- `build_dir(path)` selects the disposable probe directory. It defaults to
  `$OUT_DIR/opaque_probe`. `generate()` also writes the returned source to
  `<build_dir>/opaque_types.rs`.

## Configure the consuming workspace

Because this crate is normally built from Cargo's registry cache, it cannot
locate the consuming workspace's lockfile by walking its own parent directories.
Configure each consuming workspace once:

```console
cargo install get-cargo-lock
cargo get-cargo-lock install .
cargo check
```

This injects the workspace-local `get-cargo-lock` proxy used by the default
lockfile lookup. Without that setup, `generate()` fails with explanatory setup
instructions. A caller that supplies `cargo_lock(path)` explicitly does not use
the default lookup.

The probe is built with `--offline` and an isolated target directory. This keeps
its dependency resolution tied to the selected lockfile and avoids Cargo target
directory lock contention with the outer build.

## License

Licensed under either Apache-2.0 or MIT, at your option.
