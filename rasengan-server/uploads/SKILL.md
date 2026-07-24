---
name: rasengan-server-uploads
description: Multipart file upload patterns for @rasenganjs/server, backed by @rasenganjs/futon's fileUpload() middleware. Covers fileUpload({ storage, limits, fileFilter }), diskStorage({ destination }) from @rasenganjs/server/upload/disk, .single(fieldName) vs .array(fieldName, maxCount) vs .fields()/.any()/.none(), reading uploaded file(s) via ctx.get<UploadedFile>('file') / ctx.get<UploadedFile[]>('files'), and the UploadedFile shape (fieldname/originalname/mimetype/size/destination/filename/path). Use when adding file/image upload endpoints to a Rasengan server app.
license: MIT
metadata:
  author: Dilane Kombou
  framework: rasengan server
  version: "1.0.0-beta.0"
---

# @rasenganjs/server Upload Patterns

## When to Activate

- Adding a multipart file/image upload endpoint (single or multiple files)
- Choosing between in-memory and on-disk storage for uploaded files
- Enforcing file size/count limits or filtering by mimetype
- Reading an uploaded file's metadata (path, size, original name) inside a route handler

## Basic Setup

`fileUpload()` is Multer-style middleware, imported from `@rasenganjs/server`. `diskStorage()` lives at the `@rasenganjs/server/upload/disk` subpath (kept separate so `node:fs` stays out of non-Node/WinterCG bundles):

```ts
// upload.controller.ts
import {
  Controller,
  fileUpload,
  type RouteHandler,
  type Router,
  type UploadedFile,
} from '@rasenganjs/server';
import { diskStorage } from '@rasenganjs/server/upload/disk';

const upload = fileUpload({
  storage: diskStorage({ destination: 'uploads/' }),
  limits: { fileSize: 5 * 1024 * 1024, files: 3 },
  fileFilter: (_ctx, info) => !info.mimetype.includes('exe'),
});

export class UploadController extends Controller {
  routes(router: Router) {
    router.post('/upload/avatar', upload.single('avatar'), this.avatar);
    router.post('/upload/gallery', upload.array('photos', 3), this.gallery);
  }

  avatar: RouteHandler = async (ctx) => {
    const file = ctx.get<UploadedFile>('file');
    if (!file) {
      return ctx.res.status(400).json({ error: 'No file on field "avatar".' });
    }
    return ctx.res.json({ ok: true, file });
  };

  gallery: RouteHandler = async (ctx) => {
    const files = ctx.get<UploadedFile[]>('files') ?? [];
    return ctx.res.json({
      ok: true,
      count: files.length,
      files: files.map((f) => ({ originalname: f.originalname, path: f.path })),
    });
  };
}
```

Try it:

```bash
curl -F avatar=@some.png http://localhost:3006/upload/avatar
curl -F photos=@a.jpg -F photos=@b.jpg http://localhost:3006/upload/gallery
```

## `fileUpload()` Options

```ts
fileUpload({
  storage,     // defaults to MemoryStorage (in-memory, ctx buffer only)
  limits: {
    fileSize: 5 * 1024 * 1024, // bytes, per file
    files: 3,                   // max files per request
    fields: 10,                 // max non-file text fields
  },
  fileFilter: (ctx, info) => boolean | Promise<boolean>, // reject -> 400 FILE_FILTER_REJECTED
});
```

`fileFilter` receives a `FileInfo` (`{ fieldname, originalname, mimetype, size }`) **before** the file is stored — return `false` to reject with a 400 response.

## `Uploader` Methods

`fileUpload()` returns an `Uploader` with five middleware factories:

| Method | Populates | Use for |
|--------|-----------|---------|
| `upload.single(field)` | `ctx.get('file')` → `UploadedFile` | Exactly one file on `field` |
| `upload.array(field, maxCount?)` | `ctx.get('files')` → `UploadedFile[]` | Multiple files on one `field` |
| `upload.fields([{ name, maxCount? }, ...])` | `ctx.get('files')` → `Record<string, UploadedFile[]>` | Multiple named fields, each with its own file(s) |
| `upload.any()` | `ctx.get('files')` → `UploadedFile[]` | Accept every file regardless of field name |
| `upload.none()` | — | Text fields only; any file present is rejected |

Non-file fields in the same multipart request populate `ctx.body` as plain strings (like Multer's `req.body`).

## `diskStorage()`

```ts
import { diskStorage } from '@rasenganjs/server/upload/disk';

diskStorage({
  destination: 'uploads/', // string, or (ctx, info) => string | Promise<string>
  // filename defaults to random hex + sanitized extension of originalname
  filename: (ctx, info) => `${Date.now()}-${info.originalname}`,
});
```

Rules:
- `destination` directories are created recursively if missing
- The default `filename` never trusts the client-supplied `originalname` for path construction — only its (sanitized) extension is reused, with a random hex base, to avoid path-traversal
- `diskStorage` is NOT exported from the main `@rasenganjs/server` entry — always import it from the `@rasenganjs/server/upload/disk` subpath

## `UploadedFile` Shape

```ts
interface UploadedFile {
  fieldname: string;      // multipart field name
  originalname: string;   // client-supplied name — user-controlled, don't trust as a path
  mimetype: string;       // client-supplied content type
  size: number;           // bytes
  buffer?: Uint8Array;     // present with MemoryStorage
  path?: string;           // present with DiskStorage — where the file was written
  filename?: string;       // present with DiskStorage — name inside destination
  destination?: string;    // present with DiskStorage — the directory
  [key: string]: unknown;  // custom engines may attach extra metadata (e.g. an S3 URL)
}
```

Rules:
- Read the result with `ctx.get<UploadedFile>('file')` (single) or `ctx.get<UploadedFile[]>('files')` (array/any) — the middleware writes to `ctx.state`, and `ctx.get()` is the typed accessor for it
- `fileUpload()` defaults to `MemoryStorage` when no `storage` option is given — pass `diskStorage()` explicitly for anything beyond small in-memory files
- Parsing buffers the whole multipart body via `request.formData()` before limits apply — pair with a request-level body size cap for a hard outer ceiling
- A rejected/oversized file returns a structured error immediately: `{ error: { code, message } }` with the matching status (400 for filter/count rejections, 413 for `LIMIT_FILE_SIZE`) — `code` is one of the `UPLOAD_ERROR_CODES` constants exported from `@rasenganjs/server`
