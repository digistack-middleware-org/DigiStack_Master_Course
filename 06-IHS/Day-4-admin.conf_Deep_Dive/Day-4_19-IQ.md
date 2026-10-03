# IBM HTTP Server (IHS) Administration Server — Interview & Operations Guide

---

## ✅ Q1. "What is the IHS Administration Server and what happens in production if it goes down?"

### Part A — Build the Foundation First (You Need This to Understand the Answer)

#### What is IHS?

- **IHS = IBM HTTP Server** — a web server.
- It's the **front door**. Users hit it first.
- It forwards requests to **WebSphere Application Server (WAS)**, which actually runs the application.

```text
Browser → IHS (port 80/443) → Plugin → WAS
```

- The **Plugin** is the component inside IHS that decides which WAS server gets each request.
- The Plugin reads its instruction manual from a file: **`plugin-cfg.xml`**.

#### What is the Admin Server?

- A **completely separate process** from the main `httpd` web server.
- Runs on **port 8008**.
- Its **ONLY job**: give the **Deployment Manager (DMGR)** — WebSphere's central control brain — a way to remotely control IHS.

> [!TIP]
> **Analogy:**
> - Main IHS = the shop door (customers walk in).
> - Admin Server = the manager's back-office phone (head office calls it to give instructions).

#### What does DMGR use the Admin Server for?

Exactly **two things**:

1. **Push `plugin-cfg.xml` to IHS remotely** (called *"Propagate Plugin"*).
2. **Remotely start/stop** the main IHS process.

### Part B — The Exact Answer to "What happens if it goes down?"

> [!IMPORTANT]
> **The golden rule:** *Admin Server down = Users are 100% fine. Only management breaks.*

| What | Status if Admin Server dies |
|---|---|
| User traffic on port 80/443 | ✅ Works normally |
| Sessions / logins | ✅ Fine |
| Plugin routing to WAS | ✅ Keeps working (with the **OLD** plugin file) |
| DMGR pushing new `plugin-cfg.xml` | ❌ Broken |
| Remote start/stop IHS from console | ❌ Broken |
| Plugin freshness | ❌ Goes stale over time |

#### Why "Stale Plugin" Becomes a Real Problem

1. Imagine your team adds **3 new JVMs** to the cluster.
2. The `plugin-cfg.xml` on IHS still only knows the **old servers**.
3. IHS keeps sending traffic as if the new servers **don't exist**.

#### Real War Story (Remember This — It's the Exam/Interview Gold)

> [!WARNING]
> A firewall hardening script accidentally **blocked port 8008**.
> Nobody noticed for **4 hours** — because users weren't complaining. Everything "looked healthy."
> Meanwhile, IHS never learned about **3 new JVMs**.
>
> - **Fix:** Re-open port 8008 in `iptables`.
> - **Lesson learned:** Add monitoring that checks **port 8008 connectivity from the DMGR side**.

> [!NOTE]
> 🔑 **Memory hook:** *"Admin down = users happy, admins sad."*

---

## ✅ Q2. "Explain the exact technical difference between a Managed and Unmanaged web server definition in WAS. Which would you choose for a production bank DMZ and why?"

### Part A — The Exact Technical Difference

The difference boils down to **ONE question**:

> **Is the IHS Admin Server running and reachable from the DMGR on port 8008?**
> - **Yes** → **Managed**
> - **No** → **Unmanaged**

#### Side-by-Side Comparison

| Feature | Managed | Unmanaged |
|---|---|---|
| Admin Server on port 8008 | ✅ Running | ❌ Absent |
| Authentication | `htpasswd` file | N/A |
| Web server definition in console | Admin port configured | "Not managed" selected |
| Propagate Plugin button | ✅ Works (one click) | ❌ Greyed out |
| Where plugin gets generated | Pushed directly to IHS | Generated the DMGR machine |
| How plugin reaches IHS | Automatically over the network | Manual SCP by the admin |
| Remote start/stop IHS from console | ✅ Yes | ❌ No |

> [!TIP]
> **Analogy:**
> - Managed = head office has the shop manager's phone number → changes are in instantly.
> - Unmanaged = no phone → head office prints the memo and drives it over by hand.

