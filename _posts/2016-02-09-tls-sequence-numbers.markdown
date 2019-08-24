---
layout: post
title:  "TLS Sequence Numbers"
date:   2016-02-09 16:17:22 +0800
---

When talking about SSL/TLS most of the discussion centers around the ciphersuites, the types of messages or other complex cryptographical aspects. But there are many subtle things embedded in the protocol, which are often skipped or not discussed generally. One such thing is sequence numbers. Like in TCP, a sequence number for messages is also maintained in SSL/TLS protocol and one gets to know only is he/she delve into the RFCs.

In case of SSL/TLS, sequence number is a simple count of messages sent and received. This is maintained implicitly i.e, not sent in the messages explicitly. The protocol requires to maintain a separate sequence number counter for read and write sessions respectively.

A touch of history,  sequence numbers were not used in the SSLv1, and were introduced in SSLv2 only.  Thus making SSLv1 prone to replay attacks (against which sequence numbers protect).

The question arises, if sequence number for a connection is maintained, and given that it is not explicitly transmitted, then how it is useful? To answer, sequence numbers are used in the MAC. To prevent message replay or modification attacks, the MAC is computed using the MAC secret, the sequence number, the message length, the message contents, and two fixed character strings. When either side calculate the MAC for a given message, if sequence number does not correspond to the current message, then message authentication will fail, and the receiver will demand the sender to re-send the message.

From RFC 6101 states following about how sequence number should be calculated and also what data type should be used. Note that by using int64, chances of overflow are minimized.

```
    Each party maintains separate sequence numbers for transmitted and received messages for each connection.  When a party sends or receives a change cipher spec message, the appropriate sequence number is set to zero.  Sequence numbers are of type uint64 and may not exceed 2^64-1.
    ```

To summarize, the sequence number provides protection against attempts to delete or reorder messages.

## References:
[1] https://tools.ietf.org/html/rfc6101#page-14
[2] https://security.stackexchange.com/questions/55667/tls-sequence-number#
[3] SSL and TLS Theory and practice by Rolf Oppliger
