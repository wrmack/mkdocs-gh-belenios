# Prepare a squashfs image

## Documentation

The code in Belenios source code builds a squashfs image. Once this image is run as a container it provides the Belenios server. The source code to make the image is based on Debian Linux. The process for creating the image starts with creating a container running Debian. However I use podman rather than docker (podman is already installed in my Fedora setup). 

I amended some of the scripts - mainly to remove reference to backports (which seemed to be built for X64 systems). See [my fork](https://github.com/wrmack/belenios) for all changed files (under `contrib/fedora`).

## Build an image

I do the following from the OS host (ostree) using podman:

```bash
# change directory - move to belenios root
cd <path to belenios>

# Note the '.' at the end of the line
sudo podman build -f contrib/fedora/nspawn-build.Dockerfile -t belenios-nspawn-image .
```
## Setup
```bash
mkdir -p _docker
sudo podman run --rm belenios-nspawn-image tar -C /tmp -c build | tar -C _docker -x
```
## Run a container to create a squashfs compressed file system
The following executes `make` inside a podman container. (The Belenios documentation adds `make` to the `docker run` command line instead. My preference is to run the container without the `make` command, then from inside the container run `make`. If `make` fails, the container is not removed. There is the possibility of executing a bash shell from outside the container in another terminal as root if it is necessary to make changes with admin privileges.)

```bash
sudo podman run -it --name belcont --volume=$PWD:/tmp/belenios:Z --volume=$PWD/_docker/build:/tmp/build:Z --rm --userns=keep-id --privileged --security-opt=label=disable belenios-nspawn-image
# Inside the container do:
make
```

If you get an error message about git being unclean, exit the container and do:

```bash
git add .
git commit -m "Your message"
```
Then run the container again and execute `make`.

## Check the squashfs image provides belenios (optional)
We can run belenios locally as a service. We set the service up under /var/lib/machines rather than /srv because this is compatible with where containers in Fedora are expected: 

- in belenios-container@.service change paths, where they occur, from /srv to /var/lib/machines

```shell
mkdir -p /var/lib/machines/belenios-containers
cd <belenios root>
sudo cp contrib/fedora/belenios-nspawn /var/lib/machines/belenios-containers
sudo cp contrib/fedora/belenios-container@.service /etc/systemd/system
sudo mkdir /var/lib/machines/belenios-containers/main
sudo cp _docker/build/beleniosxxxxxxxx.squashfs /var/lib/machines/belenios-containers/main
sudo ln -s /var/lib/machines/belenios-containers/main/beleniosxxxxx.squashfs rootfs.squashfs
sudo mkdir /var/lib/machines/belenios-containers/main/belenios
sudo chown 1000:1000 /var/lib/machines/belenios-containers/main/belenios
mkdir /var/lib/machines/belenios-containers/main/belenios/etc
mkdir /var/lib/machines/belenios-containers/main/belenios/var
sudo cp demo/ocsigenserver.conf.in /var/lib/machines/belenios-containers/main/belenios/etc

sudo systemctl daemon-reload
sudo systemctl start belenios-container@main.service
```
- go to 127.0.0.1:8001 to check Belenios is being served.

