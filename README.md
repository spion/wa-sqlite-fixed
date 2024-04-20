
# wa-sqlite-fixed

This is a fork of wa-sqlite, which exposes a WebAssembly build of SQLite with experimental support
for writing to File System / IndexedDB

The fork allows for easier usage in bundlers such as Vite by declaring the right types as it
declares the async factory module in the `"exports"` field in package.json as well as eschews
ambient declaration types in favor of type modules.

Example initialiation compatible with Vite (and probably esbuild and rollup):

```typescript
import SQLiteESMFactory from 'wa-sqlite-fixed/dist/wa-sqlite-async'
import * as SQLiteAPI from 'wa-sqlite-fixed'

async function createModule() {
  const module = await SQLiteESMFactory()
  const sqlite3 = SQLiteAPI.Factory(module)
  return sqlite3
}

export let sqlite = createModule()
```

This should compile cleanly with TypeScript.

For more information, see https://github.com/rhashimoto/wa-sqlite