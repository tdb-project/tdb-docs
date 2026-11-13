# Upgrading TDB Enterprise

TDB is self-hosted. It never calls home and nothing updates on its own: a new version
reaches your servers only when you install it. This page is the whole procedure, for
whoever runs TDB at your company.

**In short:** read what changed, back up, load the new image, start it on the same data,
check it, and keep the old image until you are satisfied. It takes about fifteen
minutes and a minute or two of downtime.

!!! info "How you receive a new version"
    When a release is out we e-mail the contact on your account: the version, what
    changed, whether you need to do anything, a **download link that expires after
    7 days**, and the file's **SHA-256 checksum**. Security fixes reach you within 14 days
    for critical and high severity, 60 days otherwise ([Security & updates](../security/updates.md)).
    Only the latest release gets fixes, so stay current.

    Your licence is not part of the image. The same `TDB_LICENSE` value keeps working
    across upgrades; a renewal is a new value, not a new image. (Trial images with the
    licence built in are re-issued to you instead.)

---

## 1. Read what changed

Open the [release notes](../release-notes.md) and read the **Action on upgrade** column
for **every version between yours and the new one**, not only the newest. Skipping
versions is fine; skipping their actions is not.

Your current version:

```bash
curl -s -H "Authorization: Bearer <admin-key>" http://<host>:8000/v1/version
# {"version":"X.Y.Z","build_sha":"abc1234"}
```

## 2. Back up

Everything TDB keeps is in two places: the data directory mounted at `/app/data` and the
audit log (`TDB_LOG_FILE`, usually in the same directory). Stop TDB, then copy the
**whole** directory, hidden files included:

```bash
docker stop tdb
sudo cp -a /srv/tdb/state /srv/tdb/state.backup-$(date +%F)   # your /app/data volume
```

- **`.tdb_secret_key`** is hidden (a dot file). Without it the database passwords TDB
  stores cannot be decrypted. `cp -a` of the directory copies it; copying `*` does not.
  If you set `TDB_ENCRYPTION_KEY` instead, keep that value with the backup.
- **`tdb_registry.db-wal` and `-shm`** hold recent changes. Copy them with the database,
  which stopping TDB first guarantees.
- With tenants (`TDB_TENANTS_DIR`), back up that directory whole as well.

## 3. Download, check, load

```bash
sha256sum -c tdb-enterprise-vX.Y.Z.tar.gz.sha256     # must print: OK
# or, with the checksum from our e-mail:
echo "<sha256>  tdb-enterprise-vX.Y.Z.tar.gz" | sha256sum -c
docker load < tdb-enterprise-vX.Y.Z.tar.gz
# Loaded image: tdb-enterprise:vX.Y.Z
```

Air-gapped: download on a connected machine, check the checksum there and again after
copying the file in.

Do not load a file whose checksum does not match. Ask us for a new link.

## 4. Start the new version on the same data

Use the **same** environment variables (`TDB_LICENSE`, `TDB_API_KEYS`, `TDB_JWT_SECRET`
and the rest) and the **same** volumes. Only the image changes. On its first start the new
version updates the registry in place.

=== "docker run"

    ```bash
    docker rename tdb tdb-previous            # keep the old container for a rollback
    docker run -d --name tdb --restart unless-stopped -p 8000:8000 \
      -v /srv/tdb/state:/app/data \
      -v /srv/tdb/data:/data:ro \
      -e TDB_LICENSE -e TDB_API_KEYS -e TDB_JWT_SECRET \
      ...the rest of your settings... \
      tdb-enterprise:vX.Y.Z
    ```

    `docker inspect tdb-previous` lists the settings the old container had.

=== "Docker Compose"

    Change only the image tag in `docker-compose.yml`, then:

    ```bash
    docker compose up -d tdb
    ```

=== "Kubernetes"

    Push the loaded image to your own registry, then change the image tag in your
    manifest and apply it:

    ```bash
    docker tag tdb-enterprise:vX.Y.Z registry.example.com/tdb-enterprise:vX.Y.Z
    docker push registry.example.com/tdb-enterprise:vX.Y.Z
    kubectl -n tdb set image deployment/tdb tdb=registry.example.com/tdb-enterprise:vX.Y.Z
    ```

    TDB runs as one pod with the `Recreate` strategy (its registry and audit log are
    files on one volume), so expect a few seconds of downtime.

## 5. Check it

```bash
docker logs tdb 2>&1 | grep license_ok          # licence accepted, days left
curl -s http://<host>:8000/health
curl -s -H "Authorization: Bearer <admin-key>" http://<host>:8000/v1/version
curl -s -H "Authorization: Bearer <admin-key>" http://<host>:8000/v1/audit/verify
```

- `/v1/version` shows the new version.
- `audit/verify` returns `"valid": true`: the audit chain continues across the upgrade.
- Run one query you know the answer to, with a normal (non-admin) key, and ask one
  question through your AI tool.
- If you use single sign-on, sign in once.

## 6. If something is wrong, go back

Stop the new version and start the previous image on your backup:

```bash
docker stop tdb && docker rm tdb
sudo rm -rf /srv/tdb/state && sudo cp -a /srv/tdb/state.backup-<date> /srv/tdb/state
docker rename tdb-previous tdb && docker start tdb
```

Restoring the backup is the safe way back. Changes made on the new version (new keys,
sources, audit entries) are lost with it; the audit entries the new version wrote are
in the directory you replaced, so keep a copy of it if you need them.

Starting the previous image on the data the new version used, without restoring, has also
worked in our tests (0.32.0 and 0.36.1 after 0.37.0: keys, sources, stored passwords,
annotations and the audit chain intact). Use it when you cannot lose the changes made in
between, and tell us, so we can check that pair of versions.

Once you are satisfied, remove the old container (`docker rm tdb-previous`) and image,
and keep the backup until the next upgrade.

---

## Questions

Write to [hello@tdb.asvanlabs.com](mailto:hello@tdb.asvanlabs.com) with the output of
`/v1/version`, or to [security@tdb.asvanlabs.com](mailto:security@tdb.asvanlabs.com) about
a security update.
