# SELinux troubleshooting

## Cannot run as service
When trying to start belenios-container@main.service with systemctl, an error message showed SELinux was preventing s-nspawn from reading and executing belenios-nspawn.

I had to do:
```bash
sudo ausearch -c '(s-nspawn)' --raw | audit2allow -M my-snspawn
sudo semodule -X 300 -i my-snspawn.pp
```