# SqGent

Node.js TypeScript MCP server for structured, parameterized SQL Server access.
It uses the `mssql` driver (Tedious) and communicates over stdio. Each tool call
input and generated statement is logged under `logsDir`.

## Tools

| Tool | Description | Example arguments |
|---|---|---|
| `list_objects` | Search tables, views, procedures, functions, and other objects. | `{"schema":"sales","type":"table","search":"Order","limit":50}` |
| `test_connection` | Test connectivity; return success and server/database identity. | `{}` |
| `insert` | Insert one parameterized row using `row` or `columns` + `values`. | `{"table":"dbo.Customers","row":{"Name":"Ada","Active":true}}` |
| `update` | Update rows with `set` and a required `where`; use `force:true` to update all rows. | `{"table":"dbo.Customers","set":{"Active":false},"where":{"Id":17}}` |
| `delete` | Delete rows with a required `where`; use `force:true` to delete all rows. | `{"table":"dbo.Customers","where":{"Id":17}}` |
| `select` | Select columns with optional filters, grouping, ordering, and TOP. | `{"table":"dbo.Customers","columns":["Id","Name"],"where":{"Active":true},"orderBy":[{"column":"Name","direction":"ASC"}],"top":50}` |
| `info` | Return server version/edition, database settings, collation, and file sizes. | `{}` |
| `object_info` | Inspect columns, indexes, routine parameters/definition, and foreign keys. | `{"objectName":"dbo.usp_SaveOrder"}` |
| `deep_search` | Rank matches across object, schema, column, parameter, and definition metadata. | `{"search":"Customer","limit":25}` |

`where` accepts a map of column names to values (equality), or conditions such
as `{"Status":{"operator":"in","value":["Open","Pending"]}}`.
Supported operators: `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `like`, `notLike`,
`in`, `notIn`, `isNull`, and `isNotNull`. Values are bound parameters; table,
schema, and column names are quoted identifiers. No arbitrary SQL argument is
provided. Mutations without a non-empty `where` are refused unless `force:true`.

## Requirements and setup

- Node.js 18+
- Network access to a SQL Server instance

```bash
npm install
cp config.example.json config.json
# Edit config.json with connection credentials, or use environment variables (see below)
npm run build
npm test
npm start
```

Build output: `dist/index.js` (stdio MCP server).

## Configuration

The server loads `SQL_SERVER_CONFIG` (default: `./config.json`) unless you set
`SQL_SKIP_CONFIG_FILE=true` (alias: `SQL_CONFIG_FROM_ENV=true`) to use
environment variables only. Environment variables always override file values.

| `config.json` field | Environment variable | Description |
|---|---|---|
| `server` | `SQL_SERVER` | SQL Server host name or IP |
| `port` | `SQL_PORT` | Server port (default: `1433`) |
| `database` | `SQL_DATABASE` | Database name |
| `username` | `SQL_USERNAME` | SQL login username |
| `password` | `SQL_PASSWORD` | SQL login password |
| `trustServerCertificate` | `SQL_TRUST_SERVER_CERTIFICATE` | Trust server certificate (default: `true`) |
| `encrypt` | `SQL_ENCRYPT` | Encrypt the connection (default: `true`) |
| `logsDir` | `SQL_LOGS_DIR` | Tool-call log directory (default: `~/SqGent`) |
| — | `SQL_SERVER_CONFIG` | Path to the JSON configuration file |
| — | `SQL_SKIP_CONFIG_FILE` | When `true`, ignore the JSON file and read credentials from env only |

Example configuration file:

```json
{
  "server": "localhost",
  "port": 1433,
  "database": "MyDatabase",
  "username": "sa",
  "password": "your-password-here",
  "trustServerCertificate": true,
  "encrypt": true,
  "logsDir": "~/SqGent"
}
```

Environment-only MCP client example (no `config.json` on disk):

```json
{
  "mcpServers": {
    "sqgent": {
      "command": "node",
      "args": ["C:/Users/<you>/GitHub/SqGent/dist/index.js"],
      "env": {
        "SQL_SKIP_CONFIG_FILE": "true",
        "SQL_SERVER": "sql.example.com",
        "SQL_DATABASE": "MyDatabase",
        "SQL_USERNAME": "app_user",
        "SQL_PASSWORD": "<from-your-secret-store>"
      }
    }
  }
}
```

## Windows x64 scripts

`scripts/` builds the server into one self-contained `SqGent.exe` (Node SEA
via `@yao-pkg/pkg`, entry point bundle from `esbuild`) and installs it under
`%LOCALAPPDATA%\SqGent`.

| Script | Purpose |
|---|---|
| `scripts/installer-win-x64.ps1` | Build, stop any running instance, install/update the exe, seed `Settings.json` + `Data.db`, register the folder on the user PATH, map `-LocalAddress` into the hosts file (elevates automatically) |
| `scripts/Remove-win-x64.ps1` | Stop the app, delete the install folder, drop the PATH entry, clear `SQL_SERVER_CONFIG`, remove the managed hosts entries |
| `scripts/test-win-x64.ps1` | Offline lifecycle test: build, install, MCP handshake, update, remove, PATH/hosts assertions |
| `scripts/test-hosts-admin.ps1` | Hosts-file test for an elevated prompt (backs up and restores the hosts file) |

```powershell
# install or update (rebuilds first)
powershell -ExecutionPolicy Bypass -File .\scripts\installer-win-x64.ps1

# install and map the local name sqlserver.local to 127.0.0.1
powershell -ExecutionPolicy Bypass -File .\scripts\installer-win-x64.ps1 -LocalAddress sqlserver.local

# uninstall everything this project installed
powershell -ExecutionPolicy Bypass -File .\scripts\Remove-win-x64.ps1 -LocalAddress sqlserver.local

# prove the whole lifecycle
powershell -ExecutionPolicy Bypass -File .\scripts\test-win-x64.ps1
```

Useful switches: `-SkipBuild` (install the existing `dist\SqGent.exe`),
`-InstallRoot`, `-SettingsPath`, `-SkipPath`, `-KeepSettings` (uninstall keeps
`Settings.json`/`Data.db`), `-Quiet`. Building needs Node.js 22+ on the PATH
(`@yao-pkg/pkg` produces a Node 24 runtime); the installer fetches the Node base
binary on first use and needs Administrator only for the hosts file.

Point an MCP client at the installed executable:

```json
{
  "mcpServers": {
    "sqgent": {
      "command": "C:/Users/<you>/AppData/Local/SqGent/SqGent.exe",
      "env": { "SQL_SERVER_CONFIG": "C:/Users/<you>/AppData/Local/SqGent/Settings.json" }
    }
  }
}
```

Every tool call logs its timestamp, tool name, complete input, generated SQL,
and row count in a file named `MMDD-HHmmss.log` under `logsDir`. Log values
may contain application data; protect the directory accordingly.

## Cursor configuration

```json
{
  "mcpServers": {
    "sqgent": {
      "command": "node",
      "args": ["C:/Users/armin/GitHub/SqGent/dist/index.js"],
      "env": {
        "SQL_SERVER_CONFIG": "C:/Users/armin/GitHub/SqGent/config.json"
      }
    }
  }
}
```

Alternatively, provide `SQL_SERVER`, `SQL_DATABASE`, `SQL_USERNAME`, and
`SQL_PASSWORD` in the MCP server's `env` object (optionally with
`SQL_SKIP_CONFIG_FILE=true`).

## Development

```bash
npm run dev       # Run TypeScript over stdio
npm run build     # Compile to dist/
npm test          # Build and verify tools plus config/env smoke checks
```

## License

MIT
