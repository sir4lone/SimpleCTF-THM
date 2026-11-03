# Write-Up: Simple CTF - TryHackMe

 ### • Difficulty: Easy
 ### • OS: Linux
 ### • Platform: TryHackMe
 ### • Category: Web / FTP Enumeration / Password Cracking / Privilege Escalation

## Summary

An initial network scan revealed an HTTP web server, an anonymous FTP service, and an SSH service listening on a non-standard port (2222). Web enumeration identified a Content Management System (CMS Made Simple) vulnerable to an unauthenticated Time-Based Blind SQL Injection (CVE-2019-9053).
Simultaneously, enumerating the FTP service using passive mode allowed the retrieval of a system note containing username intelligence (mitch). A targeted SSH brute-force attack using Hydra against port 2222 successfully cracked the user's password (secret), granting initial access and the user flag. Privilege escalation was successfully executed by auditing sudo rights (sudo -l), which revealed unrestricted NOPASSWD execution of /usr/bin/vim, allowing a root shell escape via GTFOBins (:!/bin/sh).

## 1. Network Reconnaissance (Nmap)
   
The operation began with an Nmap scan of all ports to detect all open ports, running services, and the versions of those services on the target machine.

### Command Execution

```
sudo nmap -p- --open -sS -sV --min-rate 5000 -vvv -n -Pn 10.67.178.187 -oN ScanSmplectf1.txt
```

### Command Explanation:

    • sudo: Runs Nmap with superuser privileges, required for raw socket access when performing SYN Stealth scans.
    • nmap: Invokes the Network Mapper scanner tool.
    • -p-: Scans all 65,535 TCP ports on the target.
    • --open: Filters the output to display only ports with an open status.
    • -sS: Performs a TCP SYN Stealth scan (half-open scan), which is faster and less noisy than a full TCP connection.
    • -sV: Enables service version detection to identify running software versions.
    • --min-rate 5000: Forces Nmap to send packets no slower than 5000 packets per second to speed up the full port scan.
    • -vvv: Increases verbosity to level 3, printing open ports in real-time as soon as they are discovered.
    • -n: Disables DNS reverse name resolution to speed up the scanning process.
    • -Pn: Treats the target host as online, skipping ICMP ping probing.
    • 10.67.178.187: Specifies the target IP address.
    • -oN ScanSmplectf1.txt: Saves the scan output in standard normal Nmap format to ScanSmplectf1.txt.

  <img width="822" height="600" alt="scan" src="https://github.com/user-attachments/assets/0fcdf716-b225-4a9d-b4f2-f9280efecdd3" />

  ### Findings Breakdown
  
  1. Port 21 (FTP): Running vsftpd 3.0.3. Targeted for anonymous login check.
  2. Port 80 (HTTP): Running Apache httpd 2.4.18. Web application target.
  3. Port 2222 (SSH): Running OpenSSH 7.2p2. The standard SSH port (22) has been relocated to non-standard port 2222.

## 2. Web Enumeration & CVE Vulnerability Analysis (Port 80)
   
Navigating to http://10.67.178.187/ presented a default Apache HTTP server page. Directory enumeration revealed a Content Management System (CMS) installation.

### CMS Identification & CVE Mapping.

  • Target Software: CMS Made Simple (version < 2.2.10)
  • Vulnerability Class: Time-Based Blind SQL Injection (SQLi)
  • CVE Identifier: CVE-2019-9053

### Technical Analysis of CVE-2019-9053

CVE-2019-9053 is an unauthenticated Time-Based Blind SQL Injection vulnerability located in the News module of CMS Made Simple.

  • Mechanism: The application fails to properly sanitize user input supplied to module parameters. Because responses do not return raw SQL errors, an attacker must infer database contents based on response time delays (e.g., using SQL SLEEP() functions).
 
  • Impact: Exploit scripts systematically extract data character-by-character, allowing attackers to dump database credentials, administrative user hashes, and configuration details.

## 3. Anonymous FTP Enumeration (Port 21)

To check for basic configuration errors, an anonymous connection attempt was initiated against the FTP service on port 21.

### Command Execution & Passive Mode Fix
```
ftp 10.67.178.187
```
### Command Explanation:

ftp: Initiates an interactive File Transfer Protocol client session to the target IP.

Name (10.67.178.187:user): anonymous

Upon authenticating, issuing a directory listing command (ls -la) stalled due to firewall/NAT data channel restrictions. Toggling to passive mode resolved the issue:
```
ftp> passive
ftp> ls -la
```
### Command Explanation:

  • passive: Toggles FTP data transfer mode from Active to Passive, forcing the client to establish the data connection and bypassing incoming firewall blocks.
  
  • ls -la: Lists all directory contents including hidden files in long format.

<img width="485" height="454" alt="ftpscreen" src="https://github.com/user-attachments/assets/630c7548-1796-4bdd-a72e-a2658d045996" />

