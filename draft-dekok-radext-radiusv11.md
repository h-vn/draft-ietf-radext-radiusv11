---
title: RADIUS/TLS Version 1.1
abbrev: RADIUSv11
docname: draft-dekok-radext-radiusv11-01
updates: RFC6613,RFC7360

stand_alone: true
ipr: trust200902
area: Internet
wg: RADEXT Working Group
kw: Internet-Draft
cat: std
submissionType: IETF

pi:    # can use array (if all yes) or hash here
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
  RFC7930:

venue:
  group: RADEXT
  mail: radext@ietf.org
  github: freeradius/radiusv11.git

--- abstract

This document defines Application-Layer Protocol Negotiation Extensions for use with RADIUS/TLS and RADIUS/DTLS.  These extensions permit the negotiation of an additional application protocol for RADIUS over (D)TLS.  No changes are made to RADIUS/UDP or RADIUS/TCP.  The extensions allow the negotiation of a transport profile where the RADIUS shared secret is no longer used, and all MD5-based packet signing and attribute obfuscation methods are removed.  When this extension is used, the previous Authenticator field is repurposed to contain an explicit request / response identifier, called a Token.  The Token also allows more than 256 packets to be outstanding on one connection.

This extension can be seen as a transport profile for RADIUS, as it is not an entirely new protocol.  It uses the existing RADIUS packet layout and attribute format without change.  As such, it can carry all present and future RADIUS attributes.  Implementation of this extension requires only minor changes to the protocol encoder and decoder functionality.  The protocol defined by this extension is named "RADIUS version 1.1", or "radius/1.1".

--- middle

# Introduction

The RADIUS protocol {{RFC2865}} uses MD5 {{RFC1321}} to sign packets, and to obfuscate certain attributes.  Current cryptographic research shows that MD5 is insecure, and recommends that MD5 should no longer be used.  In addition, the dependency on MD5 makes it impossible to use RADIUS in a FIPS-140 compliant system.  There are many prior discussions of MD5 insecurities which we will not repeat here.  These discussions are most notably in [RFC6151], and in Section 3 of {{RFC6421}}, among many others.

This document defines an Application-Layer Protocol Negotiation (ALPN) {{RFC7301}} extension for RADIUS which removes the dependency on MD5.  Systems which implement this transport profile are therefore capable of being FIPS-140 compliant.  This extension can best be understood as a transport profile for RADIUS, rather than a whole-sale revision of the RADIUS protocol.  To support this claim, a preliminary implementation of this extension was done in an Open Source RADIUS server was done.  Formatted as a "patch", the code changes comprised approximately 2,000 lines of code.

The changes from traditional TLS-based transports for RADIUS are as follows:

* ALPN is used for negotiation of this extension,

* TLS 1.3 or later is required,

* all uses of the RADIUS shared secret have been removed,

* The now-unused Request and Response Authenticator fields have been repurposed to carry an opaque Token which identifies requests and responses,

* The Message-Authenticator attribute ({{RFC3579}} Section 3.2) is not sent in any packet, and if received is ignored,

* Attributes such as User-Password, Tunnel-Password, and MS-MPPE keys are sent encoded as "text" ({{RFC8044}} Section 3.4) or "octets" ({{RFC8044}} Section 3.5), without the previous MD5-based obfuscation.  This obfuscation is no longer necessary, as the data is secured and kept private through the use of TLS.

The following items are left unchanged from traditional TLS-based transports for RADIUS:

* the RADIUS packet header is the same size, and the Code, Identifier, and Length fields ({{RFC2865}} Section 3) all have the same meaning as before

* All attributes which do not use MD5-based obfuscation methods are encoded using the normal RADIUS methods, and have the same meaning as before,

* As this extension is a transport profile for one "hop" (client to server connection), it does not impact any other connection used by a client or server.  The only systems which are aware that this transport profile is in use are the client and server which negotiate the extension for a connection that they share,

* This extension uses the same ports (2083/tcp and 2083/udp) as are defined for RADIUS/TLS {{RFC6614}} and RADIUS/DTLS {{RFC7360}}.

A major benefit of this extenions is that a home server which implements it can also choose to also implement full FIPS-140 compliance,  That is, a home server can remove all uses of MD4 and MD5.  In that case, however, the home server will not support CHAP or MS-CHAP, or any authentication method which uses MD4 or MD5.  We note that the choice of which authentication method to accept is always left to the home server.  This specification does not change any authentication method carried in RADIUS, and does not mandate (or forbid) the use of any authentication method for any system.

