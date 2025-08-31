# My changes

## Dockerfile

```dockerfile
FROM debian:unstable
RUN apt-get update -qq && apt-get upgrade -qq && apt-get install -qq build-essential mmdebstrap bubblewrap devscripts squashfs-tools-ng zstd git sbuild
RUN useradd --create-home belenios
COPY --chown=belenios:belenios . /tmp/belenios
WORKDIR /tmp/belenios
RUN cp contrib/debian/ocaml-backports-keyring.asc /etc/apt/trusted.gpg.d
RUN contrib/debian/install-deps.sh
USER belenios
RUN contrib/debian/setup-build-dir.sh /tmp/build
WORKDIR /tmp/build
RUN git config --global user.name "Belenios Builder"
RUN git config --global user.email "belenios.builder@example.org"
RUN git config --global --add safe.directory /tmp/belenios
```
## Makefile
`_docker/build/Makefile` links to `debian/build.mk` which runs various scripts as described elsewhere. I changed the following scripts.

### `config.sh`
Just use Debian trixie.

```bash
: ${STABLE_MIRROR:="http://deb.debian.org"}
: ${STABLE_SUITE:="trixie"}
```

### `chroot.sh`
I found it necessary to remove all references to backports as these did not work on Arm64.

```bash
#!/bin/sh

# This script generates a chroot tarball suitable for compiling
# Belenios. It uses mmdebstrap. On some machines, it runs in approx. 3
# min and generates a .tar.zst of approx. 1.5 GB.

set -e

if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <target>"
    exit 1
fi

TARGET="$1"

. "$(dirname "$0")/config.sh"

export SOURCE_DATE_EPOCH="$(git log -1 --pretty=format:%ct)"

. "$(dirname "$0")/deps.sh"

TMP="$(mktemp --tmpdir --directory tmp.belenios.XXXXXXXXXX)"
trap "rm -rf $TMP" EXIT
chmod a+rx "$TMP"

cat > "$TMP/sources.list" <<EOF
deb $STABLE_MIRROR/debian $STABLE_SUITE main
EOF

mkdir "$TMP/belenios-npm"
( cd frontend && npm install && npm ci --cache "$TMP/belenios-npm" )
cp frontend/package-lock.json "$TMP/belenios-npm"
rm -rf "$TMP/belenios-npm/_logs"

echo "here1"

mmdebstrap --variant=buildd \
  --setup-hook='mkdir -p "$1"'"$TMP" \
  --include="passwd build-essential debhelper $BELENIOS_DEVDEPS $BELENIOS_DEBDEPS systemd-resolved" \
  --customize-hook='copy-in "'"$TMP"'"/belenios-npm /var/cache' \
  --customize-hook='chroot "$1" chown root:root -R /var/cache/belenios-npm' \
  --customize-hook='chroot "$1" apt-get update' \
  "$STABLE_SUITE" "$TARGET" "$TMP/sources.list"
```

### `make-squashfs.sh`

As for `chroot.sh` I removed all references to backports. Note - you need to substitute your SES user and password as indicated.

```bash
#!/bin/sh

# This script creates a squashfs image suitable for use with
# systemd-nspawn.

set -e

if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <belenios-server-deb> <target>"
    exit 1
fi

BELENIOS_SERVER_DEB="$1"
BELENIOS_SERVER_BUILDINFO="${BELENIOS_SERVER_DEB%.deb}.buildinfo"
TARGET="$2"

. "$(dirname "$0")/config.sh"

export SOURCE_DATE_EPOCH="$(git log -1 --pretty=format:%ct)"

TMP="$(mktemp --tmpdir --directory tmp.belenios.XXXXXXXXXX)"
# trap "rm -rf $TMP" EXIT
echo "I: using directory $TMP..."

chmod 755 "$TMP"

cp "$BELENIOS_SERVER_DEB" "$TMP"
BELENIOS_SERVER_DEB="$TMP/${BELENIOS_SERVER_DEB##*/}"

# Filter out build date for reproducibility
grep -v "^Build-Date: " "$BELENIOS_SERVER_BUILDINFO" > "$TMP/buildinfo.txt"
BELENIOS_SERVER_BUILDINFO="$TMP/buildinfo.txt"

cat > "$TMP/sources.list" <<EOF
deb $STABLE_MIRROR/debian $STABLE_SUITE main
EOF

cat > "$TMP/postinst.sh" <<EOF
#!/bin/sh

set -e

echo "I: setting up the rootfs..."

# ln -sfT /usr/lib/systemd/resolv.conf /etc/resolv.conf
# ln -sfT /run/systemd/resolve/resolv.conf /etc/resolv.conf

# AWS
cat > /etc/resolv.conf <<XOF
nameserver 172.31.0.2
search ec2.internal
XOF

echo belenios > /etc/hostname

cat > /etc/hosts <<XOF
127.0.0.1 localhost
127.0.1.1 belenios
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
XOF

mkdir /etc/belenios

cat > /etc/msmtprc <<XOF
defaults
tls on
tls_starttls on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
syslog on

account ses
host email-smtp.us-east-1.amazonaws.com
port 587
auth on
user ***********                 # SES user from SES-SMTP settings-Create SMTP credentials 
password **********              # SES user password
from elections@wrmack.com

account default : ses
# from %U@belenios
syslog LOG_MAIL
XOF

SBOM=/usr/share/belenios-server/sbom/runtime-deb-packages.txt
echo "Installed-Packages:" > \$SBOM
dpkg-query -W -f=',\n \${binary:Package} (= \${Version})' | tail -n +2 >> \$SBOM
echo >> \$SBOM
chown root:root -R /usr/share/belenios-server/sbom
EOF
chmod +x "$TMP/postinst.sh"

mmdebstrap --variant=essential \
  --setup-hook='mkdir -p "$1"'"$TMP" \
  --dpkgopt='path-exclude=/usr/share/man/*' \
  --dpkgopt='path-exclude=/usr/share/locale/*' \
  --dpkgopt='path-include=/usr/share/locale/locale.alias' \
  --dpkgopt='path-exclude=/usr/share/doc/*' \
  --dpkgopt='path-include=/usr/share/doc/*/copyright' \
  --dpkgopt='path-include=/usr/share/doc/*/changelog.Debian.*' \
  --hook-dir=/usr/share/mmdebstrap/hooks/file-mirror-automount \
  --include="passwd systemd dbus msmtp-mta logrotate" \
  --include="$BELENIOS_SERVER_DEB" \
  --customize-hook='copy-in "'"$BELENIOS_SERVER_BUILDINFO"'" /usr/share/belenios-server/sbom' \
  --customize-hook='copy-in "'"$TMP"'/postinst.sh" /tmp' \
  --customize-hook='chroot "$1" /tmp/postinst.sh' \
  --customize-hook='chroot "$1" rm /tmp/postinst.sh' \
  --customize-hook='chroot "$1" rm -rf '"$TMP" \
  "$STABLE_SUITE" "$TARGET" "$TMP/sources.list"
```
