![Version](https://img.shields.io/badge/version-3.0.0-blue)
![Java](https://img.shields.io/badge/Java-26-orange)
![Depends](https://img.shields.io/badge/depends%20on-axiom-informational)
![License](https://img.shields.io/badge/license-Limited%20Commercial-red)

# Axiom Warp Oracle Driver (`axiom-warp-oracle-driver`)

> The sovereign, framework-agnostic native Oracle Call Interface (OCI) and Panama memory-mapped driver for Axiom.

`axiom-warp-oracle-driver` talks to Oracle through `libclntsh` / `oci.dll` (Oracle Instant Client) using the Java Foreign Function & Memory API (Project Panama).
There is **no JDBC, no connection pool, no reflection and no third-party Java dependency** — just `axiom-spec`
(and transitively `axiom`) on the Java side and the vendor's native client on the other.

---

## Architecture overview

```
 ConfigSource ──► Oracle.connect(config)  ─ curried provisioning ─►  Oracle.Executioner
                                                                     │
          wield(conn -> …)  ── BoundScope(Arena.ofConfined) ──► engage() native connection
                                                                     │
           ┌──────────────────────────┬──────────────────────────────┼──────────────────────────┐
           ▼                          ▼                              ▼                          ▼
   shot(sql)                  dynamicQuery(sql)              fromFile(csv)               transaction(conn, …)
   Strike (raw SQL)           SqlParser + Strike             Hammer (bulk ingest)        commit / rollback
           │                  :java.name → native params     CsvEngine streaming
           └──────────────────────────┴──────────────┬───────────────┘
                                                     ▼
                         Result<PersistentList<PersistentMap<String, Object>>>
                                                     │
                                       disconnect()  ◄── always, success or failure
```

| Package (`com.ensemblu.axiom.warp.oracle.driver.*`) | Responsibility |
|---|---|
| `api.Oracle` | Static `connect(..)` entry point and the `Executioner` operations gateway |
| `provision.RawProvisioner` | Validates configuration fields and materialises the `Executioner` |
| `engine.Strike` | Raw statements (`shot`) and parameterised queries (`dynamicQuery`) |
| `engine.SqlParser` | Rewrites `:java.name` markers into the native parameter syntax (max **21** per statement) |
| `ingest.Hammer` | Contract-verified bulk CSV ingestion |
| `io.CsvEngine` | Streaming CSV reader (classpath or filesystem) |
| `scope.BoundScope` | Binds a confined `Arena` to the current call via `ScopedValue`, freed when `wield` returns |
| `catalog.OracleType` | Native type catalog used for bindings and ingest contracts |

**Connection model.** Each `wield(..)` opens a dedicated native connection, runs your lambda, and closes the
connection and its arena before returning — nothing leaks between calls, and nothing is pooled. Keep work that must
share a session (transactions, temp tables) inside a single `wield`.

Dependency graph:

```
axiom  ◄──  axiom-spec  ◄──  axiom-warp-oracle-driver
```

---

## Installation

**Maven**

```xml
<dependency>
    <groupId>com.ensemblu</groupId>
    <artifactId>axiom-warp-oracle-driver</artifactId>
    <version>3.0.0</version>
</dependency>
```

**Gradle (Kotlin DSL)**

```kotlin
implementation("com.ensemblu:axiom-warp-oracle-driver:3.0.0")
```

JPMS automatic module name: `com.ensemblu.axiom.warp.oracle.driver`

### Runtime requirements

| Requirement | Value |
|---|---|
| JDK | **26**, run with `--enable-preview` |
| Native access | `--enable-native-access=ALL-UNNAMED` (avoids restricted-method warnings) |
| Native client | `libclntsh` / `oci.dll` (Oracle Instant Client) |

On Windows the driver loads `%ORACLE_HOME%\bin\oci.dll` when `ORACLE_HOME` is set (JVM system property first, then environment variable), otherwise `oci` from `PATH`. On other platforms it looks up `clntsh`. Install Oracle Instant Client (Basic or Basic Light).

---

## Configuration guide

Create a properties file on the classpath (e.g. `src/main/resources/database.properties`):

```properties
engine.host=localhost
engine.port=1521
engine.service_name=FREEPDB1
engine.user=axiom_commander
engine.password=strike_hard
```

| Key | Required | Notes |
|---|---|---|
| `engine.host` | yes | Listener host |
| `engine.port` | yes | Listener port (1521 by default) |
| `engine.service_name` | yes | Service name, e.g. `FREEPDB1`, `ORCLPDB1` |
| `engine.user` | yes | Schema user |
| `engine.password` | no | Schema password |

Native connect string: Easy Connect descriptor `host:port/service_name`.

Provisioning is a curried chain that mirrors the required fields, so a missing step is a compile error and a missing
or invalid value fails fast at boot with `🚨 INFRASTRUCTURE BOOT FAILURE`:

```java
import static com.ensemblu.axiom.api.Axiom.Config.file;
import com.ensemblu.axiom.warp.oracle.driver.api.Oracle;

Oracle.Executioner executioner = Oracle
        .connect(file("database.properties"))//
        .requiredHost()//
        .requiredPort()//
        .requiredServiceName()//
        .requiredUser()//
        .optionalPassword()//
        .getOracleDriverOrThrow();
```

Keep credentials out of source control: load from a file outside the repo, or build the config at runtime with
`Axiom.Config.source("engine.password=" + System.getenv("DB_PASSWORD") + "\n…")`.

---

## Usage examples

### Raw statements

```java
var rows = executioner.wield(conn ->
        executioner.shot("SELECT 1 AS ok").withConnection(conn));

rows.map(list -> list.first().toStringKeyMapVal().get("ok"));   // values arrive as text
```

Run one statement per `shot(..)` call, without a trailing `;` (OCI rejects it), except for PL/SQL blocks.

### Parameterised queries

```java
import com.ensemblu.axiom.warp.oracle.driver.catalog.OracleType;

Result<PersistentList<PersistentMap<String, Object>>> accounts =
        executioner.wield(conn ->
                executioner.dynamicQuery("""
                                SELECT account_id, balance FROM accounts
                                WHERE status = :java.status
                                FETCH FIRST :java.limit ROWS ONLY""")
                        .to("ACTIVE").bind(OracleType.VARCHAR2)
                        .to(10).bind(OracleType.NUMBER)
                        .execute(conn));
```

Each `:java.name` marker is rewritten to Oracle's positional bind syntax (`:1`, `:2`, …) and bound out-of-band via `OCIBindByPos`.

Rules: bind **every** marker, **in the order it appears** in the SQL; a repeated name is a separate marker. Result
values are returned as strings — convert at the edge (`row.targetKey("balance").toDoubleVal()`).

### Transactions

```java
Result<String> outcome = executioner.wield(conn ->
        executioner.transaction(conn, () -> {
            executioner.shot("UPDATE accounts SET status = 'FROZEN' WHERE owner = 'X'")
                       .withConnection(conn).getOrThrow();   // throwing → rollback
            executioner.shot("INSERT INTO audit_logs (action_type) VALUES ('FREEZE')")
                       .withConnection(conn).getOrThrow();
            return "committed";
        }));
```

Mechanism: OCI transaction calls (`OCITransCommit` / `OCITransRollback`) — there is no SQL-text `BEGIN`. The failure message is prefixed with `Transaction failed, rolled back:`.

For schema bootstrapping there is a shortcut that runs several statements in one transaction:

```java
executioner.wieldTransactional(
        () -> "CREATE TABLE accounts (...)",
        () -> "CREATE INDEX idx_accounts_status ON accounts(status)"
).getOrThrow();
```

### Bulk CSV ingestion

```java
import com.ensemblu.axiom.warp.oracle.driver.catalog.OracleType;

Result<Long> inserted = executioner.wield(conn ->
        executioner.fromFile("csv/initial_accounts.csv")        // classpath first, then filesystem
                   .withConnection(conn)
                   .usingFileHeaders()                          // or .withHeaders("owner", "balance", ...)
                   .withDataTypes(OracleType.VARCHAR2, OracleType.NUMBER, OracleType.VARCHAR2, OracleType.VARCHAR2)
                   .onTableName("accounts"));
```

A prepared `INSERT` whose bind buffers are registered once (`SQLT_INT`, `SQLT_FLT`, `SQLT_STR`) and re-executed per row, committed every 1,000 rows via `OCITransCommit`.

Declare exactly **one type per header, in header order**; header names must match column names exactly.

### Error handling

Every operation returns a `Result`; nothing throws unless you call `getOrThrow()`:

```java
accounts
    .peekFailure(e -> Axiom.Io.log("❌ " + e.getMessage()).run())
    .recover(e -> Axiom.Data.emptyList());
```

Pair with [`axiom-telemetry`](../../axiom-telemetry) to attach a pre-failure snapshot to every database failure.

---

## License

**Limited Commercial License** — free for evaluation, testing, and non-commercial development.
Commercial or production use requires a paid annual contract.
See [LICENSE](https://raw.githubusercontent.com/ensemblu-corp/axiom/main/LICENSE) · Contact: contact@ensemblu.com

© Ensemblu Corp
