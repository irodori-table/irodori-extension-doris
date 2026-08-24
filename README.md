# irodori-extension-doris

Native [Irodori Table](https://github.com/irodori-table/irodori-table) connector
extension for [Apache Doris](https://github.com/apache/doris).

Not installed by default. Install it from **Settings ▸ Extensions**, or from the
command palette with **Open Extensions**, and the `doris` engine appears in the
connection dialog.

## What it connects to

The Doris frontend serves the **MySQL wire protocol on its query port, 9030** —
not the HTTP port on 8030, which answers a different API. This connector is a
MySQL client pointed at the FE query port.

| Field | Default |
| --- | --- |
| Host | `127.0.0.1` |
| Port | `9030` |
| User | — |
| Database | — (optional; catalogs and databases are browsable without one) |

Authentication is user/password, a connection URL, or none — which is what a
fresh `root` on a local cluster needs. TLS is off unless the profile asks for
it; a CA certificate is given as a **path**, because connector options persist
to the workspace in the clear and key material must not.

## What it reads

- **Query results**, with cell types resolved from the column type rather than
  from the bytes on the wire — MySQL's text protocol sends every value as
  bytes, so an `INT` reaches the grid as a JSON number and not as `"1"`.
- **Catalogs.** `SHOW CATALOGS` lists the internal catalog and every catalog
  attached through Doris multi-catalog, which is the reason to point Irodori
  Table at Doris rather than at the object store directly.
- **Metadata** for the object browser, from `information_schema.tables` and
  `information_schema.columns`: databases, tables, views, materialized views,
  and their columns. Doris' own `__internal_schema` is filtered out.

`version()` on a Doris frontend reports the MySQL protocol version it emulates,
not the Doris build, so the connector asks `@@version_comment` first and only
falls back to `version()` on a frontend that does not expose it.

## Local development

```sh
make check           # fmt, clippy, unit tests
make build           # release cdylib
make package         # dist/irodori-extension-doris.tar.gz
```

### Connection check

`cargo test` alone never touches the network. The live check is opt-in:

```sh
podman run -d --name doris -p 9030:9030 -p 8030:8030 \
  --memory 8g docker.io/apache/doris:all-in-one-4.1.3

IRODORI_LIVE_DORIS=127.0.0.1:9030 make live

podman rm -f doris
```

It connects, runs a typed `SELECT`, lists catalogs, creates and reads back a
table, reads the object-browser metadata, and closes.

## A note on `prefer_socket`

The Rust MySQL client defaults `prefer_socket` to true, and on a loopback
address it then reads `@@socket` to try to upgrade the TCP connection to a unix
socket. A Doris frontend has no such variable and rejects the statement, so a
frontend on `localhost` fails to connect while the same build against a remote
host succeeds. The frontend never serves a unix socket, so the driver turns the
preference off on both the host/port and the URL path — a DSN can otherwise
re-enable it.

## Licence

MIT OR 0BSD. See [LICENSE-MIT](LICENSE-MIT) and [LICENSE-0BSD](LICENSE-0BSD).
