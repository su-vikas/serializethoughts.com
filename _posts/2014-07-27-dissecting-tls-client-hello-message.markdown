---
layout: post
title:  "Dissecting TLS Client Hello Message"
date:   2014-07-27 16:17:22 +0800
---

In the [previous post](https://serializethoughts.wordpress.com/2014/06/15/tls-session-establishment/), I discussed about how TLS session is established. In the course, I also introduced to various sub-protocols involved in TLS protocol. In this post, I will look into various parameters of *Client Hellow* message. But before get going, I will lay down some basic blocks and talk about TLS Record Protocol and TLS Handshake Protocol. *Client Hello* message is part of TLS Handshake Protocol.

One thing to always keep in mind is during a TLS session negotiation all the data exchanged is unencrypted and goes in plain text. Only post this negotiation stage, the data exchanged is encrypted.  During session negotiation, the messages exchanged can be intercepted by an eavesdropper and derive information which can help in identifying the user or the domain he/she is visiting. Such attacks are mentioned in this post. Another such resource I came across is [this one](https://idea.popcount.org/2012-06-16-dissecting-ssl-handshake/).

So lets get going by delving into details of TLS Record Protocol. While further reading always remember the block diagram below and ever in confusion revert to this diagram. The crux being, TLS Record Protocol is an envelope protocol. TLS Handshake Protocol, Change Cipher Spec Protocol and Alert Protocol are 'letter' of this envelope.

![TLS's sub-protocols block representation](/assets/images/tls_protocol_layers.gif)

# TLS Record Protocol

TLS Record Protocol is a layered protocol. At each layer, messages may include fields for length, description, and content. The Record Protocol takes messages to be transmitted, fragments the data into manageable blocks, optionally compresses the data, applies a MAC, encrypts, and transmits the result.

In [RFC 5246](http://tools.ietf.org/html/rfc5246#page-19) Record Layer message is defined as following :

```C
struct{
    ContentType type;
    ProtocolVersion version;
    uint16 length;
    opaque fragment[TLSPlaintext.length];
}TLSPlaintext;
```

Lets try to understand each parameter one at a time:

- **type**: Record Layer Protocol is envelope to other TLS sub-protocols, Handshake Protocol, Change Cipher Spec Protocol, Alert Protocol and Application Data Protocol. This 'type' variable tells which sub-protocol is encapsulated in this record layer message.

- **version**: The TLS version being used. For instance, if TLS 1.1 is used, then version will be {3,2}, deriving from the use of {3,1} for TLS 1.0. Note that a client that supports multiple versions of TLS may not know what version will be employed before it receives the server_hello message.

- **length**: The length of the TLSPlaintext.fragment in bytes. The maximum length allowed is 2^14 bytes. In simple words, length of the 'letter' of this envelope message.

- **fragment**: Fragment is the application data. This data is transparent and treated as an independent block to be dealt with by the higher-level protocol specified by the type field.

A simple byte-by-byte representation of record layer message is following:

```
Byte 0 = SSL record type
Bytes 1-2 = SSL version (major/minor)
Bytes 3-4 = Length of data in the record (excluding the header itself). The maximum SSL supports is 16384 (16K).

Byte 0 can have following values:
SSL3_RT_CHANGE_CIPHER_SPEC 20 (x'14')
SSL3_RT_ALERT 21 (x'15')
SSL3_RT_HANDSHAKE 22 (x'16')
SSL3_RT_APPLICATION_DATA 23 (x'17')

Bytes 1-2 in the record have the following version values:
SSL3_VERSION x'0300'
TLS1_VERSION x'0301'
TLS2_VERSION x'0302'
TLS3_VERSION x'0303'
```

# TLS Handshake Protocol

TLS handshake protocol runs on top of TLS Record Protocol. "This protocol is used to negotiate the secure attributes of a session. Handshake messages are supplied to the TLS record layer, where they are encapsulated within one or more TLSPlaintext structures, which are processes and transmitted as specified by the current active session state."

Following handshake types are supported by this protocol:

1. hello_request
2. client_hello
3. server_hello
4. certificate
5. server_key_exchange
6. certificate_request
7. server_hello_done
8. certificate_verify
9. client_key_exchange
10. finished

A typical TLS Handshake Protocol should be of following [format](http://tools.ietf.org/html/rfc5246#page-37):

```C
 struct {
          HandshakeType msg_type;    /* handshake type */
          uint24 length;             /* bytes in message */
          select (HandshakeType) {
              case hello_request:       HelloRequest;
              case client_hello:        ClientHello;
              case server_hello:        ServerHello;
              case certificate:         Certificate;
              case server_key_exchange: ServerKeyExchange;
              case certificate_request: CertificateRequest;
              case server_hello_done:   ServerHelloDone;
              case certificate_verify:  CertificateVerify;
              case client_key_exchange: ClientKeyExchange;
              case finished:            Finished;
          } body;
      } Handshake;
```

- **msg_type**: This parameter specifies the type of TLS Handshake protocol message enclosed.

- **length**: Length parameter tells about the length of message following in the body.

With all the ground work layed, now we move towards the topic of this post, dissecting a *client_hello* message.

# Client Hello Message

As evident from above, *client_hello* message is a type of TLS Handshake Protocol. Structure for *client_hello* message is defined as following:

```C
struct{
    ProtocolVersion client_version;
    Random random;
    SessionID session_id;
    CipherSuite cipher_suites<2..2^16-2>;
    CompressionMethod compression_method<1..2^8-1>;
    select (extensions_present){
        case false:
            struct{};
        case true:
            Extension extensions<0..2^16-1>;
    };
}ClientHello;
```

Each parameter in layman terms:

- **client_version**: The version of TLS protocol using which client wishes to communicate during this session.

- **random**: It is client generated random structure, comprising of 4 bytes of time since epoch on client and 28 random bytes. Exposing timer sources may allow [clock skew measurements](http://www.cl.cam.ac.uk/~sjm217/papers/usenix08clockskew.pdf) and those in theory may be used to identify hosts.

```C
 struct{
     uint32 gmt_unix_time;
     opaque random_bytes[28];
 }Random;
 ```

- **session_id**: This field is empty if no session_id is available. In order to avoid the full process of establishing a TLS connection, the client may reuse previously established session. By providing the session_id the client intents to resume the session with previously negotiated parameters. The session cache is shared between normal and privacy modes of the browser and thus user might be identifiable due to TLS session reuse.

- **cipher_suites**: Contains the list of cryptographic options supported by the client and ordered in client's preference first. Some ciphersuites are proven to be insecure and advised against their use.

- **compression_methods**: Contains the list of compression methods supported by the client, and sorted by client preference. Compressing data does save bandwidth.
- **extensions**: TLS protocol provides the ability to extend the functionality. Notably some most used are Server Name Indication (SNI),Heartbeat etc. In case of SNI, the remote domain name goes in plaintext and thus, anyone sniffing the traffic can exactly determine which all domains you visited.

A wireshark capture showing various parameters sent for a client hello are shown in the screenshot below.

![Wireshark capture showing client hello message](/assets/images/client_hello_google.jpg)

Keep hacking :)

# References

1. [https://idea.popcount.org/2012-06-16-dissecting-ssl-handshake/](http://www.cl.cam.ac.uk/~sjm217/papers/usenix08clockskew.pdf)
2. [http://tools.ietf.org/html/rfc5246#page-37](http://www.cl.cam.ac.uk/~sjm217/papers/usenix08clockskew.pdf)
