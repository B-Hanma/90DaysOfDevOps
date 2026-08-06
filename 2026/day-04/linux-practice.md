# Day 04 — Linux Practice: Processes and Services

**Goal:** Practice the actual investigative sequence you'd use in real troubleshooting —
check if something is running, check if the service managing it is healthy, then check
its logs to understand what happened. This mirrors how incident troubleshooting
actually works, just at the OS/service level instead of network/infra level.

**Service inspected: nginx** (a web server — software that listens for and responds
to web requests, in this case serving a custom page I deployed on my EC2 instance)

---

## 1. Process Checks
**Why:** Before checking anything else, confirm the thing you're troubleshooting is
actually running at all.

    $ pgrep -a nginx
    21153 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
    21156 nginx: worker process
    21157 nginx: worker process

`pgrep` = "process grep" — searches running processes by name and returns their PIDs
(Process IDs — unique numbers the kernel assigns to every running process).
`-a` shows the full command line too, not just the PID number.

Result: nginx has one **master process** (manages everything, runs as root for
elevated privileges) and two **worker processes** (owned by the unprivileged
`www-data` user — these actually handle incoming traffic). This split is a
deliberate security design: if a worker crashes or is compromised, it has no
root access to do damage.

    $ ps aux | grep nginx

Same information, different tool: `ps aux` lists every process on the system;
piping it through `grep` filters that list down to just nginx-related lines.
Confirms the same PIDs from a second angle.

---

## 2. Service Checks
**Why:** A process can be running without systemd (the init system — the first
process started after the kernel loads, responsible for starting/managing every
other service) actually recognizing or managing it properly. This confirms nginx
is a properly registered, systemd-managed service, not just a rogue process.

    $ systemctl status nginx
    ● nginx.service - active (running) since ... 7min ago
      Loaded: enabled
      Main PID: 21153 (nginx)

- `Loaded: enabled` means nginx is set to auto-start on every future boot —
  not just running right now by coincidence.
- `Main PID: 21153` matches exactly what `pgrep` showed earlier — confirms
  systemd and the raw process list are both looking at the same real process.

    $ systemctl list-units --type=service | grep nginx
    nginx.service   loaded active running

Confirms systemd itself has nginx registered as an active, managed unit.

---

## 3. Log Checks
**Why:** Status tells you the current state; logs tell you the history —
what actually happened, and when.

    $ journalctl -u nginx -n 20
    Starting nginx.service...
    Started nginx.service.

`journalctl` reads systemd's centralized logging system. `-u nginx` filters to
just this **unit** (systemd's term for a managed service). `-n 20` limits output
to the last 20 lines. This log only tracks the service's *lifecycle*
(start/stop/restart/fail) — not the actual traffic it's serving.

    $ sudo tail -n 20 /var/log/nginx/access.log
    [GET /] 200          — real visitor, page served successfully
    [GET /favicon.ico] 404  — no favicon file exists (harmless, expected)
    [GET /robots.txt] 404   — Bing's crawler bot checking crawl rules (harmless)
    [GET /] 200          — same bot successfully indexed the homepage

This is a completely different, more detailed log — nginx's own **access log**,
which records every actual HTTP request that hits the server (who requested
what, and the result). `tail -n 20` shows just the last 20 lines rather than
the entire file history.

---

## 4. Mini Troubleshooting Notes

**Typo lesson — why this matters:**
Ran `ps aux | grep ngnix` (misspelled nginx). Got a result back — but it was
`grep` matching its own command text (since `ps aux` lists grep itself as a
running process, and the typo "ngnix" appeared inside grep's own command line).
`grep` can't tell "doesn't exist" from "you spelled it wrong" — both can look
like a valid result. Lesson: always sanity-check the search term before
trusting the output.

**Flag case-sensitivity — why this matters:**
`journalctl -u nginx` (lowercase `-u` = "unit," correct) vs.
`journalctl -U nginx` (capital `-U` = "since [timestamp]" — tried to parse
"nginx" as a date and failed). One letter's case completely changes a flag's
meaning in Linux — worth double-checking flags, not just command names.

**404s aren't automatically errors — why this matters:**
A 404 just means "the specific thing requested doesn't exist" — not that the
server itself is broken. Both 404s here (favicon, robots.txt) are expected and
harmless for a small personal project with neither file added.

**Bonus: `ssh` and socket activation — why this matters:**
Checked `systemctl status ssh`, saw "disabled" and "inactive (dead)" despite
being actively connected via SSH at that moment — initially looked broken.

Discovered this is **socket activation**: `ssh.socket` is a separate,
lightweight component that stays always-listening on port 22, doing nothing
else. The moment an actual connection request arrives, the socket triggers
`ssh.service` to start on-demand and handle that session; once the session
ends, the service can go back to sleep while the socket keeps listening for
the next one. "Disabled" here just meant "don't run constantly at boot" —
not "broken." Confirmed via the `TriggeredBy: ssh.socket` line in the status
output.
