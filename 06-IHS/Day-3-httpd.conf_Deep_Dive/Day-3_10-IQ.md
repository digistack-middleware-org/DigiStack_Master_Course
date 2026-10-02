# IQ
## Q1: A junior admin says "I added a new directive to httpd.conf and restarted IHS — but it's not working." What are the first 3 things you check?


> **Analogy:** *"I put a new rule in the office handbook, but nobody follows it."*

There are three classic causes. Check them in this order: **TEST → PLACE → RESTART**.

### 1.1 Did You Run a Config Test?

Always run:

```bash
apachectl configtest
# or
./apachectl -t
```

- If there is a typo or syntax error, IHS may **keep running with the OLD config**. It doesn't crash — it silently ignores your change.
- **Rule:** Never restart without a config test first.

### 1.2 Restart vs. Graceful Restart

| Method | Behavior | Risk |
| --- | --- | --- |
| **Graceful restart** | Politely asks running workers to finish current work, then reload | Old worker processes may linger and still serve with the **old config** |
| **Full restart** | Stops everything, starts fresh | Brief downtime, but guarantees new config |

Check for stale workers:

```bash
ps aux | grep httpd
```

- If you see worker processes of **different ages** (some started days ago), the old config is still live.

### 1.3 Is the Directive in the Right Place?

Directives are **context-sensitive**. If placed outside their valid context, IHS silently ignores them — no error, no warning.

| Directive | Valid Context |
| --- | --- |
| `AllowOverride` | Only inside `<Directory>` blocks |
| Various vHost-specific directives | Only inside `<VirtualHost>` |

> [!TIP]
> **Memory trick:** **TEST → PLACE → RESTART.** Test the config, verify directive placement, then choose the right restart type.

---

## Q2: What is the risk of setting KeepAliveTimeout to 60 seconds in a bank with 50,000 concurrent users?
### 2.1 What Is KeepAlive?

| Mode | Connection Behavior |
| --- | --- |
| **KeepAlive OFF** | Request → open connection → send page → close |
| **KeepAlive ON** | Connection stays open after the page, waiting for more requests (images, scripts, next click) |

`KeepAliveTimeout` = how long the connection stays open **doing nothing**.

### 2.2 The Math Problem (50,000 Users, 400 Workers)

- Every idle keep-alive connection **holds one worker thread hostage**.
- Suppose `MaxClients = 400` (400 "hands" total), and you have 50,000 users.
- If each connection idles for 60 seconds, your 400 hands get grabbed by idle users.
- **Result:** New users receive `503 Service Unavailable` — not because the server is busy working, but because workers are standing around waiting.

### 2.3 The Fix

- **Bank standard:** `KeepAliveTimeout 3–5` seconds.
- **Advanced trick:** Disable KeepAlive on login endpoints (heavy traffic, short requests), but keep it for static assets (images, CSS — many files requested at once).

```apache
# Example: short timeout
KeepAlive On
KeepAliveTimeout 5

# Example: disable on a high-churn endpoint
<Location /login>
    SetEnvIf Request_URI "/login" nokeepalive
</Location>
```

> [!NOTE]
> **Golden rule:** Tune with load-test data — never guesswork.

> [!TIP]
> **Memory trick:** *"Idle connections eat workers. Short timeout = happy server."*

---

## A PCI-DSS auditor asks you to prove that directory listing is disabled on all 40 IHS servers. How do you prove it and how do you fix it if it's enabled?

### 3.1 What Is Directory Listing?

If a user visits `https://www.bank.com/images/` and there is no index page, the server may display a **file-manager-like list of all files** — exposing folder structure and filenames to attackers.

### 3.2 How to Check (One Server)

```bash
grep -rn "Options" /opt/IBM/HTTPServer/conf/
```

| Setting | Status |
| --- | --- |
| `Options Indexes` | Listing **ON** — BAD |
| `Options None` / `Options -Indexes` | Listing **OFF** — GOOD |

### 3.3 How to Fix

1. Edit `httpd.conf`: change `Options Indexes` → `Options None` in **every** `<Directory>` block.
2. Run `apachectl configtest`.
3. Perform a graceful restart.

### 3.4 How to Prove It Across 40+ Servers

Script the evidence collection with SSH:

```bash
for host in $(cat serverlist.txt); do
  ssh "$host" "grep -rn 'Options' /opt/IBM/HTTPServer/conf/" >> report.txt
done
```

Deliver `report.txt` to the auditor — that is your **proof**.

### 3.5 The Mature-Bank Way

- Push configuration via automation tools (**Ansible / Chef / Puppet**).
- The tool enforces the correct config on all servers **continuously**, making **config drift** (servers slowly becoming different) impossible.

> [!TIP]
> **Memory trick:** *"Indexes = windows into your house. Close them everywhere, and prove it with a script."*

---

## Quick Reference Cheat Sheet

| Task | Command / Setting |
| --- | --- |
| Validate config | `apachectl configtest` |
| Detect stale workers | `ps aux \| grep httpd` |
| Safe KeepAlive timeout | `KeepAliveTimeout 3–5` |
| Disable directory listing | `Options None` (or `Options -Indexes`) |
| Check directive context | `<Directory>` vs `<VirtualHost>` |
| Fleet-wide compliance | SSH loop script or Ansible/Chef/Puppet |