As for proxies, there was never a requirement that proxies implement CHAP or MS-CHAP authentication.  So far as a proxy is concerned, attributes relating to CHAP and MS-CHAP are simply opaque data that is transported unchanged to the next hop.  As such, it is possible for a FIPS-140 compliant proxy to transport authentication methods which depend on MD4 or MD5, so long as that data is forwarded to a home server which supports those methods.

We reiterate that the decision to support (or not) any authentication method is entirely site local, and is not a requirement of this specification.  This specification does not not modify the content or meaning of any RADIUS attribute other than Message-Authenticator (and similar attributes), and the only change to the Message-Authenticator attribute is that is no longer used.

Unless otherwise described in this document, all RADIUS requirements apply to this extension.  That is, this specification defines a transport profile for RADIUS.  It is not a new protocol, and it defines only minor changes to the RADIUS protocol.  It does not change the RADIUS packet format, attribute format, etc.  This specification is compatible with all RADIUS attributes, past, present, and future.  In short, it simply removes the need to use MD5 when signing packets, or obfuscating certain attributes.

# Terminology

{::boilerplate bcp14}

* ALPN

> Application-Layer Protocol Negotiation, as defined in {{RFC7301}}

* RADIUS

> The Remote Authentication Dial-In User Service protocol, as defined in {{RFC2865}}, {{RFC2865}}, and {{RFC5176}} among others.

* RADIUS/UDP

> RADIUS over the User Datagram Protocol as define above.

* RADIUS/TCP

> RADIUS over the Transmission Control Protocol {{RFC6613}}

* RADIUS/TLS

> RADIUS over the Transport Layer Security protocol {{RFC6614}}

* RADIUS/DTLS

> RADIUS over the Datagram Transport Layer Security protocol  {{RFC7360}}

* RADIUSv11

> The ALPN protocol extenion defined in this document, which stands for "RADIUS version 1.1".  We use RADIUSv11 to refer interchangeably to TLS and for DTLS transport.

* TLS

> the Transport Layer Security protocol.  Generally when we refer to TLS in this document, we are referring to TLS or DTLS transport.

# The RADIUSv11 Transport profile for RADIUS

We define an ALPN extension for TLS-based RADIUS transports.   In addition to defining the extension, we also discuss how the encoding of some attributes are changed when this extension is used.  Any field or attribute not mentioned here is unchanged from RADIUS.

# Defining and configuring ALPN

This document defines two ALPN protocol names which can be used to negotiate the use (or not) of this specification:

* radius/1

> Traditional RADIUS/TLS or RADIUS/DTLS.

* radius/1.1

> This protocol defined by specification.

Where ALPN is not configured or received in a TLS connection, systems supporting ALPN MUST assume that "radius/1" is being used.

Where ALPN is configured, we have the following choices:

* use radius/1 only.

> There is no need in this case to use ALPN, but this configuration is included for completeness

* use radius/1.1 only

> ALPN is required, and this configuration must be explicitly enabled by an administrator

* negotiate either radius/1 or radius/1.1

> Where both ends support radius/1.1, that extension MUST be used.  There is no reason to negotiate a feature and then not use it.

### Configuration of ALPN

Clients or servers supporting this specification do so by extending their TLS configuration through the addition of a new configuration flag, called "radius/1.1" here.  The exact name given below does not need to be used, but it is RECOMMENDED to use similar names in user interfaces, in order to provide consistent terminology for administrators.  This flag controls the use (or not) of the application-layer protocol defined by this specification.

Configuration Flag Name

> radius/1.1

Allowed Values

> forbid
>
>> Meaning
>>
>> Forbid the use of this specification.
>>
>> The system MAY signal ALPN via using only the "radius/1" protocol
>> name.  If ALPN is not used, the system MUST use RADIUS/TLS or
>> RADIUS/DTLS as per prior specifications.

> allow
>
>> Meaning
>>
>> Allow (or negotiate) the use of this specification.
>>
>> The system MUST use ALPN to signal that both "radius/1" and
>> "radius/1.1" can be used.
>
> require
>
>> Meaning
>>
>> Require the use of this specification.
>>
>> The system MUST use ALPN to signal that "radius/1.1" is being used.
>>
>> The "radius/1" ALPN protocol name MUST NOT be sent by the system.
>>
>> If no ALPN is received, or "radius/1.1" is not received via ALPN,
>> the system MUST log an informative message and close the TLS
>> connection.

Once a system has been configured to support ALPN, it is negotiated on a per-connection basis as per {{RFC7301}}.  The definition of the "radius/1.1" transport profile is given below.

## Negotiation

