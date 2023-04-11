---
title: RADIUS Version 1.1
abbrev: RADIUSv11
docname: draft-dekok-radext-radiusv11-03
updates: 6613,7360

stand_alone: true
ipr: trust200902
area: Internet
wg: RADEXT Working Group
kw: Internet-Draft
cat: std
submissionType: IETF

pi:
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:

- ins: A. DeKok
  name: Alan DeKok
  org: FreeRADIUS
  email: aland@freeradius.org

normative:
  BCP14: RFC8174
  RFC2865:
  RFC6421:
  RFC6929:
  RFC7301:
  RFC8044:

informative:
  RFC1321:
  RFC2548:
  RFC2866:
  RFC2868:
  RFC3579:
  RFC5176:
  RFC6151:
  RFC6218:
  RFC6613:
  RFC6614:
  RFC7360:
  RFC7585:
  RFC7930:

venue:
  group: RADEXT
  mail: radext@ietf.org
  github: freeradius/radiusv11.git

--- abstract

This document defines Application-Layer Protocol Negotiation Extensions for use with RADIUS/TLS and RADIUS/DTLS.  These extensions permit the negotiation of an additional application protocol for RADIUS over (D)TLS.  No changes are made to RADIUS/UDP or RADIUS/TCP.  The extensions allow the negotiation of a transport profile where the RADIUS shared secret is no longer used, and all MD5-based packet signing and attribute obfuscation methods are removed.  When this extension is used, the previous Authenticator field is repurposed to contain an explicit request / response identifier, called a Token.  The Token also allows more than 256 packets to be outstanding on one connection.

This extension can be seen as a transport profile for RADIUS, as it is not an entirely new protocol.  It uses the existing RADIUS packet layout and attribute format without change.  As such, it can carry all present and future RADIUS attributes.  Implementation of this extension requires only minor changes to the protocol encoder and decoder functionality.  The protocol defined by this extension is named "RADIUS version 1.1", or "RADIUSv11".

--- middle

# Introduction

The RADIUS protocol {{RFC2865}} uses MD5 {{RFC1321}} to sign packets, and to obfuscate certain attributes.  Decades of cryptographic research has shown that MD5 is insecure, and MD5 should no longer be used.  In addition, the dependency on MD5 makes it impossible to use RADIUS in a FIPS-140 compliant system, as FIPS-140 forbids systems from relying on insecure cryptographic methods for security.  There are many prior discussions of MD5 insecurities which we will not repeat here.  These discussions are most notably in {{RFC6151}}, and in Section 3 of {{RFC6421}}, among others.

This document defines an Application-Layer Protocol Negotiation (ALPN) {{RFC7301}} extension for RADIUS which removes the dependency on MD5.  Systems which implement this transport profile are therefore capable of being FIPS-140 compliant.  This extension can best be understood as a transport profile for RADIUS, rather than a whole-sale revision of the RADIUS protocol.  To support this claim, a preliminary implementation of this extension was done in an Open Source RADIUS server.  Formatted as a source code patch, the changes comprised approximately 2,000 lines of code.  This effort shows that the changes to RADIUS are minimal, and are well understood.

The changes from traditional TLS-based transports for RADIUS are as follows:

* ALPN is used for negotiation of this extension,

* TLS 1.3 or later is required,

* all uses of the RADIUS shared secret have been removed,

* The now-unused Request and Response Authenticator fields have been repurposed to carry an opaque Token which identifies requests and responses,

* The Identifier field is no longer used, and has been replaced by the Token field,

* The Message-Authenticator attribute ({{RFC3579}} Section 3.2) is not sent in any packet, and if received is ignored,

* Attributes such as User-Password, Tunnel-Password, and MS-MPPE keys are sent encoded as "text" ({{RFC8044}} Section 3.4) or "octets" ({{RFC8044}} Section 3.5), without the previous MD5-based obfuscation.  This obfuscation is no longer necessary, as the data is secured and kept private through the use of TLS.

* Future RADIUS specifications are forbidden from defining new cryptographic primitives.

