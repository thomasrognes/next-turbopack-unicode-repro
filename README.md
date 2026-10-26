# Turbopack UTF-8 filename truncation reproduction

Minimal reproduction of a Turbopack panic when shortening a generated filename containing a multibyte UTF-8 character.

## Run

```sh
npm ci
npm run dev
```

Open http://127.0.0.1:3000/.

The default dependency is pinned to `next@16.5.0-canary.27`.

## Observed results

On macOS arm64 with Node 24.0.2:

| Next.js version | Directory name | Result |
| --- | --- | --- |
| `16.4.3` | `demoå` | HTTP 500, panic in `ident.rs:325:34` |
| `16.5.0-canary.27` | `demoå` | HTTP 500, the same panic |
| `16.4.3` | `demoaa` | HTTP 200 |

```text
start byte index 16 is not a char boundary;
it is inside 'å' (bytes 15..17 of string)
```

## Why it fails

The client page imports two CSS modules. With only the long-path CSS
module imported, the page succeeds. Importing both triggers generation
of single-item CSS chunks and their source maps.

The failing source path is:

```text
components/demoå/this-directory-makes-the-generated-name-long/Examples.module.css
```

This path is 82 UTF-8 bytes. With the underscore and 13-character hash,
the intermediate output name is 96 bytes. The 80-byte shortening logic
chooses byte index 16, which is inside the two-byte character `å`.

## Control test

1. Stop the server.
2. Rename `components/demoå` to `components/demoaa`.
3. Update the matching import in `app/page.jsx`.
4. Restart `npm run dev` and open the page.

The two paths have the same UTF-8 byte length. The rename removes the
invalid character boundary at byte index 16.

## Test the stable version

To test the stable version instead of canary:

```sh
npm install --no-save --package-lock=false next@16.4.3
npm run dev
```

To restore the pinned canary dependencies:

```sh
npm ci
```