If the "radius/1.1" flag is set to "allow", then both "radius/1" and "radius/1.1" application protocols MUST be signalled via ALPN.  Where a system sees that both ends of a connection support radius/1.1, then it MUST be used.  There is no reason to negotiate an extension and the refuse to use it.

If a systems supports ALPN and does not receive any ALPN signalling from the other end, it MUST behave as if the other end had sent the ALPN protocol name "radius/1".

If a system determines that there are no compatible application protocol names, then it MUST send a TLS alert of "no_application_protocol" (120), which signals the other end that there is no compatible application protocol.  The connection MUST then be torn down.

It is RECOMMENDED that a descriptive error is logged in this situation, so that an administrator can determine why a particular connection failed.  The log message SHOULD include information about the other end of the connection, such as IP address, certificate information, etc.  Similarly, a system receiving a TLS alert of "no_application_protocol" SHOULD log a descriptive error message.  Such error messages are critical for helping administrators to diagnose connectivity issues.

## Additional TLS issues

Implementations of this specification MUST require TLS version 1.3 or later.

Implementations of this specification MUST support TLS-PSK.

Any PSK used for TLS MUST NOT be used for attribute obfuscation in place of the RADIUS shared secret.  Any PSK used for TLS MUST NOT be the same as a RADIUS shared secret which is used for RADIUS/UDP or RADIUS/TLS.

# Definition of radius/1.1

This section describes the application-layer data which is sent inside of (D)TLS when using the radius/1.1 protocol.  Unless otherwise discussed herein, the application-layer data is unchanged from traditional RADIUS.  This protocol is only used when "radius/1.1" has been negotiated by both ends of a connection.

## Request and Response Authenticator fields

As packets are no longer signed with MD5, the Request and Response Authenticator fields MUST NOT be calculated as described in any previous RADIUS specification.  That 16-octet portion of the packet header is now repurposed into two logical subfields, as given below:

* 4 octets of opaque Token used to match requests and responses,

* 12 octets of Reserved.

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       Token                                                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                                
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        Reserved
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #Token title="The 16 octet Token plus Reserved field."}

Token

> The Token field is four octets, and aids in matching requests and
> replies.  The RADIUS server can detect a duplicate request if it receives
> the same Token value for two packets on a particular connection.
>
> All request / reply tracking MUST be done only on the Token field,
> all other fields in the RADIUS packet header are ignored for the
> purposes of deduplication.
>
> All deduplication MUST be done on a per-connection basis.  If two
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

Reserved

> The Reserved field is twelve (12) octets in length.
>
> These octets MUST be set to zero when sending a packet.
> These octets MUST be ignored when receiving a packet.
> These octets are reserved for future protocol extensions.

### Sending Packets

The Token field MUST change for every new unique packet which is sent.  For DTLS transport, it is possible to retransmit duplicate packets, in which case the Token value MUST NOT be changed when a duplicate packet is sent.  When the contents of a retransmitted packet change for any reason (such changing Acct-Delay-Time as discussed in {{RFC2866}} Section 5.2), the Token value MUST be changed.

The Token MUST be different for different packets on the same connection.  That is, if two packets are bit-for-bit identical (other than the Token field), then those two packets may have the same token value.  If the two packets have any differences, then the Token values for those two packets is required to be different.

For simplicity, it is RECOMMENDED that the Token values be generated from a 32-bit counter which is unique to each connection.  Such a counter SHOULD be initialized from a random number generator whenever a new connection is opened.  The counter can then be incremented for every new packet which is sent.  As there is no special meaning for the Token, there is no meaning when a counter "wraps" around from a high value to zero.

### Receiving Packets

A system which receives radius/1.1 packets MUST perform packet deduplication for all situations where it is required by RADIUS.  Where RADIUS does not require deduplication (e.g. TLS transport), the system SHOULD NOT do deduplication.

In normal RADIUS, the Identifier field can be the same for different types of packets on the same connection, e.g. for Access-Request and Accounting-Request.  This overlap leads to increased complexity for RADIUS clients and servers, as the Identifier field is not unique in a connection, and different packet types can use the same identififer.  Implementations of RADIUS therefore need do deduplication across multiple fields of the RADIUS packet header, which adds complexity.

When using radius/1.1, implementations MUST instead do deduplication solely on the Token field, on a per-connection basis.  A server MUST treat the Token as being an opaque field with no intrinsic meaning.  While the recommendation above is for the sender to use a counter, other implementations are possible, valid, and permitted.  For example, a system could use a pseudo-random number generator with a long period to generate unique values for the Token field.