The following items are left unchanged from traditional TLS-based transports for RADIUS:

* the RADIUS packet header is the same size, and the Code and Length fields ({{RFC2865}} Section 3) have the same meaning as before,

* All attributes which do not use MD5-based obfuscation methods are encoded using the normal RADIUS methods, and have the same meaning as before,

* As this extension is a transport profile for one "hop" (client to server connection), it does not impact any other connection used by a client or server.  The only systems which are aware that this transport profile is in use are the client and server which have negotiated the use of this extension on a particular shared connection.

* This extension uses the same ports (2083/tcp and 2083/udp) which are defined for RADIUS/TLS {{RFC6614}} and RADIUS/DTLS {{RFC7360}}.

A major benefit of this extensions is that a home server which implements it can also choose to also implement full FIPS-140 compliance,  That is, a home server can remove all uses of MD4 and MD5.  In that case, however, the home server will not support CHAP or MS-CHAP, or any authentication method which uses MD4 or MD5.  We note that the choice of which authentication method to accept is always left to the home server.  This specification does not change any authentication method carried in RADIUS, and does not mandate (or forbid) the use of any authentication method for any system.

As for proxies, there was never a requirement that proxies implement CHAP or MS-CHAP authentication.  So far as a proxy is concerned, attributes relating to CHAP and MS-CHAP are simply opaque data that is transported unchanged to the next hop.  As such, it is possible for a FIPS-140 compliant proxy to transport authentication methods which depend on MD4 or MD5, so long as that data is forwarded to a home server which supports those methods.

We reiterate that the decision to support (or not) any authentication method is entirely site local, and is not a requirement of this specification.  This specification does not not modify the content or meaning of any RADIUS attribute other than Message-Authenticator (and similar attributes), and the only change to the Message-Authenticator attribute is that is no longer used.

Unless otherwise described in this document, all RADIUS requirements apply to this extension.  That is, this specification defines a transport profile for RADIUS.  It is not an entirely new protocol, and it defines only minor changes to the existing RADIUS protocol.  It does not change the RADIUS packet format, attribute format, etc.  This specification is compatible with all RADIUS attributes, past, present, and future.  In short, it simply removes the need to use MD5 when signing packets, or obfuscating certain attributes.

# Terminology

{::boilerplate bcp14}

* ALPN

>> Application-Layer Protocol Negotiation, as defined in {{RFC7301}}

* RADIUS

>> The Remote Authentication Dial-In User Service protocol, as defined in {{RFC2865}}, {{RFC2865}}, and {{RFC5176}} among others.

* RADIUS/UDP

>> RADIUS over the User Datagram Protocol as define above.

* RADIUS/TCP

>> RADIUS over the Transmission Control Protocol {{RFC6613}}

* RADIUS/TLS

>> RADIUS over the Transport Layer Security protocol {{RFC6614}}

* RADIUS/DTLS

>> RADIUS over the Datagram Transport Layer Security protocol  {{RFC7360}}

* RADIUSv11

>> The ALPN protocol extension defined in this document, which stands for "RADIUS version 1.1".  We use RADIUSv11 to refer interchangeably to TLS and DTLS transport.

* TLS

>> the Transport Layer Security protocol.  Generally when we refer to TLS in this document, we are referring interchangeably to TLS or DTLS transport.

# The RADIUSv11 Transport profile for RADIUS

We define an ALPN extension for TLS-based RADIUS transports.   In addition to defining the extension, we also discuss how the encoding of some attributes are changed when this extension is used.  Any field or attribute not mentioned here is unchanged from normal RADIUS.

# Defining and configuring ALPN

This document defines two ALPN protocol names which can be used to negotiate the use (or not) of this specification:

> "radius/1.0"
>
>> Traditional RADIUS/TLS or RADIUS/DTLS.

> "radius/1.1"
>
>> The protocol defined by specification.

Where ALPN is not configured or is not received in a TLS connection, systems supporting ALPN MUST behave as if the ALPN name "radius/1.0" is being used.

