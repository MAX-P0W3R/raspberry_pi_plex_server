# Raspberry Pi Plex Server
![Raspberry Pi](https://img.shields.io/badge/-RaspberryPi-C51A4A?style=for-the-badge&logo=Raspberry-Pi) ![Plex](https://img.shields.io/badge/plex-%23E5A00D.svg?style=for-the-badge&logo=plex&logoColor=white)

## <a href="https://www.raspberrypi.org"><img src="https://www.raspberrypi.org/wp-content/uploads/2012/03/raspberry-pi-logo.png" alt="Raspberry Pi Logo" align="left"  height=150></a>  

> The Raspberry Pi is a series of credit card-sized single-board computers developed in the United Kingdom by the Raspberry Pi Foundation to promote the teaching of basic computer science in schools and developing countries. Official Link: [Raspberry Pi Homepage](https://raspberrypi.org)

#### Plex is an American streaming media service and a client–server media player platform, made by Plex, Inc. The Plex Media Server organizes video, audio, and photos from a user's collections and from online services, and streams it to the players. The official clients and unofficial third-party clients run on mobile devices, smart TVs, streaming boxes, and in web apps.

## Installation

Ensure our operating system is entirely up to date:

```shell
sudo apt-get update
sudo apt-get upgrade
```

Install the package **apt-transport-https** that allows the **apt** package manager to retrieve packages over **https** protocol that the Plex repository uses:

```shell
sudo apt-get install apt-transport-https
```

Add the Plex repositories to the **apt** package managers key list:

```shell
curl https://downloads.plex.tv/plex-keys/PlexSign.key | sudo apt-key add -
```

Add the official plex repository to the sources list:

```shell
echo deb https://downloads.plex.tv/repo/deb public main | sudo tee /etc/apt/sources.list.d/plexmediaserver.list
```

Refresh the package list:

```shell
sudo apt-get update
```

Install the Plex Media Server:

```shell
sudo apt-get install plexmediaserver
```

## Setting Up

### Fix permissions (Optional)

**NOTE:** In my case Plex worked perfectly configured with its default user.

By default, the Plex Media Server package will utilize a user named **plex**. To reduce the chances of dealing with permission issues, we could change the server’s default file:

```shell
sudo nano /etc/default/plexmediaserver
```

change the **PLEX_MEDIA_SERVER_USER** line from **plex** to **pi**:

```shell
export PLEX_MEDIA_SERVER_USER=pi
```

Restart the **plexmediaserver** service:

```shell
sudo systemctl restart plexmediaserver
```

### Setup an Static IP

We should make sure the Pi has a static IP, so it’s easy to remember the IP address. To get your current IP address, enter the following command:

```shell
hostname -I
```

Open up thecmdline.txt file:

```shell
sudo nano /boot/cmdline.txt
```

At the bottom of this file, add the following line (Replacing “YOUR IP” with the IP you got from using hostname -I):

```shell
ip=YOUR IP
```

Restart the Raspberry Pi:

```shell
sudo reboot
```

## Mount USB Storage

If you only have one external hard drive connected to the Pi, then it should be attached to **/dev/sda1** – additional drives will use **/dev/sdb1** and **/dev/sdc1** etc.

### Prepare the Mount Point

Make a directory in which to mount the USB drive:

```shell
sudo mkdir /mnt/mediastorage
```

Make pi the owner of the mounted drive and make its permissions read, write and execute for it:

```shell
sudo chown -R pi:pi /mnt/mediastorage
sudo chmod -R 775 /mnt/mediastorage
```

Set all future permissions for the mount point to pi user and group:

```shell
sudo setfacl -Rdm g:pi:rwx /mnt/mediastorage
sudo setfacl -Rm g:pi:rwx /mnt/mediastorage
```

### Determine the USB Hard Drive Format

```shell
sudo blkid
```

You will see something like this. Again it is the sda1 line we are interested in (In this case we are working with only one external hard drive). We need to take note in the attribute **TYPE** (NTFS for me) because we will need it to configure the **fstab** file.

```shell
/dev/mmcblk0p1: LABEL="boot" UUID="27D9-A951" TYPE="vfat" PARTUUID="d95dc960-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="db9fbdec-9f10-4008-95da-5062491e0659" TYPE="ext4" PARTUUID="d95dc960-02"
/dev/mmcblk0: PTUUID="d95dc960" PTTYPE="dos"
/dev/sda1: LABEL="Elements" UUID="328C83488C830597" TYPE="ntfs" PARTLABEL="Elements" PARTUUID="0f291c06-b24b-47f3-815d-fe34d120459d"
```

If the hard drive is NTFS we will need to install some utilities first:

```shell
sudo apt-get install ntfs-3g -y
```

If the drive is exfat install these utilities:

```shell
sudo apt-get install exfat-utils -y
```

For all drive types mount the hard drive with this command, -o insures pi is the owner which should avoid permission issues:

```shell
sudo mount -o uid=pi,gid=pi /dev/sda1 /mnt/mediastorage
```

If you get an error use this syntax:

```shell
sudo mount -t uid=pi,gid=pi /dev/sda1 /mnt/mediastorage
```

If the mount -t command returns an error then use this syntax:

```shell
sudo mount uid=pi,gid=pi /dev/sda1 /mnt/mediastorage
```

If you are getting **this drive is already mounted** errors then you are probably using a distro which automounts the drives. Unmount the hard drive with the next command before to mount it again to the right mount point (/mnt/mediastorage) with one of the previous commands:

```shell
sudo umount /dev/sda1
```

### Automount the USB Hard Drive on Boot

/mnt/mediastorage will be the folder in which you store your media. We want it to be automounted on boot The best way to do this is through the UUID. Get the UUID by using this commmand:

```shell
sudo ls -l /dev/disk/by-uuid/
```

You will see some output like this. The UUID you want is formatted like this XXXX-XXXX for the sda1 drive. If the drive is NTFS it can have a longer format like UUID="328C83488C830597".

```shell
total 0
lrwxrwxrwx 1 root root 15 Sep 10 10:17 27D9-A951 -> ../../mmcblk0p1
lrwxrwxrwx 1 root root 10 Sep 10 11:02 328C83488C830597 -> ../../sda1
lrwxrwxrwx 1 root root 15 Sep 10 10:17 db9fbdec-9f10-4008-95da-5062491e0659 -> ../../mmcblk0p2
```

Now we will edit fstab to mount the USB by UUID on boot:

```shell
sudo nano /etc/fstab
```

Add the next line to the bottom, replace XXXX-XXXX with your UUID and TYPE with the correct type (e.g. ntfs, exfat, vfat, ext4). You may or may not need the quotation marks wrapped around the UID, you do not need quotation marks wrapped around the file system type (ext4, vfat, NTFS etc):

```shell
UUID=XXXX-XXXX  /mnt/mediastorage TYPE   nofail,uid=pi,gid=pi   0   0
```

The umask 0002 sets 775 permissions so the pi user and group can read, write and execute files on the external USB drive. To completely eliminate permission issues you can set the umask to 0000 which equals 777 permissions so anybody can read, write and execute. Note that 777 permissions are considered a security risk.

If you have issues here then try replacing uid=pi,gid=pi with just the word defaults (typical for ext4). You can also try replacing the UUID with the /dev/sda1 line.

In my case, for NTFS:

```shell
/dev/mmcblk0p1 /boot vfat defaults 0 2
/dev/mmcblk0p2 / ext4 errors=remount-ro,noatime 0 1

UUID=328C83488C830597    /mnt/mediastorage    ntfs   nofail,uid=pi,gid=pi    0   0
```

If you get any errors you can replace uid=pi,gid=pi with defaults or remove it entirely:

```shell
/dev/mmcblk0p1 /boot vfat defaults 0 2
/dev/mmcblk0p2 / ext4 errors=remount-ro,noatime 0 1

UUID=328C83488C830597    /mnt/mediastorage    ntfs   nofail,defaults    0   0
```

For using /dev/sda1 and defaults if you have troubles with UUID

```shell
/dev/mmcblk0p1 /boot vfat defaults 0 2
/dev/mmcblk0p2 / ext4 errors=remount-ro,noatime 0 1

/dev/sda1    /mnt/mediastorage    ntfs   nofail    0   0
```

Now test the fstab file works:

```shell
sudo mount -a
```

If you didn’t get errors then reboot, otherwise try the suggestions above to get it working then mount -a again until it succeeds

```shell
sudo reboot
```

You should be able to access the mounted USB drive and list its contents

```shell
cd /mnt/mediastorage
ls
```

Every time you reboot, the drives will be mounted as long as the UUID remains the same. If you delete the partitions or format the USB hard drive or stick the UUID changes so bear this in mind. You can always repeat the process for additional hard drives in the future.

If you have multiple hard drives you will have to make separate mount points (e.g. /mnt/mediastorage2) for each drive’s partition.

## Connecting a Browser

To connect to the browser, enter the IP followed by the port 32400 and /web/. For example, in my case:

```shell
192.168.1.95:32400/web/
```

You will be prompted to log in, simply sign up or sign in to an existing plex account. Next, you will need to set up your music, movie, and TV show libraries:

1. First select add library in the left-hand side column.
2. Next, select the type of media that is in the folder. If you have more than one type, then you will need to add a new library for each type of media.
3. Next, you will need to select the folder that has all your media in it. For example, mine is on /mnt/mediastorage/Media.
4. Once added the library, Plex will now organize the files on its interface.

## Transfer Plex Library from Windows to Raspberry Pi (Raspbian)

1. Go to %LOCALAPPDATA%\Plex Media Server on Windows.
2. Backup the folders: **Media**, **Metadata**, **Plug-in Support**, **Plugins**. These folders contain the data and the media images for the library.
3. Configure Plex on Linux via the web interface. At this point you’ll have an empty library.
4. Shutdown Plex on Linux:

```shell
sudo systemctl stop plexmediaserver
```

5. Copy the backed up media files to Linux:

```shell
/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/
```
6. Fix permissions in the copied folders (Repeat this step with the folders: **Metadata**, **Plug-in Support** and **Plugins**):

```shell
sudo chown -R plex:plex /var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Media
sudo chmod -R 755 /var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Media
```

7. Start Plex:

```shell
sudo systemctl start plexmediaserver
```

8. Open Plex web interface and change the Library paths to new paths. This will trigger a rescan of the files.
9. After complete, re fresh the plex website.

## Completely Uninstall Plex Media Server

```shell
sudo apt-get purge plexmediaserver
```

What it does:
1. Uninstall the package
2. Removes the user **plex**
3. Deletes its home directory **/var/lib/plexmediaserver**

If you’re uncertain and/or want to be certain about user plex and its data (home directory):

```shell
sudo userdel plex
sudo rm -rf /var/lib/plexmediaserver
```
