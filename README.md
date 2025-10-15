# Remote File System

A cross-platform remote filesystem for Windows (WinFSP), Linux (fuser), and macOS (macFUSE), backed by a Node.js server and a Rust client. This repository is a university submission for the **API Programming** course.

- Server docs: [`server/README.md`](server/README.md)  
- Client docs: [`client/README.md`](client/README.md)

---

## Table of Contents
1. [Overview](#overview)  
2. [Key Capabilities](#key-capabilities)  
3. [Repository Layout](#repository-layout)  
4. [Quickstart](#quickstart)  
   4.1. [Server](#server)  
   4.2. [Client](#client)  
5. [Runtime Data & Automation Hooks](#runtime-data--automation-hooks)  
6. [API Overview](#api-overview)  
   6.1. [Authentication](#authentication)  
   6.2. [File & Directory Attributes](#file--directory-attributes)  
   6.3. [Directory Operations](#directory-operations)  
   6.4. [Create / Delete](#create--delete)  
   6.5. [Rename / Move](#rename--move)  
   6.6. [Read / Write (binary & streaming)](#read--write-binary--streaming)  
   6.7. [Links & Symlinks](#links--symlinks)  
   6.8. [Filesystem Size](#filesystem-size)  
   6.9. [Types, Permissions, Timestamps](#types-permissions-timestamps)  
   6.10. [Error Responses](#error-responses)  
7. [Notes & Tips](#notes--tips)  
8. [Authors](#authors)

---

## Overview

Remote File System exposes a simple HTTP API that mirrors POSIX-style filesystem semantics. All routes require an authenticated session and operate on inode-addressed entries (files, directories, symlinks). The Rust client mounts the remote view via platform-specific FUSE/WinFSP providers.

For details on setup and development workflows, see:
- **Server:** environment setup, scripts → [`server/README.md`](server/README.md)  
- **Client:** workspace members, build targets, and CLI usage → [`client/README.md`](client/README.md)

---

## Key Capabilities

- **POSIX‑aligned metadata model.** Users, groups, permissions, inodes, and timestamps are modeled after Unix semantics so tooling behaves predictably when mounted.
- **Streaming file access.** Read and write endpoints support streaming to avoid buffering large files in memory and include offset-aware semantics for random access.
- **Provisioning automation.** Special files at the filesystem root (e.g., `/create-user.txt`, `/create-group.txt`) trigger background user/group creation via the HTTP API—useful for bootstrap flows and external integrations.
- **Cross‑platform mounting.** The Rust workspace includes adapters for Linux/macOS (fuser) and Windows (WinFSP) so the same codebase serves heterogeneous fleets.
- **Database flexibility.** Metadata persistence is handled via TypeORM on SQLite. Swapping DB backend service later on requires minimal changes.

---

## Repository Layout

| Path | Description |
| --- | --- |
| `server/` | Express 5 backend (TypeScript): controllers, TypeORM entities, Passport session wiring, and dev scripts. |
| `client/` | Rust workspace with CLI, HTTP bindings, OS-specific FS adapters, and integration utilities. |
| `README.md` | This document; the consolidated API contract and project overview. |
| `server/README.md` | Backend setup, env variables, run/build commands. |
| `client/README.md` | Client build instructions, mount examples, CLI flags. |
| `LICENSE` | MIT license. |

---

## Quickstart

### Server

1) **Install dependencies**
```bash
cd server
npm install
```

2) **Run with hot reload**
```bash
npm run dev
```
Launches `nodemon` + `ts-node` for automatic rebuilds.

3) **Production build (optional)**
```bash
npm run build
npm start
```

4) **Initial login**
- A default administrator (UID `5000`) is created on first run.  
- Check server logs for the generated password, or rotate it by writing to `/create-user.txt` (see [Runtime Data & Automation Hooks](#runtime-data--automation-hooks)).

### Client

Run the CLI and mount a local path (Unix) or a drive letter (Windows):

```bash
# Unix-like systems
cargo run -- -r /path/to/local/dir

# Windows
cargo run -- -r X:
```

(See `client/README.md` for CLI flags, prerequisites, and platform notes.)

---

## Runtime Data & Automation Hooks

At the root of the mounted filesystem, **special provisioning files** can be written to trigger background tasks on the server:

- `/create-user.txt` — creating or rotating users (e.g., admin UID `5000`).  
- `/create-group.txt` — creating groups and assigning members.

These hooks let you automate initial setup without calling the API directly; the server watches for these file writes and executes the corresponding actions.

---

## API Overview

> **Base path:** `/api`  
> **Auth:** Required on all endpoints (session-based).  
> **Default format:** JSON unless otherwise specified.

### Authentication

#### `POST /api/login`

Log in using local authentication (session).

**Body (JSON):**
```json
{ "uid": 5000, "password": "your-password" }
```

**Returns:** Authenticated user ID (e.g., `5000`).

---

#### `POST /api/logout`

Ends the active session.  
**Returns:** `200 OK` with no body.

---

#### `GET /api/me`

Returns the data of the currently authenticated user.

**Example**
```json
{ "uid": 5000 }
```

---

### File & Directory Attributes

#### `GET /api/files/{ino}/attributes`

Returns metadata for a file or directory identified by `ino` (inode).  
Supports **conditional requests** via the `If-Modified-Since` header; if the resource has not changed since that time, the server **may return `304 Not Modified`** (no body), otherwise `200 OK`.

**URL params**
- `ino` (string): inode of the file or directory

**Headers (optional)**
- `If-Modified-Since: <HTTP-date>`

**Success responses**
- `200 OK` + JSON body (see below) and `Last-Modified` header
- `304 Not Modified` (no body)

**Return type (example):**
```json
{
  "ino": "12345",
  "name": "example.txt",
  "kind": 0,
  "size": 1245,
  "uid": 1000,
  "gid": 1000,
  "perm": 420,
  "atime": 1690963200000,
  "mtime": 1690963200000,
  "ctime": 1690963200000,
  "btime": 1690963200000
}
```

---

#### `PATCH /api/files/{ino}/attributes`

Updates one or more attributes (e.g., `perm`, `size`).

**URL params**
- `ino` (string)

**Body (JSON example):**
```json
{ "perm": 644, "size": 1000 }
```

**Returns:** Updated metadata (same shape as above).

---

### Directory Operations

#### `GET /api/directories/{ino}/entries`

Lists entries contained in a directory.

**URL params**
- `ino` (string): inode of the directory

**Returns:** Array of entry objects (files, directories, symlinks).

---

#### `GET /api/directories/{parentIno}/entries/lookup?name={entryName}`

Looks up a specific entry in a directory by name.

**URL params**
- `parentIno` (string)

**Query params**
- `name` (string)

**Returns:** Metadata of the found entry.

---

### Create / Delete

#### `POST /api/directories/{parentIno}/dirs/{name}`

Creates a new directory.

**URL params**
- `parentIno` (string)
- `name` (string)

**Returns:** Metadata of the created directory.

---

#### `DELETE /api/directories/{parentIno}/dirs/{name}`

Deletes an **empty** directory.

**URL params**
- `parentIno` (string)
- `name` (string)

**Returns:** `200 OK` on success.

---

#### `POST /api/directories/{parentIno}/files/{name}`

Creates a new (empty) regular file.

**URL params**
- `parentIno` (string)
- `name` (string)

**Returns:** Metadata of the created file.

---

#### `DELETE /api/directories/{parentIno}/files/{name}`

Deletes a regular file by name.

**URL params**
- `parentIno` (string)
- `name` (string)

**Returns:** `200 OK` on success.

---

### Rename / Move

#### `PATCH /api/directories/{oldParentIno}/entries/{oldName}`

Renames or moves a file/directory.

**URL params**
- `oldParentIno` (string)
- `oldName` (string)

**Body (JSON):**
```json
{ "newParentIno": "67890", "newName": "new_filename.txt" }
```

**Returns:** Updated metadata for the moved/renamed entry.

---

### Read / Write (binary & streaming)

#### `GET /api/files/{ino}`

Reads file contents (binary). Supports optional byte-range style queries.

**URL params**
- `ino` (string)

**Query params (optional)**
- `offset`: byte offset to start reading
- `size`: number of bytes to read

**Returns:** Binary data (body).

---

#### `PUT /api/files/{ino}`

Writes binary content.

**URL params**
- `ino` (string)

**Headers**
- `Content-Type: application/octet-stream`
- `X-Chunk-Offset` (optional): starting byte offset for partial writes

**Body:** Binary payload

**Returns (JSON):**
```json
{ "bytes": 1024 }
```

---

#### `GET /api/files/stream/{ino}`

Streams file contents (optimized for large files).

**URL params**
- `ino` (string)

**Returns:** Streamed binary response.

---

#### `PUT /api/files/stream/{ino}`

Stream-writes into a file (optimized for large uploads).

**URL params**
- `ino` (string)

**Body:** Binary stream

**Returns:** Upload confirmation (implementation-defined).

---

### Links & Symlinks

#### `POST /api/links/{targetIno}`

Creates a **hard link** to an existing file.

**URL params**
- `targetIno` (string)

**Body (JSON):**
```json
{ "linkParentIno": "12345", "linkName": "link_name" }
```

**Returns:** Metadata for the created hard link.

---

#### `POST /api/symlinks`

Creates a **symbolic link**.

**Body (JSON):**
```json
{
  "linkParenIno": "12345",
  "linkName": "symlink_name",
  "targetPath": "/path/to/target"
}
```

**Returns:** Metadata for the created symlink.

---

#### `GET /api/symlinks/{ino}`

Resolves the target path of a symbolic link.

**URL params**
- `ino` (string)

**Returns (JSON):**
```json
{ "target": "/path/to/target" }
```

---

### Filesystem Size

#### `GET /api/size`

Returns total and available filesystem space (**used by Windows clients**).

**Returns (JSON):**
```json
{ "total": 107374182400, "available": 85899345920 }
```

---

### Types, Permissions, Timestamps

- **Entry types:** `0` file, `1` directory, `2` symbolic link  
- **Permissions:** Unix-style octal (e.g., `755`, `644`, `420`)  
- **Timestamps:** millisecond epoch (`Date.now()` style) for `atime`, `mtime`, `ctime`, `btime`

---

### Error Responses

Common error shapes across endpoints:

**401 Unauthorized**
```json
{ "error": "Authentication required" }
```

**403 Forbidden**
```json
{ "error": "Permission denied" }
```

**404 Not Found**
```json
{ "error": "File or directory not found" }
```

**500 Internal Server Error**
```json
{ "error": "Internal server error", "details": "Error message" }
```

---

## Notes & Tips

- All API calls are authenticated; log in first to obtain a session cookie.  
- Prefer the streaming endpoints (`/files/stream`) for large reads/writes to avoid buffering.  
- See component READMEs for environment variables, platform-specific quirks, and additional examples:  
  - [`server/README.md`](server/README.md)  
  - [`client/README.md`](client/README.md)

---

## Authors

Created by **[Andrea](https://github.com/andrea-germano)** and **[Matteo](https://github.com/matteo027)**.