This change from RADIUS means that the Identifier field is no longer useful.  It is RECOMMENDED that the Identifier field be set to zero for all radius/1.1 packets.  In order to stay close to RADIUS, we still require that replies MUST use the same Identifier as was seen in the corresponding request.  There is no reason to make major changes to the RADIUS packet header.

# Attribute handling

Most attributes in RADIUS have no special encoding "on the wire", or any special meaning between client and server.  Unless discussed in this section, all RADIUS attributes are unchanged in this specification.  This requirement includes attributes which contain a tag, as defined in {{RFC2868}}.

## Obfuscated Attributes

As (D)TLS is used for this specification, there is no need to hide the contents of an attribute on a hop-by-hop basis.  The TLS transport ensures that all attribute contents are hidden from any observer

Attributes which are obfuscated with MD5 no longer have the obfuscation step applied when radius/1.1 is used.  Instead, those attributes are simply encoded as their values, as with any other attribute.  Their encoding method MUST follow the encoding for the underlying data type, with any encryption / obfuscation step omitted.

We understand that there is often concern in RADIUS that passwords are sent "in cleartext" across the network.  This allegation was never true for RADIUS, and definitely untrue when (D)TLS transport is used.  While passwords are encoded in packets as strings, the packets (and thus passwords) are protected by TLS.  For the unsure reader this is the same TLS which protects passwords used for web logins, e-mail reception and sending, etc.  As a result, any claims that passwords are sent "in the clear" are false.

### User-Password

The User-Password attribute ({{RFC2865}} Section 5.2) MUST be encoded the same as any other attribute of data type 'text' ({{RFC8044}} Section 3.4), e.g. User-Name ({{RFC2865}} Section 5.1).

### Tunnel-Password

The Tunnel-Password attribute ({{RFC2868}} Section 3.5) MUST be encoded the same as any other attribute of data type 'text' which contains a tag, such as Tunnel-Client-Endpoint ({{RFC2868}} Section 3.3).  Since the attribute is no longer obfuscated, there is no need for a Salt field or Data-Length fields as described in {{RFC2868}} Section 3,5, and the textual value of the password can simply be encoded as-is.

### Vendor-Specific Attributes

Any Vendor-Specific attribute which uses similar obfuscation MUST be encoded as per their base data type.  Specifically, the MS-MPPE-Send-Key and MS-MPPE-Recv-Key attributes ({{RFC2548}} Section 2.4) MUST be encoded as any other attribute of data type 'string' ({{RFC8044}} Section 3.5).

We note that as the RADIUS shared secret is no longer used, it is impossible for any Vendor-Specific attribute to be obfuscated using the previous methods defined for RADIUS.

If a vendor wishes to hide the contents of a Vendor-Specific attribute, they are free to do so using vendor-defined methods.  However, such methods are unnecessary due to the use of (D)TLS transport.

## Message-Authenticator

The Message-Authenticator attribute ({{RFC3579}} Section 3.2) MUST NOT be sent over a radius/1.1 connection.  That attribute is no longer used or needed.

If the Message-Authenticator attribute is received over a radius/1.1 connection, the attribute MUST be silently discarded, or treated an as "invalid attribute", as defined in {{RFC6929}}, Section 2.8.  That is, the Message-Authenticator attribute is no longer used to sign packets.  Its existence (or not) has become meaningless.

We note that any packet which contains a Message-Authenticator attribute can still be processed.  There is no need to discard an entire packet simply because it contains a Message-Authenticator attribute.

## Message-Authentication-Code

Similarly, the Message-Authentication-Code attribute defined in {{RFC6218}} Section 3.3 MUST NOT be sent over a radius/1.1 connection.  That attribute MUST be treated the same as Message-Authenticator, above.

As the Message-Authentication-Code attribute is no longer used, the related MAC-Randomizer attribute {{RFC6218}} Section 3.2 is also no longer used.  It MUST also be treated the same was as Message-Authenticator, above.

## CHAP, MS-CHAP, etc.

While some attributes such as CHAP-Password, etc. depend on insecure cryptographic primitives such as MD5, these attributes are treated as opaque blobs when sent between a RADIUS client and server.  The contents of the attributes are not obfuscated, and they do not depend on the RADIUS shared secret.

As a result, these attributes are unchanged in radius/1.1.  We reiterate that this specification is largely a transport profile for RADIUS.  Other than a  few attributes such as Message-Authenticator, the meaning of all attributes in this specification is identical to their meaning in RADIUS.  Only the "on the wire" encoding of some attributes change, and then only for attributes which are obfuscated using the RADIUS shared secret.  Those obfuscated attributes are now protected by the modern cryptography in TLS, instead of an "ad hoc" approach using MD5.