Where ALPN is configured, we have the following choices:

> use "radius/1.0" only.
>
>> There is no need in this case to use ALPN, but this configuration is included for completeness

> use "radius/1.1" only
>
>> ALPN is required.

> negotiate either "radius/1.0" or "radius/1.1"
>
>> If one end signals support for both "radius/1.0" and "radius/1.1", and the other end either does not use ALPN, then the system using ALPN MUST behave as if the other end has used the ALPN name "radius/1.0".
>>
>> If one end signals support for both "radius/1.0" and "radius/1.1", and the other end either end signals support only for one ALPN protocol, then the mutually compatible ALPN protocol MUST be used.
>>
>> Where both ends signal support for "radius/1.1", that extension MUST be used.  There is no reason to negotiate a feature and then not use it.

Implementations MUST signal ALPN "radius/1.1" in order for it to be used in a connection.  Implementations MUST NOT have an administrative flag which makes a connection use "radius/1.1" without signalling that protocol via ALPN.

### Configuration of ALPN

Clients or servers supporting this specification can do so by extending their TLS configuration through the addition of a new configuration flag, called "RADIUS-1.1" here.  The exact name given below does not need to be used, but it is RECOMMENDED that administrative interfaces or programming interfaces use a similar name in order to provide consistent terminology.  This flag controls how the implementations signal use of this protocol via ALPN.

Configuration Flag Name

> RADIUS-1.1

Allowed Values

> forbid - Forbid the use of RADIUSv11
>
>> The system MAY signal ALPN via the "radius/1.0" protocol
>> name.  If ALPN is not used, the system MUST use RADIUS/TLS or
>> RADIUS/DTLS as per {{RFC6614}} and {{RFC7360}}.
>>
>> The "radius/1.1" ALPN protocol name MUST NOT be sent by the system.

> allow - Allow (or negotiate) the use of RADIUSv11
>
>> The system MUST use ALPN to signal that both "radius/1.0" and
>> "radius/1.1" can be used.
>
> require -  Require the use of RADIUSv11
>
>> The system MUST use ALPN to signal that only "radius/1.1" is being used.
>>
>> The "radius/1.0" ALPN protocol name MUST NOT be sent by the system.
>>
>> If no ALPN is received, or "radius/1.1" is not received via ALPN,
>> the system MUST log an informative message and close the TLS
>> connection.

Once a system has been configured to support ALPN, it is negotiated on a per-connection basis as per {{RFC7301}}.  The definition of the "radius/1.1" transport profile is given below.

## Negotiation

If the "RADIUS-1.1" flag is set to "allow", then both "radius/1.0" and "radius/1.1" application protocols MUST be signalled via ALPN.  Where a system sees that both ends of a connection support "radius/1.1", then it MUST be used.  There is no reason to negotiate an extension and the refuse to use it.

If a systems supports ALPN and does not receive any ALPN signalling from the other end, it MUST behave as if the other end had sent the ALPN protocol name "radius/1.0".

If a system determines that there are no compatible application protocol names, then it MUST send a TLS alert of "no_application_protocol" (120), which signals the other end that there is no compatible application protocol.  The connection MUST then be torn down.

It is RECOMMENDED that a descriptive error is logged in this situation, so that an administrator can determine why a particular connection failed.  The log message SHOULD include information about the other end of the connection, such as IP address, certificate information, etc.  Similarly, a system receiving a TLS alert of "no_application_protocol" SHOULD log a descriptive error message.  Such error messages are critical for helping administrators to diagnose connectivity issues.

## Additional TLS issues

Implementations of this specification MUST require TLS version 1.3 or later.

Implementations of this specification MUST support TLS-PSK.

## Session Resumption

{{RFC7301}} Section 3.1 states that ALPN is negotiated on each connection, even if session resumption is used:

> When session resumption or session tickets (RFC5077) are used, the previous contents of this extension are irrelevant, and only the values in the new handshake messages are considered.

