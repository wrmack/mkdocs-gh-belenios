# Set up email using Amazon SES (Simple Email Service)

## `msmtp`

In Amazon SES, set up verified identities. Under "Email receiving" create a rule set with a condition containing a recipient email address. This is used as the 'from' address in msmtprc configuration (below).

The Belenios code calls `/usr/lib/sendmail` unless the environment variable `BELENIOS_SENDMAIL` is set:

```ocaml
(* src/web/server/common/send_message.ml *)
let mailer =
  match Sys.getenv_opt "BELENIOS_SENDMAIL" with
  | None -> "/usr/lib/sendmail"
  | Some x -> x
```

When the squashfs image is built, msmtp is installed. The `sendmail` executable is linked to msmtp:

```bash
root@rootfs:~# ls -al /usr/lib/sendmail
lrwxrwxrwx. 1 root root 12 Apr 16 21:42 /usr/lib/sendmail -> ../bin/msmtp
```

Therefore, unless the environment variable BELENIOS_SENDMAIL is set, the code calls msmtp to send emails.

`msmtp` configuration is in `/etc/msmtprc` which is written when the squashfs image is built. The following shows my changes in `contrib/fedora/make-squashfs.sh`:

```bash
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
user AKIA3BQFZYK6ZWIIXT7P
password ******************
from elections@wrmack.com

account default : ses
# from %U@belenios
syslog LOG_MAIL
XOF
```
A valid "from" address having the same domain as the domain from which emails are sent is important to avoid emails being sent to spam.

The domain should have SPF, DKIM and DMARC TXT records. If the domain is set up using Amazon Route 53 then configuring SES should provide these.

## Security summary

The msmtp settings ensure:
- tls encryption is applied to emails
- user authentication is by password

Amazon settings:
 - SPF (Sender Policy Framework): 
    - valid emails must come from domains authorised in this record (ie my domain: wrmack.com)
    - this provides a check that someone else is not pretending to be me 
 - DKIM (DomainKeys Identified Mail): emails are digitally signed and DKIM holds the public key
 - DMARC (Domain-based Message Authentication, Reporting, and Conformance):
    - tells receiving server what action to take if SPF and DKIM fail
    - this might be to reject the email or quarantine it (send it to spam) 
