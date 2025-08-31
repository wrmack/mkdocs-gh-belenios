# Containers

## Isolation
`chroot`, `unshare`, `systemd-nspawn`, `podman`, `docker` all provide isolation, to different degrees, of files, processes, user namespaces, networks, and more.   Containers "contain" or "isolate" something in some way.  Code is on the same file system as it always was.  It is just "contained" or isolated and hence more secure and portable (to the extent it doesn't have external dependencies).  

All of these isolation mechanisms are used in producing a squashfs image for runnning Belenios.

The outcome, is a squashfs image which can be run by systemd-nspawn as a systemd service on the host. 

Comparison table:


| Feature / Tool       | **chroot**                                                             | **unshare**                                                        | **systemd-nspawn**                                                                        | **docker**                                                                                  |
| -------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------ | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Purpose**          | Change root directory for a process (filesystem isolation)             | Run a program with new Linux namespaces (fine-grained isolation)   | Lightweight container manager built on Linux namespaces & cgroups (part of systemd)       | Full container platform with runtime, networking, orchestration, and registry integration   |
| **Isolation Level**  | Filesystem only (unless combined with other tools)                     | User-specified namespaces (PID, NET, UTS, IPC, USER, MNT, etc.)    | Filesystem + namespaces (PID, NET, IPC, UTS, user) + cgroups resource limits              | Full isolation with namespaces, cgroups, seccomp, capabilities                              |
| **Security**         | Weak (processes can escape with root privileges unless extra measures) | Stronger if combined namespaces are used, but depends on config    | Stronger — drops capabilities, applies cgroups, integrates with systemd security features | Strongest by default — adds seccomp filters, AppArmor/SELinux profiles, capability dropping |
| **Ease of Use**      | Very simple, but low-level; needs a prepared root filesystem           | Low-level, requires explicit namespace setup; flexible but complex | Higher-level, single command to start an isolated container; integrates with systemd      | Higher-level, with CLI, Dockerfiles, image registry, orchestration (Swarm/Kubernetes)       |
| **Networking**       | Shares host network unless combined with `unshare -n` or extra setup   | Configurable (can isolate network stack or use host)               | Defaults to private veth bridge; can integrate with host networking                       | Advanced networking (bridge, overlay, host, macvlan) with built-in tooling                  |
| **Resource Control** | None                                                                   | None (unless you manually apply cgroups)                           | Yes, via cgroups (CPU, memory, I/O)                                                       | Yes, via cgroups (CPU, memory, I/O) with fine-grained limits                                |
| **Image Management** | None (you prepare the root fs manually)                                | None                                                               | Supports machine images (`systemd-nspawn -D /path` or `--image`)                          | Built-in image distribution via Docker Hub / registries                                     |
| **Init / PID 1**     | None (you must run your own init manually)                             | None (just runs the command you give)                              | Runs systemd (or chosen init) as PID 1 in the container                                   | Runs whatever you specify (commonly an app, or full init system)                            |
| **Typical Use Case** | Testing a different root filesystem, rescue/repair                     | Advanced sandboxing, debugging namespaces                          | Lightweight containers on a systemd host, testing distros, dev environments               | Application containerization, microservices, production deployments                         |




## What do the scripts do?

### podman build
```bash
podman build -f contrib/docker/nspawn-build.Dockerfile -t belenios-nspawn-image .
```
Looking at `nspawn-build.Dockerfile` this, among other things,:

- builds a Debian image and installs dependencies
- sets up the belenios user and ownership
- runs the script `contrib/debian/setup-build-dir.sh /tmp/build` which creates in `_docker/build`,:
    - `Makefile` which is a symbolic link to `contrib/debian/build.mk`
    - `Makefile.config`

### podman run
```bash
podman run -it --name belcont --volume=$PWD:/tmp/belenios:Z --volume=$PWD/_docker/build:/tmp/build:Z --rm --userns=keep-id --privileged --security-opt label=disable belenios-nspawn-image
```
This runs in privileged mode with SELinux disabled in the container.

To get a shell inside the container with root privileges do:
```bash
podman exec -it -u 0 belcont bash
```

When the command `make` is executed it runs `_docker/build/Makefile` (a link to `contrib/debian/build.mk`) which calls:

- $(SQUASHFS), which calls $(DEB), which calls $(DSC) and $(CHROOT)
- $(DSC) runs the script `contrib/debian/make-dsc.sh` which builds `belenios-server_XXX.dsc`
- $(CHROOT) runs the script `contrib/debian/make-chroot.sh`
    - it calls `config.sh`
- $(DEB) runs the `sbuild` command to build Debian packages
- finally $(SQUASHFS) runs the script `contrib/debian/make-squashfs.sh`

## To run rootfs.squashfs with systemd-nspawn

```bash
sudo systemd-nspawn -i /var/lib/machines/belenios-containers/main/rootfs.squashfs
# Or
sudo systemd-nspawn -i /var/lib/machines/belenios-containers/main/rootfs.squashfs --boot
```
## To run as systemd service
```bash
sudo systemctl start belenios-container@main.service
```

## Inspect with machinectl

```bash
machinectl list
machinectl show rootfs.squashfs
machinectl status rootfs.squashfs
# Run interactive shell as root in running container
sudo machinectl shell <container name>
```