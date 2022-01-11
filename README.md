# FlexNet as systemd service

FlexNet is a common license server from Flexera Software used with large number of software e.g. Abaqus. It is a server-client architecture license management software which provides floating licenses to multiple users of software. The license is checked-out when a user starts to use the software and checked-in when a user finishes working with it.

The FlexNet server is designed to give remote access to licenses usually in a local network. The FlexNet server consists of the license manager daemon `lmgrd` and vendor(s) daemon(s). Both license manager and vendor daemons work with open TCP ports to communicate with clients - programs which check license on the server.

Usually the FlexNet installation procedure is combined with the installation of the software which uses FlexNet as license server. It's generally quite straightforward and a well documented procedure. Unfortunately the default, post-installation configuration of the FlexNet server is very basic, inconvenient to administrate and last but not least it is not secure.

Please find below the instructions for FLexNet configuration on the systemd-based Linux machine which:
- is run as a dedicated user and group,
- saves log files in specified directory (e.g. /var/log/),
- is started automatically at the system restart,
- can be started, stopped and reloaded using systemctl command.

The short instruction on how to configure Flexnet and firewall is provided as well.

## Create user & group

The FlexNet server should be run as a dedicated (non-root) user with restricted privileges (e.g. no login, limited access to files and directories). Moreover, on Linux usage of `lmdown`, `lmreread`, and `lmremove` is restricted to a license administrator who is by default root. Because we don't want to manage the FlexNet server using the root account the dedicated group called lmadmin is created. Thanks to this using `-2 -p` and `-local` option the usage of the server management commands is restricted to members of that group only.

To create lmadmin group use command:
```
$ sudo groupadd lmadmin
```
To create flexnet user use command:
```
$ sudo useradd flexnet -d /opt/abaqus/License -c "FlexnetLM User" -g lmadmin -s /sbin/nologin
```
## Create configuration and log files

Placed log file in dedicated directory using command:
```
$ sudo > /var/log/flexnet.log
```
Create configuration file to set up license and log files locations:
```
$ cat /etc/flexlm.conf
FLEXLICDIR=/opt/abaqus/License
FLEXLOGDIR=/var/log
FLEXLICFILE=abaquslm.lic
FLEXLOGFILE=flexnet.log
```
Set permissions:
```
$ sudo chown -R flexnet:lmadmin /usr/SIMULIA/License/2021/linux_a64/code/bin
$ sudo chown -R flexnet:lmadmin /opt/abaqus/License
$ sudo chown -R flexnet:lmadmin /var/log/flexnet.log
$ sudo chown flexnet:lmadmin /etc/flexlm.conf
```
Check permissions:
```
$ ls -al /etc/flexlm.conf
-rw-rw-r-- 1 flexnet lmadmin 206 01-05 01:27 /etc/flexlm.conf
…
```
##Creating and installing a systemd service unit

Create a systemd service unit in any (e.g. home) directory:
```
$ cat flexlm.service
[Unit]
Description=FlexNet Licence Server
Requires=network.target
After=local_fs.target network.target

[Service]
Type=simple
User=flexnet
Group=lmadmin
Restart=always
WorkingDirectory=/usr/SIMULIA/License/2021/linux_a64/code/bin
ExecStart=/usr/SIMULIA/License/2021/linux_a64/code/bin/lmgrd -z -2 -p -c /opt/abaqus/License/abaquslm.lic -l +/var/log/flexnet.log -local
ExecReload=/usr/SIMULIA/License/2021/linux_a64/code/bin/lmreread -c /opt/abaqus/License/abaquslm.lic -all
ExecStop=/usr/SIMULIA/License/2021/linux_a64/code/bin/lmdown -c /opt/abaqus/License/abaquslm.lic -q
SuccessExitStatus=15
Restart=always
RestartSec=120

[Install]
WantedBy=multi-user.target
```
To install the service - copy the service file into the /etc/systemd/system directory:
```
$ sudo cp flexlm.service /etc/systemd/system
$ sudo chmod 664 /etc/systemd/system/flexlm.service
```
Notify systemd that a new flexlm.service file exists: 
```
$ sudo systemctl daemon-reload
```
## Mange a FlexNet service

Enable the service to start it automatically at the next system restart:
```
$ sudo systemctl enable flexlm
```
To run the service use command:
```
$ sudo systemctl start flexlm
```
To check if the service is running you can use the following command:
```
$ sudo systemctl is-active flexlm
```
To check the service status use command:
```
$ sudo systemctl status flexlm
```
To reload license file (e.g. to extend or updated license) first modify the license file and then run command:
```
$ sudo systemctl reload flexlm
```
To stop the service just use command:
```
$ sudo systemctl stop flexlm
```
## FlexNet & firewall configuration

The correctly configured and run firewall  is a critical part of server security for most cases. Firewall configuration on the Linux server  is not discussed here. However, if the firewall is activated its configuration has to be modified to enable access to the license server for other computers in the local network.

To see the status of the firewall service use the following command:
```
$ sudo firewall-cmd --state
running
```
If command output is running that means the firewall is run. Please follow the instructions below to configure the firewall to make the license server accessible in the local network. Please note, that open access to the license server from the public network is not recommended due to legal and security reasons.

First set up two ports for lmgrd and vendor demons in the license file:
```
$ cat /opt/abaqus/License/abaquslm.lic
SERVER this_host 001a4b53fdfc 27000
VENDOR ABAQUSLM PORT=53153
…
```
The port 27000 is default for `lmgrd` (server) daemon. By default vendor daemon is assigned automatically and is changed on the occasion of each lmgrd restart. For firewall configuration vendor daemon has to be set up permanently and can be any valid number of unused unprivileged ports in the range 1025 to 65535. To check if the selected port is free use command:
```
$ netstat -ltn | grep 53153
$
```
No output means the port is free and can be used for vendor daemon.

To open ports use the following commands:
```
$ sudo firewall-cmd --zone=specific_zones --add-port=27000/TCP
$ sudo firewall-cmd --zone=specific_zones --add-port=53153/TCP
```
Check if license servers is available for expected remont machines and make the new settings persistent:
```
$ sudo firewall-cmd --runtime-to-permanent
```
