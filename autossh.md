
# SSH Reverse Tunnel Setup Guide

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


## Jump Server Configuration

**1. Modify SSH Configuration**

Edit /etc/ssh/sshd_config to allow SSH to bind to all interfaces (0.0.0.0), so the forwarded port can be accessed from any IP, not just 127.0.0.1.

`GatewayPorts yes`


**2. Restart the SSH Service**

`sudo systemctl restart sshd.service`



## Internal Host Configuration

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

```
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

The -M option specifies a monitoring port (use any available port).

-N means “no remote command”—it only establishes the tunnel without opening a shell.

-f runs AutoSSH in the background, but -f is not supported by systemd.

The option -R 0.0.0.0:22001:localhost:2002 maps the internal host port 2002 to the jump server’s port 22001.
Once connected, any access to port 22001 on the jump server will automatically forward to port 2002 on the internal host.





**4. Reload Systemd Daemon**

`sudo systemctl daemon-reload`


**5. Start the AutoSSH Service**

`sudo systemctl start autosshd.service`


**6. Enable AutoSSH on Boot**

`sudo systemctl enable autosshd.service`


## Testing the Connection

Use the following command to test access through the reverse tunnel:

`ssh -p 22001 root@180.97.80.104`

When prompted, enter the password for the internal host user root.

If the connection is successful, the reverse SSH tunnel is working correctly — you can now access the internal host from the public server’s port 22001.
