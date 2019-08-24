---
layout: post
title:  "What is POODLE and How to Detect It"
date:   2014-10-19 16:17:22 +0800
---

SSL was again center attention recently. After series of vulnerabilities, Heartbleed and OpenSSL CCS to a count a few, another vulnerability rocked one of the most important communication protocol of the internet. **POODLE** - *Padding Oracle On Downgraded Legacy Encryption*, affects SSLv3 (RFC6101) protocol and assigned [CVE-2014-3566](https://access.redhat.com/security/cve/CVE-2014-3566). SSL 3.0 is an old protocol, proposed by Netscape in 1996 and now have been succeeded by new protocols, TLS 1.0, TLS 1.1 and TLS 1.2. SSL 3.0 is obsolete now and only supported because many legacy systems still use it. For example, some old embedded devices still use it and it is not very straightforward to update such devices. But after surfacing of POODLE attack, it is highly recommended to disable SSL 3.0 for any form of communication. After the demise of IE 6, all the modern browsers support TLS 1.0 and thus it is safe and logical to deprecate this old protocol.

POODLE hogged the most of the limelight, but in the [OpenSSL advisory](https://www.openssl.org/news/secadv_20141015.txt) there were some more bugs reported, which should also be taken care by the administrators and the users. In total there are 4 bugs disclosed, other than poodle, rest 3 relates to implementation issues in OpenSSL. It is highly recommended to update your OpenSSL library version if you are using it.

# POODLE
SSL is a very big and complex protocol. I tried to cover the basics of SSL/TLS protocol in the [previous posts](https://serializethoughts.wordpress.com/2014/06/14/introduction-to-secure-socket-layer/). Most of the complexities lies in the initial *handshake phase*, where various protocol parameters like protocol version, ciphersuite, compression etc are negotiated. POODLE attack is concerned with the next phase of the SSL protocol, the *bulk encryption phase*, where the communication end points have already established the SSL connection and ready to exchange application's data, e.g HTTP request and responses. POODLE is the latest in a long line of similar attacks against known weaknesses in SSL's use of *Cipher block chaining* (CBC) mode for block ciphers used in encryption. It is important to note that POODLE is more of a design issue than an implementation bug, unlike previous bugs like [Heartbleed](http://heartbleed.com/) and [OpenSSL CCS](https://serializethoughts.wordpress.com/2014/06/09/detecting-ccs-injection-vulnerability/) vulnerability. Some excellent explanation about the working and explanation of the attacks is available at following places:

- The [security advisory](https://www.openssl.org/~bodo/ssl-poodle.pdf) by the authors of the attack, Bodo Moller, Thai Duong and Krzysztof Kotowicz.
- [Thomas Pornin explaining](https://security.stackexchange.com/questions/70719/ssl3-poodle-vulnerability) very succinctly at security stackexchange.
-  A detailed [explanation](https://www.dfranke.us/posts/2014-10-14-how-poodle-happened.html) of the vulnerability and its history .
- [This](https://security.stackexchange.com/questions/70719/ssl3-poodle-vulnerability%20) and [this](https://security.stackexchange.com/questions/70719/ssl3-poodle-vulnerability) contains information about configuration changes in various servers and browsers to be taken care of in order to protect against POODLE.

# The Workaround
There are couple of workarounds suggested by experts. One recommended by most is to outright **totally disable SSLv3**. As argued, it is a very old protocol and now most modern devices support its successor, thus it makes sense to do so. But if this is not an option, then using **TLS_FALLBACK_SCSV** proposal by Google's Adam Langley is another option.

SCSV stands for "signaling cipher suite value", and its essentially a hack which allows TLS clients to indicate to servers that they support some extensions to TLS, while ensuring that servers that don't understand the extension will simply ignore. SCSV works by appending a special, bogus value to the list of ciphersuites that the client advertises support for. The working of this proposal is a IETF draft and not standardised yet, but many implementation are supporting it
now.

The working of SCSV is very simple, but tend to confuse in the first shot. In case of an attack via downgrading the SSL version, TLS_FALLBACK_SCSV indicates that the client is knowingly repeating  a SSL/TLS connection attempt over a lower protocol version than its supports, as the last one failed for some reason.  When the server sees this TLS_FALLBACK_SCSV in ciphersuite list, it compares the highest protocol version it supports to the version indicated in the Client Hello. If the client's version is lower, then the server responds with a new Alert defined by the RFC called 'inappropriate_fallback' (code 86). The logic being that the server knows the client supports something better so the connection should have negotiated that. The 'inappropriate_fallback' Alert is a "fatal" error and the connection attempt should be aborted. In practice, this protects from all forced downgrades attack completely. Unfortunately, this is not the silver bullet to the problem. In case the client-server negotiate SSLv3 protocol then this attack can be carried out. To further read about TLS_FALLBACK_SCSV, you can visit [here](http://www.exploresecurity.com/poodle-and-the-tls_fallback_scsv-remedy/) and [RFC draft](http://tools.ietf.org/html/draft-ietf-tls-downgrade-scsv-00).

# How To Detect
The detection is also straightforward. Send the TLS_FALLBACK_SCSV {0x56,0x00} in the list of supported ciphersuites in the Client Hello, if the server responds with an alert message *inappropriate_fallback*, then the server is not vulnerable to downgrade attack. Additionally, it is best if the server does not support SSLv3 outright.

Keep hacking!! :D


<Paste>