### Part B — Which One for a Production Bank DMZ?

> [!WARNING]
> **The trap:**
> - Junior answer = *"Managed, obviously — it's automated!"*
> - Senior answer = **"It depends on the security architecture."**

#### The Problem with Managed in a Bank DMZ

- A DMZ sits between **two firewall tiers** (strict PCI-DSS environments).
- Managed requires an **inbound firewall hole from the internal App zone into the DMZ** (port 8008).
- Security teams typically reject this — management traffic must **NOT** flow from internal zones into the DMZ.

#### The Verdict

| Environment | Choice | Reason |
|---|---|---|
| Production bank DMZ (strict PCI-DSS) | **Unmanaged** | It's the correct architecture, **not a limitation** |
| DevTest / IHS on same network segment as WAS | **Managed** | Faster, fewer human errors, better visibility |

#### How We Survive Unmanaged in Production (ensating Controls)

1. **Documented change process** — generate plugin, validate it on DMGR first.
2. **Transfer via a privileged jump server.**
3. **File integrity verification (checksums)** after transfer — proves the file wasn't corrupted orpered with.

> [!NOTE]
> 🔑 **Memory hooks:**
> - *"Managed = DMGR pushes. Unmanaged = you carry the file."*
> - *"Bank DMZ = Unmanaged, because security teams hate holes into the DMZ."*

---

## ✅ Q3. "I see two httpd processes in ps output — both using the httpd binary. How do I tell which is the main web server and which is the Admin Server? And what config files do each use?"

### Part A — Why There Are Two Processes

- Both processes run the **same binary**: `/opt/IBM/HTTPServer/bin/httpd`
- They are **NOT duplicates or errors** — they are two completely independent servers.
- This trips up juniors all the time. **You're not alone — it's a normal confusion.### Part B — The Trick: Look at the `-f` Flag

Run:

```bash
ps -ef | grep httpd
```

You'll see something like:

```text
httpd -f /opt/IBM/HTTPServer/conf/httpd.conf   ← MAIN IHS
httpd -f /opt/IBM/HTTPServer/conf/admin.conf   ← ADMIN SERVER
```

- The `-f` flag tells each process **which config file to load**.
- `httpd.conf` = main web server.
- `admin.conf` = Admin Server.

> [!TIP]
> **The `-f` flag never lies.**

### Part C — The Full Separation Table (Memorize This)

| Property | Main IHS | Admin Server |
|---|---|---|
| Binary | Same: `/opt/IBM/HTTPServer/bin/httpd` | Same binary |
| Config file | `httpd.conf` | `admin.conf` |
| Port | 80 / 443 | 8008 |
| Logs | `error_log`, `access_log` | `admin_error_log`, `admin_access_log` |
| Independence | Stop one → other is unaffected ✅ | Same |

### Part D — The Troubleshooting Pro Tip

**Scenario:** DMGR can't connect to the Admin Server.

**Where do you look first?**
→ `admin_error_log` — **NOT** the main `error_log`.

**Why?**

| Issue | Logged In |
|---|---|
| Authentication failures | `admin_error_log` |
| Port 8008 conflicts | `admin_error_log` |
| Connection errors on 8008 | `admin_error_log` |

- The main `error_log` only knows about **port 80/443** problems.
- Looking there for Admin Server issues is like checking the shop door log when the back-office phone is broken.

> [!NOTE]
> 🔑 **Memory hooks:**
> - *"`httpd.conf` vs `admin.conf` — the `-f` flag never lies."*
> - *"Admin problems → `admin_error_log`. Always."*

---
# IBM HTTP Server (IHS) — `admin.conf` Interview Guide

## Q1. "What is the admin.conf file? How is it different from httpd.conf? Can you explain 3 key directives from it?"

### Before anything: What is IHS?

**IBM HTTP Server (IHS)** = IBM's version of the Apache web server.

```text
Browser → Internet → [IHS] → plugin → [WebSphere (WAS)] → database
```

IHS is the front-door waiter. It serves static pages/images and forwards app requests to WAS.

### The Big Secret: TWO servers run on one IHS box

Beginners always miss this. There are two separate processes:

