# Systemd 

Systemd is used to manage operating system startup, for the host where wgsync
is running.  Since Systemd exists, we might as well take advantage of its
mechanisms for orderly service startup.

This document describes the different System units that each wgsync component
uses, as well as a few other Systemd features that we take advantage of.

## Systemd Features Used

There are a number of Systemd features leveraged by wgsync:

* Notification

  Systemd's `sd_nofiy` mechanism is used by each service, to communicate to
  Systemd if the service was able to start successfully.  This is useful, as
  services will not have to daemonize just to tell Systemd that the service is
  running properly.  Instead, `sd_notify` may be used to confirm that the
  service has fully started up, and is ready for the next service to be run.

* Journaling

  Each service logs directly to Journald.  This allows other services to grab
  logs as they wish.  For example:

  * If rsyslog is installed, Journald may be configured to send journal entries
    to syslog.

  * If the Splunk Universal forwarder is used, FIFOs may be used to ship
    JSON-formatted journal entries from Journald to Splunk.

# LDAP Consumer

The LDAP Consumer is represented by the Systemd service unit
`wgsync-ldap.service`.

`wgsync-ldap.service` depends on `wgsync-event.service`: Since the LDAP
Consumer opens connections to the Event Generator, the Event Generator must be
up and running before the LDAP Consumer is able to start.

`wgsync-ldap.service` also depends on `wgsync-ldap-keytab.service`.  This
service is responsible for running `k5start`, which keeps the credentials cache
active.  If this service fails, then `wgsync-ldap.service` will be stopped.
This service depends on `network-online.target`, since network connectivity is
required in order to maintain the credentials cache.

`wgsync-ldap-keytab.service` depends on `wgsync-ldap-keytab-obtain.service`.
This is a one-shot service, and is responsible for locating the encrypted
keytab file on-disk, obtaining the AES-256-GCM key (from AWS KMS) needed to
decrypt the keytab, and doing the decryption/validation.  The decrypted keytab
is kept in a subdirectory of `/run`, so it is necessary for this to happen on
every boot.  This service also depends on `network-online.target`, because
network connectivity is required to AWS KMS.

`wgsync-ldap.service` also depends on `wgsync-ebs.service`.  This is another
one-shot service.  It is responsible for ensuring that the dedicated data store
EBS volume is mounted.

# Event Generator

The Event Generator is represented by the Systemd service unit
`wgsync-event.service`.  It represents the multiple Python processes which
listen to workgroup change messages from the LDAP Consumer, and which generate
the SNS notifications to clients.  This service also reads from DynamoDB
(cached using the local Dogpile cache), and creates table entries for
newly-discovered stems.

`wgsync-event.service` does not have any local dependencies, other than
`network-online.target`.  There is a dependency on DynamoDB, but that is
validated as part of service start.

# Remctl Generator

The remctl generator is one part of wgsync which is able to exist on a separate
system from the other components.  This is because it doesn't communicate
directly with the other components: `wgsync-remctl` has its own keytab, and
reads changes from an SQS queue.

The remctl generator is represented by the Systemd service unit
`wgsync-remctl.service`.  This service is "wanted by" `multi-user.target`.
That means it is possible to be started and stopped at the same time as the
other services.

`wgsync-remctl.service` depends on `wgsync-remctl-keytab.service`.  This
service is responsible for running `k5start`, which keeps the credentials cache
active.  If this service fails, then `wgsync-remctl.service` will be stopped.
This service depends on `network-online.target`, since network connectivity is
required in order to maintain the credentials cache.

`wgsync-remctl-keytab.service` depends on
`wgsync-remctl-keytab-obtain.service`.  This is a one-shot service, and is
responsible for locating the encrypted keytab file on-disk, obtaining the
AES-256-GCM key (from AWS KMS) needed to decrypt the keytab, and doing the
decryption/validation.  The decrypted keytab is kept in a subdirectory of
`/run`, so it is necessary for this to happen on every boot.  This service also
depends on `network-online.target`, because network connectivity is required to
AWS KMS.

# Web Site

TBD
