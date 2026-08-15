# 07 · Deployment with Docker

Rust compiles to a single native binary with no runtime to ship alongside
it — no interpreter, no VM, often no dynamic library dependencies beyond
the OS's own libc. That makes it one of the best-suited languages for
small, fast-starting container images. This module looks at what that
binary actually contains, why multi-stage builds matter, and the specific
Docker patterns that go wrong for Rust services.

## What actually ships: inspecting the binary

```bash
cargo build --release
ls -lh target/release/docker-demo
file target/release/docker-demo
otool -L target/release/docker-demo   # ldd on Linux
```

```text
-rwxr-xr-x@ 1 bhanuja  wheel   421K Aug 15 14:57 target/release/docker-demo
target/release/docker-demo: Mach-O 64-bit executable arm64
target/release/docker-demo:
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1356.0.0)
```

A `println!`-only "hello world" binary is 421K in release mode, linked
against exactly one system library. `otool -L` (macOS) / `ldd` (Linux) is
the tool to actually check this rather than assume it — a binary using
`bundled` rusqlite or OpenSSL bindings pulls in more; a binary built against
`musl` instead of `glibc` on Linux can end up with *zero* dynamic
dependencies, which is what makes Rust binaries able to run in
`FROM scratch` containers with no OS at all.

## Stripping debug symbols shrinks it further

```bash
strip target/release/docker-demo
ls -lh target/release/docker-demo
```

```text
-rwxr-xr-x@ 1 bhanuja  wheel   334K Aug 15 15:03 target/release/docker-demo
```

421K → 334K just from removing debug symbols the running binary doesn't
need. In `Cargo.toml`, this can be automated at build time instead of a
separate `strip` step:

```toml
[profile.release]
strip = true
lto = true
codegen-units = 1
```

`lto = true` (link-time optimization) and `codegen-units = 1` trade slower
release builds for smaller, faster binaries — worth it for a service image
that's built once in CI and run many times, not worth it for local
iteration, which is why these settings live under `[profile.release]`
specifically and don't affect `cargo build` (debug) at all.

## Multi-stage Dockerfile

```dockerfile
# ---- build stage ----
FROM rust:1.97-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release

# ---- runtime stage ----
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*
RUN useradd -m appuser
COPY --from=builder /app/target/release/docker-demo /usr/local/bin/docker-demo
USER appuser
ENV PORT=8080
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/docker-demo"]
```

The build stage has the full Rust toolchain (~1.5GB+ of image); the runtime
stage starts from a minimal Debian base and copies in *only the compiled
binary* — `COPY --from=builder` is what makes this "multi-stage": nothing
from the builder stage except that one file ends up in the final image. The
final image is small because it never contains `rustc`, `cargo`, source
code, or the intermediate build artifacts — those all stay behind in a
discarded intermediate layer.

## Rust-specific traps

**Rebuilding dependencies on every source change without a caching
layer.** `COPY src ./src` before `cargo build` means any source edit
invalidates Docker's cache for that layer — but naively, `Cargo.toml` /
`Cargo.lock` were already copied first, so dependency compilation (usually
the slow part) is still cached separately as long as those files didn't
change. Skipping that ordering (copying everything in one `COPY . .`)
forces a full dependency rebuild on every single source edit, which is the
single biggest Docker-Rust build-time mistake.

**`FROM scratch` needs a genuinely static binary.** Compiling normally on
Linux links against `glibc` dynamically — a `FROM scratch` image (literally
empty, no OS at all) will fail at container startup with "no such file or
directory" even though the binary exists, because the dynamic linker itself
is missing. `FROM scratch` requires building against the `musl` target
(`rustup target add x86_64-unknown-linux-musl`, `cargo build --target
x86_64-unknown-linux-musl`) to get a fully static binary first.

**Running as root in the container by default.** A `Dockerfile` with no
`USER` directive runs the process as root inside the container — Rust's
memory safety doesn't protect against a container escape or a
misconfigured volume mount. `RUN useradd` + `USER appuser` above costs
nothing and removes a whole category of container-escape severity.

**`ENV PORT=8080` baked into the image vs. read at runtime.** The example
binary reads `PORT` via `std::env::var` at process startup, so the `ENV`
line in the Dockerfile is just a *default* — `docker run -e PORT=9000
image` still overrides it. Baking a value into the binary at compile time
(e.g., via `env!("PORT")`, a compile-time macro) would require rebuilding
the image to change it — a common confusion between compile-time and
runtime configuration.

## Cheat sheet

| Concern | Fix |
|---|---|
| Slow rebuilds on every source change | Copy `Cargo.toml`/`Cargo.lock` and build deps before copying `src/` |
| Large final image | Multi-stage build, copy only the compiled binary |
| Binary size | `strip = true`, `lto = true` in `[profile.release]` |
| `FROM scratch` fails at runtime | Build against `x86_64-unknown-linux-musl` for a static binary |
| Running as root | `useradd` + `USER` directive in the Dockerfile |
| Runtime vs. compile-time config | `std::env::var` (runtime) vs. `env!()` macro (compile-time, baked in) |

## Exercise

Add a `HEALTHCHECK` instruction to the Dockerfile that curls an `/healthz`
endpoint (reuse the health-check server from module 04) every 10 seconds,
and confirm with `docker inspect --format='{{json .State.Health}}'
<container>` that Docker reports the container as `healthy` once it starts
responding. If you don't have Docker available locally, write out the exact
`HEALTHCHECK` instruction and explain in a comment what each of its flags
(`--interval`, `--timeout`, `--retries`) controls.