| | Server 1: Web Server | Server 2: Admin Server |
|---|---|---|
| Purpose | Serves website traffic to users | Lets DMGR manage IHS remotely |
| Config file | `httpd.conf` | `admin.conf` |
| Port | `80` / `443` | `8008` (typical) |
| Who talks to it | Customers (browsers) | Only WebSphere DMGR |
| Command | `apachectl` | `adminctl` |
| Logs | `access_log` / `error_log` | `admin_access.log` / `admin_error.log` |

**Real-life analogy:** Bank branch.

- Web server = teller at the counter serving customers all day.
- Admin server = manager's back-office door. Customers never touch it. Only head office (DMGR) enters to give instructions.

### So what is `admin.conf`?

`admin.conf` is the configuration file for the **IHS Administration Server** — the second process that runs alongside the main httpd web server.

- `httpd.conf` → controls how IHS serves web traffic (port 80/443).
- `admin.conf` → controls how the Admin Server manages IHS (port 8008).

They live side by side:

```text
/opt/IBM/HTTPServer/conf/
      ├── httpd.conf
      └── admin.conf
```

> [!TIP]
> Memory trick: `admin.conf` = admin door. `httpd.conf` = http door.

**Why does the Admin Server exist?** Without it, you'd SSH into every box manually to change things. With it, the Deployment Manager (DMGR) — WebSphere's "head office" console — can remotely push the plugin file (Propagate Plugin), restart IHS, etc.

### The 3 Key Directives

#### Directive 1: `Listen 8008`

```apache
Listen 8008
```

**Plain English:** "Admin Server, open your door on port 8008."

- **Golden rule:** This must *exactly* match the "Administration Port" in the WAS Console's web server definition.
- Mismatch = DMGR knocks on the wrong door = Console shows IHS status as **"Unknown"** — the most common beginner failure.
- **Analogy:** You tell the courier flat 8008, but the doorbell is wired to flat 8009. No delivery ever arrives.

#### Directive 2: The `<Location /wasadmin>` block — the password gate

```apache
<Location /wasadmin>
    AuthType Basic
    AuthName "WebSphere Administration"
    AuthUserFile /opt/IBM/HTTPServer/conf/admin.passwd
    Require user ihsadmin
</Location>
```

**Plain English, line by line:**

1. `<Location /wasadmin>` — These rules apply only to the management URL path DMGR uses.
2. `AuthType Basic` — Username/password authentication.
3. `AuthUserFile ... admin.passwd` — Allowed users + encrypted passwords live in this file (created with `htpasswd`).
4. `Require user ihsadmin` — Only user `ihsadmin` may enter.

**The mirror rule:** The password typed into the WAS Console's web server definition must match `admin.passwd`. Two locks, one key.

> [!WARNING]
> Real production incident: Security rotates passwords. Someone updates `admin.passwd` on the box but forgets the Console. Every management operation fails with `401 Unauthorized` until the Console is updated. **Always update both sides.**

#### Directive 3: `Allow from <DMGR IP>` — the guest list

```apache
Order deny,allow
Deny from all
Allow from 10.10.2.10
```

**Plain English:**

- `Deny from all` — Nobody enters. Nobody.
- `Allow from 10.10.2.10` — Except the DMGR's IP.

So a stolen password alone isn't enough — the attacker's machine must also be the DMGR IP, else `403 Forbidden`.

- In **PCI-DSS** (card payment security) environments, this is a mandatory control.
- Apache 2.2 syntax: `Allow from`. Apache 2.4: `Require ip`. Know both — old IHS uses 2.2 style.

**Analogy:** Back-office door has a peephole with a guest list. Password + being on the list, both required.

### Q1 Revision Card

- Two processes: web server + admin server.
- `admin.conf` = admin server's config; `httpd.conf` = web server's config.
- `Listen 8008` → must match Console port.
- Auth block → password gate (Console ↔ `admin.passwd` must match).
- `Allow from` → IP guest list → only DMGR.

---

## Q2. "In admin.conf you have an Allow from directive with the DMGR IP. You've just set up a DR DMGR server at a new IP. What exact steps do you follow to update this — without causing downtime to the production web traffic?"

### Understanding

