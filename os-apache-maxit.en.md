# The apache plugin

*[Deutsche Version](os-apache-maxit.md)*

An Apache HTTP Server based alternative to the `www/nginx` plugin: HTTP
servers (vhosts), locations, reverse proxy / load balancing, security
headers, basic auth, IP ACLs, custom error pages, a mod_security2 (OWASP
Core Rule Set) WAF, and a log viewer.

This README has two parts: a **User Guide** (reference for every page and
field) and a **Walkthrough** (a worked example, start to finish). Design
rationale and implementation notes for contributors are further down under
[Design Notes](#design-notes). For an in-depth look at the WAF
specifically (implementation, troubleshooting, writing custom rules,
false positives), see **[WAF.md](os-apache-maxit-waf.en.md)**.

- [User Guide](#user-guide)
  - [Prerequisites](#prerequisites)
  - [Menu overview](#menu-overview)
  - [General Settings](#general-settings)
  - [Reverse Proxy](#reverse-proxy)
  - [Access](#access)
  - [WAF](#waf) -- see also [WAF.md](os-apache-maxit-waf.en.md) for the in-depth guide
  - [Logs](#logs)
  - [Applying changes](#applying-changes)
  - [Tips and gotchas](#tips-and-gotchas)
- [Walkthrough: publishing an internal app over HTTPS](#walkthrough-publishing-an-internal-app-over-https)
- [Design Notes](#design-notes)

## User Guide

### Prerequisites

Installing the plugin (**System &rarr; Firmware &rarr; Plugins &rarr;
os-apache**) pulls in the Apache HTTP Server package and its WAF/proxy
modules automatically (`apache24`, `ap24-mod_security`,
`ap24-mod_proxy_msrpc`). No manual package installation is required.

If you plan to serve HTTPS vhosts, import or generate the relevant
certificate first under **System &rarr; Trust &rarr; Certificates** -- the
Apache plugin only lets you *select* an existing certificate, it does not
issue one itself (the `os-acme-client` plugin works fine alongside it; see
the ACME Passthrough field below).

### Menu overview

The plugin adds a **Services &rarr; Apache** menu with these pages:

| Page | What it's for |
|---|---|
| General Settings | Enable/disable Apache, global logging/compression, WAF on/off, worker tuning |
| Reverse Proxy | HTTP Servers (vhosts), Locations, Upstreams (load balancer pools) |
| Access | Basic-auth users/user lists, IP ACLs, security header sets, custom error pages |
| WAF | mod_security2 rulesets and custom rules |
| Logs / Access, Logs / Error | Per-vhost access/error log viewer |
| Logs / WAF Audit | `modsec_audit.log` viewer (global, not per-vhost -- one mod_security2 engine for the whole instance) |
| Log File | Raw syslog view (Diagnostics &rarr; Log Files &rarr; apache) |

Every page has an **Apply** button that saves the form and reconfigures
the running service (regenerates `httpd.conf` and reloads/restarts Apache
as needed) -- see [Applying changes](#applying-changes).

### General Settings

Three tabs, all backed by a single form; each has its own Apply button.

**General Settings tab**

| Field | Description |
|---|---|
| Enable Apache | Master on/off switch for the whole service. |
| Server Tokens | How much version detail Apache reveals in the `Server` header and generated pages (`Minimal` vs `Full`). |
| Error Log Level | Minimum severity written to the global error log (`emerg` … `debug`). Can be overridden per HTTP Server. |
| Enable Compression | Turns on `mod_deflate` output compression for compressible content types. |

**WAF tab**

| Field | Description |
|---|---|
| Enable WAF | Global switch for mod_security2. This only loads the engine and the OWASP CRS base config -- it does **not** protect anything by itself. You still need to assign a WAF ruleset to each HTTP Server or Location you want protected (see [Reverse Proxy](#reverse-proxy) and [WAF](#waf)). |
| Regex Match Limit *(advanced)* | Upper bound on PCRE match steps per rule (`SecPcreMatchLimit`) -- a DoS/performance safeguard. Default 1500 is fine for almost everyone. |
| Regex Recursion Limit *(advanced)* | Upper bound on PCRE match recursion depth per rule (`SecPcreMatchLimitRecursion`). Normally kept equal to Regex Match Limit. |

**Worker Settings tab**

| Field | Description |
|---|---|
| Threads per Child | `ThreadsPerChild` -- worker threads per child process. |
| Max Clients | `MaxRequestWorkers` -- maximum simultaneous connections. |
| Keepalive Timeout | `KeepAliveTimeout` -- seconds to hold a persistent connection open between requests. |

### Reverse Proxy

Three tabs: **HTTP Servers**, **Locations**, **Upstreams**. A typical
vhost is built bottom-up: Upstream Server(s) &rarr; Upstream &rarr;
Location &rarr; HTTP Server (see the [Walkthrough](#walkthrough-publishing-an-internal-app-over-https)
for the concrete order).

**HTTP Servers** (one row per vhost)

| Field | Description |
|---|---|
| Server Name(s) | Comma separated hostnames. The first is `ServerName`, the rest become `ServerAlias`. |
| Default Server | Serve this vhost when no `ServerName`/`ServerAlias` matches on the same listening address. Only one default server is allowed per listening address -- the form rejects a conflicting second one. |
| Listen Address (HTTP) | Comma separated list of addresses/ports for plain HTTP, e.g. `*:80`, `127.0.0.1:8080`. |
| Listen Address (HTTPS) | Same, for TLS. Only takes effect once a certificate is also selected below. |
| Certificate | Server certificate for the HTTPS listener(s), picked from the Trust store. |
| Client Auth CA *(advanced)* | CA(s) used to verify client certificates, for mutual TLS. |
| Client Certificate Verification *(advanced)* | `off` / `optional` / `require` -- whether a client certificate is requested or mandatory. |
| Client Certificate Verify Depth *(advanced)* | How many intermediate CA certificates to follow when validating the client's chain (`SSLVerifyDepth`, default 10). Raise this if client certificates are issued by an intermediate CA rather than directly by a CA in Client Auth CA. |
| Client Certificate Revocation List (CRL) *(advanced)* | Optional pasted PEM-encoded CRL(s) from the Client Auth CA. When set, revoked client certificates are rejected even if otherwise valid (`SSLCARevocationFile`/`SSLCARevocationCheck chain`). Leave empty to skip revocation checking. |
| TLS Profile | `Custom` (default, use the two fields below), `Modern` (TLS 1.3 only) or `Intermediate` (TLS 1.2 + 1.3) -- applies Mozilla's current [SSL Configuration Generator](https://ssl-config.mozilla.org/) recommendations for `SSLProtocol`/`SSLCipherSuite` outright when set to Modern/Intermediate. |
| TLS Protocols *(advanced)* | Allowed TLS versions. Only used when TLS Profile is Custom. |
| TLS Cipher Suite *(advanced)* | Custom cipher list; leave empty for the system default. Only used when TLS Profile is Custom. |
| Enable HTTP/2 | Advertise and accept HTTP/2 on this vhost. |
| Real Client IP Header | Header carrying the real client IP when this vhost sits behind another proxy. |
| Trusted Proxies | Networks allowed to set that header. |
| Access Log Format | `combined` / `common` / `disabled`. |
| Error Log Level | Per-vhost override of the global error log level. |
| Document Root *(advanced)* | Fallback `DocumentRoot` when a matching Location has none of its own. |
| Max Body Size *(advanced)* | `LimitRequestBody`; empty means the Apache default. |
| Locations | Which Location entries this vhost serves. |
| ACME Passthrough *(advanced)* | Proxies `/.well-known/acme-challenge/` to `os-acme-client`'s built-in HTTP-01 listener so it can issue certificates for this vhost. |
| Enable Outlook Anywhere Passthrough | Recognizes and correctly proxies classic Outlook Anywhere (RPC-over-HTTP/MAPI) traffic on top of ordinary `ProxyPass` Locations, via `mod_proxy_msrpc`. Not needed for MAPI/HTTP-only Exchange (2016+). |
| Outlook Anywhere User-Agents *(advanced)* | Extra User-Agent strings to treat as Outlook Anywhere traffic; empty uses `mod_proxy_msrpc`'s built-in default (`MSRPC`). |
| Security Headers | A security header set to apply to the whole vhost. |
| Error Pages | Custom error pages to serve. |
| IP ACL | Restrict access to this vhost by client IP. |
| WAF Ruleset | mod_security2/OWASP CRS ruleset for the whole vhost (requires WAF enabled globally). |
| WAF Custom Rules *(advanced)* | Additional custom mod_security2 rules for this vhost. |

**Locations** (one row per URL path rule, attached to one or more HTTP Servers)

| Field | Description |
|---|---|
| Description | Free text label. |
| URL Pattern | The path this location matches, e.g. `/` or `/app/`. |
| Match Type *(advanced)* | Prefix match (`Location`), exact match, or a regular expression (`LocationMatch`). |
| Upstream | Reverse-proxy requests here to this upstream pool. Leave empty to serve static files or PHP locally, or to issue a Redirect instead. |
| Path Prefix *(advanced)* | Optional path appended to the balancer URL. |
| Preserve Host Header *(advanced)* | Forward the original `Host` header to the backend. On by default, and what most apps want -- but see the note on proxying another firewall's own WebUI below, where it needs to be off. |
| WebSocket Support | Upgrade matching connections to a WebSocket tunnel towards the upstream. |
| Proxy Timeout *(advanced)* | Network timeout to the backend, in seconds. Raise this for long-polling clients, e.g. Exchange ActiveSync/MAPI push notifications or Nextcloud's `notify_push`. |
| Keep Backend Connection Alive *(advanced)* | Reuses the TCP connection to the backend across requests (`ProxyPass keepalive=On`) instead of one per request. On by default; required for NTLM/Negotiate (Kerberos) authentication to survive the proxy -- Exchange Outlook Anywhere/MAPI over HTTP, EWS, and ActiveSync all depend on this. |
| Pass Request URI Unmodified *(advanced)* | Forwards the request URI to the backend byte-for-byte (`ProxyPass nocanon`), without Apache normalizing/decoding it first. Needed for some Exchange endpoints (EWS, Autodiscover, ActiveSync) whose URLs carry encoded characters that must reach the backend unchanged. |
| Redirect Target URL / Redirect Status Code | Issues an HTTP redirect instead of proxying or serving locally; only used when no Upstream is selected. Handy for e.g. sending `/` to `/owa` on an Exchange vhost, or wiring up Nextcloud's `/.well-known/carddav` and `/.well-known/caldav` convenience redirects to `/remote.php/dav`. |
| Extra Request Headers *(advanced)* | One `Name: Value` pair per line, added to the request sent to the backend. |
| Extra Response Headers *(advanced)* | One `Name: Value` pair per line, added to the response sent to the client -- e.g. CORS (`Access-Control-Allow-Origin: ...`) for CalDAV/CardDAV clients. |
| Document Root *(advanced)* | Filesystem path to serve when no upstream is selected. |
| Enable PHP | Pass `*.php` requests to the local php-fpm socket. |
| Index Files *(advanced)* | Comma separated `DirectoryIndex` file names. |
| Directory Listing *(advanced)* | Show a directory listing when no index file is present. |
| Custom Rewrite Rule *(advanced)* | One raw `RewriteRule` line, verbatim. |
| Basic Auth Realm | Realm shown in the browser's login prompt; leave empty to disable basic auth here. Leave this empty for locations proxying to an app with its own auth (Exchange, Nextcloud, ...) -- Apache basic auth would otherwise consume the `Authorization` header before NTLM/Negotiate or the app's own login ever sees it. |
| Basic Auth User List | Which user list is allowed to authenticate. |
| IP ACL | Restrict access to this location by client IP. |
| WAF Ruleset | Overrides the vhost's WAF ruleset for just this location. |
| WAF Custom Rules *(advanced)* | Additional custom rules for this location. |

A Location only takes effect once it is attached to an HTTP Server's
**Locations** field.

**Upstreams** (load balancer pools, split into two grids)

*Upstream Servers* -- individual backends:

| Field | Description |
|---|---|
| Description | Free text label. |
| Server | Backend hostname or IP. |
| Port | Backend TCP port. |
| Load Factor *(advanced)* | Relative weight in the balancer. |
| Status | `Active`, `Disabled`, or `Hot Standby` (only used when all active servers are down). |

*Upstreams* -- pools of the servers above:

| Field | Description |
|---|---|
| Description | Free text label, shown when picking this upstream from a Location. |
| Servers | Which Upstream Servers belong to this pool. |
| Load Balancing Method | `By Requests` (round robin, weighted), `By Traffic`, or `By Busyness`. |
| Enable TLS | Talk HTTPS to the backends instead of plain HTTP. |
| Verify Certificate *(advanced)* | Verify the backend's certificate: trust chain against the CA list below, hostname/CN match, and expiry. Disable if the backend uses a self-signed or expired certificate you can't replace (e.g. another OPNsense box's default web GUI certificate). |
| Client Certificate *(advanced)* | Optional client certificate presented to the backend. |
| Trusted CA *(advanced)* | CA(s) used to verify the backend; empty uses the system trust store. |
| TLS Protocols *(advanced)* | Protocol versions allowed towards the backend. |

### Access

Four tabs.

**Users & Auth** -- two grids: **Users** (`Username`/`Password`, hashed on
save) and **User Lists** (`Name` + which Users belong to it). Reference a
user list from a Location's *Basic Auth User List* field to protect it
with HTTP basic auth.

**IP ACLs** -- `Description`, `Default Action` (allow/deny when nothing
below matches), `Allow Networks`, `Deny Networks` (comma separated
IPs/CIDRs, always take priority over the default action). Assign an ACL
from an HTTP Server's or Location's *IP ACL* field.

**Security Headers** -- a named set of response headers: Referrer-Policy,
X-XSS-Protection, X-Content-Type-Options, HSTS (max-age /
include-subdomains / preload), and a full Content-Security-Policy builder
(`default-src`, `script-src`, `style-src`, `img-src`, `font-src`,
`connect-src`, `media-src`, `frame-src`, `frame-ancestors`,
`form-action`, `worker-src`, plus a report-only toggle). Assign a set from
an HTTP Server's *Security Headers* field.

**Error Pages** -- `Name`, `Status Codes` (comma separated, e.g. `404` or
`500,502,503,504`), an optional `Redirect URL`, or raw HTML `Page
Content`. Reference one or more pages from an HTTP Server's *Error Pages*
field.

### WAF

Two grids, both intentionally global (not per-vhost) so the same tuning
can be reused across many HTTP Servers/Locations. This section covers the
fields only -- for how the WAF is actually implemented, day-to-day
operation, troubleshooting, writing custom rules, and handling false
positives, see **[WAF.md](os-apache-maxit-waf.en.md)**.

**Rulesets**

| Field | Description |
|---|---|
| Description | Free text label, shown when assigning this ruleset elsewhere. |
| Learning Mode | Detection-only: violations are logged but nothing is blocked. Use this while tuning a new ruleset. |
| Paranoia Level | 1 (baseline, default) through 4 (maximum) -- higher enables stricter OWASP CRS rules at the cost of more false positives. |
| Inbound / Outbound Anomaly Threshold *(advanced)* | Request/response is blocked once its cumulative anomaly score reaches this value. |
| Allowed HTTP Verbs *(advanced)* | HTTP methods CRS accepts without flagging. Add the WebDAV verbs here (PROPFIND, MKCOL, COPY, ...) for a WebDAV/CalDAV/CardDAV backend such as Nextcloud -- leave empty to keep CRS's own default (GET, HEAD, POST, OPTIONS). |
| Application Exclusion Packages *(advanced)* | Known false-positive exclusions for specific applications (WordPress, Nextcloud, ...), if the matching OWASP CRS exclusion package is installed. |
| Whitelisted Source IPs *(advanced)* | IPs/CIDRs that bypass the WAF entirely. |
| Disable Security Rules by ID *(advanced)* | Comma separated CRS rule IDs/ranges (e.g. `920420,933100-933200`) to exclude wherever this ruleset is assigned -- a lighter-weight alternative to a full Custom Rule for the common "just exclude these rule IDs" case. |

**Custom Rules** -- `Description`, `Phase` (1 Request Headers, 2 Request
Body -- default, 3 Response Headers, 4 Response Body, 5 Logging), and the
raw `SecRule`/`SecAction` text.

None of this takes effect until **Enable WAF** is turned on under General
Settings *and* the ruleset/custom rules are assigned to an HTTP Server or
Location on the Reverse Proxy page.

### Logs

**Logs / Access** and **Logs / Error** let you pick a vhost, a log file
(current or a rotated `.gz`), page through it, and filter by text: the
filter box matches against the request line (Access) or the error message
(Error). **Logs / WAF Audit** shows `modsec_audit.log` the same way, minus
the vhost picker -- mod_security2's audit log is a single global file
covering every vhost, not one per vhost -- with columns for the matched
rule IDs, messages and whether the request was blocked; the filter box
matches against the joined rule messages. **Log File** (under Diagnostics
&rarr; Log Files) shows the raw syslog stream for anything the setup
script itself logs (certificate export problems, config test failures,
and so on) -- it is not the same as the per-vhost access/error logs or
the WAF audit log.

### Applying changes

Every page's **Apply** button saves that page's fields and then
reconfigures Apache: `httpd.conf` (and the per-vhost blocks) are
regenerated from the current configuration, TLS keys/certificates and
basic-auth password files are (re-)exported to disk, and Apache is
reloaded (or started, if it wasn't running). A small status widget near
the Apply button shows whether the service is currently running and lets
you start/stop/restart it directly.

### Tips and gotchas

* Fields marked **(advanced)** above are hidden by default -- click the
  small "advanced mode" toggle at the top of a tab to reveal them.
* Comma-separated fields (Server Name(s), Listen Address, Trusted
  Proxies, Index Files, networks/CIDRs, status codes, ...) take a plain
  comma separated string typed directly into the box, not a tag/token
  picker.
* A Location only serves traffic once it is attached to an HTTP Server's
  *Locations* field, and an HTTP Server only accepts HTTPS once it has
  both a *Listen Address (HTTPS)* and a *Certificate* set.
* WAF is a two-step opt-in: enable it globally (General Settings), then
  assign a ruleset per HTTP Server/Location (Reverse Proxy page). Turning
  on the global switch alone protects nothing.
* Only one *Default Server* is allowed per listening address (combining
  HTTP and HTTPS) -- the form will reject a second one until the conflict
  is resolved.

### Reverse-proxying real-world apps (Exchange, Nextcloud, ...)

The Location fields above cover the handful of things that most
production web apps behind a reverse proxy actually need. Two backends
that exercise nearly all of them are Microsoft Exchange and Nextcloud:

* **Exchange (OWA, ECP, EWS, Outlook Anywhere/MAPI, ActiveSync,
  Autodiscover)** -- leave *Basic Auth Realm* empty on these Locations so
  Apache doesn't intercept the `Authorization` header itself; Exchange
  needs NTLM/Negotiate to reach it unmodified. Keep *Keep Backend
  Connection Alive* on (the default) so that handshake survives across
  requests. Turn on *Pass Request URI Unmodified* for the EWS,
  Autodiscover and ActiveSync (Microsoft-Server-ActiveSync) Locations --
  their URLs contain encoded characters Apache would otherwise mangle.
  Raise *Proxy Timeout* on the ActiveSync Location so long-polling
  ("push") connections aren't cut off. A *Redirect* Location from `/` to
  `/owa` is a common convenience on the mail vhost. If the mail flow also
  needs to support classic Outlook Anywhere (RPC-over-HTTP/MAPI, as
  opposed to MAPI/HTTP-only Exchange 2016+), turn on *Enable Outlook
  Anywhere Passthrough* on the HTTP Server and add proxying Locations for
  `/rpc/rpcproxy.dll` and `/rpcwithcert/rpcproxy.dll` pointing at the
  Exchange CAS.
* **Nextcloud** -- set a generous *Max Body Size* on the HTTP Server (or
  `0` for unlimited) so large file uploads aren't rejected, and raise
  *Proxy Timeout* on the Location proxying `/` so big uploads/downloads
  and the `notify_push` app's long-polling connection don't time out
  (`notify_push` also needs *WebSocket Support* enabled). If a WAF
  ruleset is assigned to this vhost/Location, add the WebDAV verbs
  (PROPFIND, PROPPATCH, MKCOL, COPY, MOVE, LOCK, UNLOCK) to that
  ruleset's *Allowed HTTP Verbs*, or CalDAV/CardDAV/WebDAV clients will
  get blocked by CRS's method-enforcement rule. The automatic
  `X-Forwarded-Host` header (see below) lets Nextcloud's `overwritehost`
  detect the public hostname correctly behind this proxy. Two small
  *Redirect* Locations, `/.well-known/carddav` and `/.well-known/caldav`
  to `/remote.php/dav`, satisfy CalDAV/CardDAV auto-discovery in most
  clients.
* **Another OPNsense/pfSense-style firewall's WebUI** -- turn *Preserve
  Host Header* **off** for this Location, unlike Exchange/Nextcloud
  above. These WebUIs validate the incoming `Host` header against their
  own configured hostname/IP; forwarding the client's original `Host`
  (this proxy's own name) instead of the backend's produces a specific,
  confusing failure: the login page itself loads, but completely
  unstyled (every CSS/JS/image request 500s), and logging in returns a
  500 too. If you hit exactly that combination, this setting -- not the
  WAF, not a TLS problem -- is the first thing to check.
* Every proxied Location automatically sends `X-Forwarded-Proto`,
  `X-Forwarded-Port`, `X-Forwarded-Host` and `X-Real-IP` to the backend --
  most apps (including both of the above) use these to build correct
  self-referencing URLs behind a reverse proxy without any extra
  configuration. *Extra Request/Response Headers* are there for
  anything an app still needs beyond that (a custom trusted-header name,
  CORS for CalDAV/CardDAV clients, cache-control tweaks, ...).

## Walkthrough: publishing an internal app over HTTPS

This walks through a realistic first setup: reverse-proxying a single
internal web application (`app.internal.example`, listening on
`10.0.0.20:8080`) through Apache with TLS termination, then layering on
basic auth and a WAF ruleset.

### 1. Enable Apache

**Services &rarr; Apache &rarr; General Settings &rarr; General Settings
tab** &rarr; check **Enable Apache** &rarr; **Apply**.

Leave Server Tokens/Error Log Level/Compression at their defaults for
now; they can be tuned later.

### 2. Make sure a certificate is available

**System &rarr; Trust &rarr; Certificates**: import or generate a
certificate for `app.example.com` (the public name visitors will use).
If you don't have one yet, an internal CA-signed cert or an
`os-acme-client`-issued one both work fine -- come back to this step once
you have one.

### 3. Define the backend (Upstream Server + Upstream)

**Services &rarr; Apache &rarr; Reverse Proxy &rarr; Upstreams tab**:

1. Under **Upstream Servers**, add one:
   - Description: `app backend`
   - Server: `10.0.0.20`
   - Port: `8080`
   - Status: `Active`
2. Under **Upstreams**, add a pool:
   - Description: `app pool`
   - Servers: select `app backend`
   - Load Balancing Method: `By Requests`
   - Leave *Enable TLS* off (the backend here is plain HTTP)

Even a single backend needs an Upstream Server + Upstream pair -- Apache
always proxies via a named balancer, there is no direct "proxy to this
one host" shortcut.

### 4. Define the Location

**Services &rarr; Apache &rarr; Reverse Proxy &rarr; Locations tab** &rarr;
add one:

- Description: `app root`
- URL Pattern: `/`
- Upstream: `app pool`
- Preserve Host Header: on (default)
- Leave WebSocket Support off unless the app needs it

### 5. Define the HTTP Server (vhost)

**Services &rarr; Apache &rarr; Reverse Proxy &rarr; HTTP Servers tab**
&rarr; add one:

- Server Name(s): `app.example.com`
- Listen Address (HTTP): `*:80` (kept so plain-HTTP requests still reach
  Apache; combine with a redirect-to-HTTPS Location later if you want,
  or simply rely on visitors using the HTTPS URL)
- Listen Address (HTTPS): `*:443`
- Certificate: the certificate imported in step 2
- Enable HTTP/2: on
- Locations: select `app root`

**Apply**. Point a browser at `https://app.example.com/` -- you should
now see the internal app served through Apache.

### 6. Check it worked

- **Services &rarr; Apache &rarr; General Settings**: the status widget
  near any Apply button should show the service running.
- **Services &rarr; Apache &rarr; Logs / Access**: pick the vhost and
  confirm requests are showing up.
- From the firewall CLI, `configctl apache test_config` re-validates the
  generated `httpd.conf` without restarting anything.

### 7. (Optional) Protect a path with basic auth

**Services &rarr; Apache &rarr; Access &rarr; Users & Auth tab**:

1. Add a **User**: username/password for whoever needs access.
2. Add a **User List** (e.g. `app-admins`) containing that user.

Back on **Reverse Proxy &rarr; Locations**, either edit `app root` or add
a new Location for a specific path (e.g. `/admin/`):

- Basic Auth Realm: `Admin area`
- Basic Auth User List: `app-admins`

**Apply**. That path (or the whole vhost, if you edited `app root`) now
prompts for credentials.

### 8. (Optional) Restrict by source IP

**Services &rarr; Apache &rarr; Access &rarr; IP ACLs tab** &rarr; add one:

- Description: `office only`
- Default Action: `Deny Access`
- Allow Networks: `203.0.113.0/24`

Assign it via the HTTP Server's or Location's **IP ACL** field, then
**Apply**.

### 9. (Optional) Turn on the WAF

1. **Services &rarr; Apache &rarr; WAF &rarr; Rulesets tab** &rarr; add
   one: Description `default`, Learning Mode **on** to start (so nothing
   gets blocked while you watch for false positives), Paranoia Level `1`.
2. **General Settings &rarr; WAF tab** &rarr; **Enable WAF** &rarr; Apply.
3. Back on **Reverse Proxy &rarr; HTTP Servers**, edit the vhost and set
   **WAF Ruleset** to `default`. **Apply**.
4. Watch **Logs / WAF Audit** (or `/var/log/apache/modsec_audit.log`) for
   a few days, then turn **Learning Mode** off once you're confident it
   isn't blocking legitimate traffic.

### 10. (Optional) Require a client certificate instead of/alongside basic auth

For something more sensitive than the basic-auth path above -- an admin
tool, an internal API -- require a client certificate instead. On the
HTTP Server:

- Client Auth CA: the CA that issues your client certificates (from the
  Trust store).
- Client Certificate Verification: `require`.

**Apply**. Browsers without a matching client certificate installed can't
complete the TLS handshake at all -- this rejects them before Apache even
serves a response, which is stronger than basic auth (no login prompt to
brute-force) but means losing the certificate locks you out too, so keep
a second admin path available while testing. If certificates come from
an intermediate CA rather than directly from the CA selected above,
raise **Client Certificate Verify Depth** accordingly; if you also
maintain a CRL for that CA, paste it into **Client Certificate Revocation
List (CRL)** so revoked certificates stop working immediately rather than
whenever they expire naturally.

That's a complete, TLS-terminated, authenticated, WAF-protected reverse
proxy in front of an internal app -- everything else on the Reverse
Proxy/Access/WAF pages (multiple locations, multiple backends per
upstream, security headers, custom error pages, custom mod_security2
rules, mutual TLS) layers on top of this same basic shape.

## Design Notes

The sections below are implementation notes for anyone extending or
reviewing this plugin, not needed to just use it.

### Relationship to www/nginx

This plugin intentionally does **not** try to be a feature-for-feature port
of `www/nginx`. Where nginx has no Apache equivalent, the feature is left
out rather than faked:

* TCP/UDP stream proxying (nginx `stream {}` blocks)
* VTS traffic-statistics module
* TLS client-fingerprint MITM detection
* HTTP/3 / QUIC, 0-RTT
* the `resolver` directive and SNI-based upstream routing map
* NAXSI's per-rule score builder -- replaced by mod_security2/OWASP CRS's
  own tuning model (paranoia level, anomaly score threshold, rule
  exclusions, custom rule snippets), since that is how mod_security2 is
  actually configured in practice

Where a feature does carry over, it is expressed with the closest Apache
module rather than mimicking nginx's directive names: `mod_proxy_balancer`
for upstreams, `mod_authz_core`'s `Require` family for IP ACLs,
`mod_headers` for security headers, `mod_security2` for the WAF, etc.

### Frontend

Unlike `www/nginx`'s bundled Backbone/webpack frontend, this plugin uses
OPNsense's standard grid/dialog framework (`UIBootgrid`,
`base_bootgrid_table`, `base_dialog`, `base_tabs_header/content`) -- the
same pattern used by `www/caddy` and `devel/grid_example`. There is no
separate JS build step.

### The apache plugin as infrastructure

Mirroring the nginx plugin's include-hook convention: any file dropped in
`/usr/local/etc/apache24/opnsense_vhost_plugins/*.conf` is automatically
included, for plugins that need to serve something over Apache without
owning a vhost themselves.

### Hooking a vhost

Create a directory called `UUID_pre` or `UUID_post` (UUID is the
`http_server`'s UUID) under `/usr/local/etc/apache24/` and place a `.conf`
file in it; it is included at the top or bottom of that vhost's body.

### Verification caveats

This plugin was authored without access to a FreeBSD/OPNsense host, so it
could not be functionally tested end-to-end (`apachectl configtest`, an
actual browser round-trip through a reverse-proxied vhost, mod_security2
blocking a request, etc). Everything was checked for XML well-formedness
and PHP syntax balance, and template directives were cross-checked against
Apache/mod_security2 documentation, but real-world verification on an
OPNsense box or VM is still needed before production use -- in particular:

* the exact `etc/apache24/modules.d/*.conf` / `Modules/*.conf` layout the
  `www/mod_security` port drops its files into
* whether `mod_http2` is available in a stock `apache24` (Apache 2.4) package build

Confirmed on a real box: `etc/apache24/modules.d/*.conf` is empty/minimal on
OPNsense's `apache24` package (no MPM, no `mod_unixd`, ...), unlike a plain
FreeBSD ports build. `httpd.conf` no longer assumes any base module comes
from there -- every module the generated config actually uses (MPM, auth,
logging, mime/dir/alias, gzip, headers, rewrite, the proxy/balancer/TLS
stack) is now loaded explicitly, each guarded with `<IfModule !x>` so
nothing double-loads if a given install does provide one. `mod_http2` and
`mod_security2` are intentionally left out of that list since they ship in
optional add-on packages/ports and are only ever used through `<IfModule>`
guards, so an install without them keeps working minus HTTP/2 or the WAF.

Also confirmed and fixed on a real box since: missing `Listen` directives
(Apache never actually bound to configured vhost ports), several Jinja2
template-include newline bugs that glued directives together on one line,
TLS key/cert export never running outside of the plugin's own GUI actions
(fixed via the `apache24_setup` rc.conf.d hook, the same convention
`www/nginx`/`mail/postfix`/`net/freeradius` use), the `scripts/apache/*.php`
helpers missing their executable bit, a missing `SecDataDir` directory
that made mod_security2 fail to load once WAF was enabled, `<type>textarea</type>`
on the Custom Rules "Rule" and Error Pages "Page Content" fields (not a
valid `form_input_tr.volt` type -- core silently renders no input element
at all for an unrecognized type, so the field just looked empty; the
correct type is `textbox`), the HTTP Servers dialog's "Client
Address" and "Logging" sections having every field marked `advanced`,
making those whole sections look empty by default, and an Upstream's
*Verify Certificate* toggle not touching `SSLProxyCheckPeerExpire`
(a directive independent from `SSLProxyVerify`, defaulting to `on`), so a
backend with an expired certificate still failed its TLS handshake even
with *Verify Certificate* fully unchecked, and the Logs pages never
showing any content: the page/rows-per-page/filter values were sent as a
query string (`$.get(url, {page, ...})`), but OPNsense's router only binds
action parameters from URL path segments, so `accessesAction`/`errorsAction`
always received their defaults (page 0, 0 rows per page -- meaning
"return every line unbounded" -- and an empty filter) no matter what was
selected in the UI; fixed by sending those three values as extra path
segments instead, matching what the controller expected all along (its
`urldecode()` call on the filter parameter only makes sense for a path
segment, since the framework doesn't auto-decode those). Also fixed the
filter box itself while at it: it sent plain text, but the log parsers
expect a JSON object mapping column name to filter text, so the text was
silently ignored -- now wrapped server-side per log type (`request_line`
for Access, `message` for Error). The path-segment fix alone still
wasn't enough on PHP 8.2+, though: `LogParserBase` set
`$this->file_name`/`page`/`per_page`/`query`/`page_count`/`total_lines`/
`query_lines` without ever declaring them as class properties, so every
request emitted a batch of "Deprecated: Creation of dynamic property"
warnings to stdout ahead of the actual JSON. Since
`LogsController::sendConfigdToClient()` writes configd's raw output
straight into the HTTP response body, those warnings ended up
concatenated in front of the JSON, so the browser received an invalid,
unparseable response and the table silently stayed empty even though
`configctl apache log ...` on the CLI clearly returned good data. Fixed
by declaring all of `LogParserBase`'s properties up front.
