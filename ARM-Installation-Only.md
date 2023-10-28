# Installation steps of Automatic Ripping Machine (Docker Install) inside a Proxmox LXC container

This guide will install the Automatic Ripping Machine (ARM) inside a Proxmox Container LXC using the docker install script.

## Storage Considerations

This guide concerns itself strictly with the installation of ARM. No considerations are given to where the completed files will be stored once ARM completed the rips.  Other guides include storage considerations.

## Graphics Card Pass-through

This guide will not include any instructions for the pass-through of a graphics card.  Other guides will include a graphics card pass-through.

## Tested Environment

- Proxmox VE 8.0.4
- Container OS: Debian 12
- ARM V2.6.60
- Computer
  - Asus P5QL-E Motherboard
  - Core(TM) 2 Quad CPU Q6600 @ 2.40GHz
  - 16 GiB of DDR2 Ram
  - Pioneer DVD-RW DVR-107D Optical Drive

## Procedure

### Identifying your Optical Drives in Proxmox

The first step is getting information about your system.  Properly identifying your optical drives now will make the process easier later.

1. Log into Proxmox as root (makes the rest easier) and open your node's Shell ("Datacenter" -> [_YourNodeName_] -> ">_ Shell"  This should give you a Shell screen logged in as root.
2. `apt update && apt upgrade -y`
3. Install lsscsi `apt install lsscsi` 
4. `lsscsi -g`
The command returns a screen like below, write down your optical drive's SR id (in the blue box of the image) and SG id (in the green box of the image)
![01-lsscsi screenshot](https://github.com/automatic-ripping-machine/automatic-ripping-machine/assets/21991849/159241e7-d985-4d85-a86f-175721b3c00d)
5. `ls -l /dev/sr* && ls -l /dev/sg*`
This command will list your drive's major (in the blue box) and minor (in the green box) number for both the SR id, the SG id as well as the device's category (the letter "b" and "c" in the yellow box) Record all three values.

![02-Major-minor](https://github.com/automatic-ripping-machine/automatic-ripping-machine/assets/21991849/86f1841d-00b1-42c3-bf8d-5fbad45230b0)

### Creating the LXC container

This guide uses the latest version of Debian, which at this time is Debian 12 Bookworm. While not tested this may work with any Debian derivative.
1. Download the Debian-12 Container image
    1. Choose "local" storage
    2. "CT Templates"
    3. "Templates"
    4. Search for "Debian"
    5. Select "debian-12-standard"
    6. Download
2. Create the container
    1. "Create CT"
    2. Enter a [CTID], Hostname and Password of your choice.
    3. ***IMPORTANT*** Deselect "Unprivileged container"  
    4. "Next"
    5. Choose the previously downloaded Debian-12 template and click "Next"
    6. Give plenty of room for your storage I choose 100 GiB. (But that is not nearly enough for a movie library of any size...)
    7. Decide how many cores to let your container access.  I suggest as many as you can, I gave it all four.  Minimum of 2 is recommended.
    8. Give your container access to some RAM, I went with 2GiB (2048MiB) and 512MiB Swap Drive
    9. Enter your network details.  I suggest a static IP as this is the IP you will use to access ARM
    10. Enter your DNS settings (or just use the Host Settings)
    11. Confirm everything and start the container.
    12. Log-in to your container's Console ("Container Name" > ">_ Console") as root
    13. `apt update && apt upgrade -y`
    14. `apt install sudo -y`
    15. Create a non-root user to complete the installation.
        1. `adduser <UserName>`
        2. Complete the questionnaire and give it a password.
        3. `usermod -aG sudo <UserName>`
    16. Stop the container

### Passing the Optical Drives the container

1. Go back to your node's Shell interface
2. Edit the container's .conf file `nano /etc/pve/lxc/<Your Container ID>.conf` (It should look similar to below)
```
arch: amd64
cores: 4
hostname: arm
memory: 2048
net0: name=eth0,bridge=vmbr0,firewall=1,gw=<Your Gateway IP>,hwaddr=<Your MAC Address>,ip=<Your IP>,type=veth
ostype: debian
rootfs: local-lvm:vm-300-disk-0,size=100G
swap: 512
```
3. Add the following lines.
```
lxc.apparmor.profile: unconfined
lxc.cap.drop:
lxc.autodev: 1
lxc.mount.auto: sys:rw
```
4. Add the following lines for each of the Optical Drives you are passing to the Container (Replace the values inside `<Value>` to what is appropriate for your situation)
```
lxc.cgroup2.devices.allow: <SR Device Category> <SR Major Number>:<SR Minor Number> rwm
lxc.mount.entry: /dev/sr<SR ID Number> dev/sr<SR ID Number> none bind,create=file,optional 0 0
lxc.cgroup2.devices.allow: <SG Device Category> <SG Major Number>:<SG Minor Number> rwm
lxc.mount.entry: /dev/sg<SG ID Number> dev/sg<SG ID Number> none bind,create=file,optional 0 0
```
This is my configuration
```
lxc.cgroup2.devices.allow: b 11:0 rwm
lxc.mount.entry: /dev/sr0 dev/sr0 none bind,create=file,optional 0 0
lxc.cgroup2.devices.allow: c 21:0 rwm
lxc.mount.entry: /dev/sg0 dev/sg0 none bind,create=file,optional 0 0
```
5. Start the container again and log-in to the container's console with the user that was previously created.
6. *OPTIONAL* - Confirm that you have successfully passed-through your optinal drives.  This is a good troubleshooting step.
    1. `sudo apt install lsscsi -y` 
   2. Confirm that your Optical Drive is passed through with `lsscsi -g` confirm that the output for your Optical Drives are the same as they are on the Proxmox Host. If not check your LXC Config file and restart the container.
   3. Confirm that you have access to the drive with `ls -l /dev/sr* && ls -l /dev/sg*`  The lines about your Optical Drives should be identical as they were on the Proxmox Host.  If not check your LXC Config file and restart the container.

### Install ARM using the Docker Script

1. `sudo apt install curl -y`
2. Create the arm user and group
   1. `adduser <UserName>`
3. `curl https://raw.githubusercontent.com/automatic-ripping-machine/automatic-ripping-machine/v2_devel/scripts/installers/docker-setup.sh | sudo bash`
4. The script will (assuming the container is properly setup)
    1. Install Docker
    2. Pull the appropriate image from Dockerhub
    3. Save a copy of the template to start the arm docker container in `/home/arm/start_arm_container.sh`

### Docker Post-Installation Steps

1. Create a configurations directory
   1. `sudo mkdir -p /etc/arm/config`
   2. `sudo chown -R arm:arm /etc/arm`
2. Get the User ID and the Group ID for the arm user and record the values.
   1. `id -u arm`
   2. `id -g arm`
3. `sudo nano /home/arm/start_arm_container.sh`
#### Editing the start_arm_container.sh script
This script creates the docker container and gives it access to the Optical Drives, Graphics Card (if setup), the UserID and GroupID of Arm as well as some folders (Volumes) where ARM will place files or look for files.
   1. Replace the values `<id -u arm>` and `<id -g arm>` with the correct values.

##### Docker Volume Settings

The file contains a number of volume entries. ARM will run if you do not modify these entries.  Docker will simply create ephemeral volumes to satisfy it's needs.  This means that everytime you destroy and rebuild the Docker container you will loose all your settings and completed rips. To simply this process you can now add mount points to locations within your LXC container so that ARM will keep it's files even after the Docker container is rebuilt. Th mount points take this form `-v "<LXC Container's Path to location>:<Docker Path to location>" \`  Keep in mind that here the location are from the LXC Container's and Docker's perspective respectively.  The defaults are below:
```
    -v "<path_to_arm_user_home_folder>:/home/arm" \
    -v "<path_to_music_folder>:/home/arm/music" \
    -v "<path_to_logs_folder>:/home/arm/logs" \
    -v "<path_to_media_folder>:/home/arm/media" \
    -v "<path_to_config_folder>:/etc/arm/config" \
```

ARM uses the arm user home directory (`/home/arm/`) by default for most of it's needs.  If you wish to keep the defaults, then you can delete the lines found above and replace them with these.

```
    -v "/home/arm:/home/arm" \
    -v "/etc/arm/config:/etc/arm/config" \
```
With these you will find the ARM settings in: `/etc/arm/config`
The ripped DVDs and Blu Rays will be in `/home/arm/media/completed`
The Music CDs will be in `/home/arm/Music`


##### Docker Optical Drive Settings
For the Optical drives the default contents of the script is misleading.  Locate the following lines and erase them:
```
    --device="/dev/sr0:/dev/sr0" \
    --device="/dev/sr1:/dev/sr1" \
    --device="/dev/sr2:/dev/sr2" \
    --device="/dev/sr3:/dev/sr3" \
```
The correct entries provide two entries per drives. One for it's SR ID and one for it's SG ID.  Both are required for ARM to function.  They take this form.
```
    --device="/dev/sr<SR ID>:/dev/sr<SR ID>" \
    --device="/dev/sg<SG ID>:/dev/sg<SG ID>" \
```
Mine looks like this
```
    --device="/dev/sr0:/dev/sr0" \
    --device="/dev/sg0:/dev/sg0" \
```
Adjust yours as necessary.

##### Grant Docker Access to CPUs

Locate this line:
```
    --cpuset-cpus='2,3,4,5,6,7...' \
```
This line is very specific it grants docker access to the cores numbered.  Cores are numbered starting at 0.  So if you have a 4 core CPU and 4 thread, then your available cores are 0,1,2,3. If you have a 4 core and 8 thread CPU then your available cores are 0,1,2,3,4,5,6,7.  It is recommended that you don't give ARM access to each available core.  During transcoding, if all cores are available it can make you host unresponsive.  Since I have a 4 core and 4 thread CPU this is my setting.  Change yours as appropriate.
```
    --cpuset-cpus='1,2,3' \
```
This gives access to the last 3 cores of my CPU to ARM.  Keeping one for all the other things my Proxmox machine is doing.

### Start Arm

From your LXC console, log-in as your sudo user
1. `sudo /home/arm/start_arm_container.sh`


Assuming all was done correctly (and my guide is correct) you should now be able to use ARM on your Proxmox machine.  Just give ARM a few seconds to start, go to the address `http://<your arm ip address>:8080` log in with the default user (admin) and password (password) and once you see the arm interface pop-in a disk in your optical drive and see if it runs!