- **DR DMGR** = a backup "head office" server at a new IP, used during disaster recovery.
- Your `admin.conf` guest list only has the production DMGR's IP.
- When DR kicks in, the new DMGR tries to manage IHS → `403 Forbidden`.
- Your job: add the new IP **without touching live traffic**.

### The 5 Exact Steps

#### Step 1 — Look before you touch

```bash
grep -n "Allow from" /opt/IBM/HTTPServer/conf/admin.conf
```

Find the `<Location /wasadmin>` block and confirm current IPs.

#### Step 2 — Add the new IP. DO NOT remove the old one.

```apache
Allow from 10.10.2.10   # existing prod DMGR (keep!)
Allow from 10.10.2.20   # new DR DMGR IP (new)
```

**Why keep both?** So production management keeps working even if DR isn't tested yet.

#### Step 3 — Syntax check (your seatbelt)

```bash
adminctl configtest
```

Must say **`Syntax OK`**. A typo + restart = dead admin server.

#### Step 4 — Restart ONLY the Admin Server

```bash
adminctl restart
```

> [!IMPORTANT]
> **This is the magic moment of the whole topic.** Because web server and admin server are two separate processes:
>
> - `adminctl restart` touches only the admin server (`admin.conf`).
> - The main httpd process serving port 80/443 keeps running completely untouched.
> - User web traffic = **zero impact**. Customers notice nothing.
>
> If you'd instead run `apachectl restart`, you'd drop live customer connections — thousands of angry users at 2 AM.

#### Step 5 — Verify

```bash
tail -f /opt/IBM/HTTPServer/logs/admin_access.log
```

Watch the DR DMGR IP get `200` responses. Also from the DR DMGR box:

```bash
telnet <IHS_IP> 8008
```

### Q2 Revision Card

```text
grep → edit (add, don't remove) → configtest → adminctl restart → check admin_access.log
```

> [!TIP]
> **One-line interview answer:** "`adminctl restart` only restarts the Admin Server process; the main web server on 80/443 keeps serving — that's the beauty of two separate processes."

---

## Q3. "You're called at 2 AM. The team says 'Propagate Plugin is failing in Console but IHS is serving traffic fine.' Walk me through your diagnosis."

### The key insight (say this FIRST)

- Website serving fine = web server + plugin are healthy.
- So the problem is ONLY in the admin path: `DMGR → port 8008 → Admin Server`.
- This halves the problem space immediately. Smart admins think this way.

### The 4-Step Diagnosis Ladder (climb in order)

#### Step 1 — Is the Admin Server even alive?

```bash
ps -ef | grep admin.conf | grep -v grep
netstat -tlnp | grep 8008
```

- Port 8008 not listening? → `adminctl start` → done. Go back to bed.

#### Step 2 — Read `admin_access.log` (the detective's notebook)

```bash
tail -50 /opt/IBM/HTTPServer/logs/admin_access.log
```

Every DMGR attempt leaves a line with an HTTP code. The code tells you exactly which layer broke:

| What you see | Meaning | Fix |
|---|---|---|
| No entries at all | DMGR's request never reached IHS | Firewall problem. Test: `telnet <IHS_IP> 8008` from DMGR |
| `401` | Arrived, wrong password | Console password ≠ `admin.passwd` → update Console or reset with `htpasswd` |
| `403` | Arrived, password OK, IP blocked | Add DMGR IP to `Allow from` → `configtest` → `adminctl restart` |
| `200` | Admin Server is FINE | Problem is upstream in Console/DMGR → check DMGR logs |

> [!TIP]
> Memory trick: `401` = "I don't know you" (auth). `403` = "I know you, but you're not invited" (IP). `200` = "OK, all good."

#### Step 3 — If firewall is suspected, check on the IHS box

```bash
iptables -L INPUT -n | grep 8008
```

No rule for 8008 = firewall is silently dropping DMGR's requests.

#### Step 4 — Check `admin_error.log` for startup/runtime errors

```bash
tail -100 /opt/IBM/HTTPServer/logs/admin_error.log
```

Crashes, permission issues, config problems appear here.

### The one sentence to tattoo on your brain

> [!IMPORTANT]
> "When Propagate Plugin fails, look at `admin_access.log` FIRST — the response code in that log tells you exactly which layer is failing."
