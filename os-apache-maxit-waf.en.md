# WAF (mod_security2 / OWASP CRS) in depth

*[Deutsche Version](os-apache-maxit-waf.md)*

This document covers the Web Application Firewall integration in the
Apache plugin: how it's wired together under the hood, how to operate it
day to day, how to troubleshoot it, how to write your own rules, and how
to work through false positives. For the basic "what does this checkbox
do" reference, see the [WAF section](os-apache-maxit.en.md#waf) of the main README.

- [How it's implemented](#how-its-implemented)
  - [Package and files on disk](#package-and-files-on-disk)
  - [What gets generated, and when](#what-gets-generated-and-when)
  - [Rule ID allocation](#rule-id-allocation)
  - [Precedence: global vs vhost vs location](#precedence-global-vs-vhost-vs-location)
- [Using it](#using-it)
  - [Turning it on](#turning-it-on)
  - [Rulesets: the tuning knobs](#rulesets-the-tuning-knobs)
  - [Custom rules](#custom-rules)
- [Troubleshooting](#troubleshooting)
  - [WAF has no effect at all](#waf-has-no-effect-at-all)
  - [Apache won't start/reload with WAF enabled](#apache-wont-startreload-with-waf-enabled)
  - [Reading the audit log](#reading-the-audit-log)
  - [Finding which rule blocked a request](#finding-which-rule-blocked-a-request)
- [Writing your own rules](#writing-your-own-rules)
  - [SecRule anatomy](#secrule-anatomy)
  - [Phases](#phases)
  - [Useful variables](#useful-variables)
  - [Useful operators](#useful-operators)
  - [Common actions](#common-actions)
  - [Worked examples](#worked-examples)
  - [Picking a rule ID for custom rules](#picking-a-rule-id-for-custom-rules)
- [Working with false positives](#working-with-false-positives)
  - [The escalation path](#the-escalation-path)
  - [Excluding a specific rule for a specific request](#excluding-a-specific-rule-for-a-specific-request)
  - [Application exclusion packages](#application-exclusion-packages)
  - [Whitelisting by source IP](#whitelisting-by-source-ip)
  - [Tuning the paranoia level and anomaly thresholds](#tuning-the-paranoia-level-and-anomaly-thresholds)

## How it's implemented

### Package and files on disk

The WAF engine itself (`mod_security2`) comes from the `ap24-mod_security`
FreeBSD package, pulled in automatically as a plugin dependency.
`ap24-mod_security` does **not** provide the OWASP Core Rule Set, though
-- it ships zero rule files, only a fetcher script (`rules-updater.pl`)
that requires manually installing its own Perl dependency and is not run
automatically by anything. Rather than depend on that at runtime, this
plugin vendors CRS directly: the upstream `coreruleset-<version>-minimal`
release tarball (the OWASP project's own curated distribution meant for
exactly this kind of downstream packaging -- rules + `crs-setup.conf.example`
+ `LICENSE`, no docs/tests/CI) is fetched once at plugin-development time
and committed under
`src/opnsense/service/templates/OPNsense/Apache/owasp-crs/`, each file
wrapped in a Jinja `{% raw %}...{% endraw %}` block (verbatim otherwise --
a couple of CRS's own regexes contain literal `{{`/`{%`, which would
otherwise be parsed as template syntax) and mapped one-to-one in
`+TARGETS` to its real destination under
`/usr/local/etc/apache24/modsecurity.d/owasp-crs/`. Currently vendored:
CRS **v4.28.0**. There is genuinely nothing to install separately for
CRS itself; **updating** it means re-running the same fetch-and-wrap
process against a newer release and committing the result -- there is no
in-place auto-update.

The port ships its own `modules.d/280_mod_security.conf` with the
`LoadModule` line commented out, so this plugin owns that file outright
(like it does `httpd.conf`) instead of relying on the port's default --
`280_mod_security.conf` (source) / `+TARGETS` renders it to
`/usr/local/etc/apache24/modules.d/280_mod_security.conf`, loading
`security2_module` exactly when **Enable WAF** is on and removing it
(back to a no-op file) when it's off. The same file also loads
`unique_id_module` -- ModSecurity hard-requires it (refuses to start
with "ModSecurity requires mod_unique_id to be installed." otherwise)
but it ships in the base `apache24` port, not `ap24-mod_security`, and
nothing else in this plugin's generated config needs it.

Runtime files the plugin manages:

| Path | Purpose |
|---|---|
| `/usr/local/etc/apache24/modsecurity.d/owasp-crs/crs-setup.conf` | CRS defaults, vendored by this plugin |
| `/usr/local/etc/apache24/modsecurity.d/owasp-crs/rules/*.conf` / `*.data` | The actual OWASP CRS rule/data files, vendored by this plugin |
| `/var/db/apache/modsecurity/` | `SecDataDir` -- mod_security's persistent state (IP collections used by some CRS rules) |
| `/var/log/apache/modsec_audit.log` | `SecAuditLog` -- the audit log, see [Reading the audit log](#reading-the-audit-log) |

### What gets generated, and when

Five sets of templates cooperate, generated fresh on every Apply (see the
main README's [Applying changes](os-apache-maxit.en.md#applying-changes)):

- **`280_mod_security.conf`** -- a standalone target (see `+TARGETS`),
  rendered straight to
  `/usr/local/etc/apache24/modules.d/280_mod_security.conf`, replacing
  the `ap24-mod_security` port's own copy of that file. Loads
  `security2_module` when **Enable WAF** is on, emits nothing when it's
  off -- this is what actually makes `mod_security2` available at all;
  everything else below is inert without it (see
  [WAF has no effect at all](#waf-has-no-effect-at-all)).
- **`owasp-crs/crs-setup.conf`, `owasp-crs/LICENSE` and the ~48 files
  under `owasp-crs/rules/`** -- each a standalone `+TARGETS` entry,
  rendered verbatim (wrapped in `{% raw %}`) to their matching path under
  `/usr/local/etc/apache24/modsecurity.d/owasp-crs/`. This is the vendored
  CRS release itself (see [Package and files on disk](#package-and-files-on-disk));
  unlike the other templates here these don't contain any actual Jinja
  logic, they're just deployed through the same mechanism.
- **`waf.conf`** -- included once, globally, from `httpd.conf`. Only
  emits anything at all if **Enable WAF** is on. Sets up `SecDataDir`,
  `SecAuditLog` and friends, then includes the CRS base config and rule
  files. This is the only place `SecRuleEngine` is *not* set -- engine
  on/off is always decided per-vhost/per-location (see below), never
  globally.
- **`waf_server.conf`** -- included once per HTTP Server (vhost), inside
  its `<VirtualHost>` block. If that vhost has a **WAF Ruleset**
  assigned, turns `SecRuleEngine` on (or `DetectionOnly` if the
  ruleset's Learning Mode is on) and emits the paranoia
  level/thresholds/exclusions/whitelist as `SecAction` tuning
  directives, followed by the raw text of any assigned **WAF Custom
  Rules**. If no ruleset is assigned, explicitly sets
  `SecRuleEngine Off` for that vhost.
- **`waf_location.conf`** -- included once per Location that has its own
  **WAF Ruleset** assigned, inside that `<Location>`/`<LocationMatch>`
  block. Same shape as `waf_server.conf`, but only emitted when the
  location actually overrides the ruleset (an empty *WAF Ruleset* field
  on a location means "inherit whatever the vhost decided", see
  [Precedence](#precedence-global-vs-vhost-vs-location)).

All three are no-ops (emit nothing) when **Enable WAF** is off, and all
three are wrapped in `<IfModule security2_module>` so a build without
mod_security keeps working (just without a WAF).

### Rule ID allocation

Every CRS/mod_security rule needs a numeric ID, and IDs must be unique
across the whole config. This plugin derives them deterministically from
each object's UUID rather than keeping a running counter, so IDs don't
shift around as you add/remove unrelated vhosts:

```
waf_server.conf:   15,000,000 + (first 6 hex chars of the vhost's UUID)   -> range 15,000,000-31,777,215
waf_location.conf: 40,000,000 + (first 6 hex chars of the location's UUID) -> range 40,000,000-56,777,215
```

Within that base, small fixed offsets are used:

| Offset | Purpose |
|---|---|
| `+0` | paranoia level |
| `+1` | inbound anomaly threshold |
| `+2` | outbound anomaly threshold |
| `+3` | allowed HTTP methods override (`tx.allowed_methods`) |
| `+5` | source IP whitelist rule |
| `+11` .. `+17` | one per selected application exclusion package |

`Disable Security Rules by ID` doesn't need an offset of its own -- it
renders a plain `SecRuleRemoveById`, not a `SecAction`/`SecRule`, so it
has no `id:` to allocate.

Collisions are astronomically unlikely (6 hex chars = 16.7M possible
values per vhost/location) but not mathematically impossible; `setup.php`
runs `apachectl configtest` after every regeneration specifically to
catch this (and anything else) before Apache is reloaded -- see
[Apache won't start/reload with WAF enabled](#apache-wont-startreload-with-waf-enabled).
Your own [custom rules](#picking-a-rule-id-for-custom-rules) need an ID
outside both of these ranges.

### Precedence: global vs vhost vs location

1. **Enable WAF** (General Settings) is the master switch. Off means
   `waf.conf`/`waf_server.conf`/`waf_location.conf` all emit nothing --
   mod_security may still be loaded (if some other config references it)
   but this plugin issues no `SecRuleEngine`/tuning directives at all.
2. With WAF enabled, each **HTTP Server** decides its own baseline: a
   ruleset assigned means `SecRuleEngine On` (or `DetectionOnly`) with
   that ruleset's tuning; no ruleset means `SecRuleEngine Off` for that
   whole vhost, explicitly.
3. Each **Location** *inherits* the vhost's decision unless it sets its
   own **WAF Ruleset**, in which case `waf_location.conf` re-issues
   `SecRuleEngine`/tuning inside that specific `<Location>` block,
   overriding the vhost's setting for just that path. There is no way to
   force a location back to "Off" if the vhost is "On" other than giving
   it its own ruleset with rules you consider harmless, or making it a
   ruleset with Learning Mode on.
4. **Custom rules** are additive at whichever level they're assigned
   (vhost and/or location) -- they don't replace the ruleset's CRS rules,
   they run alongside them.

## Using it

### Turning it on

1. Create at least one **WAF Ruleset** (Services &rarr; Apache &rarr; WAF
   &rarr; Rulesets tab) -- even just `Description: default` with all other
   fields left at their defaults is enough to start.
2. **General Settings &rarr; WAF tab** &rarr; **Enable WAF** &rarr; Apply.
3. Assign that ruleset to an **HTTP Server** (or a specific **Location**)
   on the Reverse Proxy page &rarr; Apply.

Nothing is inspected or blocked until all three of these are true. This
is deliberate -- see [WAF has no effect at all](#waf-has-no-effect-at-all)
if you've done all three and still see no blocking.

### Rulesets: the tuning knobs

A ruleset is a named bundle of CRS tuning, reusable across many
vhosts/locations (see the [User Guide](os-apache-maxit.en.md#waf) for the full field
list). The two you'll touch most:

- **Learning Mode** -- forces `SecRuleEngine DetectionOnly`: every rule
  still runs and every match still gets logged to the audit log, but
  nothing is ever blocked. Always start a new ruleset here.
- **Paranoia Level** -- CRS ships four tiers of rules; level 1 (the
  default) is the baseline and has the fewest false positives, level 4
  is maximum strictness and will flag a lot of legitimate traffic on
  most applications. Raise this gradually, watching the audit log each
  time (see [Tuning the paranoia level](#tuning-the-paranoia-level-and-anomaly-thresholds)).

### Custom rules

A custom rule is one raw `SecRule`/`SecAction` line (or several), typed
exactly as it would appear in a `.conf` file, assigned directly to a
vhost or location (not bundled into a ruleset). Use these for anything
CRS doesn't cover: blocking a specific known-bad request pattern,
allow-listing a specific path your own CRS tuning can't express cleanly,
or virtual-patching a vulnerability while you wait for an application
fix. See [Writing your own rules](#writing-your-own-rules).

## Troubleshooting

### WAF has no effect at all

Work through this checklist in order -- each step depends on the one
before it:

1. Is **Enable WAF** actually on (General Settings &rarr; WAF tab)? This
   is easy to miss because assigning a ruleset to a vhost is a separate
   step and doesn't warn you if the global switch is still off.
2. Does the vhost (or the specific location) actually have a **WAF
   Ruleset** assigned? An HTTP Server with no ruleset gets an explicit
   `SecRuleEngine Off` -- there's no implicit "on by default".
3. Is the ruleset's **Learning Mode** accidentally left on? Everything
   will be logged as a match in the audit log but nothing blocked -- this
   looks identical to "WAF isn't running" from the outside.
4. Did you **Apply** after the last change? Check
   `/usr/local/etc/apache24/httpd.conf` (or `configctl apache test_config`)
   to confirm the `SecRuleEngine`/`SecAction` lines you expect are
   actually present for that vhost.
5. Is `mod_security2` actually loaded? `apachectl -M | grep security2` on
   the firewall's console/SSH session. With **Enable WAF** on, this
   plugin's own `280_mod_security.conf` target should have loaded it
   unconditionally -- if it's still missing, something is wrong with the
   `ap24-mod_security` package install itself (it's a hard plugin
   dependency, so a missing `mod_security2.so` means a broken/incomplete
   package install, not a config gap). Every `<IfModule security2_module>`
   block in this plugin's generated config silently no-ops when the
   module isn't loaded, which is exactly why step 5 needs a direct check
   rather than inferring it from the config file.

### Apache won't start/reload with WAF enabled

- **`SecDataDir ... is not a directory` / permission errors on
  `/var/db/apache/modsecurity`** -- `setup.php` creates this directory on
  every run; if it's missing, `setup.php` itself didn't run. See the main
  README's note on the `apache24_setup` rc.conf.d hook -- a raw
  `/usr/local/etc/rc.d/apache24 <cmd>` from the CLI only triggers it for
  `start`/`restart`/`reload`, never for `configtest`.
- **Duplicate rule ID** -- extremely unlikely (see
  [Rule ID allocation](#rule-id-allocation)), but if `apachectl
  configtest` reports a duplicate `id:`, it's almost certainly a
  [custom rule](#picking-a-rule-id-for-custom-rules) using an ID that
  collides with the derived ranges. Move it outside `15,000,000-56,777,215`.
- **`Unable to open ... crs-setup.conf` / CRS rules not found** -- CRS is
  vendored with this plugin itself now (see
  [Package and files on disk](#package-and-files-on-disk)), so this
  should only happen if the plugin package itself is broken/incomplete;
  both includes are `IncludeOptional`, so this fails silently rather than
  blocking startup -- you'll just get an engine with zero CRS rules
  loaded. Confirm with `ls /usr/local/etc/apache24/modsecurity.d/owasp-crs/rules/`
  (should show ~48 `REQUEST-*`/`RESPONSE-*.conf` and `*.data` files) and,
  if it's actually empty, reinstall/reapply the plugin.

### Reading the audit log

`/var/log/apache/modsec_audit.log`, `SecAuditEngine RelevantOnly` (only
requests that triggered a rule or returned an error are logged, not every
request), `SecAuditLogParts ABIJDEFHZ` (request/response headers and
bodies plus the trailer with matched rules). Each entry is a multi-part
record separated by a boundary marker; the parts that matter most day to
day:

- **Part A** -- timestamp, connection info, a unique transaction ID.
- **Part B** -- the request line and headers as the client sent them.
- **Part F** -- the response headers Apache would have sent.
- **Part H** -- the trailer: which rule(s) matched, the message, the
  anomaly score, and (crucially) whether the transaction was `Intercepted`
  or (Learning Mode) only logged.

Tail it live while reproducing an issue:

```
tail -f /var/log/apache/modsec_audit.log
```

or filter for a specific transaction ID once you have one from a 403
response/browser dev tools.

### Finding which rule blocked a request

1. Reproduce the request, note the approximate time.
2. `grep -B5 'Intercepted' /var/log/apache/modsec_audit.log` (or tail
   the file live) to find the matching Part H trailer -- it lists every
   rule ID that matched and the final decision.
3. The message for each match usually names the CRS rule (e.g.
   `[id "942100"] [msg "SQL Injection Attack Detected via libinjection"]`)
   -- that ID is what you exclude in the next section, not the
   `waf_rule_base`-derived tuning IDs (those never block anything
   themselves, they only set variables).

## Writing your own rules

This section assumes some familiarity with mod_security's `SecRule`
language; it's a big topic and the
[ModSecurity Reference Manual](https://github.com/owasp-modsecurity/ModSecurity/wiki/Reference-Manual-(v2.x))
is the authoritative source. What follows is the subset that covers most
custom-rule needs in this plugin.

### SecRule anatomy

```
SecRule VARIABLE "OPERATOR" "ACTIONS"
```

Example:

```
SecRule REQUEST_HEADERS:User-Agent "@contains BadBot" \
    "id:60000001,phase:1,deny,status:403,log,msg:'Blocked known bad crawler'"
```

- `VARIABLE` -- what to inspect (`ARGS`, `REQUEST_URI`, a specific
  header, `REQUEST_BODY`, ...).
- `OPERATOR` -- how to test it, prefixed with `@` (`@contains`, `@rx`
  for regex, `@eq`, `@ipMatch`, ...). No `@` prefix means a plain string
  match.
- `ACTIONS` -- comma separated, always including a unique `id`.

### Phases

mod_security processes a request in five phases; a custom rule's `phase`
action controls when it runs (this matches the **Phase** dropdown on the
Custom Rules grid):

| Phase | Runs at | Typical use |
|---|---|---|
| 1 | Request headers | Fast checks before the body is even read (headers, URL, method) |
| 2 | Request body | Form fields, JSON/XML bodies, file uploads -- most CRS rules live here |
| 3 | Response headers | Inspecting what the backend is about to send back |
| 4 | Response body | Data-leakage checks, response content inspection |
| 5 | Logging | Runs regardless of earlier phases' outcome; can't block, only log/tag |

### Useful variables

| Variable | Matches |
|---|---|
| `ARGS` | All request parameters (query string + body), by value |
| `ARGS_NAMES` | All request parameter names |
| `REQUEST_URI` | The request path + query string |
| `REQUEST_HEADERS:Name` | A specific request header |
| `REQUEST_COOKIES:name` | A specific cookie |
| `REQUEST_BODY` | The raw request body |
| `RESPONSE_HEADERS:Name` | A specific response header (phase 3+) |
| `REMOTE_ADDR` | Client IP |
| `TX.varname` | A transaction variable set by an earlier rule (`setvar`) |

### Useful operators

| Operator | Meaning |
|---|---|
| `@contains` | substring match |
| `@rx` | regular expression (PCRE) |
| `@eq` / `@gt` / `@lt` | numeric comparison |
| `@streq` | exact string match |
| `@ipMatch` | IP/CIDR match, comma separated list |
| `@pm` | fast parallel match against a list of phrases |
| `@detectSQLi` / `@detectXSS` | mod_security's built-in libinjection-based detectors, the same ones CRS itself uses |

### Common actions

| Action | Effect |
|---|---|
| `id:NNN` | required, unique numeric ID (see [Picking a rule ID](#picking-a-rule-id-for-custom-rules)) |
| `phase:N` | which phase this rule runs in |
| `deny` | block the request |
| `status:403` | HTTP status to send when denying |
| `pass` | let the request continue (used with `setvar`-only rules that never block) |
| `log` / `nolog` | whether this match is written to the audit log |
| `msg:'...'` | human-readable message shown in the audit log |
| `t:none` / `t:lowercase` / `t:urlDecode` | transformation applied to the variable before the operator runs |
| `chain` | this rule's match is only meaningful combined with the *next* rule (logical AND) |
| `ctl:ruleEngine=Off` | turn the engine off for the rest of this transaction (used for whitelisting, see below) |
| `setvar:tx.varname=value` | set a transaction variable other rules can read |

### Worked examples

Block a specific path outright:

```
SecRule REQUEST_URI "@streq /phpmyadmin" \
    "id:60000010,phase:1,deny,status:404,log,msg:'Blocked disallowed path'"
```

Allow a large upload only on one specific endpoint (raising the body
limit just for that path, everywhere else keeps the vhost's default):

```
SecRule REQUEST_URI "@streq /api/upload" \
    "id:60000011,phase:1,pass,nolog,ctl:requestBodyLimit=52428800"
```

Rate/anomaly-independent virtual patch for a known vulnerable parameter
in a specific application, while you wait for a real fix:

```
SecRule ARGS_NAMES "@streq vulnerable_param" \
    "id:60000012,phase:2,chain,deny,status:403,log,msg:'Virtual patch for CVE-XXXX-YYYY'"
    SecRule ARGS:vulnerable_param "@rx \.\./" "t:none"
```

### Picking a rule ID for custom rules

Custom rule IDs must avoid two ranges this plugin already uses:
`15,000,000-31,777,215` (vhost tuning) and `40,000,000-56,777,215`
(location tuning) -- see [Rule ID allocation](#rule-id-allocation). CRS
itself uses IDs below 1,000,000. Anything in the `60,000,000+` range (as
used in the examples above) is a safe, unused block; pick your own
starting point and increment for each new custom rule to avoid
colliding with your own other rules.

## Working with false positives

### The escalation path

When something legitimate gets blocked, don't just disable the WAF.
Work through these from least to most disruptive:

1. **Confirm it's really CRS, not your own custom rule** --
   [find the matching rule ID](#finding-which-rule-blocked-a-request) in
   the audit log first.
2. **Check for an application exclusion package** -- if you're running
   WordPress, Nextcloud, or another app with a matching entry in
   **Application Exclusion Packages**, enabling it may resolve a whole
   class of false positives in one step (see below).
3. **Exclude just that rule** -- the most surgical fix, see below.
4. **Lower the paranoia level** -- only if you're seeing many false
   positives across paranoia-level-specific rules, not a single one-off.
5. **Set Learning Mode** on that ruleset temporarily -- only as a last
   resort while you work through the above, since it also stops blocking
   genuine attacks in the meantime.

### Excluding a specific rule for a specific request

For the common case -- one or more rule IDs are wrong for this
application, full stop, everywhere this ruleset is assigned -- use the
ruleset's own **Disable Security Rules by ID** field (a comma separated
list of IDs/ranges, e.g. `920420,933100-933200`) rather than writing a
custom rule; it renders the same `SecRuleRemoveById` directive for you at
whichever level (vhost or Location) the ruleset is assigned.

If the exclusion needs to be narrower than "this whole ruleset" --
scoped to one specific Location rather than every vhost/Location sharing
the ruleset, or scoped to one specific parameter rather than the whole
rule -- drop to a raw [custom rule](#custom-rules) instead. `SecRuleUpdateTargetById`
narrows what a rule inspects when only one parameter is the problem:

```
SecRuleRemoveById 942100
```

```
SecRuleUpdateTargetById 942100 "!ARGS:comment_body"
```

assigned as a custom rule on the specific Location where the false
positive occurs, rather than on the vhost, keeps the exclusion as narrow
as the problem.

### Application exclusion packages

**Application Exclusion Packages** on a ruleset sets
`tx.<name>-rule-exclusions-enabled=1` for each selected app (WordPress,
Drupal, Nextcloud, cPanel, DokuWiki, phpBB, XenForo). This only has an
effect if the OWASP CRS exclusion plugin package for that application is
also installed alongside the base CRS rules -- the checkbox alone does
nothing if the corresponding rule file isn't present. **This plugin
currently vendors only the base CRS ruleset, not any of the
application-specific exclusion plugins** (those live in separate
`coreruleset/<app>-rule-exclusions-plugin` repos upstream), so today this
checkbox has no effect for any application until the matching plugin file
is added to `owasp-crs/plugins/` (source) / `+TARGETS` the same way the
base rules were. Check
`/usr/local/etc/apache24/modsecurity.d/owasp-crs/plugins/` for a
`<name>-*` file to confirm one's actually available before relying on
this checkbox.

### Whitelisting by source IP

**Whitelisted Source IPs** on a ruleset takes the WAF out of the
transaction entirely (`ctl:ruleEngine=Off`) for matching client IPs --
use this for trusted internal scanners, monitoring systems, or admin
networks that need to bypass inspection, not as a general false-positive
fix (it disables *all* protection for that source, not just the rule
that was misfiring).

### Tuning the paranoia level and anomaly thresholds

CRS scores each request instead of blocking on the first match; a
request is only actually blocked once its cumulative anomaly score
reaches the **Inbound/Outbound Anomaly Threshold**. Two independent
knobs, often confused:

- **Paranoia Level** controls *which rules run at all* -- level 1 only
  runs the lowest-false-positive rules, higher levels add progressively
  stricter (and noisier) ones.
- **Anomaly Threshold** controls *how many/how severe the matches need to
  be before blocking*, at whatever paranoia level you've chosen. Raising
  the threshold (the default is 5 inbound / 4 outbound, matching CRS's
  own defaults) makes the same rule set more tolerant without changing
  which rules run -- useful when you're seeing several low-severity
  matches on legitimate traffic that individually don't warrant a rule
  exclusion.

If you're seeing many different false positives after raising the
paranoia level, that's a sign the level itself is too high for this
application rather than something to individually exclude your way out
of -- drop it back down first.
