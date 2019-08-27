---
layout: post
title:  "Linux Thread is a Standard Process"
date:   2014-06-10 16:17:22 +0800
---

In few upcoming posts I will be talking about the how the SSL/TLS works. In this article I will cover very basic stuff about SSL. Though [Wikipedia](http://en.wikipedia.org/wiki/Secure_Sockets_Layer<Paste>) might be a better resource for this kind of information, but this is very condensed information and also serves as my personal notes :).

Secure Socket Layer or Transport Layer Security Protocol is the backbone of Internet's security. It provides *authentication, confidentiality and data integrity* (CIA of security). The 1st version of SSL was designed by [Netscape](http://en.wikipedia.org/wiki/Netscape) in 1994 in order to make the client-server communication secure. SSLv1 never saw light of day and was very soon upgraded by SSLv2. SSLv2 was incorporated in Netscape's [Navigator](http://en.wikipedia.org/wiki/Netscape_Navigator) browser. SSL is client-server protocol, i.e, it provides a secure channel of communication between a client and a server. It is a OS agnostic protocol and hence secure communication is not platform dependent. SSL was designed to work very similar to [Berkeley Socket](http://en.wikipedia.org/wiki/Berkeley_sockets) so that applications that were intially designed can be easily ported to use this new protocol.

Over the course of last 20 years several iterations of SSL have been released. Post SSLv3, SSL was renamed as TLS and further versions as TLS 1.0, 1.1 and 1.2. The abbreviations SSL and TLS are used interchangeably, though using TLS is a bit more correct.

In [OSI reference model](http://en.wikipedia.org/wiki/OSI_model), TLS sits between application and transport layer. TLS session initiation is at Layer 5 (session layer) and while it works at Layer 6 (presentation layer). Since, TLS security protocol is sandwiched between the application protocol layer and transport protocol layer, it can secure and send application data to the transport layer. Also, this ensures that TLS can support multiple application layer protocols.

It is a **single hop protocol**, i.e, it provides security between a single hop of client-server. For every message sent from client, the secure channel will end at the server. If the server needs to forward this data further to another server, it needs to negotiate another TLS session.

TLS/SSL assumes that a connection-oriented transport protocol, typically TCP, is in use. The protocol allows client/server applications to detect the following security risks, which are in line with the CIA of security:
- Message tampering
- Message interception
- Message Forgery

Talking about CIA of security, in [X.509 certificates](http://en.wikipedia.org/wiki/X.509), i.e, asymmetric cryptography, is used for authentication and symmetric cryptography is used for confidentiality of the communication data. [Message Authentication Code](http://en.wikipedia.org/wiki/Message_authentication_code) (MAC) is used to ensure message integrity.


This sums up the basic introductory information about TLS. Upcoming posts will talk about how a TLS session is established between two peers.
