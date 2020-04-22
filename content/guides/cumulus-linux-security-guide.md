---
title: Cumulus Linux Security Guide
author: Cumulus Networks
product: Cumulus Networks Guides
version: "1.0"
Draft: True
---
Cumulus Linux is a powerful operating system for routers that comes with secure defaults and is ready to use. This whitepaper discusses additional security measures to further secure your switch and meet corporate, regulatory, and governmental standards. The topics focus on three types of security measures:

- Big impact to security and low impact to usability
- Medium impact to both security and usability
- Low impact to security and high impact to usability

These impacts are all relative but Cumulus Networks uses them to help you prioritize how to get the most value from your changes for the time you have available and for the impact to your management practices.

## Big Impact to Security and Low Impact to Usability

This section discusses issues that have the biggest security impacts with the least impact to management and user experiences.

### Hardware Security

Securing the hardware is vital because an attacker with physical access to the hardware will eventually have access to the entire device, allowing them to change configurations, capture all the traffic moving through the switch, or even steal the switch itself. If there is a security breach in the hardware, the entire system is compromised. Additionally, securing the router from various attacks or misconfigurations is very important.

### Prevent Denial of Service (DOS)

Denial of Service attacks, or DOS attacks, aim to disrupt normal use of a service or device. To create a DOS attack, an attacker sends a very large number of redundant and unnecessary requests to a target system to overwhelm it and block intended users from accessing the service or application being served by the target system. Routers and firewalls are commonly targeted by DOS attacks. Cumulus Linux comes with a built-in check system for these types of attacks. When enabled, the switch can intelligently analyze packets coming into the system and drop packets that match specific criteria.

To enable automatic checks on your switch, open the `/etc/cumulus/datapath/traffic.conf` file in a text editor and change the value of the `dos_enable` setting to `true`.

To specify which DOS checks you want to enable, open the `/usr/lib/python2.7/dist-packages/cumulus/__chip_config/bcm/datapath.conf` file and enable the desired DOS check(s) by setting their corresponding value(s) to true. Restart the `switchd` service with the `sudo systemctl restart switchd.service` command for the changes to take effect.

#### Switch Port Configuration

Cyber-attackers often steal information through vulnerable switch ports. Many companies with VLANs use VLAN 1 instead of choosing a custom VLAN ID because VLAN 1 is the default VLAN ID on most network devices. Because this default is very well known, it is the first place attackers look to gain VLAN access.

By default, the router configuration protects against VLAN hopping attacks, where an attacker can try to fool the target switch into sending traffic from other networks by using generic tags or by using dynamic VLAN negotiation protocols. If successful, the attacker can gain access to networks that are connected to the switch, but otherwise unavailable to the attacker. Cumulus Linux is built to mitigate this threat by ignoring packet requests coupled with generic tags.

To protect your ports, ensure that access ports are not assigned to VLAN 1 or any unused VLAN ID.
The following shows an example of how you configure switch ports 1 to 48 as access ports for VLAN 99:

```
net add bridge bridge ports swp1-48
net add interface swp1-48 bridge access 99
```

Ensure that no trunk ports use VLAN 1 and be thoughtful when assigning and pruning (removing) VLANs to ports. In this example, swp3 is added as a trunk port and assigned to VLANs 100 and 200.

```
cumulus@switch:~$  net add interface swp3 bridge vids 100,200
```

#### Control Plane Policy Policing

