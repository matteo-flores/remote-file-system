# Remote File System – Server
This package contains the Express + TypeScript backend that powers the Remote File System project. It authenticates users via Passport, stores metadata with TypeORM/SQLite and keeps file contents under a configurable root directory (`FS_ROOT`).

## Quick start
```bash
cd server
npm install
npm run dev    # Run with ts-node-dev
```
To produce a compiled build:
```bash
npm run build
npm start
```

## Environment variables
| Variable | Default | Description |
|----------|---------|-------------|
| `PORT`   | `3000`  | HTTP port exposed by the Express server. |
| `FS_ROOT`| `<repo>/server/file-system` | Base directory mirrored by the API. |

## Project structure
```
src/
├── controllers/          # Request handlers (authentication, attributes, file I/O)
├── routes/               # Express route registration
├── entities/             # TypeORM entities (User, Group, File, Path)
├── utilities.ts          # Helper functions shared across controllers
├── data-source.ts        # TypeORM data source definition
└── index.ts              # Application bootstrap
```

Metadata is persisted in `metadata.sqlite`. The `file-system/` directory is created automatically if missing.

## API documentation
The full API reference (request bodies, responses and HTTP status codes) lives in the project root `README.md`.