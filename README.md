# Steps we use to build a Malcolm Server
These are the steps I take to install Malcolm (https://malcolm.fyi/) from Idaho National Labs on Ubuntu server.

Based on installation example using Ubuntu 24.04 LTS 
(https://malcolm.fyi/docs/ubuntu-install-example.html#InstallationExample)

***Ingredients***
Ubuntu server LTS
Malcolm

***Instructions***

Create bootable USB with Ubuntu server using LTS version

## Ubuntu Server OS install

Do not have a network cable connected to the promiscuous port

-> enable SSH with password

By default Ubuntu will only create a 100G hard disk.  Malcolm will need as much hard disk space as is available for storing data and PCAPs.
#### Expand lvm to use entire disk
* In FILE SYSTEM SUMMARY
* Down arrow to USED DEVICES
* Find ubuntu-lv hit enter and select Edit
* Change size to be the max available
* Tab down to Save
* Tab down to Done


-> **Admin name**: Malcolm

-> **System name**: mlclm01 (*change number as needed*)

-> **User name**: malcolm

-> **User password**: (*create a sufficiently strong password*)

Select Install OpenSSH server

-> Use defaults for all install options
-> Finish install

## Add authentication cert to server

1. After reboot, SSH in using password
2. Create private/public key pair on desktop
3. Copy public key to `./.ssh/authorizedkeys` on server
4. Edit `/etc/ssh/sshd_config.d/50-cloud-init.conf` and set enable password to **no**

## Update distro
```
apt update
apt upgrade
reboot
```

## Install Malcolm 

### Install dependencies
```
sudo apt-get -y update
sudo apt-get -y install curl unzip python3 python3-dialog python3-dotenv python3-pip python3-ruamel.yaml git python3-dialog dialog
sudo reboot
```

Identify the network port to be promiscuous

Get the port ID and write it down

`ip a | grep en`

### Get Malcolm install; files
```
git clone https://github.com/idaholab/Malcolm
```

### Run install script
```
cd Malcolm
sudo ./scripts/install.py
```

### Install configuration options
```
Container Runtime Settings
-> Malcolm Restart Policy -> Always	

Clean Up Artifacts Settings
-> Arkime PCAP Management Settings -> Delete PCAP Threshold -> 7%
-> Delete Old Indices Settings -> Index Prune Threshold -> 75%

Zeek Analysis Settings
-> Enable Zeek File Extraction -> Yes

Zeek File Extraction Settings -> File Extraction Settings -> Extracted File Percent Threshold -> 5G

Zeek File Extraction Settings -> Preserved Files HTTP Server Settings -> Downloaded Preserved File Password -> infected

Zeek File Extraction Settings -> Preserved Files HTTP Server Settings -> Zip Downloads -> Yes

Zeek File Extraction Settings -> Update Scan Rules -> Yes

Zeek Analysis Settings -> Enable Zeek ICS/OT Analyzers -> Yes

Zeek ICS/OT Analyzers Settings -> Enable Zeek ICS "Best Guess" -> Yes

NetBox Settings
-> Auto-Create Subnet Prefixes -> Yes
-> Auto-Populate NetBox Inventory -> Yes

Capture Live Network Traffic -> Yes

Capture Live Network Traffic Settings -> Capture Interface(s) ->. <<<<enter the promiscuous port ID found earlier>>>
```  

`Save and Continue`    twice

Review the proposed configuration then select `Yes`.  It will take about a minute for the configuration to complete.  It will be complete when you see (SUCCESS) [INSTALLER] message

### Build docker components

`sudo docker compose --profile malcolm pull`

Reboot

### Configuration Authentication

`cd Malcolm
./scripts/auth_setup`

Accepts the defaults. Fill in the information as needed.  **Note**: After the message about adding additional account pops up, it will take about a minute before it moves on to the next configuration item.

## Create service to auto-start promisc network port

Get the port ID (should be same as set during configuration)

`ip a | grep en`

Create service file

`sudo nano /etc/systemd/system/promisc-port.service`

Add following text.  Change <<network interface>> to the port configured as promiscous
```
[Unit]
Description=Auto start promiscuous port
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/ip link set <<network interface>> up
ExecStop=/sbin/ip link set <<network interface>> down
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Save the file then get the service started using the following commands.

```
sudo systemctl daemon-reload
sudo systemctl enable promisc-port
sudo systemctl start promisc-port
```
## Start Malcolm
`./scripts/start`

First start will take about 5 minutes.  When it is complete, it will provide a message showing...
```
Started Malcolm


Malcolm services can be accessed at https://<<server ip address>>/
```

To verify that Malcolm is fully running, run

`./scripts/status`

Each item should be showing as healthy.  If not issue `./scripts/restart`

When it completes, run the status script again.

You should now be able to access the Malcolm dashboard from the IP address for the server using https.
