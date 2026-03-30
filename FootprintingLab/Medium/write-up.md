# 1. Reconnaissance

## Nmap

Starting with a basic Nmap scan

![alt text](images/image.png)

![alt text](images/image-1.png)

We found many services with open ports:

Port 111 - rpcbind

Port 2049 - nfs/nlockmgr - Sharing files using NFS

Port 135 - msrpc - Microsoft Remote Procedure Call

Port 139 and 445 - NetBIOS and SMB - File, printer sharing

Port 3389 - ms-wbt-server - Remote Desktop Protocol, graphical login.

Port 5985 - http - Windows Remote Management WinRM - remote shell.

#

Based on this scan we can come up with a strategy of order based on the priority since some services are more robust and might require credentials.

NFS/SMB - Potential open doors

RPC - Gather information

RDP/WinRM - If info/password log in

## NFS

We start with NFS because many times the shares are accessible to anyone on then network.

![alt text](images/image-2.png)

![alt text](images/image-3.png)

We can see in both scanns a TechSupport mount.

## RPC & SMB

We see how the service is operating as we use RPC but we found a login

![alt text](images/image-6.png)

Also tried Samba Share Enumerator tool

![alt text](images/image-7.png)

And some manual SMB client connections

![alt text](images/image-8.png)


# 2. Vulnerability Discovery

## NFS

After discovering thhe public share, we mount it locally to investigate it

![alt text](images/image-4.png)

And found a batch of tickets.txt

![alt text](images/image-5.png)

Investigating a bit more the tickets we run:

![alt text](images/image-9.png)

To see if we get any match, and we do.

# 3. Exploitation and Post Exploitation

After finding a ticket.txt that exposed some password we try to read it all to get more context

![alt text](images/image-10.png)

![alt text](images/image-11.png)

We found many data about a user "alex" like mail and credentials

This gives us access to the RPC service

![alt text](images/image-12.png)

Trying the found credentials on smbclient:

![alt text](images/image-13.png)

We find the admin shares

We manage to get inside the directories and we start exploring finding:

![alt text](images/image-14.png)

an "important.txt" file

![alt text](images/image-15.png)

sa usually stands for System Administrator and is the default account for Microsoft SQL Server (MSSQL).

We confirm it

![alt text](images/image-16.png)

Using WinRM service and the credentials

We get full access as administartor to the target machine

![alt text](images/image-17.png)

With the credentials found we also try to access the SQL server that we check that is running on local machine since we could not find user "HTB" on system.

We run a check on running services

![alt text](images/image-18.png)

Then we run the sql commands to enumerate the directories

![alt text](images/image-19.png)

and find our flag

![alt text](images/image-20.png)

# 4. Remediation

Key lessons:

- Secure Network File Shares: The entry point was a public TechSupport share that exposed credentials from a user.

- The escalation was posible because with the credentials found on ticket.txt we could access the files and directories shared in network. Allowing us to find a devshare with System Administrator credentials in plain text.

- Also service, specially critical ones, should not share passwords.

- The Database access was closed to the network but accesible locally with higher privileges.

- We should encrypt passwords in data bases also. Never save passwords as plain text.


