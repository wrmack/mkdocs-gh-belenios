# Troubleshoot `msmtp`

## How to:

### Try msmtp from inside the Belenios systemd-nspawn container

In one terminal:

- log into the AWS EC2 instance
- start the belenios-container service if it is not already active

```bash
sudo systemctl start belenios-container@main.service
```

- then:

```bash
# Get container name
machinectl list
# MACHINE       CLASS     SERVICE        OS     VERSION ADDRESSES
# belenios-main container systemd-nspawn debian 13      -

# Get shell access to the running container
sudo machinectl shell belenios-main

# In the container, display server information
msmtp --serverinfo

# Send an email message 
# - simply provide recipient email address,
#   all other settings are taken from /etc/msmtprc
# - end message by pressing Control-d
msmtp recipient@example.com 
Subect: Test

This is a test
CTL-D
```

- after sending an email check if it goes into the recipients spam folder
- check raw source of email of the received email for whether it passed checks for SPF, DKIM, DMARC
- in the raw source look for entries like this:

```
Authentication-Results: xxxxxxxxxxxxxxxx;
 dkim=pass header.i=xxxxxxxxxxxxx;
 dkim=pass header.i=xxxxxxxxxxxxx;
 spf=pass smtp.mailfrom=xxxxxxxxxxxx;
 dmarc=pass(p=QUARANTINE) header.from=xxxxxxxxxxx;
```