A server implementing this specification can proxy CHAP, MS-CHAP, etc. without any issue.  A home server implementing this specification can authenticate CHAP, MS-CHAP, etc. without any issue.

## Original-Packet-Code

The Original-Packet-Code attribute ({{RFC7930}} Section 4) MUST NOT be sent over a radius/1.1 connection.  That attribute is no longer used or needed.

If the Original-Packet-Code attribute is received over a radius/1.1 connection, the attribute MUST either be silently discarded, or be treated an as "invalid attribute", as defined in {{RFC6929}}, Section 2.8.  That is, existence of the Token field means that the Original-Packet-Code attribute is no longer needed to correlate Protocol-Error replies with outstanding requests.  As such, the Original-Packet-Code attribute is not used in radius/1.1.

We note that any packet which contains an Original-Packet-Code attribute can still be processed.  There is no need to discard an entire packet simply because it contains an Original-Packet-Code attribute.

# Proxies

A RADIUS proxy normally decodes and then re-encodes all attributes, included obfuscated ones.  A RADIUS proxy will not generally rewrite the content of the attributes it proxies (unless site-local policy requires such a rewrite).  While some attributes may be modified due to administrative or policy rules on the proxy, the proxy will generally not rewrite the contents of attributes such as User-Password, Tunnel-Password, CHAP-Password, MS-CHAP-Password, MS-MPPE keys, etc.  All attributes are therefore transported through a radius/1.1 connection without changing their values or contents.

The proxy may negotiate radius/1.1 (or not) with a particular client or clients, and it may negotiate radius/1.1 (or not) with a server or servers it connect to, in any combination.  As a result, this specification is fully compatible with all past, present, and future RADIUS attributes.

# Crypto-Agility

The crypto-agility requirements of {{RFC6421}} are addressed in {{RFC6614}} Appendix C, and in Section 10.1 of {{RFC7360}}.  This specification makes no changes from, or additions to, those specifications.

This document adds the requirement that any new RADIUS specification MUST NOT introduce new "ad hoc" cryptographic primitives as was done with User-Password and Tunnel-Password.  There is insufficient expertise in the RADIUS community to securely design new cryptography.

# Implementation Status

(This section to be removed by the RFC editor.)

This specification is being implemented (client and server) in the FreeRADIUS project which is hosted on GitHub.  URL TBD.  The code implemention "diff" is approximately 2,000 lines, including build system changes and changes to configuration parsers.

# Privacy Considerations

This specification requires secure transport for RADIUS, and this has all of the privacy benefits of RADIUS/TLS {{RFC6614}} and RADIUS/DTLS {{RFC7360}}.  All of the insecure uses of RADIUS bave been removed.

# Security Considerations

The primary focus of this document is addressing security considerations for RADIUS.

# IANA Considerations

IANA is request to update the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry with two new entries:

~~~~
Protocol: radius/1.0
Id. Sequence: 0x72 0x61 0x64 0x69 0x75 0x73 0x2f 0x31 ("radius/1")
Reference: This document

Protocol:radius/1.1
Id. Sequence: 0x72 0x61 0x64 0x69 0x75 0x73 0x2f 0x31 0x2e 0x31
Reference: ("radius/1.1")  This document
~~~~

# Issues and Questions

Should we add a header flag which says "require TLS transport"?  This can be set by client to indicate that the data should not be sent over UDP/TCP at any stage of proxying.  It can also be added by a server to indicate that the packet contains similar data.

One use-case for this is attributes which require the protection of TLS, but which do not have any RADIUS obfuscation methods defined.

As a related note, do we want to allow new specifications to define attributes containing private information, but where we do not define RADIUS obfuscation?

Maybe we want to reserve part of the attribute space (or extended attribute flags) for attributes which require TLS.

It is difficult for a client to know when to set this flag, unless there is a specification which requires secure transport of clear-text private information.  Servers can set the flag in similar situations.

Perhaps a home server could require that the flag is set for all packets sent to it?



# Acknowledgements

In hindsight, the decision to retain MD5 for RADIUS/TLS was likely wrong.  It was an easy decision to make in the short term, but it has caused ongoing problems which this document addresses.

Thanks to Bernard Aboba, Karri Huhtanen, and Alexander Clouter for reviews and feedback.

# Changelog

* draft-dekok-radext-sradius-00

> Initial Revision

* draft-dekok-radext-radiusv11-00

> Use ALPN from RFC 7301, instead of defining a new port.  Drop the name "SRADIUS".
>
> Add discussion of Original-Packet-Code

--- back
