# Serve TDB over HTTPS with Caddy

*Both editions. About 10 minutes. Checked with Caddy 2.8.4 in front of enterprise 0.21.0.*

TDB serves plain HTTP and leaves TLS to a proxy in front of it. Caddy is the shortest
path: it gets and renews a certificate on its own.

## 1. Keep TDB off the public interface

Run TDB so only the proxy can reach it, for example on a private Docker network with no
published port, or bound to `127.0.0.1`:

```bash
docker network create tdb-net
docker run -d --name tdb --network tdb-net \
  -e TDB_API_KEYS=... \
  -e TDB_SERVER_URL=https://tdb.example.com \
  ghcr.io/tdb-project/tdb-community:latest
```

`TDB_SERVER_URL` is the public HTTPS address. TDB advertises it to OAuth clients
(enterprise), which require `https://` for anything but `localhost`.

## 2. Put Caddy in front

`Caddyfile`:

```
tdb.example.com {
	reverse_proxy tdb:8000
}
```

```bash
docker run -d --name caddy --network tdb-net -p 80:80 -p 443:443 \
  -v $PWD/Caddyfile:/etc/caddy/Caddyfile:ro \
  -v caddy-data:/data \
  caddy:2.8.4
```

With a public DNS name pointing at the host, Caddy obtains a certificate from Let's
Encrypt on first request. For a test on your own machine, use a name such as
`tdb.localhost` and add `tls internal` inside the block; Caddy then signs its own
certificate, which your client must trust or skip (`curl -k`).

## 3. Check it

```bash
curl https://tdb.example.com/health
# {"status":"ok"}
```

Then point every client — AI tools, scripts, the VS Code extension — at
`https://tdb.example.com`.

## Notes

- Keep the `caddy-data` volume: it holds the certificates and keys.
- Using nginx instead? See the [IT Administrator Guide](../deployment/it-admin-guide.md#7-tls-ssl).
- The PostgreSQL wire gateway (enterprise) is not HTTP and does its own TLS:
  `TDB_PG_TLS_CERT` and `TDB_PG_TLS_KEY`. See [wire gateway](../integrations/postgres-wire.md).
