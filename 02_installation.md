## Installation of Docker
* MAC
* Windows
* Linux

----

### MAC
* Download Docker for Mac [link](https://docs.docker.com/engine/installation/mac/)
* Installing Docker for Mac does not affect machines you created with Docker Machine. You’ll get the option to copy containers and images from your local default machine (if one exists) to the new Docker for Mac HyperKit VM.
* The install includes: Docker Engine, Docker CLI client, Docker Compose, and Docker Machine.

----

### Windows
* Download Docker for Windows [link](https://docs.docker.com/engine/installation/windows/)
* Docker for Windows requires Microsoft Hyper-V to run. After Hyper-V is enabled, VirtualBox will no longer work, but any VirtualBox VM images will remain. 
* You can import a default VirtualBox VM after installing Docker for Windows by using the Settings menu in the System Tray.
* The current version of Docker for Windows runs on 64bit Windows 10 Pro, Enterprise and Education (1511 November update, Build 10586 or later). In the future we will support more versions of Windows 10.
* The Hyper-V package must be enabled for Docker for Windows to work. The Docker for Windows installer will enable it for you, if needed. (This requires a reboot).

----

### Linux
* No virtualization Layer needed [link](https://docs.docker.com/engine/installation/)
* Follow the instruction provided by Docker on their website, specific to your distribution