Cumulus Linux comes out of the box with a default control plane security policy that is located in the `/etc/cumulus/acl/policy.d/` directory. You can see a full list of the default rules [here](https://docs.cumulusnetworks.com/cumulus-linux/System-Configuration/Netfilter-ACLs/Default-Cumulus-Linux-ACL-Configuration/).

Best practices dictate that:

- All ACL drops are logged
- The use of IP Tables is required in configuration
- Any line with the action *LOG* must be immediately followed with the same line with the action *DROP*

Be sure to apply changes to the default control plane policies with `cl-acltool` so that they are properly hardware accelerated.

The following command applies all the ACLs and control plane policy rules in the `/etc/cumulus/acl/policy.d/` directory:

```
cumulus@switch:~$ sudo cl-acltool -i
```

You can verify that the rules are applied with the following command:

```
cumulus@switch:~$ sudo cl-acltool -L all
```

#### Disable Insecure SSL and TLS Protocol Versions in Nginx

Cumulus Linux is packaged with Nginx, an open source web server that supports the Cumulus Linux RESTful API via HTTPS. By default, Nginx is enabled and listening on localhost (127.0.0.1) port 8080. For more information, go [here](https://docs.cumulusnetworks.com/cumulus-linux/System-Configuration/HTTP-API/).

For backward compatibility, Nginx natively supports the outdated and vulnerable SSLv3 and TLSv1 protocols. To secure against potential exploits using those protocols, disable them using this procedure:

Open the `/etc/nginx/nginx.conf` file in a text editor and edit the `ssl_protocols` line to allow only the TLSv1.1 and TLSv1.2 protocols:

```
cumulus@switch:~$ sudo nano /etc/nginx/nginx.conf
...
ssl_protocols TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
```

Restart the Nginx service for the setting to take effect:

```
cumulus@switch:~$ sudo systemctl restart nginx.service
```

If you are not using the Cumulus REST API, you can uninstall Nginx. This completely eliminates the TLS attack vector:

```
cumulus@switch:~$ sudo apt-get remove python-cumulus-restapi nginx-common
cumulus@switch:~$ sudo apt-get autoremove
```

### Management Security

#### Management VRF

Management virtual routing tables and forwarding (VRF) separates out-of-band management network traffic from the in-band data plane network. Management VRF increases network security by separating the management routing tables from the main routing tables. It also protects the management network from interference or attacks directed at the routing infrastructure of the main network, which might prevent you from managing the router or switch.

To take advantage of management VRF, connect interface eth0 to the out-of-band management network. Management VRF is enabled by default in Cumulus Linux 4.0 and later. For earlier versions of Cumulus Linux, use the following commands to configure a management VRF and eth0 to be a member of that VRF:

```
cumulus@switch:~$ net add vrf mgmt
cumulus@switch:~$ net commit
```

For more information about working with a management VRF, such as running services in a specific VRF,  refer to the Management VRF documentation.

Also remember to enable all the network services inside each VRF including the following:

- Traffic flow reporting
- Syslog
- SNMP
- DNS
- NTP

#### Management ACL

Management Access Control List (ACL) is the main list of user permissions for Cumulus Linux. Review and customize the management ACL as soon as possible during the installation process to help prevent user errors or malicious behavior by restricting the abilities of administrative users. Due to many unique needs and environments, the management ACL is highly customizable; it is important that you change the defaults.

Use the following guidelines as a starting point to build your management ACL. These guidelines reference example IP addresses and ports that are shown in detail in the firewall rules in the next section.

- Allow SSH inbound only for the management network (192.168.200.0/24).
- Allow UDP ports for the DHCP client on the management network (UDP port 67 & 68).
- Allow SNMP polling only from specific management stations (192.168.200.1).
- Allow NTP only from internal network time servers (192.168.200.1).
- Allow DNS only from configured DNS servers (192.168.200.1).
- Allow TACACS from the management network on eth0 (TCP port 49).
- Allow MLAG traffic on the backup interface eth0 (UDP port 5342).
- Allow outbound syslog only to known logging stations (UDP port 514).
- Allow outbound Flow only to known flow collectors (UDP port 6343).
- Allow outbound connection for the NetQ agent to the NetQ server (TCP port 31980).
- Block transit traffic on the management network (allow ingress to the switch or egress from the switch).
- Allow traffic to and from the local subnets to be forwarded through the data plane switch ports.

Create iptables rules to restrict traffic flow in both directions; inbound and outbound. In the following example, the switch is assigned the management IP address 192.168.200.29.

To create the sample iptables firewall rules, create the `/etc/cumulus/acl/policy.d/50management-acl.rules` file, then add the following contents to the file:

```
cumulus@switch:~$ sudo nano /etc/cumulus/acl/policy.d/50management-acl.rules
[iptables]
INGRESS_INTF = swp+
-A INPUT -i eth0 -p udp --dport 68 -j ACCEPT
-A INPUT -i eth0 -s 192.168.200.0/24 -d 192.168.200.29 -p tcp --dport 22 -j ACCEPT
-A INPUT -i eth0 -s 192.168.200.10 -d 192.168.200.29 -p udp --dport 161 -j ACCEPT
-A INPUT -i eth0 -s 192.168.200.1 -d 192.168.200.29 -p udp --dport 123 -j ACCEPT
-A INPUT -i eth0 -s 192.168.200.1 -d 192.168.200.29 -p udp --dport 53 -j ACCEPT
-A INPUT -i eth0 -s 192.168.200.0/24 -d 192.168.200.29 -p udp --dport 5342 -j ACCEPT
-A INPUT -i eth0 -s 192.168.200.0/24 -d 192.168.200.29 -p udp --sport 5342 -j ACCEPT
-A INPUT -i eth0 -j LOG
-A INPUT -i eth0 -j DROP
-A OUTPUT -o eth0 -p udp --dport 67 -j ACCEPT
-A OUTPUT -o eth0 -s 192.168.200.29 -d 192.168.200.0/24 -p tcp --sport 22 -j ACCEPT
-A OUTPUT -o eth0 -s 192.168.200.29 -d 192.168.200.0/24 -p udp --sport 5342 -j ACCEPT
-A OUTPUT -o eth0 -s 192.168.200.29 -d 192.168.200.0/24 -p udp --dport 5342 -j ACCEPT
-A OUTPUT -o eth0 -s 192.168.200.29 -d 192.168.200.1 -p udp --dport 161 -j ACCEPT
-A OUTPUT -o eth0 -s 192.168.200.29 -d 192.168.200.1 -p udp --dport 123 -j ACCEPT
-A OUTPUT -o eth0 -s 192.168.200.29 -d 192.168.200.1 -p udp --dport 53 -j ACCEPT
-A OUTPUT -o eth0 -s 192.168.200.29 -d 192.168.200.1 -p udp --dport 514 -j ACCEPT
-A OUTPUT -o eth0 -s 192.168.200.29 -d 192.168.200.1 -p udp --dport 6343 -j ACCEPT
-A OUTPUT -o eth0 -s 192.168.200.29 -d 192.168.200.250 -p tcp --dport 31980 -j ACCEPT
-A OUTPUT -o eth0 -j LOG
-A OUTPUT -o eth0 -j DROP
-A INPUT --in-interface $INGRESS_INTF -s 10.0.0.0/8 -d 10.0.0.0/8 -j ACCEPT
-A INPUT --in-interface $INGRESS_INTF -j LOG
-A INPUT --in-interface $INGRESS_INTF -j DROP
```

{{%notice note%}}

This ACL does not take effect until you apply it with the `cl-acltool -i` command.

{{%/notice%}}

### User Security

#### Lock the root Account

Safeguarding access to the root account is imperative to a secure system. Users should not have direct access to the root account. This helps to prevent universal changes from being made to the system accidentally. By default, the root account is locked.

To verify the root account status, run the following command:

```
cumulus@switch:~$ sudo passwd -S root
```

In the command output, the `L` (immediately following the word `root`) indicates the account is locked:

```
cumulus@switch:~$ sudo passwd -S root
root L 08/07/2019 0 99999 7 -1
cumulus@switch:
```

The `P` (immediately following the word `root`) indicates the account is *not* locked:

```
cumulus@switch:~$ sudo passwd -S root
root P 09/10/2019 0 99999 7 -1
cumulus@cumulus:mgmt-vrf:~$
```

To lock the root account, run the following command:

```
cumulus@switch:~$ passwd -l root
```

#### Harden sudo Access

The sudo command allows you to execute programs in Linux with the security privileges of a superuser. It is important to enforce sudo rules to avoid abuse. By default, sudo credentials are cached for a set amount of time after you execute a command with privileges via sudo.

To increase security, configure Cumulus Linux to require the superuser password with the `sudo` command. Run the `visudo` command to edit the default settings and change the `timestamp_timeout` option to `0`:

```
cumulus@switch:~$ sudo visudo
Defaults        env_reset,timestamp_timeout=0
```

Ensure there are no occurrences of `NOPASSWD` or `!authenticate` in the `/etc/sudoers` file or in files in the `/etc/sudoers.d` directory:

```
cumulus@switch:~$  sudo grep NOPASSWD /etc/sudoers
cumulus@switch:~$  sudo grep NOPASSWD /etc/sudoers.d/*
cumulus@switch:~$  sudo grep \!authenticate /etc/sudoers.d/*
cumulus@switch:~$  sudo grep \!authenticate /etc/sudoers
```

#### Session Timeouts and Limits

It is important to limit the number of sessions a user can run at the same time so that a single user or account does not make contradictory changes. Good practice is to set the limit to 10.

Add the following line at the top of the `/etc/security/limits.conf` file:

```
cumulus@switch:~$ sudo nano /etc/security/limits.conf
hard maxlogins 10
```

Add the following lines to the `/etc/profile.d/autologout.sh` script to set the inactivity timeout to five minutes (600 seconds). Create the file if it does not exist already:

```
cumulus@switch:~$ sudo nano /etc/profile.d/autologout.sh
TMOUT=600
readonly TMOUT
export TMOUT
```

Add the following line to the `/etc/profile` file to set a console timeout:

```
cumulus@switch:~$ sudo nano /etc/profile.d/autologout.sh
export TMOUT=600
```

### Remote Access Security

#### Restrict SSH

Secure Shell (SSH) is a protocol for secure remote login and other secure network services over an insecure network. It is most commonly used to allow administrators to securely access remote systems, such as Linux.

By default, Cumulus Linux includes the following ssh security settings:

- Version 2 of the SSH protocol is on.
- Empty passwords are not permitted.
- User environment is not permitted.
- Print last login is enabled.
- Strict modes is enabled.
- User privilege separate is enabled.
- Encryption for connections for interactive users.

SSH public key files are protected with permissive mode 0644:

```
cumulus@switch:~$ ls -l /etc/ssh/*.pub
-rw-r--r-- 1 root root 602 Apr 29  2017 /etc/ssh/ssh_host_dsa_key.pub
-rw-r--r-- 1 root root 174 Apr 29  2017 /etc/ssh/ssh_host_ecdsa_key.pub
-rw-r--r-- 1 root root  94 Apr 29  2017 /etc/ssh/ssh_host_ed25519_key.pub
-rw-r--r-- 1 root root 394 Apr 29  2017 /etc/ssh/ssh_host_rsa_key.pub
SSH private host key files under /etc/ssh to 0600 with the following command:
ls -alL /etc/ssh/ssh_host*key
-rw------- 1 root root  668 Apr 29  2017 /etc/ssh/ssh_host_dsa_key
-rw------- 1 root root  227 Apr 29  2017 /etc/ssh/ssh_host_ecdsa_key
-rw------- 1 root root  399 Apr 29  2017 /etc/ssh/ssh_host_ed25519_key
-rw------- 1 root root 1679 Apr 29  2017 /etc/ssh/ssh_host_rsa_key
```

To secure SSH further, consider enabling or reviewing the following options in the `/etc/ssh/sshd_config` file:

- SSH should listen on the eth0 or management VRF interfaces only. Option: `ListenAddress`
- Disable root SSH login access. Option: `PermitRootLogin no`
- Review default SSH cryptographic policy
- Review enabled ciphers
- Review Message Authentication Codes
- Review HostKeyAlgorithms
- Review KexAlgorithms
- Enable SSH session timeouts
- Can be accomplished with a combination of the two options: `ClientAliveInterval`, `ClientAliveCountMax`
- Use key based authentication
- Disable SSH compression

For more information about these options in `sshd`, click [here] (https://linux.die.net/man/5/sshd_config).

### Secure Network Protocols

#### Enable NTP Authentication

Network time protocol synchronizes the time between a computer client and or server to another time source or server. Time synchronization is critical for authentication and for log management. To mitigate attacks involving forged time synchronization, connect your Cumulus Linux switch to an authenticated NTP server.

Add authentication keys to the `/etc/ntp.keys` file:

```
cumulus@switch:~$ sudo nano /etc/ntp.keys
#
# PLEASE DO NOT USE THE DEFAULT VALUES HERE.
#
#65535  M  akey
#1      M  pass
1  M  cumulus123
```

After you add the keys to the file above, add at least two servers and their associated trusted keys to the `/etc/ntp.conf` file:

```
cumulus@switch:~$ sudo nano /etc/ntp.conf
server 192.168.0.254 key 1
keys /etc/ntp.keys
trustedkey 1
controlkey 1
requestkey 1
```

To configure time synchronization to occur at least once every 24 hours, add or correct the following lines in the `/etc/ntp.conf` file. Replace `[source]` in the following line with an authoritative time source:

```
cumulus@switch:~$ sudo nano /etc/ntp.conf
maxpoll = 17
server [source] iburst
```

Restart NTP for the changes to take effect:

```
cumulus@switch:~$ sudo systemctl restart ntp@mgmt.service
```

#### Routing Protocol Security

Open Shortest Path First (OSPF) and Border Gateway Protocol (BGP) are dynamic routing protocols that allow routers to negotiate with each other to determine the best path to send traffic through the network. During this negotiation, routers learn about the networks connected to the other routers. The routers then use this information to determine where to send traffic on the network.

If left unsecured, an attacker can exploit a dynamic routing protocol such as OSPF or BGP to reroute packets to a rogue system instead of its intended destination. To help mitigate this threat, enable authentication on these protocols.

To configure OSPF authentication, two NCLU commands are required: one to add the key and a second to enable authentication on the interface:

```
cumulus@switch:~$ net add interface swp# ospf message-digest-key 1 md5 thisisthekey
cumulus@switch:~$ net add interface swp# ospf authentication message-digest
```

For more information, click [here](https://docs.cumulusnetworks.com/version/cumulus-linux-37/Layer-3/Open-Shortest-Path-First-OSPF/#configure-md5-authentication-for-ospf-neighbors).

To configure BGP authentication for an existing BGP neighbor, only a single password is required:

```
cumulus@switch:~$ net add bgp neighbor 1.1.1.1 password BGPPWD
```

For more information, see [here](https://docs.cumulusnetworks.com/version/cumulus-linux-37/Layer-3/Border-Gateway-Protocol-BGP/#configure-md5-enabled-bgp-neighbors).

#### File Services

Trivial File Transfer Protocol (TFTP) is a simple and unauthenticated alternative to File Transfer Protcol (FTP) and is often used to update the configuration of network devices. By nature, TFTP contains no means to authenticate the user. If using TFTP is not mandatory within your organization, disable or uninstall it.

To remove the TFTP service:

```
cumulus@switch:~$ sudo apt-get remove atftp atftpd
cumulus@switch:~$ sudo apt-get remove tftpd-hpa
```

To remove the FTP service:

```
cumulus@switch:~$ sudo apt-get remove vsftpd
```

To verify that neither TFTP nor FTP is installed:

```
cumulus@switch:~$ sudo dpkg -l | grep *ftp*
```

## Medium Impact to Security, Medium Impact to Usability

This section discusses items that have similar impacts to both security, and management and user experiences.

### Hardware Security

#### 802.1X

802.1X is a popular technology because it authenticates devices that physically attach to the switch. It can also assign these devices different levels of access to the network after they authenticate. There are many use cases for this technology and each configuration varies widely. For additional details, go [here](https://docs.cumulusnetworks.com/cumulus-linux/Layer-1-and-Switch-Ports/802.1X-Interfaces/) and [here](https://cumulusnetworks.com/blog/campus-design-feature-set-up-part-4/).

The following example is a starting point to build on. This is a base 802.1X configuration:

1. Run the following commands:

    ```
    cumulus@switch:~$ net add dot1x radius server-ip 192.168.200.254 vrf mgmt
    cumulus@switch:~$ net add dot1x radius client-source-ip 192.168.200.29
    cumulus@switch:~$ net add dot1x radius shared-secret supersecret
    cumulus@switch:~$ net add dot1x send-eap-request-id
    cumulus@switch:~$ net add dot1x dynamic-vlan
    ```

2. Open the `hostapd.conf` file in a text editor and change the last two values in the file from `1` to `0`:

    ```
    cumulus@switch:~$ sudo nano hostapd.conf
    ...
    radius_das_require_event_timestamp=0
    radius_das_require_message_authenticator=0
    ```

3. Restart `hostapd.service` with the `sudo systemctl restart hostapd.service` command.

4. Enable 802.1X on switch ports:

    ```
    cumulus@switch:~$ net add interface swp1-4 dot1x
    ```

5. Configure the 802.1X reauthentication period:

    ```
    cumulus@switch:~$ net add dot1x eap-reauth-period 3600
    ```

6. Configure the maximum number of stations:

    ```
    cumulus@switch:~$ net add dot1x max-number-stations 1
    ```

#### USB

The Cumulus Linux switch comes with several USB ports as part of the external hardware. USB drives are standard among many industries and therefore easily accessible to those who want to do harm to the switch. While a best practice for any switch, disabling the USB ports is especially important if Cumulus Linux is set up in a publicly available area.

### Management Security

#### Password Policy and User Management

User passwords are the easiest way to break into any system. After a hacker steals a password, they have access to whatever the user has and can obtain information without raising too much suspicion. Therefore, many companies enforce specific user password requirements.

The default password requirements for Cumulus Linux are strong cryptographic hash (SHA-512). No accounts with nullok are in the `/etc/pam.d` file.

Password configurations should be consistent with NIST [password complexity guidelines](https://en.wikipedia.org/wiki/Password_policy#NIST_guidelines), but companies can set their own individual requirements for users.

#### Remove unnecessary Services

Unnecessary services that remain installed can cause open sockets that become target attack vectors. These services can be accidentally misused and can cause malfunctions. It is important to uninstall any programs or services that are not in use.

#### Emergency User Account

If your organization relies on a central authentication system such as TACACS or RADIUS, consider enabling an emergency administration account to access Cumulus Linux during times when the authentication systems are unavailable. Create the emergency admin account with its age set to never expire.

Run the following command to set the password policy for the emergency administrator account to never expire. Replace `[Emergency_Administrator]` with the correct emergency administrator account. You can create an emergency administrator user account with your standard user creation workflow.

```
cumulus@switch:~$ sudo chage -I -1 -M 99999 [Emergency_Administrator]
```

#### Login Banner

To prominently disclose the security and restrictions in place on your switch, enable login banners for all users so that upon login, each user sees your text message. Proper disclosure of your security policies upon login can help rule out legal defenses for inappropriate use of the equipment. Consult with the legal representative in your organization to obtain the proper wording of the login banner message.

To enable a login banner for all SSH login sessions:

1. Edit the `/etc/issue.net` file and add the login banner text approved by your organizational security policy:

    ```
    cumulus@switch:~$ sudo nano /etc/issue.net
    You are accessing an Information System (IS) that is provided for authorized use only.
    ```

2. Edit the  `/etc/ssh/sshd_config` file to enable the logon banner. Uncomment the following line:

    ```
    cumulus@switch:~$ sudo nano /etc/ssh/sshd_config
    ...
    Banner /etc/issue.net
    ```

3. Restart the ssh service:

    ```
    cumulus@switch:~$ sudo systemctl restart ssh@mgmt.service
    ```

#### Audit

Configure your system to log administrative events, then periodically audit those logs to ensure your system security policies are working as desired, as well as to detect any unauthorized attempts to gain access to your system. Auditing the system can also be helpful when troubleshooting issues. By looking at specific log events, you can identify consistent problems.

Cumulus Linux has many audit logs enabled by default. Make sure that the overall level of audit logging required conforms to your organizational security policy, as well as its need to track performance information about the system.

To view the default configurations:

```
cumulus@switch:~$ sudo grep log_file /etc/audit/auditd.conf
log_file = /var/log/audit/audit.log
max_log_file = 6
max_log_file_action = ROTATE
```

To view the size of the audit logs:
```
df /var/log/audit/ -h
Filesystem  	Size  Used Avail Use% Mounted on
/dev/sda4   	5.8G  931M  4.6G  17% /var/log
```

### Secure Network Protocols

#### Source Route

Source routing is a common security threat that allows attackers to send packets to your network and then use the returned information to break into your network. If your organization is not purposefully using source routing, disable it.

To disable IPv4 source-routed packets, set the current behavior with the following command:

```
cumulus@switch:~$ sudo sysctl -w net.ipv4.conf.default.accept_source_route=0
```

Check the default (boot up) setting:

```
cumulus@switch:~$ sudo sysctl net.ipv4.conf.default.accept_source_route
```

If the default value is not 0, add or update the following line in the `/etc/sysctl.conf` file so the setting persists after a reboot:

```
net.ipv4.conf.default.accept_source_route=0
```

Alternatively, you can create a new file in the `/etc/sysctl.d` directory and add the following text to the file:

```
net.ipv4.conf.default.accept_source_route=0
```

#### ICMP Redirects

Internet Control Message Protocol (ICMP) is a great troubleshooting tool, but can be a security threat if your router automatically accepts an ICMP redirect message. Attackers can use this to their advantage by sending unrecognized redirects to either capture your traffic or create a DOS attack.

To prevent IPv4 ICMP redirect messages from being accepted, set the current behavior with the following command:

```
cumulus@switch:~$ sudo sysctl -w net.ipv4.conf.default.accept_redirects=0
```

Check the default (boot up) setting:

```
cumulus@switch:~$ sudo sysctl net.ipv4.conf.default.accept_redirects
```

If the default value is not 0, add or update the following line in the  `/etc/sysctl.conf` file so the setting persists after a reboot:

```
net.ipv4.conf.default.accept_redirects =0
```

Alternatively, you can create a new file in the `/etc/sysctl.d` directory and add the following line:

```
net.ipv4.conf.default.accept_redirects =0
```

To prevent IPv4 ICMP redirect messages from being sent, set the current behavior with the following commands:

```
cumulus@switch:~$ sudo sysctl -w net.ipv4.conf.default.send_redirects=0
cumulus@switch:~$ sudo sysctl -w net.ipv4.conf.all.send_redirects=0
```

Check the default (boot up) setting:

```
cumulus@switch:~$ sudo sysctl net.ipv4.conf.default.send_redirects
net.ipv4.conf.default.send_redirects = 1
```

```
cumulus@switch:~$ sudo sysctl net.ipv4.conf.all.send_redirects
net.ipv4.conf.all.send_redirects = 1
```

If the default value is not 0, add or update the following lines in the `/etc/sysctl.conf` file so the setting persists after a reboot:

```
net.ipv4.conf.default.send_redirects=0
net.ipv4.conf.all.send_redirects=0
```

Alternatively, you can create a new file in the `/etc/sysctl.d` directory and add the following lines:

```
net.ipv4.conf.default.send_redirects=0
net.ipv4.conf.all.send_redirects=0
```

## Low Impact to Security, High Impact to Usability

This section discusses items that have a relatively low impact on security but have the potential to greatly disturb the user experience.

### Password Protect bootloader

A bootloader is the program that launches the operating system when the switch is powered on or rebooted. Adding a password to the bootloader does not significantly improve the security of a system but it can cause accidental outages.

For example, if you configure the switch to have a bootloader password, then later you work on the switch remotely and make a change that requires a system reboot, when the system begins to boot, it halts and waits for the bootloader password, which you cannot enter unless you are physically at the switch. Instead of a quick reboot, the switch sits offline until someone can enter the bootloader password on the switch and allow it to launch the operating system and bring the switch back online.

### Debian Packages

One of the most tempting services to configure is the Debian package manager that controls the software and updates installed on your switch. For example, you might think it is be a good idea to configure the package manager to remove all outdated software packages after a new update is completed. While it makes more disk space available, it also prevents you from quickly rolling back to a previous version if a software glitch causes the system to malfunction or stop communicating.

## Not All Security Measures are Created Equal

Cumulus Linux comes out of the box ready for use and with built-in security. However, if you need to customize the default security configuration or add additional security controls, the measures described in this document will help achieve the required result. Not all security measures are created equal, and many can ruin a user experience without adding a significant amount of security. First, focus on security measures with the greatest benefits and the fewest drawbacks. Later, expand to include other security mitigations as your comfort level increases.