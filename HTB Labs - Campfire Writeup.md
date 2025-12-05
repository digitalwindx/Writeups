## Before diving into our investigation let's highlight a high level overview how Kerberos works when accessing a service:
1. Client logs into the domain and requests an Ticket Granting Ticket (TGT) from the Key Distribution Center (KDC)
2. The KDC verifies the credentials and sends back an encrypted TGT and a session key
3. Client requests access to service by sending the current TGT to the Ticket Granting Service (TGS) with the Service Principal Name (SPN) of the resource
4. TGS issues a TGS ticket for the service to the client encrypted with the service account password. 
5. Client then presents the TGS ticket to the target service, which authenticates the user and begins a secure session. 
## How do Kerberoasting attacks work?
- Kerberoasting attacks exploit a combination of weak or easily guessable service account passwords, particularly when older encryption standards like RC4 are used. 
- These attacks typically follow the process outlined below:
	1. The threat actor, using any valid domain user credentials, can use a variety of tools to identify AD accounts that have SPNs (Service Principal Names) tied to them. 
	2. The threat actor then requests a Kerberos service ticket for one more more of these identified accounts from the ticket granting service (TGS) using tools like GhostPack’s Rubeus.
	3. The threat actor receives a ticket from the KDC. The ticket is encrypted with a hashed version of the account's password.
	4. The threat actor captures the TGS ticket and takes it offline.
	5. The threat actor attempts to crack the SPN credential hash to obtain the service account's plaintext password using brute force techniques or tools like Hashcat or JohnTheRipper.
	6. With the service account password in hand, the threat actor attempts to authenticate as the service account and is granted access to any service, network or system associated with the compromised account
	7. The attacker is then able to steal data, escalate privileges, or set backdoors on the network to ensure future access. 
## Analyzing Domain Controller Security Logs, can you confirm the UTC date & time when the Kerberoasting activity occurred?
- To begin this investigation, we can leverage `Event ID 4769` which indicates `A Kerberos service ticket was requested` and  `TicketEncryptionType 0x17` which indicates the `RC4` encryption standard. 
- Using this information I was able to find our log of interest: 
  ![Campfire](images/Pasted%20image%2020251205014002.png)
## What is the Service Name that was targeted?
  ![Campfire](images/Pasted%20image%2020251205014002.png)
## What is the IP Address of the workstation?
  ![Campfire](images/Pasted%20image%20251205014106.png)
## What is the name of the file used to Enumerate Active directory objects and possibly find Kerberoastable accounts in the network?
- For this next part I shifted over to the `Powershell-Operational` event file and filtered for `Event ID 4104` to view the PowerShell Script blocks. 
- Piggybacking off of the log of interest found earlier we can try to establish some sort of timeline by looking at logs that occurred close to our log of interest.
- The first log after filtering we see `powershell -ep bypass` being ran. 
  ![Campfire](images/Pasted%20image%2020251205012853.png)
- Attackers bypass PowerShell script execution policies to run malicious scripts without restrictions. Basically it is saying ignore all execution policies for this session and run whatever I give you without warnings.
- The following log displays `PowerView.ps1` being ran, which is an PowerShell designed for Active Directory enumeration.
  ![Campfire](images/Pasted%20image%20251205014429.png)
## When was this script executed? (UTC)
- The answer to this question is within the same log.
  ![Campfire](images/Pasted%20image%20251205014629.png)
## What is the full path of the tool used to perform the actual kerberoasting attack?
- To begin this section I parsed the prefetch files using the PEcmd Tool which exported everything into a CSV file, which I then opened in the Timeline Explorer tool to look for any executables of interest that executed around the timeline I have established this far. 
- Poking around in Timeline Explorer using our timeline I found `Rubeus.exe`, which is a C# toolset designed for raw Kerberos interaction. 
  ![Campfire](images/Pasted%20image%20251205015752.png)
## When was the tool executed to dump credentials? (UTC)
- Correlating with our earlier findings Rubeus was executed and one second later a Kerberos service ticket request was logged for the MSSQLService.
  ![Campfire](images/Pasted%20image%20251205020251.png)
## What can be done to combat Kerberoasting attacks?
- Ensure strong password hygiene, especially for service accounts that have SPNs related to them.
- Usage of AES encryption to encrypt Kerberos service tickets, since when AES encryption is used a stronger password hash is also used, making password cracking very difficult. 
- This [article](https://www.crowdstrike.com/en-us/cybersecurity-101/cyberattacks/kerberoasting/) from crowdstrike delves deeper into the topic if you would like to read more
