# Development setup

This documentation describes how I installed Belenios on a Fedora linux workstation on Arm64 and deployed it to an AWS EC2 instance running Amazon Linux 2023 also on Arm64. 

## My setup

I have a Macbook Pro M1 (Arm64) running a Fedora Cosmic Atomic desktop in a virtual machine (VMWare Fusion). 

Fedora atomic desktops run an immutable file system based on ostree. (Only the /etc and /var directories are writable. The home directories are under /var/home.) Podman is preferred over Docker for containerisation.

For development work I have created a 'toolbx' container (as recommended by Fedora for atomic desktops). I have installed VSCode, OCaml and the Linux packages recommended by Belenios in the toolbx container. In the toolbx container I use VSCode to make any modifications to Belenios.

Any work involving containers I do from the Fedora OS host to avoid issues (if any exist) of running containers within containers. 

I have created a [fork of Belenios](https://github.com/wrmack/belenios), with my modifications under contrib/fedora.

## Assumptions
- Operating system is Fedora Linux
- OCaml is installed 
- The modifications in my fork are used to produce the squashfs image
- Deployment is to an AWS EC2 instance which has command line access using `ssh` (for logging in) and `rsync` (for uploading files).




