# SSLproxy - transparent SSL/TLS proxy for decrypting and diverting network traffic to other programs for deep SSL inspection [![Build Status](https://travis-ci.org/sonertari/SSLproxy.svg?branch=master)](https://travis-ci.org/sonertari/SSLproxy)

Copyright (C) 2017-2018, [Soner Tari](http://comixwall.org).  
https://github.com/sonertari/SSLproxy

Copyright (C) 2009-2018, [Daniel Roethlisberger](//daniel.roe.ch/).  
https://www.roe.ch/SSLsplit

## Overview

SSLproxy is a proxy for SSL/TLS encrypted network connections.  It is intended 
to be used for decrypting and diverting network traffic to other programs, such 
as UTM services, for deep SSL inspection. See [this 
presentation](https://drive.google.com/open?id=12YaGIGs0-xfpqMNAY3rzUbIyed-Tso8W) 
for a summary.

SSLproxy is designed to transparently terminate connections that are redirected 
to it using a network address translation engine.  SSLproxy then terminates 
SSL/TLS and initiates a new SSL/TLS connection to the original destination 
address. Packets received on the client side are decrypted and sent to the 
program listening on a port given in the proxy specification. SSLproxy inserts 
in the first packet the address and port it is expecting to receive the packets 
back from the program. Upon receiving the packets back, SSLproxy re-encrypts 
and sends them to their original destination. The return traffic follows the 
same path back to the client in reverse order.

![Mode of Operation 
Diagram](https://drive.google.com/uc?id=1N_Yy5nMPDSvY8YaNFd4sHvipyLWq5zDy)

This is similar in principle to [divert 
sockets](https://man.openbsd.org/divert.4), where the packet filter diverts the 
packets to a program listening on a divert socket, and after processing the 
packets the program reinjects them into the kernel. If there is no program 
listening on that divert socket or the program does not reinject the packets 
into the kernel, the connection is effectively blocked. In the case of 
SSLproxy, SSLproxy acts as both the packet filter and the kernel, and the 
communication occurs over networking sockets.

For example, given the following proxy specification:

	https 127.0.0.1 8443 up:8080

The SSLproxy listens for HTTPS connections on 127.0.0.1:8443. Upon receiving a 
connection from the Client, it decrypts and diverts the packets to a Program 
listening on 127.0.0.1:8080. After processing the packets, the Program gives 
them back to the SSLproxy listening on a dynamically assigned address, which 
the Program obtains from the first packet in the connection. Then the SSLproxy 
re-encrypts and sends the packets to the Server. The response from the Server 
follows the same path to the Client in reverse order.

The program that packets are diverted to should support this mode of operation.
Specifically, it should be able to recognize the SSLproxy address in the first
packet, and give the first and subsequent packets back to the SSLproxy 
listening on that address, instead of sending them to the original destination 
as it normally would.

A sample line SSLproxy inserts into the first packet in the connection is the 
following:

	SSLproxy: [127.0.0.1]:34649,[192.168.3.24]:47286,[192.168.111.130]:443,s

The first IP:port pair is a dynamically assigned address that the SSLproxy 
expects the program send the packets back to it. The second and third IP:port 
pairs are the actual source and destination addresses of the connection 
respectively. Since the program receives the packets from the SSLproxy, it 
cannot determine the source and destination addresses of the packets by 
itself, hence must rely on the information in this SSLproxy line. The last 
letter is either s or p, for SSL/TLS encrypted or plain traffic respectively. 
This information is also important for the program, because it cannot reliably 
determine if the actual network traffic it is processing was encrypted or not.

SSLproxy supports plain TCP, plain SSL, HTTP, HTTPS, POP3, POP3S, SMTP, and 
SMTPS connections over both IPv4 and IPv6.  It also has the ability to 
dynamically upgrade plain TCP to SSL in order to generically support SMTP 
STARTTLS and similar upgrade mechanisms.  SSLproxy fully supports Server Name 
Indication (SNI) and is able to work with RSA, DSA and ECDSA keys and DHE and 
ECDHE cipher suites.  Depending on the version of OpenSSL, SSLproxy supports 
SSL 3.0, TLS 1.0, TLS 1.1 and TLS 1.2, and optionally SSL 2.0 as well.

For SSL and HTTPS connections, SSLproxy generates and signs forged X509v3 
certificates on-the-fly, mimicking the original server certificate's subject 
DN, subjectAltName extension and other characteristics.  SSLproxy has the 
ability to use existing certificates of which the private key is available, 
instead of generating forged ones.  SSLproxy supports NULL-prefix CN 
certificates but otherwise does not implement exploits against specific 
certificate verification vulnerabilities in SSL/TLS stacks.

SSLproxy implements a number of defenses against mechanisms which would 
normally prevent MitM attacks or make them more difficult.  SSLproxy can deny 
OCSP requests in a generic way.  For HTTP and HTTPS connections, SSLproxy 
mangles headers to prevent server-instructed public key pinning (HPKP), avoid 
strict transport security restrictions (HSTS), and prevent switching to 
QUIC/SPDY, HTTP/2 or WebSockets (Upgrade, Alternate Protocols).  HTTP
compression, encodings and keep-alive are disabled to make the logs more
readable.

Another reason to disable persistent connections is to reduce file descriptor 
usage. Accordingly, connections are closed if they remain idle for a certain 
period of time. The default timeout is 120 seconds, which can be changed in a 
configuration file.

SSLproxy verifies upstream certificates by default. If the verification fails, the 
connection is terminated immediately. This is in contrast to SSLsplit, because 
in order to maximize the chances that a connection can be successfully split, 
SSLsplit accepts all certificates including self-signed ones. See [The Risks of 
SSL Inspection](https://insights.sei.cmu.edu/cert/2015/03/the-risks-of-ssl-inspection.html)
for the reasons of this difference.

As SSLproxy is based on SSLsplit, this is a modified SSLsplit README file.
See the manual page sslproxy(1) for details on using SSLproxy and setting up
the various NAT engines.


## Requirements

SSLproxy depends on the OpenSSL and libevent 2.x libraries.
The build depends on GNU make and a POSIX.2 environment in `PATH`.
If available, pkg-config is used to locate and configure the dependencies.
The optional unit tests depend on the check library.

SSLproxy currently supports the following operating systems and NAT mechanisms:

-   FreeBSD: pf rdr and divert-to, ipfw fwd, ipfilter rdr
-   OpenBSD: pf rdr-to and divert-to
-   Linux: netfilter REDIRECT and TPROXY
-   Mac OS X: pf rdr and ipfw fwd

Support for local process information (`-i`) is currently available on Mac OS X
and FreeBSD.

SSL/TLS features and compatibility greatly depend on the version of OpenSSL
linked against; for optimal results, use a recent release of OpenSSL proper.
OpenSSL forks like BoringSSL may or may not work.


## Installation

With OpenSSL, libevent 2.x, pkg-config and check available, run:

    make
    make test       # optional unit tests
    make sudotest   # optional unit tests requiring privileges
    make install    # optional install

Dependencies are autoconfigured using pkg-config.  If dependencies are not
picked up and fixing `PKG_CONFIG_PATH` does not help, you can specify their
respective locations manually by setting `OPENSSL_BASE`, `LIBEVENT_BASE` and/or
`CHECK_BASE` to the respective prefixes.

You can override the default install prefix (`/usr/local`) by setting `PREFIX`.
For more build options see `GNUmakefile` and `defaults.h`.


## Documentation

See the manual page `sslproxy.1` for user documentation.
See `NEWS.md` for release notes listing significant changes between releases.


## License

SSLsplit is provided under a 2-clause BSD license.
SSLsplit contains components licensed under the MIT and APSL licenses.
See `LICENSE`, `LICENSE.contrib` and `LICENSE.third` as well as the respective
source file headers for details.
The modifications for SSLproxy are licensed under the same terms as SSLsplit.


## Credits

See `AUTHORS.md` for the list of contributors.

SSLsplit was inspired by `mitm-ssl` by Claes M. Nyberg and `sslsniff` by Moxie
Marlinspike, but shares no source code with them.