### Downloading and Reading Intelligence Artifacts
```
ftp> cd pub
ftp> get ForMitch.txt
ftp> exit
```
### Command Explanation:

  • cd pub: Navigates into the pub (public) remote folder.
  
  • get ForMitch.txt: Downloads ForMitch.txt from the remote server to the local system directory.
  
  • exit: Terminates the FTP session.

### Inspecting the note contents:
```
cat ForMitch.txt
```
### Command Explanation:
  
  • cat: Concatenates and displays the text contents of ForMitch.txt in the terminal.
    
### Intelligence Recovered:

  • Username: mitch
  
  • Note Content: Confirmed that user mitch had set an extremely weak password, validating a brute-force approach.

<img width="925" height="208" alt="ftpscreen2" src="https://github.com/user-attachments/assets/ba42f151-cf58-448f-ae22-e70cf17a2808" />
<img width="1334" height="62" alt="messagelol" src="https://github.com/user-attachments/assets/5ed9c088-e1fe-4b2e-93d9-18966263bf41" />

## 4. SSH Brute-Force Attack (Port 2222)
   
With a valid target username (mitch) and a non-standard SSH port (2222), Hydra was executed using the standard rockyou.txt password dictionary.

### Command Execution
```
hydra -l mitch -P /usr/share/wordlists/rockyou.txt ssh://10.67.178.187:2222 
```
### Command Explanation:

  • hydra: Invokes the online password cracking tool.
  
  • -l mitch: Specifies the single target username (mitch).
  
  • -P /usr/share/wordlists/rockyou.txt: Points to the local wordlist containing common passwords.
  
  • ssh://10.67.178.187:2222: Defines the protocol (ssh), target IP, and target port (2222).

### Discovered Credentials: mitch:secret

<img width="1409" height="256" alt="hydrapasswd" src="https://github.com/user-attachments/assets/3d2d0293-8315-4b89-827d-31aeeba22d52" />

## 5. Initial Access & User Flag

With cracked credentials, an interactive remote SSH session was opened on port 2222.

### Command Execution
```
ssh mitch@10.67.178.187 -p 2222
```
### Command Explanation:

  • ssh: Launches the Secure Shell client.
  
  • mitch@10.67.178.187: Specifies connection as user mitch to the target IP.
  
  • -p 2222: Forces SSH to connect over port 2222 instead of default port 22.
  
<img width="762" height="543" alt="sshmitch1" src="https://github.com/user-attachments/assets/901a4ea8-72c2-46b6-8d68-05355618c113" />

### Reading User Flag
```
mitch@SimpleCTF:~$ cat user.txt
G00d j0b, keep up!
```
### Enumerating Local Users
```
mitch@SimpleCTF:~$ ls -la /home
```
### Command Explanation:

  • ls -la /home: Lists all user directories present under /home to identify co-existing system accounts.

<img width="461" height="116" alt="seconname" src="https://github.com/user-attachments/assets/4d620ee4-57a3-4dfb-8014-03382d6b4a9c" />

### Second System Account Identified: sunbath

## 6. Privilege Escalation to Root

To establish a path to root, sudo permissions assigned to user mitch were audited.

### Auditing Sudo Privileges
```
mitch@SimpleCTF:~$ sudo -l
```
### Command Explanation:

  • sudo -l: Lists the allowed (and forbidden) commands for the invoking user on the current host.

### Output

User mitch may run the following commands on SimpleCTF:
   ``` 
  (root) NOPASSWD: /usr/bin/vim
  ```

### Exploitation via GTFOBins (Vim Shell Escape)

According to GTFOBins, text editors like Vim possess built-in escape functionality that allows users to spawn system shells from within the application interface.
   
   1. Launch Vim with Root Privileges:
       sudo /usr/bin/vim

### Command Explanation:

  sudo /usr/bin/vim: Executes Vim directly as the root superuser.
  
  2. Execute Shell Escape Sequence Inside Vim:

     Press Esc to ensure Normal mode, then type:
      ```
      :!/bin/sh
      ```
 ### Syntax Explanation:
 
   : enters command-line mode in Vim.
   
   ! instructs Vim to pass the following command to the underlying operating system shell.
   
   /bin/sh launches a standard Bourne shell binary. Because Vim runs as root, the spawned shell inherits root permissions.
     
  3. Verify Identity:

      whoami
     
      root

### Reading Root Flag
```
# cat /root/root.txt
W3ll d0n3. Y0u m4d3 1t!
```
<img width="445" height="135" alt="rootflag" src="https://github.com/user-attachments/assets/4b8be85d-91a6-47fa-822d-fb0fc7202cdf" />

## Conclusion & Recommendations

  1. Disable Anonymous FTP Access: Anonymous login should be disabled on FTP servers unless explicitly required. Sensitive file notes must never reside in publicly accessible locations.
 
  2. Enforce Strong Password Policies: Users should be required to set complex passwords that resist dictionary attacks via SSH or web logins.
     
  4. Restrict Sudo Privileges: Text editors (vim, nano), pagers (less, more), and command interpreters must never be assigned unrestricted NOPASSWD execution rights in /etc/sudoers.

     
