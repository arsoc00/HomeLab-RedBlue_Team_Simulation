### **Note:** *If you want to view the complete project with all the screenshots, please check the project's PDF version available in this repository.*

# HOME LAB: RED & BLUE TEAM SIMULATION - arsoc00
This project was created to test defensive and offensive security. The entire network infrastructure and the attacks have been simulated locally on virtualized machines.

## 1. INTRODUCTION AND OBJECTIVES
The fictional background of the project is that the company “Corporation” has a server (corp-server) located in a DMZ, on which DHCP, DNS, and HTTP services are installed to support its business operations, as well as an internal LAN for its employees; both are connected to each other via a switch, and the switch connects them to the company’s router. The server has a Wazuh agent installed, which is monitored by a computer on the internal LAN that acts as a manager.

The objective is to practice enumeration, intrusion, and privilege escalation techniques in terms of offensive security, as well as log analysis, detection, and hardening in terms of defensive security. It is also designed as a homemade CTF and includes a flag that indicates the success of the attack on the server.

## 2. NETWORK ARCHITECTURE
We’ll begin by explaining the network architecture, created using Cisco Packet Tracer. Here, you can clearly see how the server is located on a network separate from the internal LAN via a switch and how the switch, in turn, connects to the internet via the corporate router. The network components have been configured as if they were operating on a real network, solely for practice purposes. The project itself was carried out using virtual machines on an internal network. The project was carried out using virtual machines on an internal network in VirtualBox.

The network assignment table would look like this:

## 3. NETWORK EQUIPMENT CONFIGURATION
ISP ROUTER CONFIGURATION
The only settings configured on the ISP router are the IP address and subnet mask, so that the corporate router can connect.

CORPORATE ROUTER CONFIGURATION
The IP address, subnet mask, and interfaces have been configured on the corporate router, and the “encapsulation dot1Q” command has been used to separate network traffic for each VLAN. Security measures, such as password encryption, have also been implemented, along with passwords for accessing the router’s configuration and for remote access.

CORP-SWITCH CONFIGURATION
The DMZ and internal LAN VLANs have been defined on the switch, and “switchport trunk encapsulation dot1Q” and “switchport mode trunk” have been used to enable data to be sent to each VLAN separately. Security measures were then defined in the same way as on the router.

## 4. CORP-SERVER SERVICES INSTALLATION
As previously mentioned, virtual machines in VirtualBox were used to carry out this project; therefore, it would not be necessary to configure the server according to the network topology. However, to follow the same approach, this section explains how the network and DHCP would be configured if such topology existed.

I chose Ubuntu Server version 26.04.1 as the server. I had to configure DHCP, DNS, HTTP, and SSH servers. For the server to function as a DHCP server, you must first set a static IP address, which would be 192.168.10.10.
I installed the “isc-dhcp-server” DHCP server. Next, you need to edit the configuration file at “/etc/dhcp/dhcpd.conf.” As we mentioned earlier, the server is located in the DMZ, which is VLAN 10, and the internal LAN is VLAN 20. Since the router has a static IP address, addresses are assigned starting from 192.168.10.11.

For the DNS server, I used “bind9.” The DNS server is configured by creating a configuration file in the /etc/bind folder; in this case, “db.corporationcom”.

Unfortunately, I forgot to take screenshots of the HTTP server configuration—I used Apache2—but it was a very straightforward setup with a very simple index.html file; I also created a few more folders to perform a directory enumeration. Here are the screenshots showing the services running correctly.

I also set up an SSH service as a gateway for the attack.

After installing all the services on the server, I needed a way to monitor any security issues that might arise, so I installed Wazuh on a computer on the internal LAN (Ubuntu Desktop) to act as the manager, and the Wazuh agent on the server.

## 5. ATTACK SIMULATION
To carry out the attack on the server, I used a Kali Linux virtual machine, version 2026.2. The simulation was based on the premise that the attacker had already gained access to the network where the server was located. Keep in mind that in the screenshots, there is a 6-hour time difference between Kali Linux and the server.

I began the attack by scanning the network to locate the server using “arp-scan.” Two IP addresses appeared: the first was the server’s, and the second belonged to another VM that was not part of the project but had connected to the server by mistake, so we ignored it.

Now that we have the server's IP address, we'll perform a port scan to identify the services it runs, their versions, and so on. Since I didn't need a lot of information and didn't need to be discreet, I used the “-p-” options to view all open ports, “-A” to identify the operating system and versions, and “-T5” to speed up the scan.

Since it appears there is an SSH service running on port 22, I made an initial brute-force attempt using the Hydra tool with the username “admin” to see how the information appears in Wazuh. I narrowed down the rockyou.txt list a bit so it would run more smoothly. It didn’t work because OpenSSH has built-in protection against brute-force attacks, and since I used the “-T4” option, multiple connections were attempted simultaneously, causing the service to start rejecting them. Fortunately, some of them were logged by Wazuh.

Since HTTP servers are often an entry point and sometimes provide some clues, I proceed to enumerate the directories using Gobuster and the common.txt list, identifying three interesting directories: admin, backup, and uploads.

Wazuh also displays a list of directories. In the alert details, you can see the requests that were sent to the server to retrieve the directories. Furthermore, the fact that “gobuster” appears as the User-Agent in the requests also gives us a clue that we are under attack.

Now that we have the Apache server directories, let's check what's in them. I'll start by checking the index.html file to see what appears, and in the first directory I check (/admin), I find a secret.txt file that says we should delete a supposed test user (testuser).

Once we had obtained a username, we repeated the brute-force attack on the SSH service, including that username in the edited list, in case it was used as the password. I forgot to add the “-T1” option so the attack wouldn’t fail, but since I had added “testuser” to the beginning of the list, it did manage to obtain the credentials.

