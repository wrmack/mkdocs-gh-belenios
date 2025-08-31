# Install Belenios locally

Belenios source code is on [Github](https://github.com/glondu/belenios), [Gitlab](https://gitlab.com/vcast.vote/belenios) and an Inria [Gitlab repository](https://gitlab.inria.fr/belenios/belenios).  It can also be downloaded from the Belenios [website](https://www.belenios.org/software.html).

My [fork](https://github.com/wrmack/belenios) contains modified files for creating a squashfs image on a Fedora Linux workstation. These files are found under `contrib/fedora`.

## Get the Belenios code

On Github, the options for getting the source code are:

1. [clone my fork](#clone-my-fork) (recommended)
1. [download a zipped file](#download-a-zipped-file)
1. [clone the origin repository](#clone-the-origin-repository)
1. [fork the origin repository](#fork-the-origin-repository)


### Download a zipped file

A zipped file contains only the source code.  It does not contain the git history. This is a suitable option if you only want the source code as-is in order to run it and you do not want to amend it.  If you amend the source code and you later download another version of Belenios, you will need to copy your changes to it.

Download the zip archive from [Github](https://github.com/glondu/belenios/tags).

Extract to an appropriate folder.  

**Set up git in the Belenios root folder**

The Belenios code assumes a git repository and, when building Belenios, looks for it.

In VSCode open the folder with the Belenios code then, under the Source tab, initialise a git repository and make an initial commit.

Then in a terminal:

```bash
# Configure git if not already configured
git config --global user.email "user@example.com"
git config --global user.name "username"

# Add a version tag
git tag -a -m "Belenios version 3.1" "3.1"
```
### Clone the Belenios repository
This creates a copy of the Belenios origin repository on your local workstation including the full git history. 

You can make changes. 

You can pull updates and deal with any merges conflicting with your changes.

You can create a remote on Github and push your code to it.

From the green 'Code' button on the Github Belenios repository, copy the https url. In a terminal change to an appropriate directory and

```bash
git clone <https url>
```

### Fork the Belenios repository
This creates a copy of the Belenios repository on Github under your own account name. 

You can then clone your fork to work on the code locally. Your own fork then becomes the "origin" and the original belenios repository becomes the "upstream".

You can pull updates from the upstream and deal with any merge conflicts.

You can push your local changes to your fork.  

From the Fork button on the Github Belenios repository, click the down arrow and select 'Create a new fork'.

Then, from your own fork, create a local clone as above. 

### Clone my fork
My [fork of Belenios](https://github.com/wrmack/belenios) has the changes that worked for me under the directory `contrib/fedora`.

Even if you don't use my fork, you might make the changes I made.  The files I changed are all in `contrib/fedora`. The changes are also described in [Troubleshooting - My changes](troubleshoot/my_changes.md) 

## Set up the non-OCaml dependencies
See the file INSTALL.md in the Belenios code for installation instructions. These recommend a Debian Sid distribution. For my Fedora 42 distribution I did:
```bash
sudo dnf install gcc-c++ gmp-devel libsodium-devel pkgconfig-pkg-config m4 openssl-devel sqlite3 sqlite-devel wget1 ca-certificates ncurses-devel zlib-ng-devel gd-devel cracklib jq nodejs-npm patch
```
## Run the opam-bootstrap.sh script
This script requires that dune is not installed. If you have dune installed in your current opam switch and don't want to remove it, you could create a new switch without it:

```bash
# Optional
opam switch create "nodune" 5.3.0
eval $(opam env)
```

```bash
./opam-bootstrap.sh
```
## Set the environment
```bash
source ./env.sh
eval $(opam env)
```
## Build the server
The script `frontend/Makefile` has two references to `nodejs`. Fedora knows `nodejs` as `node`. Change both of `nodejs` to `node`.

The script requires the git repository to have a tag for the current version.
```bash
# Check if a tag exists
git tag --list

# If a version tag is not present, create one like
git tag -a -m "Belenios version 3.1" "3.1"
```
Then

```bash
make build-release-server
```
## Check server works
```bash
demo/run-server.sh
```
In a browser go to `127.0.0.1:8001` to see if Belenios is up.
