# Credentials at Rest

*Enterprise only. The community edition registers CSV file paths, which are not secrets.*

When you register a database source, its connection details — host, user, and
**password** — are stored in TDB's registry database. TDB encrypts that blob, so
possession of the database file alone is not possession of your credentials.

There is no plaintext mode. Encryption is always on.

---

## What is encrypted

The entire `connection` blob for every registered source: password, host,
username, database name, and any driver-specific fields. Encrypting the whole
blob rather than picking out password-like keys means a connector that
introduces a new secret field is covered automatically.

Everything else in the registry is either not a secret or already one-way
hashed — API keys are stored as SHA-256 digests and cannot be recovered from the
database at all, and OAuth clients are public PKCE clients with no client secret.

**Algorithm:** Fernet (AES-128-CBC with an HMAC-SHA256 authentication tag), from
the `cryptography` library. Ciphertext is stored with an `enc:v1:` prefix so the
format can be versioned later.

---

## Where the key comes from

In this order:

1. **`TDB_ENCRYPTION_KEY`** — set it, and the key never touches the data volume.
   Point it at your secrets manager. This is the production configuration.
2. **A generated key file** — if the variable is unset, TDB generates a key on
   first start and writes it beside the registry database as `.tdb_secret_key`
   with mode `0600`.

Generate a key with:

```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

---

## What the default actually protects against

Be clear-eyed about the generated-key default, because a security reviewer will
be: **the key sits in the same directory as the database it protects.**

| Scenario | Generated key file | `TDB_ENCRYPTION_KEY` |
|---|---|---|
| Someone obtains only the `.db` file | Protected | Protected |
| A database backup leaks | Protected | Protected |
| A whole-volume snapshot or `docker cp` of the data directory leaks | **Not protected** — key travels with it | Protected |
| An attacker has code execution on the host | Not protected | Not protected¹ |

¹ No at-rest scheme protects against this: a running TDB must be able to decrypt
its own credentials, so an attacker on the host can read them out of the process.
At-rest encryption addresses data leaving the host, not an intruder on it.

The default is worth having — a leaked database file on its own is the common
case, and it covers that. But if you are subject to an audit, set
`TDB_ENCRYPTION_KEY` and keep the key in a secrets manager.

---

## Rotating the key

1. Stop TDB.
2. Start it with the **old** key still configured and re-register the affected
   sources, or restore from a backup taken with the old key.
3. Set the new `TDB_ENCRYPTION_KEY` and restart.

There is no online re-key command today. If this matters for your deployment,
tell us — it is a small addition and demand decides priority.

---

## Losing the key

There is no recovery path and no backdoor. If the registry database and its key
are separated, TDB **fails loudly** on the affected sources rather than silently
returning corrupt connection details:

```
Stored source credentials could not be decrypted with the active key. The
registry database and TDB_ENCRYPTION_KEY (or .tdb_secret_key) have most likely
been separated — restore the matching key, or re-register the affected sources.
```

Recovery is to restore the matching key, or to re-register the sources with
their passwords.

!!! warning "Back up the key separately from the database"
    A backup that contains both the database and its key provides no protection
    if that backup leaks. Store them in different places.

---

## Upgrading an existing deployment

No action required. On first start after upgrading, TDB encrypts any credentials
still stored in plaintext, in place, and logs how many sources were migrated:

```
INFO:     credentials_encrypted sources_migrated=3
```

Plaintext values remain readable until they are migrated, so the upgrade is safe
to roll back.