In Wazuh, this event is a high-severity alert (level 12 or higher), as can be seen in the “Threat Hunting” section of the dashboard. In the log, we can verify that this is an SSH connection from the IP address 192.168.10.14 using the username “testuser” and that the connection occurred on port 51618. The alert description is “Multiple authentication failures followed by a successful login,” which accurately describes how Hydra operates. In the alert details, Wazuh also maps the event to the Mitre ATT&CK framework and provides the tactics and techniques the attacker may be following and using, along with the technique ID.

I continued the attack by initiating an SSH connection using the “testuser” credentials. I also found this in Wazuh. The alert details show once again that the connection originated from the Kali Linux IP address, that the connection was made via interactive keyboard for the “testuser” account (meaning the password was typed in), and that the connection was made on port 43528.

To finish the attack, I check the permissions for “testuser,” who has been mistakenly granted full permissions, so I simply need to switch to the root user and look for flag.txt in their /home directory.

## 6. HARDENING
Since we have already simulated the attack, we are going to harden the server so that it cannot be attacked again. I have started by blocking password-based access to the SSH server so that connections are only possible using public keys.

When I restart the SSH server to apply the configuration, I cut off the attacker's connection. I also verify that I can no longer access the server with the password.

Next, I changed the SSH server port from 22 to 2222.

To prevent the “testuser” account from being used again, I delete it along with all its folders and verify that the deletion was successful.

Next, I checked the permissions each user had in the sudoers file and removed those for the “testuser” account.

I also established some rules for creating and changing passwords. Now, passwords must be changed every 90 days, with a minimum interval of 7 days between changes and 14 days’ notice before the change. I installed “pam_pwquality” and edited the configuration file /etc/pam.d/common-password to enforce complex passwords—with a minimum length of 12 characters and including one uppercase letter, one lowercase letter, one number, and one special character—even for the root user. Afterward, I verified that the configuration was working as intended.

And finally, I set up iptables to control connections to the server. The idea was to secure the SSH server by limiting connection attempts and thus prevent further brute-force attacks, but also to close all ports that weren't necessary for the server to function properly. I broke them down into sections to make them easier to see in the screenshot; you don’t actually need to do that. Keep in mind that iptables rules are read as a list from top to bottom.

Now let's explain the iptables rules:

    sudo iptables -P INPUT DROP
    sudo iptables -P FORWARD DROP
    sudo iptables -P OUTPUT ACCEPT
These iptables rules block all incoming traffic and forwarded traffic and allow outgoing traffic.

    sudo iptables -A INPUT -i lo -j ACCEPT
    sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
The first rule allows incoming traffic on the loopback interface because services sometimes communicate with themselves; otherwise, they would fail. The second rule allows already established connections so that the Wazuh agent installed on the server can maintain its connection with the manager after it starts up.

    sudo iptables -A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --set --name SSH_ATTACK
    sudo iptables -A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --update --seconds 60 --hitcount 3 --name SSH_ATTACK -j DROP
These two iptables rules are used to prevent new brute-force attacks against the SSH server. They establish a policy for the maximum number of connection attempts per minute using iptables. The first rule logs the IP addresses attempting to open a new connection into a list called “SSH_ATTACK,” while the second rule checks the list; if it finds an IP address that has attempted to connect more than 3 times in less than 60 seconds, the connection is rejected.

    sudo iptables -A INPUT -p tcp --dport 2222 -j ACCEPT
    sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
    sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
    sudo iptables -A INPUT -p tcp --dport 53 -j ACCEPT
    sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
    sudo iptables -A INPUT -p udp --dport 67 -j ACCEPT
    sudo iptables -A INPUT -p udp --dport 68 -j ACCEPT
All of these iptables rules are designed to allow incoming traffic only to the services running on the server: SSH, HTTP and HTTPS, DNS, and DHCP.
To save the iptables rules, you must install iptables-persistent. Otherwise, they would be lost when the server is shut down because, after they are created, they remain only in RAM.

To secure the Apache server, I made the following configuration changes:

- I disabled the “Indexes” option in the /etc/apache2/apache2.conf file to prevent users from viewing files in a folder if there is no index.html file in that folder.

- I installed Fail2ban to block directory traversal. The default configuration file is copied to prevent changes from being overwritten, and it is edited in /etc/fail2ban/jail.local. Each section is a “jail”; I edited the “apache-badbots” and “apache-noscript” jails. The first one detects crawlers that appear as User-Agent in the HTTP header and blocks them. The second one monitors the /etc/apache2/error.log file and detects and blocks IP addresses that receive too many errors during their searches.

## 7. HARDENING TEST
The first thing to check is whether the SSH server port has been successfully changed from 22 to 2222.

Next, I wanted to try a brute-force attack to verify that the attacker's IP address was being blocked, but Hydra didn't work because of the password authentication block, so we treated the IP block based on failed attempts as a double safeguard.

The next step would be to verify that the configuration to prevent exploitation of the HTTP server is working. The “-Indexes” setting does work, so the contents of the “/admin” folder—where the secrets.txt file was located—can no longer be accessed.

Unfortunately, Gobuster was able to enumerate the directories without any problems, so the directory scan block is not configured properly.

After doing some research, it turns out that Fail2ban’s default filters block outdated User-Agents and do not work with Gobuster, even if it appears in the request. To block enumeration attacks, you need to add a custom rule that monitors access logs to record the IP addresses that generate multiple requests with a 403 (Forbidden) or 404 (Page Not Found) status code. The custom rule “apache-custom.conf” is added as a file in the filters folder at “/etc/fail2ban/filter.d” and then added as a jail to the “jail.local” file.

With this new custom configuration, the Gobuster connection is rejected, thereby blocking directory enumeration.
