
# wa-sqlite-fixed

This is a fork of wa-sqlite, which exposes a WebAssembly build of SQLite with experimental support
for writing to File System / IndexedDB

The fork allows for easier usage in bundlers such as Vite by declaring the right types as it
declares the async factory module in the `"exports"` field in package.json as well as eschews
ambient declaration types in favor of type modules.

Example initialiation compatible with Vite (and probably esbuild and rollup):

```typescript
import SQLiteESMFactory from 'wa-sqlite-fixed/wa-sqlite-async'
import * as SQLiteAPI from 'wa-sqlite-fixed'

export let Configurations = {
  memory: {
    vfsModule: import('wa-sqlite-fixed/vfs/memory'),
    vfsClass: 'MemoryAsyncVFS',
    vfsArgs: [],
  },
  idb: {
    vfsModule: import('wa-sqlite-fixed/vfs/idb'),
    vfsClass: 'IDBBatchAtomicVFS',
    vfsArgs: ['wa-sqlite-database'],
  },
  opfs: {
    vfsModule: import('wa-sqlite-fixed/vfs/opfs'),
    vfsClass: 'OriginPrivateFileSystemVFS',
    vfsArgs: [],
  },
  'access-handle-pool': {
    vfsModule: import('wa-sqlite-fixed/vfs/ahp'),
    vfsClass: 'AccessHandlePoolVFS',
    vfsArgs: ['/demo-AccessHandlePoolVFS'],
  },
}

export type BackendType = keyof typeof Configurations

async function createModule(backend: BackendType = 'idb') {
  let module = await SQLiteESMFactory()
  let sqlite3 = SQLiteAPI.Factory(module)
  let config = Configurations[backend]
  // Create the VFS and register it as the default file system.
  let namespace = (await config.vfsModule) as any
  let vfs = new namespace[config.vfsClass](...(config.vfsArgs ?? []))
  await vfs.isReady
  sqlite3.vfs_register(vfs, true)

  return sqlite3
}

export let sqlitePromise = createModule()

```

This should compile cleanly with TypeScript.

# example usage

The following example implements a simplified driver that's easily made compatible with Drizzle ORM

```typescript
import { SQLiteDriver, SQLiteMigration } from '../driver-types'
import type SQLiteAPI from 'wa-sqlite-fixed'
import { SQLITE_OPEN_CREATE, SQLITE_OPEN_READWRITE, SQLITE_ROW } from 'wa-sqlite-fixed'

export class SQLiteDatabaseDriverWeb implements SQLiteDriver {
  private sqliteWasm: Promise<SQLiteAPI>

  private db: Promise<number>
  private didMigrate: Promise<void>

  constructor(private database: string, private upgradeStatements: SQLiteMigration[] = []) {
    this.sqliteWasm = import('./sqlite-wasm').then(m => m.sqlitePromise)
    this.db = this.sqliteInit()
    this.didMigrate = this.runMigration()
  }

  private async sqliteInit() {
    let sqlite = await this.sqliteWasm
    let db = await sqlite.open_v2(this.database, SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE)
    return db
  }

  private async q(sql: string, values: any[]) {
    let db = await this.db
    let sqlite3 = await this.sqliteWasm

    let rows = []

    const str = sqlite3.str_new(db, sql)
    try {
      // Traverse and prepare the SQL, statement by statement.
      let sqlStr = sqlite3.str_value(str)

      let prepared: { stmt: number; sql: number } | null = null
      while ((prepared = await sqlite3.prepare_v2(db, sqlStr))) {
        for (let i = 0; i < values.length; i++) {
          sqlite3.bind(prepared.stmt, i + 1, values[i])
        }
        try {
          // Step through the rows produced by the statement.
          while ((await sqlite3.step(prepared.stmt)) === SQLITE_ROW) {
            // Do something with the row data...
            let row = sqlite3.row(prepared.stmt)
            rows.push(row)
          }
        } finally {
          await sqlite3.finalize(prepared.stmt)
        }
      }
    } finally {
      sqlite3.str_finish(str)
    }
    return { rows }
  }

  async runMigration() {
    let sortedVersions = this.upgradeStatements.slice().sort((a, b) => a.toVersion - b.toVersion)
    let maxVersion = this.upgradeStatements.reduce(
      (max, migration) => Math.max(max, migration.toVersion),
      0
    )
    for (let migration of sortedVersions) {
      await this.q('BEGIN TRANSACTION', [])
      for (let statement of migration.statements) {
        await this.q(statement, [])
      }
      await this.q('COMMIT', [])
    }
  }
  async query(query: string, values: any[]): Promise<{ rows: any[][] }> {
    await this.didMigrate
    return await this.q(query, values)
  }
  async run(query: string, values: any[]): Promise<{ rows: any[][] }> {
    return this.query(query, values)
  }
  async get(query: string, values: any[]): Promise<{ rows: any[] }> {
    let result = await this.query(query, values)
    return { rows: result.rows[0] }
  }
  async values(query: string, values: any[]): Promise<{ rows: any[] }> {
    return await this.get(query, values)
  }
}
```

# more info
For more information, see https://github.com/rhashimoto/wa-sqlite