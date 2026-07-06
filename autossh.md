# SSH Reverse Tunnel Setup Guide

## The Architecture
- Internal Server: The private target machine (e.g., behind a home router or corporate firewall).
- Public VPS: A publicly accessible server with a static IP (e.g., AWS, DigitalOcean).
- Your Remote Laptop: The machine you are using outside the network to jump back in.

[Remote Laptop] ---> [Public VPS:Port 22001] ===(Reverse Tunnel)=== [Internal Server:Port 2002]


## Network Configuration Overview

**Internal Host (Private Network)**

IP: (not specified)

SSH Port: 2002

Username: root

Password: 

Internal Configuration Port: 2002


**Public Host (Jump Server)**

IP: 180.97.80.104

SSH Port: 2134

Username: tunnel

Password: 

Public Configuration Port: 22001


## Prepare the Public VPS

**1. Modify SSH Configuration**

Edit /etc/ssh/sshd_config to allow SSH to bind to all interfaces (0.0.0.0), so the forwarded port can be accessed from any IP, not just 127.0.0.1.

`GatewayPorts yes`


**2. Restart the SSH Service**

`sudo systemctl restart sshd.service`



## Establish the Reverse Tunnel

**1. Generate SSH Key and Configure Authentication**

Generate an SSH key pair on the internal host, then copy it to the jump server for authentication.

```
ssh-keygen -t rsa -C "tenag_hirmb@hotmail.com"
ssh-copy-id -p 2134 tunnel@180.97.80.104
```

Enter the password of the jump server user tunnel when prompted.


**2. Install AutoSSH**

`sudo apt install autossh`


**3. Create a Systemd Service File**

Create a new systemd unit file for AutoSSH:

`sudo vi /etc/systemd/system/autosshd.service`

Example configuration:

```bash
[Unit]
Description=AutoSSH tunnel service
After=network.target

[Service]
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 55555 -o TCPKeepAlive=yes -NR 0.0.0.0:22001:localhost:2002 tunnel@180.97.80.104 -p 2134

[Install]
WantedBy=multi-user.target
```

> Notes:
> 1. The -M option specifies a monitoring port (use any available port).
> 2. -N means “no remote command”—it only establishes the tunnel without opening a shell. (ideal for port forwarding only)
> 3. -f runs AutoSSH in the background, but -f is not supported by systemd.
> 4. The option -R 0.0.0.0:22001:localhost:2002 maps the internal host port 2002 to the jump server’s port 22001. Routes traffic arriving at port 22001 on the VPS back to port 2002 on your internal server.
> 5. Once connected, any access to port 22001 on the jump server will automatically forward to port 2002 on the internal host.
> 6. Even though you type `-R` directly into the `autossh` command, `autossh` doesn't actually process that flag itself. Instead, `autossh` takes every single flag it doesn't recognize and silently hands them over to the standard `ssh` client hidden underneath.
> ```# This is what autossh actually runs in the background for you:
ssh -o TCPKeepAlive=yes -NR 0.0.0.0:22001:localhost:2002 tunnel@180.97.80.104 -p 2134
```


**4. Reload Systemd Daemon**

`sudo systemctl daemon-reload`


**5. Start the AutoSSH Service**

`sudo systemctl start autosshd.service`


**6. Enable AutoSSH on Boot**

`sudo systemctl enable autosshd.service`


## Jump into the Internal Server

From **your Remote Laptop** anywhere in the world, you can now jump through the VPS directly into your internal machine.

Use the following command to test access through the reverse tunnel:

`ssh -p 22001 root@180.97.80.104`

When prompted, enter the password for the internal host user root.

If the connection is successful, the reverse SSH tunnel is working correctly — you can now access the internal host from the public server’s port 22001.

## The Principle

The principle behind a reverse SSH tunnel is reversing the direction of the connection establishment.
Instead of an outside client trying to punch through a strict firewall to reach an internal server, the internal server initiates an outgoing connection to a public server, leaving an open channel for traffic to flow back inward.
Here is a breakdown of how this mechanism functions:
### 1. Firewalls Allow Outbound, Block Inbound

* Standard firewalls and NAT (Network Address Translation) routers automatically block unsolicited incoming connections from the internet.
* However, they almost always allow internal devices to make outbound connections (like browsing a website or connecting to a VPS).

### 2. The Internal Server Initiates the Connection

* The internal server reaches out to the public VPS.
* Because this connection is outbound, the strict local firewall approves it and keeps the state tracker open.

### 3. Creating the Return Pathway (Port Forwarding)

* During that outbound connection, the internal server tells the VPS: "Please listen on Port 2222. If anyone sends traffic to your Port 2222, pass it back through this existing connection to my Port 22."
* This essentially creates a two-way pipeline (or tunnel) inside the authorized outbound connection.

### 4. Bypassing the Firewall

* When your remote laptop connects to the VPS on Port 2222, the VPS drops that data directly into the open tunnel.
* The data travels backward through the local firewall seamlessly, because the local firewall views it as part of the safe, established session originally started by the internal server.

### 5. The Role of Autossh

* SSH connections naturally time out or drop during network glitches. If the connection dies, the pathway disappears.
* autossh acts as a watchdog. It constantly sends test packets through the tunnel. If it detects that the VPS is no longer responding, it immediately issues a new outbound connection to recreate the pathway.
