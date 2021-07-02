# rockylinux-on-wsl

Following are instructions on how to get RockyLinux installed into a Windows10 WSL distro.
It should provide a good solid base install of RockyLinux that you can customise further once installed.

## upstream references
Most of this comes from [https://docs.microsoft.com/en-us/windows/wsl/use-custom-distro]
The main difference is getting the rootfs from docker

## Prereq's
Install Windows terminal
[https://aka.ms/terminal]

Install Cascadia Code PL fonts
download the zip file and extract it in Downloads directory , then browse to the files and go into the ttf directory. Highlight all of the .ttf files right click and select install for all users.  This should install the fonts needed for powerline to show the little graphics.

[https://github.com/microsoft/cascadia-code/releases/download/v2105.24/CascadiaCode-2105.24.zip]

Install WSL on Windows10, follow these instrauctions:
[https://docs.microsoft.com/en-us/windows/wsl/install-win10]
Note: the machine must support virtualization to be able to run WSL. I tried installing on a Windows10 vm in proxmox with little success.
You can trick it to install but I just installed onto a physical windows machine for this guide. As usual YMMV 

Run these from a powershell terminal with Admin privs:
```
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```
A reboot is required to iniate WSL before updating further 

Install the wsl2 kernel update from this link [https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi]

Set WSL2 as the default:
```
wsl --set-default-version 2
```

A Linux VM or machine with docker-ce installed.

The rootfs.tar file will be scp'd using winscp or filezilla from this vm onto the windows 10 machine to be imported into WSL.

## Quick Docker Setup
I used a Rocky Linux 8.4 vm that I built from my own custom pxeboot kickstart deployment method but a standard minimal next next next ISO 
Install will suffice for this purpose.

On the Rocky 8.x Linux vm install docker-ce

The extras repo is required but is installed by default on Rocky 8.x.
setup the docker-ce repo files and start docker:
```
$ sudo yum remove podman -y
$ sudo yum install -y yum-utils
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum install docker-ce docker-ce-cli containerd.io
$ sudo systemctl enable docker
$ sudo systemctl start docker
```

Get the Rocky 8.4 docker image:
```
[root@localhost ~]# docker pull rockylinux/rockylinux
Using default tag: latest
latest: Pulling from rockylinux/rockylinux
1b474f8e669e: Pull complete
Digest: sha256:8122f31fbdd5c1368c6b7d5b9ae99fec2eb5966a5c967339d71e95c4a3ab7846
Status: Downloaded newer image for rockylinux/rockylinux:latest
docker.io/rockylinux/rockylinux:latest
```

List the iamges:
```
$ docker image ls
REPOSITORY              TAG       IMAGE ID       CREATED      SIZE
rockylinux/rockylinux   latest    333da17614b6   7 days ago   227MB
```

Run a container:
```
$ docker run -it rockylinux/rockylinux /bin/bash
```

Get the container ID
from another session on the vm find the docker container id using:
```
$ sudo docker ps
CONTAINER ID   IMAGE                   COMMAND       CREATED         STATUS         PORTS     NAMES
6edab05f8055   rockylinux/rockylinux   "/bin/bash"   3 minutes ago   Up 3 minutes             hungry_goldwasser
```

Export the container:
```
$ sudo docker export 6edab05f8055 > /tmp/rocky.tar
$ ls -la /tmp/rocky.tar
-rw-r--r--. 1 root root 233887744 Jun 29 13:20 /tmp/rocky.tar
```

SCP the container to your windows machine:
using winscp or filezilla scp the /tmp/rocky.tar file to your c:\temp directory on your windows machine

## Windows side of things
Create a root install folder for your linux distros I used c:\linux
use powershell as it has more linux like commands:
```
mkdir c:\temp
mkdir c:\linux
mkdir c:\linux\rocky
```
Import to wsl:
```
wsl --import rocky c:\linux\rocky c:\temp\rocky.tar
```

list linux distros:
```
wsl -l -v
NAME       STATE           VERSION
rocky    Running           2
```

Run it:
```
wsl -d rocky
```
Setup a default user account:
```
dnf -y install epel-release && dnf -y update && dnf -y install passwd sudo
myUsername=<insert username here>
adduser -G wheel $myUsername
echo -e "[user]\ndefault=$myUsername" >> /etc/wsl.conf
passwd $myUsername
```

Set the root passwd:
```
passwd root
```

Install all the things:
```
dnf -y install \
vim \
wget \
curl \
traceroute \
rsync \
dos2unix \
zip \
unzip \
bash-completion \
bc \
bzip2 \
file \
time \
glibc-langpack-en \
libmodulemd \
libzstd \
passwd \
sudo \
cracklib-dicts \
openssh-clients \
htop \
ansible \
git \
mc \
nmap-ncat \
bind-utils \
connect-proxy \
powerline \
powerline-fonts
```

Do a full yum upgrade:
```
yum -y upgrade
```

If you want to get a more standard install you can install the @Base software group which will make it more like a standard fatter install.
Not installing @Base makes the install much leaner but you may need to install packagfes as you go along if they are missing.
Up to you.
```
dnf -y groupinstall base
```

Setup Windows terminal to use the new Rocky distro:
```
wsl -s rocky
```
Make the following changes in windows terminal to make the home directory /home/<insert username here> not /mnt/c/Users/<insert username here>:
```
In settings set the default startup profile to be rocky84
Select rocky
then set these
#General#
Commandline:  wsl.exe -u <insert username here> -d rocky
Starting Directory: //wsl$/rocky/home/<insert username here>
#Appearance#
Text Colour scheme: tango dark
Font face: Cascadia Code PL
```
Setup powerline in bash:
add the following to your .bashrc file to enable powerline
```
if [ -f `which powerline-daemon` ]; then
  powerline-daemon -q
  POWERLINE_BASH_CONTINUATION=1
  POWERLINE_BASH_SELECT=1
  . /usr/share/powerline/bash/powerline.sh
fi
```

# No need for the Linux VM anymore either shutddown or destroy it.

If rocky was already installed and was using wsl v1 you can upgrade to wslv2
Set rocky to use wsl v2:
```
wsl -l -v
NAME       STATE           VERSION
rocky    Running           1

wsl --set-version rocky 2
Conversion in progress, this may take a few minutes...
For information on key differences with WSL 2 please visit https://aka.ms/wsl2
Conversion complete.
```
After this you should have a very functional yet minimal RockyLinux install to customise to your hearts content.

