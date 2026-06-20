# Security & Updates

How TDB Enterprise handles security issues, how you receive fixes, and how to
report a vulnerability.

---

## Reporting a vulnerability

Please report security issues **privately** — do not open a public issue.

- Email **security@tdb.jiracorp.co.in**, or
- Open a private security advisory on the relevant repository.

Include the version you're running (see [Checking your version](#checking-your-version))
and steps to reproduce. We follow coordinated disclosure: advisory details are
published after a fix is available and customers have had time to update.

### Response targets

| Stage | Target |
|---|---|
| Acknowledgement | within 48 hours |
| Assessment & severity rating | within 5 business days |
| Fix released — critical / high | within 14 days |
| Fix released — medium / low | within 60 days |

## Supported versions

Security fixes ship in the **latest release**. There are no backports to older
versions — to stay supported, run the current version and update when a new one is
published. Updating is a `docker pull` + restart (see below).

## How you receive a fix

TDB is **self-hosted** — it never calls home and nothing auto-updates, so you stay
in control (including in air-gapped environments). When we publish a fix, you update
to the patched image. The steps depend on how you receive TDB:

=== "Image tarball (current)"

    We deliver the patched version as a versioned image tarball. The image is generic
    and your license is supplied at runtime, so updating never touches your license:

    ```bash
    docker load < tdb-enterprise-vX.Y.Z.tar.gz
    # restart your container — your existing -e TDB_LICENSE is unchanged
    ```

    This works the same way for air-gapped deployments.

=== "Registry pull (future)"

    As we grow we'll offer registry-based updates, so you can pull a new version
    directly:

    ```bash
    docker pull <registry>/tdb-enterprise:vX.Y.Z
    # restart your container — your token is unchanged
    ```

=== "Trial image"

    We re-issue your trial image on the patched version; load it and restart.

We notify the contact on your account when an update affecting you is available,
with the exact version and steps.

## Checking your version

Confirm what you're running at any time:

```bash
# Authenticated — includes the build commit:
curl -s -H "Authorization: Bearer <your-api-key>" http://<host>:8000/v1/version
# → {"version": "X.Y.Z", "build_sha": "abc1234"}

# Unauthenticated liveness banner also carries the version:
curl -s http://<host>:8000/
```

The running version is also printed in the container logs at startup
(`tdb_startup version=… build_sha=…`).

## Where advisories are published

Once a fix is released and customers have had time to update, advisories are
published as GitHub Security Advisories and summarized here. For the Community
Edition's security policy and known design constraints, see the
[`SECURITY.md`](https://github.com/tdb-project/tdb-community/blob/main/SECURITY.md)
in the open-source repository.
