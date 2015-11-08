logmanagement-rsyslog
+++++++++++++++++++++

This repository contains rsyslog configuration files useful in many logmanagement environments. The goal is to have a set of easily reusable configuration files that are tested, proven and well documented. For these configurations to be really useful in real world environments sufficient attention to issues related to confidentiality, integrity, authenticity and scalability have been given.


Contributing
============
Contributing configuration samples and/or use-cases is encouraged! Just fork this repository and send me a pull request.


Versions
========
We try to support all our use-cases / configurations for several supported release branches of rsyslog:

*   7-stable
*   8-stable


7.x stable
----------
These configurations have only been tested with the stock rsyslog packages as provided with RHEL/CentOS 7.x. The use of the official distro packages is recommended since they are the most stable in regards to versioning and thus the configuration options. Tracking the upstream repo requires you to test your configuration as upstream keeps releasing new versions.


8.x stable
----------
Not yet supported.


How does it work?
=================
The rsyslog configuration has been split into several files. Hopefuly, this makes it easy to just pick the components you need to implement your specific use-case. Just grab the ``.conf`` files you need and drop them into the appropiate location; the global configuration file (``rsyslog.conf``) should go into ``/etc/`` all other files should go into ``/etc/rsyslog.d/``. Make sure you first verify your configuration (``rsyslogd -N2``) before you reload/restart rsyslogd.

*   syslog.conf - the global configuration file
*   input_*.conf - input configuration files
*   output_*.conf - output configuration files
*   forward_*.conf - forwarding configuration files

Input
-----

*   journald - systemd's journal log
*   relp - reliable log transfer protocol
*   syslog / tcp - legacy syslog over tcp
*   syslog / tls - syslog over tls tunnel
*   syslog / udp - traditional syslog over udp
*   uxsocket - unix domain socket, eg. ``/dev/log``

Output
------

*   file - write log to local files

Forward
-------

*   syslog / udp - traditional syslog forwarding
*   syslog / tcp - improved syslog forwarding
*   syslog / tls - secure syslog forwarding


Use-cases
=========
Different use-cases can easily be implemented by combining several of the above configuration files. This section simply describes several example use-cases, and how to implement them using the provided configuration examples.

| Use-case | Short description                      | Input                 | Output                | Forward               |
| :------- | :------------------------------------- | :-------------------- | :-------------------- | :-------------------- |
| [1](#1)  | local logging                          | journald, uxsocket    | file                  | n/a                   | 
| [2](#2)  | local logging + traditional forwarding | journald, uxsocket    | file                  | syslog_udp            | 
| [3](#3)  | local logging + improved forwarding    | journald, uxsocket    | file                  | syslog_tcp            | 
| [4](#4)  | local logging + secure forwarding      | journald, uxsocket    | file                  | syslog_tls            | 


<a name="1">
1. local logging
----------------
This configuration gathers log from the systemd journal and the traditional syslog socket and writes it's output to local files only. This setup is most common for environments where central logging is not required (eg. your personal laptop).
</a>


<a name="2">
2. local logging + traditional forwarding
-----------------------------------------
Like the first use-case but also sends it's log using the traditional syslog forwarding method (syslog over UDP). Forwarding syslog over UDP is the most unreliable forwarding method available. It's also the most common and best supported forwarding method, if your log server supports better protocols like TCP or TLS use that instead.
</a>

<a name="3">
3. local logging + improved forwarding
--------------------------------------
Like the previous use-case but using TCP as the forwarding protocol. This method has a higher level of reliability because rsyslog can now detect when the destination is unavailable. In this case log messages will be queued and resend when the destination becomes available again. Although a lot better than using UDP; but still there's no 100% guarentee that messages can't be lost.
</a>

<a name="4">
4. local logging + secure forwarding
------------------------------------
This use-case implements local loggin with secure (TLS) forwarding. This method of forwarding is pretty good when dealing with strict requirements regarding confidentiality, integrity and authenticity (CIA) of logs. Unfortunately this method does also suffer from the same limitations as the TCP forwarding use-case, there's still no guarantee that messages can't be lost.
</a>


Setting up TLS
==============
This section describes how to create a Certificate Authority (CA) and server certificate to use with TLS transport methods. To begin with, you need to generate the root CA key, this is what signs all issued certificates.

    openssl genrsa -out rootCA.key 4096


Generate the self-signed root CA certificate, using the previousely generated key.

    openssl req -x509 -new -nodes -key rootCA.key -days 365 -out rootCA.crt


Once you have the root CA certificate generated, you can use that to generate additional SSL certificates for systems and/or services. We can now create the certificate for the rsyslog service on this system, but first we need to create a key.

    openssl genrsa -out host.key 2048


Then we can create a certificate signing request (CSR). Make sure the Common Name (CN) is set to the FQDN, hostname or IP address of the machine you're going to put this on. These values should be used in the rsyslog permittedpeers configuration directive when using x509/name authentication.

    openssl req -new -key host.key -out host.csr


The next step is to take the CSR and generate a signed certificate using the root CA certificate and key you generated previously. Now you have an SSL certificate (in PEM format) called host.crt. This is the certificate you want your rsyslog service to use.

    openssl x509 -req -in host.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out host.crt -days 365


