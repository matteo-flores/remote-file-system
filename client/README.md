# Remote File System – Client workspace
Rust workspace that hosts the CLI application and the supporting crates required to mount the remote filesystem on Linux, macOS, and Windows.

## Prerequisites
- Rust toolchain (Sespite the stable channel is recommended, in windows nigthly features are mandatory)
- Platform FUSE provider
  - Linux: [fuser]https://github.com/cberner/fuser
  - macOS: [macFUSE](https://osxfuse.github.io/)
  - Windows: [WinFSP](https://winfsp.dev/)
- On Windows add the WinFSP binaries to `PATH` before launching the CLI (`$env:PATH += ";C:\\Program Files (x86)\\WinFsp\\bin"`).

## Workspace layout
| Crate | Target | Description |
| --- | --- | --- |
| `rfs-cliApp` | Binary | User-facing CLI that authenticates, mounts, and manages the remote filesystem. |
| `rfs-api` | Library | HTTP client (Reqwest) + session handling for the server API. |
| `rfs-models` | Library | Shared data structures, error enums, streaming helpers. |
| `rfs-cache` | Library (Unix only) | In-memory cache used by the FUSE implementation. |
| `rfs-fuse` | Library (Unix only) | FUSE integration (`fuser`) to mount the remote filesystem. |
| `rfs-winfsp` | Library (Windows only) | WinFSP bindings and helpers to present a drive on Windows. |

Default workspace members (`cargo build` / `cargo test`) include `rfs-api`, `rfs-cliApp`, and `rfs-models`. Platform-specific crates are compiled automatically when targeting Unix or Windows.

## Build & run
```bash
cd client
cargo run -- --help
```

The CLI parameters (see `--help`):
- `-m, --mount-point <PATH>` – local mount point/drive (defaults vary per OS).
- `-r, --remote-address <URL>` – server base URL (defaults to `http://fzucca.com:25570`).
- `-s, --speed-testing` – enable throughput logging on Unix (writes to `/tmp/remote-fs.speed-test.out`).

During startup the CLI performs an interactive authentication flow via `rfs-api`, launches the async runtime, and mounts the filesystem through the appropriate backend (`rfs-fuse` on Unix, `rfs-winfsp` on Windows). Linux builds daemonize automatically and drop PID/log files under `/tmp/`.