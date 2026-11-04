# Single sign-on and SCIM

**Enterprise, Mid-Market, from 0.24.0. Off unless `TDB_OIDC_ISSUER` is set.**

People sign in through your identity provider (IdP). Their group decides their role, their
column and row rules, and the MCP tools they can use. Remove them in the IdP and their access
ends.

!!! warning "What this has been tested with"
    Sign-in and refresh are tested against **Keycloak 26.8** in CI, and with **Claude Code**
    by hand. SCIM is tested with an independent conformance tool, **not yet against a real
    IdP**. **Okta and Entra have not been run.** Both should work, since they speak the same
    standards, but check them in a test tenant before you rely on them, and tell us what you
    find. SAML is not supported; it is built on request.

## How it works

TDB stays the OAuth server for its own clients, so MCP clients connect as before. When one
needs a token, TDB shows its own consent page, sends the person to your IdP, and then issues
a TDB token. Your IdP's tokens are never accepted as bearer tokens.

```text
MCP client → TDB consent page → your IdP → TDB → a TDB token for the client
```

## Set it up

**1. Register TDB as a confidential OIDC client in your IdP.** The redirect URI is
`https://<your TDB>/sso/callback`. Have the ID token include a groups claim: many IdPs do not
send one by default, and without it everyone is refused.

**2. Write a group file.** Being in your directory grants nothing: a person in no group listed
here is refused, and admin needs its own mapping.

```yaml title="sso.yml"
groups_claim: groups        # optional; the claim holding the group list
groups:
  data-readers:
    role: read
    attributes: {region: EMEA}
    allowed_tools: [query_source, schema_source]  # optional; every tool if absent
  data-admins:
    role: admin
```

Across several groups, the highest role and the widest tool list win. Attributes are merged,
and for the same name the earlier group in the file wins. The file is read at startup, and a
mistake in it stops TDB from starting.

**3. Set the environment.**

```bash
TDB_OIDC_ISSUER=https://idp.example.com/realms/company
TDB_OIDC_CLIENT_ID=tdb
TDB_OIDC_CLIENT_SECRET=<from your IdP>
TDB_SERVER_URL=https://tdb.example.com   # https, or loopback for local testing
TDB_JWT_SECRET=<32+ random bytes>
TDB_SSO_FILE=/etc/tdb/sso.yml
TDB_OIDC_SCOPES="openid email profile offline_access"   # see below
```

TDB refuses to start if one of these is missing, if `TDB_SERVER_URL` or the issuer is plain
http off this machine, or if the licence is not Mid-Market.

**Refresh tokens need your IdP to issue them.** Many IdPs only do so when asked for
`offline_access`. Without one, a person's MCP client has to sign in again each time a token
expires, which in our tests with Claude Code meant a failed call and a manual
re-authentication. Keycloak issues them with the default scopes.

!!! note "Keycloak and `offline_access`"
    On Keycloak `offline_access` issues a token that outlives the person's session, so ending
    the session would not end their access. Leave it out there.

## How long access lasts after someone leaves

| What happens in the IdP | With SCIM | Without SCIM |
|---|---|---|
| Person disabled or removed | Refused on their next request | Refused at their next refresh, so within the token lifetime (60 minutes by default, `TDB_JWT_EXPIRE_MINUTES`) |
| Person leaves a mapped group | Refused on their next request, once groups are pushed | Refused at their next refresh |
| IdP session ended by an admin | Applies at their next refresh | Applies at their next refresh |

**API keys are not affected.** Keys are service credentials and keep working when the person
who made them leaves, so a production agent does not stop without warning. Each key records
`created_by`, and SCIM's deactivation entry in the audit log lists that person's keys so you
can decide. Without SCIM, list them with `GET /v1/auth/keys`.

An open wire gateway session is checked before each statement, so it ends too.

## REST and psql: the token page

There is no CLI to log in with, so people open **`https://<your TDB>/sso/token`** in a browser,
sign in, and copy a token. Use it as `Authorization: Bearer <token>`, or as the password for
psql if the [wire gateway](../integrations/postgres-wire.md) is on and their role is `read`.

It lasts the token lifetime and has no refresh. It acts as that person, so it should not be
pasted anywhere shared. For a tool with a stored connection, such as a BI tool, use a read
[API key](api-keys.md) instead.

## Consent

The first time a person uses a client, TDB asks them to allow it and shows where its access is
sent. This is required, not a courtesy: client registration is open, and an IdP with a live
session asks nothing, so without this page one link could hand a person's access to whoever
registered the address. The answer is remembered in that browser for 30 days, per client and
redirect address.

## Passwords

With SSO on, the password form on `/oauth/authorize` is gone. `POST /v1/auth/token` stays as a
way in when the IdP is down; turn it off with `TDB_SSO_DISABLE_PASSWORD=true`. Keys in
`TDB_API_KEYS` are not affected.

## Rules and tools by group

[Column and row rules](../security/policies.md) can name groups:

```yaml
applies_to: {groups: [data-readers]}
```

A key name beats a group, and a group beats a role. A person in two groups gets the **first
matching rule in the file**, so you decide by ordering them: rules are not merged, because that
could widen what either grants. A rule's row filter reads the attributes from the group file.
Attributes from IdP claims are not supported.

## SCIM provisioning

Set `TDB_SCIM_TOKEN` (32 characters or more) and point your IdP at
`https://<your TDB>/scim/v2`. **That token can grant admin**, by putting someone in a group
mapped to it, so treat it as an admin credential.

Users and Groups are supported, with PATCH and `eq` filters on `userName`, `externalId`,
`displayName` and `id`. `.search`, bulk, sorting and ETags are not.

**Which groups count.** Once your IdP pushes any group, pushed membership decides for the people
SCIM knows, and it applies at once. Until then the groups in their ID token decide. So push
every group your file maps, under the same names. On Entra, where the ID token can carry group
IDs, the names in your file must match what is pushed.

**Linking the first sign-in.** SCIM creates a person before they sign in and cannot know their
OIDC subject. So their first sign-in links once to the SCIM user whose `userName` equals their
**verified** email. After that only the IdP's issuer and subject identify them, so a later
change of address cannot move an account.

The address must be vouched for, or someone who can set their own could take over a provisioned
account: `email_verified` must be true. If your IdP sends no such claim, sign-in to a provisioned
account is refused (`sso_email_unverified`) until you decide. Set `TDB_SSO_TRUST_EMAIL=true` only
if your users cannot change the `email` claim themselves. An unverified address also stays out of
the audit log, where entries show the user's ID instead.

`TDB_SSO_JIT=false` refuses anyone SCIM has not provisioned. By default a first sign-in creates
the user.

Deleting a user in SCIM deactivates and hides them. The row stays, because audit entries refer
to it.

## What the audit log records

Every entry made with a JWT has `user`, and SSO entries add `user_id`; `user_inactive` marks a
refused removed person. Events are `login`, `token_refresh` and `scim_*`, with refusals under
the [`sso_*` reasons](../reference/errors.md#single-sign-on-and-scim).

Names and addresses in the audit log sit in a hash chain, which cannot be edited without showing.
An erasure request cannot be met by removing entries, so plan for that in your retention policy.
