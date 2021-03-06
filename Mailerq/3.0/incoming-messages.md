# Processing incoming mail

Mail can be injected into MailerQ in two ways: either by using the AMQP protocol to publish
messages directly to the outbox message queue, or by using MailerQ's 
built-in SMTP port. Your application can connect to this port, and
use the SMTP protocol to inject emails.


## Available ports

In the config file there are a number of variables that you can use
to set the ports on which MailerQ should listen, and from which ips MailerQ should
accept incoming SMTP connections:

````
smtp-port:          25,2525,1024-1026
smtp-ip:            1.2.3.4
````

The `smtp-port` variable holds the port or ports that MailerQ opens and
on which incoming connections are accepted. You can assign a single port,
but comma-seperated values and port-ranges are also accepted. The
default SMTP port is 25, which is the one that you probably want to use.

Normally, MailerQ opens this port on all IP address that are available on
the server. This means that if a server has multiple IP addresses, it does
not matter to which of its IP addresses you connect: MailerQ listens on
all of them. If you want to limit this, you can assign an explicit IP
address using the `smtp-ip` variable. If you set this, MailerQ will
only accept incoming connections to that specific IP.

The IP address to which you send a message to MailerQ, is the same as 
the address *from* which it is forwarded. Thus, if you send an email to 
MailerQ listening on IP address 5.6.7.8, the message will also be sent 
out from this IP.


## Secure connections

You probably want to secure your SMTP traffic, and inject your emails
using encrypted connections. To enable encryption, you need to assign
a private key that is used for the encryption. You can use a self-signed
private key, but it is much better to use a key from a certificate 
authority. However, using a self-signed key is still better than using 
no key at all, as most SMTP clients will still accept the self-signed 
key.

````
smtp-certificate:       /path/to/certificate.crt
smtp-privatekey:        /path/to/privatekey.key
smtp-ciphers:           !aNULL:!eNULL:!LOW:!SSLv2:!EXPORT:!EXPORT56:FIPS:MEDIUM:HIGH:@STRENGTH
````

If you have enabled secure connections, it still is up to the SMTP 
client to decide whether to use this by sending the "STARTTLS" 
instruction over the SMTP protocol. Even if you have enabled encryption,
it still is possible to send in mail over plain SMTP.

To set up a secure connection, an extra "STARTTLS" handshake is necessary,
which makes setting up the connection slower than necessary. MailerQ 
allows you to open up a socket that is in an encrypted state right away,
without sending the "STARTTLS" command:

````
smtp-secure-port:       465
````

The port assigned to the `smtp-secure-port` variable is in an encrypted 
state right away, just as if the "STARTTLS" was already sent. Such an 
encrypted connection is not part of the SMTP standard, and regular SMTP
clients do not expect this. However, if you write your own SMTP handshake
code, it sometimes is simpler and faster to have access to a connection 
that is already encrypted without using "STARTTLS".

Just like the `smtp-port` variable, you can also assign comma seperated
lists of ports, or port ranges.


## Restricting access

The SMTP port is opened for the entire world, meaning that everyone can
connect to it and inject mails. You therefore need to set up a firewall
to restrict access, or configure MailerQ so that only connections from
trusted sources or trusted users are allowed.

````
smtp-ranges:            1.2.3.4/31;4.5.6.7/30
smtp-username:          username
smtp-password:          password
````

The `smtp-ranges` setting in the config file can be used to limit the
IP addresses from which incoming connections are accepted, and the 
`smtp-username` and `smtp-password` settings can be used to install
a login for accessing the SMTP port.

Keep in mind that if you set a username and password, the SMTP handshake
becomes a little slower (incoming connections first have to authenticate
before a message can be delivered). If you do not have to, it is therefore 
ofter faster to set up a firewall or an IP range to restrict access, 
than to require authentication.


## Running behind a proxy

You can run MailerQ behind a HAProxy proxy server that has the 
[PROXY protocol](http://www.haproxy.org/download/1.5/doc/proxy-protocol.txt)
enabled. You must mention this in the config file however, so that 
MailerQ knows that the first message received over the TCP connection
is not already the SMTP handshake, but the initial header from the
proxy server in which the TCP details (remote IP address and remote port
for example) of the incoming connection are transfered.

````
smtp-proxy:             1
````

If you run MailerQ behind a proxy server, you should also enable this 
PROXY protocol in the configuration file of your proxy server. Only 
version 2 of the PROXY protocol is supported.


## Publishing incoming messages

All incoming messages on the SMTP port are published in JSON format to 
message queues in RabbitMQ. In the MailerQ configuration file you can 
set the names of the message queues to which these messages are 
published.

````txt
rabbitmq-inbox:         name-of-queue-for-valid-messages
rabbitmq-refused:       name-of-queue-for-rejected-messages
````

Every valid incoming message is published to the inbox queue. Many users
assign the same queue to both the `rabbitmq-inbox` and 
`rabbitmq-outbox` variables, so that messages that are sent to MailerQ's
SMTP port automatically end up in the outbox, and are thus immediately
forwarded to the actual recipient. If you want to preprocess incoming 
messages before you forward them, you can however assign a different 
inbox queue and write a script that processes the incoming messages.

When a message is rejected (for example because a user was not 
authenticated), the message is normally lost. If you want to inspect 
such rejected messages to check what went wrong, or whether the messages
came from an untrusted source and your system is under attack from 
hackers, you can collect them in a special dedicated queue for
rejected messages. The `rabbitmq-refused` config file variable can be 
set to instruct MailerQ to store the rejected messages in this queue.

Be careful here: the queue with refused messages does not automatically
get emptied, and can fill up fast in case of an attack. MailerQ only 
adds messages to it, and it is up to you to periodically check the
contents of the queue and empty it, or to set a max length and/or max
age for messages in the queue (you can use [RabbitMQ's web based 
management console](rabbitmq-config) to set such limits).


## Collecting bounces and notifications

```txt
rabbitmq-reports:       name-of-queue-for-delivery-reports
``` 

Every incoming message is checked by MailerQ to see if it is a Delivery 
Status Notification, or some other sort of delivery report or feedback 
loop message. If MailerQ recognizes such a message, it will 
not publish it to the regular inbox, but to a special queue with 
delivery reports instead. In the MailerQ configuration file you can use
the `rabbitmq-reports` variable to assign a queue for these delivery
reports. 

MailerQ accepts all delivery reports, even the ones that come
from connections that are not authenticated. This means that MailerQ 
allows every incoming connection to finish the entire SMTP handshake, 
including sending the envelope, recipient and full MIME message data. 
After the message is received, MailerQ checks whether the message 
was a delivery report, in which case it is turned into JSON and 
published to the reports queue. A regular message is published to the 
inbox queue if it came from a valid authenticated connection, or to the
queue with refused messages if it came from an non-authenticated 
client.

## Local email addresses

MailerQ only accepts messages through SMTP when the connection has been 
successfully authorized to send messages. In some cases you need to be 
able to accept messages to a specific (local) email address without having 
to authenticate first. 

These can be set in the [MailerQ management console](management-console).
All emails sent to specific addresses in the local email address list, will not 
have to authenticate before sending and will be entered directly into the `rabbitmq-local`-queue. 

