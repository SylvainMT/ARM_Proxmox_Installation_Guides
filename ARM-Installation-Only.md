# Installation steps of Automatic Ripping Machine (Docker Install) inside a Proxmox LXC container

This guide will install the Automatic Ripping Machine (ARM) inside a Proxmox Container LXC using the docker install script.

## Storage Considerations

In this setup I use a shared volume for ARM to deposit completed files into which is shared with other containers installed on for use, in my case Jellyfin.  LXD containers are persistent which means that settings can be saved in the container's volume.  In my setup Proxmox is running on an SSD which is the fastest storage available so my transcode and raw folders are remaining inside the container.  That does mean however that I needed to give enough storage space to my container to work.  (100 GiB).

## Tested Environment

- Proxmox VE 8.0.4
- Computer
  - Asus P5QL-E Motherboard
  - Core(TM) 2 Quad CPU Q6600 @ 2.40GHz
  - 16 GiB of DDR2 Ram
  - Pioneer DVD-RW DVR-107D Optical Drive
  - NVidia GeForce GTX 1050Ti (Asus Strix) (Note that this guide doesn't include GPU Passthrough... yet...)

## Procedure

First one needs to prepare the Proxmox Host and get Information

1. Log into Proxmox as root (makes the rest easier) and open your node's Shell ("Datacenter" -> [_YourNodeName_] -> ">_ Shell"  This should give you a Shell screen logged in as root.
2. `apt update && apt upgrade -y`
3. Install lsscsi `apt install lsscsi` 
4. `lsscsi -g`
The command returns a screen like below, write down your optical drive's SR id (in the blue box of the image) and SG id (in the green box of the image)
![01-lsscsi screenshot](https://github.com/automatic-ripping-machine/automatic-ripping-machine/assets/21991849/159241e7-d985-4d85-a86f-175721b3c00d)
5. `ls -l /dev/sr* && ls -l /dev/sg*`
This command will list your drive's major (in the blue box) and minor (in the green box) number for both the SR id, the SG id as well as the device's category (the letter "b" and "c" in the yellow box) Record all three values.
![02-Major-minor](https://github.com/automatic-ripping-machine/automatic-ripping-machine/assets/21991849/86f1841d-00b1-42c3-bf8d-5fbad45230b0)
6. Download the Debian-12 Container image
    1. Choose "local" storage
    2. "CT Templates"
    3. "Templates"
    4. Search for "Debian"
    5. Select "debian-12-standard"
    6. Download
7. Create the container
    1. "Create CT"
    2. Enter a CTID, Hostname and Password of your choice.
    3. Deselect "Unprivileged container"  *IMPORTANT*
    4. "Next"
    5. Choose the previously downloaded Debian-12 template and click "Next"
    6. Give plenty of room for your root storage I choose 100 GiB.  It will become your scratch drive (We will mount a shared volume later)
    7. Decide how many cores to let your container access.  I suggest as many as you can, I gave it all four.  Minimum of 2 is recommended.
    8. Give your container access to some RAM, I went with 2GiB (2048MiB) and 512MiB Swap Drive
    9. Enter your network details.  I suggest a static IP as this is the IP you will use to access ARM
    10. Enter your DNS settings (or just use the Host Settings)
    11. Confirm everything but do not start the container yet
    12. Once your container is done building, click on the container in the UI > Options > Features > Edit.  Add a checkbox next to Nesting.  Click OK.
6. Passing through the Optical drive(s) to your container.
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
~
    3. Add the following lines.
```
lxc.apparmor.profile: unconfined
lxc.cap.drop:
lxc.autodev: 1
lxc.mount.auto: sys:rw
```
~
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
7. Start your container and log-in to your container's Console ("Container Name" > ">_ Console") as root
    1. `apt update && apt upgrade -y'
    2. `apt install lsscsi` 
    3. Confirm that your Optical Drive is passed through with `lsscsi -g` confirm that the output for your Optical Drives are the same as they are on the Proxmox Host. If not check your LXC Config file and restart the container.
    4. Confirm that you have access to the drive with `ls -l /dev/sr* && ls -l /dev/sg*`  The lines about your Optical Drives should be identical as they were on the Proxmox Host.  If not check your LXC Config file and restart the container.
    5. Create the Arm User and Group.  *IMPORANT* To be able to use Shared Storage Volumes, the group ID and user ID must be the same for all the users and groups (in all containers) that will used the shared volume.  (So your jellyfin user and group in the jellyfin container should have the same **UserID** and **GroupID**)
        1. `groupadd -g <GroupID> arm`
        2. `useradd -u <UserID> -g arm arm`
        3. `passwd arm`
        4. Set a password (you will need it later)
    6. Install Docker
        1. `apt install curl -y`
        2. `curl -sSL https://get.docker.com/ | sh`
        3. Optionally Install Portainer a UI took for managing Docker containers. _This step is useful if you need to do some troubleshooting_ 
            1. `docker run --restart always -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce`
            2. Navigate to: `http://[LXC_Container_ID]:9000` to complete portainer setup.
8. Install ARM as per the wiki (But do not Start AMR Yet!, Wait to complete the "Post Install" step of the wiki guide until later.) [docker](https://github.com/automatic-ripping-machine/automatic-ripping-machine/wiki/docker)
9. If the volume you intend to save the completed rips in doesn't already exist follow below, else skip to step 10.
    1. From the Proxmox UI click on your container
    5. Click on "Resources"
    6. "Add" > "Mount Point"
    7. Choose a size, set a mount point id if desired.  
    8. Set the path (this is the path inside the container) to `/mnt/media`
    9. If you wish to include this volume in backup ensure the "backup" checkbox is selected.
    10. Click "Create"
11. In this step we make the desired volume shareable and add it to each LXC container that needs access to it.
    1. Enter your Proxmox Node Shell (and log-in as root if not already logged in)
    3. `nano /etc/pve/lxc/<ContainerID>.conf` here container ID is for the container that has the volume you wish to share, if you completed step 9. this is your ARM container.
    4. Find the mount point entry that equates to the container you wish to share for example `mp0: local-lvm:vm-300-disk-1,mp=/mnt/media,backup=1,size=16T`  Hint: the "mp0" should match the Mount Point ID seen in the UI.
    5. Make it shareable by adding `shared=1` to it.  Copy the entire line.
    6. Edit the config file for each container you wish to share this volume with and paste the line in it.  *IMPORTANT* The Mount Point ID 'mp#' needs to be unique within each container!!!  Modify the Mount Point ID as needed!  *IMPORTANT* To make linux permissions easier.  Be sure to give each user and group, in each container the **SAME** `<UserID>` and `<GroupID>`  how to do this is outside the scope of this guide.
12. Complete the Wiki "Post Install" steps (But do not start ARM yet!)
13. Open your Container's Console and log in as the "arm" user and add the Optical Drive and your mount point to Docker.
    1. `nano start_arm_container.sh`
    2. Add the optical drive (repeat for each optical drive you have)
```
    --device="/dev/sr<SR ID>:/dev/sr<SR ID>" \
    --device="/dev/sg<SG ID>:/dev/sg<SG ID>" \
```
 ~
    3. Add your mount points these take the form of `-v "<LXC Container's Path to location>:<Docker Path to location>" \`  Keep in mind that here the location are from the LXC Container's and Docker's perspective respectively.  (you may have to delete some lines already there or modify them...)
 
```
    -v "/home/arm:/home/arm" \
    -v "/mnt/media/Music:/home/arm/Music" \
    -v "/home/arm/logs:/home/arm/logs" \
    -v "/home/arm/media:/home/arm/media" \
    -v "/mnt/media/arm_completed:/home/arm/media/completed" \
    -v "/etc/arm/config:/etc/arm/config" \
```
14. Start ARM
    1. Log out from the arm account with `exit`
    2. Log in as root
    3. `/home/arm/start_arm_container.sh`


Assuming all was done correctly (and my guide is correct) you should now be able to use ARM on your Proxmox machine.  Just give ARM a few seconds to start, go to the address `http://<your arm ip address>:8080` log in with the default user (admin) and password (password) and once you see the arm interface pop-in a disk in your optical drive and see if it runs!

###### Edits
-  Edited for legibility
- Noticed I forgot the steps to install docker.  Added those steps to the guide.