In order to prevent down-bidding attacks, RADIUS servers which negotiate RADIUSv11 MUST associate that information with the session ticket.  On session resumption, the server MUST advertise only the capability to do "radius/1.1" for that session.  That is, even if the server configuration permits both ALPN methods "radius/1.0"" and "radius/1.1" to be used for new connections, the server MUST NOT permit "radius/1.0" to be used on session resumption where the session had previously negotiated "radius/1.1".

# Definition of RADIUSv11

This section describes the application-layer data which is sent inside of (D)TLS when using the RADIUSv11 protocol.  Unless otherwise discussed herein, the application-layer data is unchanged from traditional RADIUS.  This protocol is only used when "radius/1.1" has been negotiated by both ends of a connection.

## RADIUS/1.1 Packet Format

When RADIUSv11 is used, the RADIUS header is modified from standard RADIUS.  While the header has the same size, some fields have different meaning.  The Identifier and the Request Authenticator and Response Authenticator fields are no longer used.  Any operations which depend on those fields MUST NOT be performed.  As packet signing and security are handled by the TLS layer, RADIUS-specific cryptographic primitives are no longer used.

A summary of the RADIUS/1.1 packet format is shown below.  The fields are transmitted from left to right.

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Code      |  Reserved-1   |            Length             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Token                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                           Reserved-2                          |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Attributes ...
+-+-+-+-+-+-+-+-+-+-+-+-+-
~~~~
{: #Header title="The RADIUS/1.1 Packet Format"}

Code

> The Code field is one octet, and identifies the type of RADIUS packet.
>
> The meaning of the Code field is unchanged from the previous definition in RADIUS.

Reserved-1

> The Reserved-1 field is one octet.  It MUST be set to zero for all packets.
>
> This field was previously called "Identifier" in RADIUS.  It is now unused, as the Token field is now used to identify requests and responses.

Length

> The Length field is two octets.
>
> The meaning of the Length field is unchanged from the previous definition in RADIUS.

Token

> The Token field is four octets, and aids in matching requests and
> replies.  The RADIUS server can detect a duplicate request if it receives
> the same Token value for two packets on a particular connection.
>
> All request / reply tracking MUST be done only on the Token field,
> all other fields in the RADIUS packet header are ignored for the
> purposes of deduplication.  This requirement simplifies implementations.
>
> All packet deduplication MUST be done on a per-connection basis.  If two
> RADIUS packets which are received on different connections contain
> the same Token value, then those packets MUST be treated as distinct
> (i.e. different) packets.
>
> The Token field MUST be different for every unique packet sent over
> a particular connection.  This unique value can be used to
> match responses to requests, and to identify duplicate requests.
> Other than those two requirements, there is no special meaning for
> the Token field.
> 
> Systems generating a Token value for placement in a packet can do so
> via any method they choose.  Systems receiving a Token value in a
> packet MUST NOT interpret its value as anything other than an opaque
> token.

Reserved-2

> The Reserved-2 field is twelve (12) octets in length.
>
> These octets MUST be set to zero when sending a packet.
> 
> These octets MUST be ignored when receiving a packet.
> 
> These octets are reserved for future protocol extensions.

## The Token Field

This section describes in more detail how the Token field is used.

### Sending Packets

A client which sends packets uses the Token field to increase the number of RADIUS packets which can be sent over one connection.

The Token field MUST change for every new unique packet which is sent.  For DTLS transport, it is possible to retransmit duplicate packets, in which case the Token value MUST NOT be changed when a duplicate packet is (re)sent.  When the contents of a retransmitted packet change for any reason (such changing Acct-Delay-Time as discussed in {{RFC2866}} Section 5.2), the Token value MUST be changed.

The Token MUST be different for different packets on the same connection.  If a packet is retransmitted without any changes, then the retransmitted packet MUST use the same token value.  If the two packets have any differences, then the Token values for those two packets is required to be different.  Note that on reliable transports, packets are never retransmitted, and therefore every new packet sent has a unique Token value.

For simplicity, it is RECOMMENDED that the Token values be generated from a 32-bit counter which is unique to each connection.  Such a counter SHOULD be initialized to a random value, taken from a random number generator whenever a new connection is opened.  The counter can then be incremented for every new packet which is sent.  As there is no special meaning for the Token, there is no meaning when a counter "wraps" around from a high value back to zero.

If a RADIUS client has multiple independent subsystems which send packets to a server, each subsystem MAY open a new port which is unique to that subsystem.  There is no requirement that all packets go over one particular connection.  That is, despite the use of a 32-bit Token field, RADIUSv11 clients are still permitted to open multiple source ports as discussed in {{RFC2865}} Section 2.5.

### Receiving Packets

A server which receives RADIUSv11 packets MUST perform packet deduplication for all situations where it is required by RADIUS.  Where RADIUS does not require deduplication (e.g. TLS transport), the server SHOULD NOT do deduplication.

In normal RADIUS, the Identifier field can be the same for different types of packets on the same connection, e.g. for Access-Request and Accounting-Request.  This overlap requires that RADIUS clients and servers, track the Identifier field not only on a per-connection basis, but also on a per-packet type basis.  This behavior adds complexity to implementations.

When using RADIUSv11, implementations MUST instead do deduplication only on the Token field, on a per-connection basis.  A server MUST treat the Token as being an opaque field with no intrinsic meaning.  While the recommendation above is for the sender to use a counter, other implementations are possible, valid, and permitted.  For example, a system could use a pseudo-random number generator with a long period to generate unique values for the Token field.

This change from RADIUS means that the Identifier field is no longer useful.  It is RECOMMENDED that the Identifier field be set to zero for all RADIUSv11 packets.  In order to stay close to RADIUS, we still require that replies MUST use the same Identifier as was seen in the corresponding request.  There is no reason to make major changes to the RADIUS packet header.

# Attribute handling

Most attributes in RADIUS have no special encoding "on the wire", or any special meaning between client and server.  Unless discussed in this section, all RADIUS attributes are unchanged in this specification.  This requirement includes attributes which contain a tag, as defined in {{RFC2868}}.

## Obfuscated Attributes

As (D)TLS is used for this specification, there is no need to hide the contents of an attribute on a hop-by-hop basis.  The TLS transport ensures that all attribute contents are hidden from any observer

Attributes which are obfuscated with MD5 no longer have the obfuscation step applied when RADIUSv11 is used.  Instead, those attributes are simply encoded as their values, as with any other attribute.  Their encoding method MUST follow the encoding for the underlying data type, with any encryption / obfuscation step omitted.

We understand that there is often concern in RADIUS that passwords are sent "in cleartext" across the network.  This allegation was never true for RADIUS, and definitely untrue when (D)TLS transport is used.  While passwords are encoded in packets as strings, the packets (and thus passwords) are protected by TLS.  For the unsure reader this protocol is the same TLS which protects passwords used for web logins, e-mail reception and sending, etc.  As a result, any claims that passwords are sent "in the clear" are false.

There are risks from sending passwords over the network, even when they are protected by TLS.  One such risk somes from the common practice of multi-hop RADIUS routing.  As all security in RADIUS is on a hop-by-hop basis, every proxy which receives a RADIUS packet can see (and modify) all of the information in the packet.  Sites wishing to avoid proxies SHOULD use dynamic peer discovery {{RFC7585}}, which permits clients to make connections directly to authoritative servers for a realm.

These others ways to mitigate these risks .  One is by ensuring that the RADIUS/TLS session parameters are verified before sending the password, usually via a method such as verifying a server certificate.  That is, passwords should only be sent to verified and trusted parties.  If the TLS session parameters are not verified, then it is trivial to convince the RADIUS client to send passwords to anyone.

Another way to mitigate these risks is for the system being authenticated to use an authentication protocol which never sends passwords (e.g. EAP-PWD {{?RFC5931}}), or which sends passwords protected by a TLS tunnel (e.g. EAP-TTLS {{?RFC5281}}).  The processes to choose and configuring an authentication protocol are strongly site-dependent, so further discussion of these issues are outside of the scope of this document.  The goal here is to ensure that the reader has enough information to make an informed decision.

### User-Password

The User-Password attribute ({{RFC2865}} Section 5.2) MUST be encoded the same as any other attribute of data type 'string' ({{RFC8044}} Section 3.4).

The contents of the User-Password field MUST be at least one octet in length, and MUST NOT be more than 128 octets in length.  This limitation is maintained from {{RFC2865}} Section 5.2 for compatibility with legacy transports.

Note that the User-Password attribute is not of data type 'text'.  The original reason in {{RFC2865}} was because the attribute was encoded as an opaque and obfuscated binary blob.  We maintain that data type here, even though the attribute is no longer obfuscated.  The contents of the User-Password attribute do not have to printable text, or UTF-8 data as per the definition of the 'text' data type in {{RFC8044}} Section 3.5.

However, implementations should be aware that passwords are often printable text, and where the passwords are printable text, it can be useful to store and display them as printable text.  Where implementations can process non-printable data in the 'text' data type, they MAY use the data type 'text' for User-Password.

### CHAP-Challenge

{{RFC2865}} Section 5.2 allows for the CHAP challenge to be taken from either the CHAP-Challenge attribute ({{RFC2865}} Section 5.40), or the Request Authenticator field.  Since RADIUSv11 connections no longer use a Request Authenticator field, proxies may receive an Access-Request containing a CHAP-Password attribute ({{RFC2865}} Section 5.2) but without a CHAP-Challenge attribute ({{RFC2865}} Section 5.40).  Proxies which forward that CHAP-Password attribute over a RADIUSv11 connection MUST create a CHAP-Challenge attribute in the proxied packet using the contents from the Request Authenticator.

### Tunnel-Password

The Tunnel-Password attribute ({{RFC2868}} Section 3.5) MUST be encoded the same as any other attribute of data type 'text' which contains a tag, such as Tunnel-Client-Endpoint ({{RFC2868}} Section 3.3).  Since the attribute is no longer obfuscated, there is no need for a Salt field or Data-Length fields as described in {{RFC2868}} Section 3,5, and the textual value of the password can simply be encoded as-is.

### Vendor-Specific Attributes

Any Vendor-Specific attribute which uses similar obfuscation MUST be encoded as per their base data type.  Specifically, the MS-MPPE-Send-Key and MS-MPPE-Recv-Key attributes ({{RFC2548}} Section 2.4) MUST be encoded as any other attribute of data type 'string' ({{RFC8044}} Section 3.5).

We note that as the RADIUS shared secret is no longer used, it is no longer possible or necessary for any attribute to be obfuscated on a hop-by-hop basis using the previous methods defined for RADIUS.

## Message-Authenticator

The Message-Authenticator attribute ({{RFC3579}} Section 3.2) MUST NOT be sent over a RADIUSv11 connection.  That attribute is no longer used or needed.

If the Message-Authenticator attribute is received over a RADIUSv11 connection, the attribute MUST be silently discarded, or treated an as "invalid attribute", as defined in {{RFC6929}}, Section 2.8.  That is, the Message-Authenticator attribute is no longer used to sign packets.  Its existence (or not) in this transport is meaningless.

We note that any packet which contains a Message-Authenticator attribute can still be processed.  There is no need to discard an entire packet simply because it contains a Message-Authenticator attribute.  Only the Message-Authenticator attribute itself is ignored.

## Message-Authentication-Code

Similarly, the Message-Authentication-Code attribute defined in {{RFC6218}} Section 3.3 MUST NOT be sent over a RADIUSv11 connection.  That attribute MUST be treated the same as Message-Authenticator, above.

As the Message-Authentication-Code attribute is no longer used, the related MAC-Randomizer attribute {{RFC6218}} Section 3.2 is also no longer used.  It MUST also be treated the same was as Message-Authenticator, above.

## CHAP, MS-CHAP, etc.

While some attributes such as CHAP-Password, etc. depend on insecure cryptographic primitives such as MD5, these attributes are treated as opaque blobs when sent between a RADIUS client and server.  The contents of the attributes are not obfuscated, and they do not depend on the RADIUS shared secret.

As a result, these attributes are unchanged in RADIUSv11.  We reiterate that this specification is largely a transport profile for RADIUS.  Other than a few attributes such as Message-Authenticator, the meaning of all attributes in this specification is identical to their meaning in RADIUS.  Only the "on the wire" encoding of some attributes change, and then only for attributes which are obfuscated using the RADIUS shared secret.  Those obfuscated attributes are now protected by the modern cryptography in TLS, instead of an "ad hoc" approach using MD5.

A server implementing this specification can proxy CHAP, MS-CHAP, etc. without any issue.  A home server implementing this specification can authenticate CHAP, MS-CHAP, etc. without any issue.

## Original-Packet-Code

The Original-Packet-Code attribute ({{RFC7930}} Section 4) MUST NOT be sent over a RADIUSv11 connection.  That attribute is no longer used or needed.

If the Original-Packet-Code attribute is received over a RADIUSv11 connection, the attribute MUST either be silently discarded, or be treated an as "invalid attribute", as defined in {{RFC6929}}, Section 2.8.  That is, existence of the Token field means that the Original-Packet-Code attribute is no longer needed to correlate Protocol-Error replies with outstanding requests.  As such, the Original-Packet-Code attribute is not used in RADIUSv11.

We note that any packet which contains an Original-Packet-Code attribute can still be processed.  There is no need to discard an entire packet simply because it contains an Original-Packet-Code attribute.

# Other Considerations

Most of the differences between normal RADIUS and RADIUSv11 are in the packet header and attribute handling, as discussed above.  The remaining issues are a small set of unrelated topics, and are discussed here.

## Status-Server

{{RFC6613}} Section 2.6.5, and by extension {{RFC7360}} suggest that the Identifier value zero (0) be reserved for use with Status-Server as an application-layer watchdog.  This practice MUST NOT be used for RADIUSv11, as the Identifier field is no longer used.

The rational for reserving one value of the Identifier field was the limited number of Identifiers available (256), and the overlap in Identifiers between Access-Request packets and Status-Server packers..  If all 256 Identifier values had been used to send Access-Request packets, there would be no Identifier value available for sending a Status-Sercer Packet.

In contrast, the Token field allows for 2^32 outstanding packets on one RADIUSv11 connection.  If there is a need to send a Status-Server packet, it is always possible to allocate a new value for the Token field.  Similarly, the value zero (0) for the Token field has no special meaning.  The edge condition is that there are 2^32 outstanding packets on one connection with no new Token value available for Status-Server.  In which case there are other issues which are more serious, such allowing billions of packets to be oustanding.  The safest way forward is likely to just close the connection.

## Proxies

A RADIUS proxy normally decodes and then re-encodes all attributes, included obfuscated ones.  A RADIUS proxy will not generally rewrite the content of the attributes it proxies (unless site-local policy requires such a rewrite).  While some attributes may be modified due to administrative or policy rules on the proxy, the proxy will generally not rewrite the contents of attributes such as User-Password, Tunnel-Password, CHAP-Password, MS-CHAP-Password, MS-MPPE keys, etc.  All attributes are therefore transported through a RADIUSv11 connection without changing their values or contents.

A proxy may negotiate RADIUSv11 (or not) with a particular client or clients, and it may negotiate RADIUSv11 (or not) with a server or servers it connect to, in any combination.  As a result, this specification is fully compatible with all past, present, and future RADIUS attributes.

# Crypto-Agility

The crypto-agility requirements of {{RFC6421}} are addressed in {{RFC6614}} Appendix C, and in Section 10.1 of {{RFC7360}}.  This specification makes no changes from, or additions to, those specifications.

All crypto-agility needed or use by this specification is implemented in TLS.  This specification also removes all cryptographic primitives from the application-layer protocol (RADIUS) being transported by TLS.  As discussed in the next section below, this specification also bans the development of all new cryptographic or crypto-agility methods in the RADIUS protocol.

## Future Standards

This specification defines a new transport profile for RADIUS.  It does not define a completely new protocol.  As such, any future attribute definitions MUST first be defined for RADIUS/UDP, after which those definitions can be applied to this transport profile.

New specifications MAY define new attributes which use the obfuscation methods for User-Password as defined in {{RFC2865}} Section 5.2, or for Tunnel-Password as defined in {{RFC2868}} Section 3.5.  There is no need for those specifications to describe how those new attributes are transported in RADIUSv11.  Since RADIUSv11 does not use MD5, any obfuscated attributes will by definition be transported as their underlying data type ("text" ({{RFC8044}} Section 3.4) or "octets" ({{RFC8044}} Section 3.5).

New RADIUS specifications MUST NOT define attributes which can only be transported over RADIUS/TLS.  The RADIUS protocol has no way to signal the security requirements of individual attributes.  Any existing implementation will handle these new attributes as "Invalid Attributes" ({{RFC6929}} Section 2.8), and could forward them over an insecure link.  As RADIUS security and signalling is hop-by-hop, there is no way for a RADIUS/TLS client or server to even know if such forwarding is taking place.  For these reasons and more, it is therefore inappropriate to define new attributes which are only secure if they use a secure transport layer.

This document updates {{RFC2865}} at al. to state that any new RADIUS specification MUST NOT introduce new "ad hoc" cryptographic primitives as was done with User-Password and Tunnel-Password.  That is, RADIUS-specific cryptographic methods existing as of the publication of this document can continue to be used for historical compatibility.  However, all new cryptographic work in RADIUS is forbidden.  There is insufficient expertise in the RADIUS community to securely design new cryptography.

# Implementation Status

(This section to be removed by the RFC editor.)

This specification is being implemented (client and server) in the FreeRADIUS project which is hosted on GitHub.  URL TBD.  The code implementation "diff" is approximately 2,000 lines, including build system changes and changes to configuration parsers.

# Privacy Considerations

This specification requires secure transport for RADIUS, and this has all of the privacy benefits of RADIUS/TLS {{RFC6614}} and RADIUS/DTLS {{RFC7360}}.  All of the insecure uses of RADIUS have been removed.

# Security Considerations

The primary focus of this document is addressing security considerations for RADIUS.

# IANA Considerations

IANA is request to update the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry with two new entries:

~~~~
Protocol: radius/1.0
Id. Sequence: 0x72 0x61 0x64 0x69 0x75 0x73 0x2f 0x31 0x2e 0x30
    ("radius/1.0")
Reference: This document

Protocol: radius/1.1
Id. Sequence: 0x72 0x61 0x64 0x69 0x75 0x73 0x2f 0x31 0x2e 0x31
    ("radius/1.1")
Reference:  This document
~~~~

# Acknowledgements

In hindsight, the decision to retain MD5 for RADIUS/TLS was likely wrong.  It was an easy decision to make in the short term, but it has caused ongoing problems which this document addresses.

Thanks to Bernard Aboba, Karri Huhtanen, Heikki Vatiainen, and Alexander Clouter for reviews and feedback.

# Changelog

draft-dekok-radext-sradius-00

> Initial Revision

draft-dekok-radext-radiusv11-00

> Use ALPN from RFC 7301, instead of defining a new port.  Drop the name "SRADIUS".
>
> Add discussion of Original-Packet-Code

draft-dekok-radext-radiusv11-01

> Update formatting.

draft-dekok-radext-radiusv11-02

> Add Flag field and description.
>
> Minor rearrangements and updates to text.

draft-dekok-radext-radiusv11-03

> Remove Flag field and description based on feedback and expected use-cases.
>
> Use "radius/1.0" instead of "radius/1"
>
> Consistently refer to the specification as "RADIUSv11", and consistently quote the ALPN name as "radius/1.1"
>
> Add discussion of future attributes and future crypto-agility work.

--- back
