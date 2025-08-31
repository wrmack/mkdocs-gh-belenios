# DNS name resolution

I had problems sending email. `msmtp` could not resolve the email endpoint name (`email-smtp.us-east-1.amazonaws.com`). The systemd Belenios service executes systemd-nspawn which can take arguments to point to a resolved configuration (see `man systemd-nspawn` and the `--resolv-conf` argument).  I could not get these options to work. In the end I hardwired what my EC2 uses into the post-inst.sh script in the squashfs image.

What my EC2 instance uses for DNS resolution:

```bash
[ec2-user@ip-172-31-30-217 ~]$ ls -al /etc/resolv.conf
lrwxrwxrwx. 1 root root 32 Jul 30  2024 /etc/resolv.conf -> /run/systemd/resolve/resolv.conf

[ec2-user@ip-172-31-30-217 ~]$ cat /run/systemd/resolve/resolv.conf
# This is /run/systemd/resolve/resolv.conf managed by man:systemd-resolved(8).
# ...

nameserver 172.31.0.2
search ec2.internal
```
Then in contrib/fedora/make-squashfs.sh:

```bash
cat > "$TMP/postinst.sh" <<EOF
#!/bin/sh

...

# AWS
cat > /etc/resolv.conf <<XOF
nameserver 172.31.0.2
search ec2.internal
XOF

...

EOF
```