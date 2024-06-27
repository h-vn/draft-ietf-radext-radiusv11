---
title: RADIUS ALPN and removing MD5
abbrev: RADIUSv11
docname: draft-ietf-radext-radiusv11-09
updates: 5176, 6614, 7360

stand_alone: true
ipr: trust200902
area: Internet
wg: RADEXT Working Group
kw: Internet-Draft
cat: exp
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
  RFC8174:
  RFC2865:
  RFC6421:
  RFC6929:
  RFC6614:
  RFC7301:
  RFC7360:
  RFC8044:

informative:
  RFC1321:
  RFC2548:
  RFC2866:
  RFC2868:
  RFC3539:
  RFC3579:
  RFC5176:
  RFC6151:
  RFC6218:
  RFC6613:
  RFC7585:
  RFC7593:
  RFC7930:
  EDUROAM:
     title: "eduroam"
     author:
       name: eduroam
     format:
       TXT:  https://eduroam.org
  OPENROAMING:
     title: "OpenRoaming: One global Wi-Fi network"
     author:
       name: Wireless Broadband Alliance
     format:
       TXT:  https://wballiance.com/openroaming/

venue:
  group: RADEXT
  mail: radext@ietf.org
  github: radext-wg/draft-ietf-radext-radiusv11.git

--- abstract

This document defines Application-Layer Protocol Negotiation Extensions for use with RADIUS/TLS and RADIUS/DTLS.  These extensions permit the negotiation of an additional application protocol for RADIUS over (D)TLS.  No changes are made to RADIUS/UDP or RADIUS/TCP.  The extensions allow the negotiation of a transport profile where the RADIUS shared secret is no longer used, and all MD5-based packet signing and attribute obfuscation methods are removed.

This document updates RFC5176, RFC6614, and RFC 7360.

--- middle

# Introduction

The RADIUS protocol {{RFC2865}} uses MD5 {{RFC1321}} to sign packets, and to obfuscate certain attributes.  Additional transport protocols were defined for TCP ({{RFC6613}}), TLS ({{RFC6614}}), and DTLS ({{RFC7360}}).  However, those transport protocols still relied on MD5.  That is, the shared secret was used along with MD5, even when the RADIUS packets were being transported in (D)TLS.  At the time, the consensus of the RADEXT working group was that this continued use of MD5 was acceptable.  TLS was seen as a simple "wrapper" around RADIUS, while using a fixed shared secret.  The intention at the time was to allow the use of (D)TLS while making essentially no changes to the basic RADIUS encoding, decoding, signing, and packet validation.

Issues of MD5 security have been known for decades, most most notably in {{RFC6151}}, and in {{RFC6421, Section 3}}, among others.  The reliance on MD5 for security makes it impossible to use RADIUS in a FIPS-140 compliant system, as FIPS-140 forbids systems from relying on insecure cryptographic methods for security.  In addition, the use of MD5 in RADIUS/TLS and RADIUS/DLTS adds no security or privacy over that provided by TLS.

This document defines an Application-Layer Protocol Negotiation (ALPN) {{RFC7301}} extension for RADIUS over (D)TLS which removes the need to use MD5 for (D)TLS.  We make no changes to UDP or TCP transport.  This extension can best be understood as a transport profile for RADIUS over TLS, rather than a whole-sale revision of the RADIUS protocol.

Systems which implement this transport profile can be more easily verified to be FIPS-140 compliant.  A preliminary implementation has shown that only minor code changes are required to support RADIUS/1.1 on top of an existing RADIUS/TLS server implementation, which are:

* A method to set the list of supported ALPN protocols before the TLS handshake starts

* After the TLS handshake has completed, a method to query if ALPN has chosen a protocol, and if yes, which protocol was chosen.

* Changes to the packet encoder and decoder, so that the individual packets are not signed, and no attribute is encoded with the historic obfuscation methods.

That is, the bulk of the ALPN protocol can be left to the underlying TLS implementation.  This document discusses the ALPN exchange in detail in order to give simplified descriptions for the reader, and so that the reader does not have to read or understand all of {{RFC7301}}.

The detailed list of changes from historic TLS-based transports to RADIUS/1.1 is as follows:

* ALPN is used for negotiation of this extension,

* TLS 1.3 or later is required,

* All uses of the RADIUS shared secret have been removed,

* The now-unused Request and Response Authenticator fields have been repurposed to carry an opaque Token which identifies requests and responses,

* The functionality of the Identifier field has been replaced by the Token field, and the space previously taken by the Identifier field is now reserved and unused,

* The Message-Authenticator attribute ({{RFC3579, Section 3.2}}) is not sent in any packet, and if received is ignored,

* Attributes such as User-Password, Tunnel-Password, and MS-MPPE keys are sent encoded as "text" ({{RFC8044, Section 3.4}}) or "octets" ({{RFC8044, Section 3.5}}), without the previous MD5-based obfuscation.  This obfuscation is no longer necessary, as the data is secured and kept private through the use of TLS,

* The conclusion of the efforts stemming from {{RFC6421}} is that crypto-agility in RADIUS is best done via a TLS wrapper, and not by extending the RADIUS protocol.

* {{RFC5176}} is updated to allow the Error-Cause attribute to appear in Access-Reject packets.

The following items are left unchanged from traditional TLS-based transports for RADIUS:

* The RADIUS packet header is the same size, and the Code and Length fields ({{RFC2865, Section 3}}) have the same meaning as before,

* The default 4K packet size is unchanged, although {{RFC7930}} can still be leveraged to use larger packets,

* All attributes which have simple encodings (that is, attributes which do not use MD5 obfuscation) have the same encoding and meaning as before,

* As this extension is a transport profile for one "hop" (client to server connection), it does not impact any other connection used by a client or server.  The only systems which are aware that this transport profile is in use are the client and server who have negotiated the use of this extension on a particular shared connection,

* This extension uses the same ports (2083/tcp and 2083/udp) which are defined for RADIUS/TLS {{RFC6614}} and RADIUS/DTLS {{RFC7360}}.

A major benefit of this extension is that a home server which implements it can also be more easily verified for FIPS-140 compliance.  That is, a home server can remove all uses of MD4 and MD5, which means that those algorithms are provably not used for security purposes.  In that case, however, the home server will not support CHAP, MS-CHAP, or any authentication method which uses MD4 or MD5.  The choice of which authentication method to accept is always left to the home server.  This specification does not change any authentication method carried in RADIUS, and does not mandate (or forbid) the use of any authentication method for any system.

As for proxies, there was never a requirement that proxies implement CHAP or MS-CHAP authentication.  So far as a proxy is concerned, attributes relating to CHAP and MS-CHAP are simply opaque data that is transported unchanged to the next hop.  It is therefore possible for a FIPS-140 compliant proxy to transport authentication methods which depend on MD4 or MD5, so long as that data is forwarded to a home server which supports those methods.

We reiterate that the decision to support (or not) any authentication method is entirely site local, and is not a requirement of this specification.  The contents or meaning of any RADIUS attribute other than Message-Authenticator (and similar attributes) are not modified.  The only change to the Message-Authenticator attribute is that it is no longer used in RADIUS/1.1.

Unless otherwise described in this document, all RADIUS requirements apply to this extension.  That is, this specification defines a transport profile for RADIUS.  It is not an entirely new protocol, and it defines only minor changes to the existing RADIUS protocol.  It does not change the RADIUS packet format, attribute format, etc.  This specification is compatible with all RADIUS attributes, past, present, and future.

This specification is compatible with existing implementations of RADIUS/TLS and RADIUS/DTLS.  Systems which implement this standard can fall back to historic RADIUS/TLS if no ALPN signaling is performed, and the local configuration permits such fallback.

This specification is compatible with all existing RADIUS specifications.  There is no need for any RADIUS specification to mention this transport profile by name, or to make provisions for this specification.  This document defines how to transform RADIUS into RADIUS/1.1, and no further discussion of that transformation is necessary.

We note that this document makes no changes to previous RADIUS specifications.  Existing RADIUS implementations can continue to be used without modification.  Where previous specifications are explicitly mentioned and updated, those updates or changes apply only when the RADIUS/1.1 transport profile is being used.

In short, when negotiated on a connection, the RADIUS/1.1 transport profile permits implementations to avoid MD5 when signing packets, or when obfuscating certain attributes.

# Terminology

{::boilerplate bcp14}

The following list describes the terminology and abbreviations which are used in this document.

* ALPN

> Application-Layer Protocol Negotiation, as defined in {{RFC7301}}.

* RADIUS

> The Remote Authentication Dial-In User Service protocol, as defined in {{RFC2865}}, {{RFC2866}}, and {{RFC5176}} among others.
>
> While this protocol can be viewed as "RADIUS/1.0", for simplicity and historical compatibility, we keep the name "RADIUS".

* RADIUS/UDP

> RADIUS over the User Datagram Protocol {{RFC2865}}, {{RFC2866}}, {{RFC5176}}, among others.

* RADIUS/TCP

> RADIUS over the Transmission Control Protocol {{RFC6613}}.

* RADIUS/TLS

> RADIUS over the Transport Layer Security protocol {{RFC6614}}.

* RADIUS/DTLS

> RADIUS over the Datagram Transport Layer Security protocol  {{RFC7360}}.

* RADIUS over TLS

> Any RADIUS packets transported over TLS or DTLS.  This terminology is used instead of alternatives such as "RADIUS/(D)TLS", or "either RADIUS/TLS or RADIUS/DTLS".  This term is generally used when referring to TLS-layer requirements for RADIUS packet transport.

* historic RADIUS/TLS

> RADIUS over (D)TLS as defined in {{RFC6614}} and {{RFC7360}}.  This term does not include the protocol defined in this specification.

* RADIUS/1.1

> The transport profile defined in this document, which stands for "RADIUS version 1.1".  We use RADIUS/1.1 to refer interchangeably to TLS and DTLS transport.

* TLS

> The Transport Layer Security protocol.  Generally when we refer to TLS in this document, we are referring interchangeably to TLS or DTLS transport.

# The RADIUS/1.1 Transport profile for RADIUS

This section describes the ALPN transport profile in detail.  It first gives the name used for ALPN, and then describes how ALPN is configured and negotiated by client and server.  It then concludes by discussing TLS issues such as what to do for ALPN during session resumption.

## ALPN Name for RADIUS/1.1

The ALPN name defined for RADIUS/1.1 is as follows:

> "radius/1.1"
>
>> The protocol defined by this specification.

Where ALPN is not configured or is not received in a TLS connection, systems supporting ALPN MUST NOT use RADIUS/1.1.

Where ALPN is configured, the client signals support by sending ALPN strings signaling which protocols it supports..  The server can accept one of these proposals and reply with a matching ALPN string, or reject this proposal, and not reply with any ALPN string.  A full walk-through of the protocol negotiation is given below.

Implementations MUST signal ALPN "radius/1.1" in order for it to be used in a connection.  Implementations MUST NOT have an administrative flag which causes a connection to use "radius/1.1", but which does not signal that protocol via ALPN.

The next step in defining RADIUS/1.1 is to review how ALPN works.

## Operation of ALPN

In order to provide a high-level description of ALPN for readers who are not familiar with the details of {{RFC7301}}, we provide a brief overview here.

Once a system has been configured to support ALPN, it is negotiated on a per-connection basis as per {{RFC7301}}.  The negotiation proceeds as follows:

1) The client sends an ALPN extension in the ClientHello.  This extension lists one or more application protocols by name.  These names are the protocols which the client is claiming to support.

2) The server receives the extension, and validates the application protocol name(s) against the list it has configured.

> If the server finds no acceptable common protocols (ALPN or otherwise), it closes the connection.

3) Otherwise, the server returns a ServerHello with either no ALPN extension, or an ALPN extension containing only one named application protocol, which needs to be one of the names proposed by the client.

> If the client did not signal ALPN, or the server does not accept the ALPN proposal, the server does not reply with any ALPN name.

4) The client receives the ServerHello, validates the received application protocol (if any) against the name(s) it sent, and records which application protocol was chosen.

> This check is necessary in order for the client to both know which protocol the server has selected, and to validate that the protocol sent by the server is one which is acceptable to the client.

The next step in defining RADIUS/1.1 is to define how ALPN is configured on the client and server, and to give more detailed requirements on its configuration and operation.

## Configuration of ALPN for RADIUS/1.1

Clients or servers supporting this specification can do so by extending their TLS configuration through the addition of a new configuration flag, called "Version" here.  The exact name given below does not need to be used, but it is RECOMMENDED that administrative interfaces or programming interfaces use a similar name in order to provide consistent terminology.  This flag controls how the implementation signals use of this protocol via ALPN.

When set, this flag contains the list of permitted RADIUS versions as numbers, e.g. "1.0" or "1.1".  The implementation may allow multiple values in one variable, or allow multiple variables, or instead use two configuration for "minimum" and "maximum" allowed versions.  We assume here that there is one variable, which can contain either no value, or else a list of one or more versions which the current implementation supports.  In this specification, the possible values, ALPN strings, and corresponding interpretations are:

~~~~
Flag Value  |   ALPN String(s)       | Interpretation
------------------------------------------------------
unset       |                        | no ALPN strings are sent.
1.0         | radius/1.0             | require historic RADIUS/TLS
1.0, 1.1    | radius/1.0, radius/1.1 | allow either historic
            |                        | RADIUS/TLS or RADIUS/1.1.
1.1         | radius/1.1             | require RADIUS/1.1.
~~~~

This configuration is also extensible to future RADIUS versions if that extension becomes necessary.  New flag values and ALPN names can simply be added to the list.  Implementations can then negotiate the highest version which is supported by both client and server.

Implementations SHOULD support both historic RADIUS/TLS and RADIUS/1.1.  Such implementations MUST set the default value for this configuratiun flag to "1.0, 1.1".  This setting ensures that both versions of RADIUS can be negotiated.

Implementations MAY support only RADIUS/1.1.  In which case the default value for this configuration flag MUST be "1.1".  This behavior is NOT RECOMMENDED, as it is incompatible with historic RADIUS/TLS.  This behavior can only be a reasonable default when all (or nearly all) RADIUS clients have been updated to support RADIUS/1.1.

A more detailed definition of the variable and the meaning of the values is given below.

Configuration Flag Name

> Version

Values

> When the flag is unset, ALPN is not used.
>
> Any connection MUST use historic RADIUS/TLS.
>
> This flag is included here only for logical completeness.  Implementations of this specification SHOULD be configured to always send one or more ALPN strings.  This data signals that the implementation is capable performing ALPN negotiation, even if it is not currently configured to use RADIUS/1.1
>
>> Client Behavior
>>
>>> The client MUST NOT send any protocol name via ALPN.
>>
>> Server Behavior
>>
>>> The server MUST NOT signal any protocol name via ALPN.
>>>
>>> If the server receives an ALPN name from the client, it MUST NOT close the connection.  Instead, it simply does not reply with ALPN, and finishes the TLS connection setup as defined for historic RADIUS/TLS.
>>>
>>> Note that if a client sends "radius/1.1", the client will see that the server failed to acknowledge this request, and will close the connection.  For any other client configuration, the connection will use historic RADIUS/TLS.

> Other values ("1.0", "1.0, 1.1", "1.1", etc.)
>
>> Client Behavior
>>
>>> The client MUST send the ALPN string(s) associated with the configured version.  e.g. For "1.0", send "radius/1.0".
>>>
>>> The client will receive either no ALPN response from the server, or an ALPN response of one version string with MUST match one of the strings it sent, or else a TLS alert of "no_application_protocol" (120).
>>>
>>> If the connection remains open, the client MUST treat the connection as using the matching ALPN version.
>>
>> Server Behavior
>>
>>> If the server receives no ALPN name from the client, it MUST use historic RADIUS/TLS.
>>>
>>> If the server receives one or more ALPN names from the client, it MUST reply with the highest mutually supported version and then use the latest supported version for this connection.
>>>
>>> If the server receives one or more ALPN names from the client, but none of the names match the versions supported by (or configured on) the server, it MUST reply with a TLS alert of "no_application_protocol" (120), and then MUST close the TLS connection.
>>>
>>> These requirements for negotiation are not specific to RADIUS/1.1, and therefore can be used unchanged if any new version of RADIUS is defined.

By requiring the default configuration to allow historic RADIUS/TLS, implementations will be able to negotiate both historic RADIUS/TLS connections, and also RADIUS/1.1 connections.  Any other recommended default setting would prevent either the negotiation of historic RADIUS/TLS, or prevent the negotiation of RADIUS/1.1.

Once administrators verify that both ends of a connection support RADIUS/1.1, and that it has been negotiated successfully, the configurations SHOULD be updated to require RADIUS/1.1.  The connections should be monitored after this change to ensure that the systems continue to remain connected.  If there are connection issues, then the configuration should be reverted to using allow both "radius/1.0" and "radius/1.1" ALPN strings, until such time as the connection problems have been resolved.

We reiterate that systems implementing this specification, but which are configured with setting that forbid RADIUS/1.1, will behave largely the same as systems which do not implement this specification.  The only difference is that clients may send the ALPN name "radius/1.0".

Systems implementing RADIUS/1.1 SHOULD NOT be configured by default to forbid that protocol.  That setting exists mainly for completeness, and to give administrators the flexibility to control their own deployments.

While {{RFC7301}} does not discuss the possibility of the server sending a TLS alert of "no_application_protocol" (120) when the client does not use ALPN, we believe that this behavior is useful.  As such, servers MAY send a a TLS alert of "no_application_protocol" (120) when the client does not use ALPN.

However, some TLS implementations may not permit an application to send a TLS alert of its choice, at a time of its choice.   This limitation means that it is not always possible for an application to send the TLS alert as discussed in the previous section.  The impact is that an implementation may attempt to connect, and then see that the connection fails, but not be able to determine why that failure has occurred.  Implementers and administrators should be aware that unexplained connection failures may be due to ALPN negotiation issues.

The server MAY send this alert during the ClientHello, if it requires ALPN but does not receive it.  That is, there may not always be a need to wait for the TLS connection to be fully established before realizing that no common ALPN protocol can be negotiated.

Where the client does perform signaling via ALPN and the server determines that there is no compatible application protocol name, then as per {{RFC7301, Section 3.2}}, it MUST send a TLS alert of "no_application_protocol" (120).

Whether or not the server sent a TLS alert for no compatible ALPN, it MUST close the connection.  The above requirements on ALPN apply to both new sessions, and to resumed sessions.

In contrast, there is no need for the client to signal that there are no compatible application protocol names.  The client sends zero or more protocol names, and the server responds as above.  From the point of view of the client, the list it sent results in either a connection failure, or a connection success.

It is RECOMMENDED that the server logs a descriptive error in this situation, so that an administrator can determine why a particular connection failed.  The log message SHOULD include information about the other end of the connection, such as IP address, certificate information, etc.  Similarly, when the client receives a TLS alert of "no_application_protocol" it SHOULD log a descriptive error message.  Such error messages are critical for helping administrators to diagnose connectivity issues.

### Using Protocol-Error for Signaling ALPN Failure

When it is not possible to send a TLS alert of "no_application_protocol" (120), then the only remaining method for one party to signal the other is to send application data inside of the TLS tunnel.  Therefore, for the situation when a one end of a connection determines that it requires ALPN while the other end does not support ALPN, the end requiring ALPN MAY send a Protocol-Error packet {{RFC7930}} inside of the tunnel, and then MUST close the connection.  If this is done, the Token field of the Protocol-Error packet cannot be copied from any request, and therefore that field MUST be set to all zeros.

The Protocol-Error packet SHOULD contain a Reply-Message attribute with a textual string describing the cause of the error.  The packet SHOULD also contain an Error-Cause attribute, with value Unsupported Extension (406).  The packet SHOULD NOT contain other attributes.

An implementation sending this packet could bypass any RADIUS encoder, and simply write this packet as a predefined, fixed set of data to the TLS connection.  That process would likely be simpler than trying to call the normal RADIUS packet encoder to encode a reply packet with no corresponding request packet.

As this packet is an unexpected response packet, existing client implementations of RADIUS over TLS will ignore it.  They may either log an error and close the connection, or they may discard the packet and leave the connection open.  If the connection remains open, the end supporting ALPN will close the connection, so there will be no side effects from sending this packet.  Therefore, while using a Protocol-Error packet in this way is unusual, it is both informative and safe.

The purpose of this packet is not to have the other end of the connection automatically determine what went wrong, and fix it.  Instead, the packet is intended to be (eventually) seen by an administrator, who can then take remedial action.

### Tabular Summary

The preceding text gives a large number of recommendations.  In order to give a simpler description of the outcomes, a table of possible behaviors for client/server values of the Version flag is given below.  This table and the names given below are for informational and descriptive purposes only.

~~~~
                             Server
             no ALPN  |   1.0    | 1.0, 1.1 |    1.1
Client    |--------------------------------------------
----------|
No ALPN   |   TLS        TLS        TLS        Close-S
          |
1.0       |   TLS        TLS        TLS        Alert
          |
1.0, 1.1  |   TLS        TLS        1.1        1.1
          |
1.1       |   Close-C    Alert      1.1        1.1
~~~~
{: title="Possible outcomes for ALPN Negotiation"}

The table entries above have the following meaning:

> Alert
>
>> The client sends ALPN, and the server does not agree to the clients ALPN proposal.  The server replies with a TLS alert of "no_application_protocol" (120), and then closes the TLS connection.
>>
>> As the server replies with a TLS alert, the Protocol-Error packet is not used here.
>
> Close-C
>
>> The client sends ALPN, but the server does not respond with ALPN.  The client closes the connection.
>>
>> As noted in the previous section, the client MAY send a Protocol-Error packet to the server before closing the connection.
>
> Close-S
>
>> The client does not send ALPN string(s), but the server requires ALPN.  The server closes the connection.
>>
>> As noted in the previous section, the server MAY send a Protocol-Error packet to the client before closing the connection.  The server MAY also send a TLS alert of "no_application_protocol" (120) before closing the connection.
>
> TLS
>
>> Historic RADIUS/TLS is used.  The client either sends no ALPN string, or sends "radius/1.0".  The server either replies with no ALPN string, or with "radius/1.0".  The connection MUST use historic RADIUS/TLS.
>
> 1.1
>
>> The client sends the ALPN string "radius/1.1.  The server acknowledges this negotiation with a reply of "radius/1.1", and then RADIUS/1.1 is used.

Implementations should note that this table may be extended in future specifications.  The above text is informative, and does not mandate that only the above ALPN strings are used.  The actual ALPN negotiation takes place as defined in the preceding sections of this document, and in {{RFC7301}}.

## Miscellaneous Items

Implementations of this specification MUST require TLS version 1.3 or later.

The use of the ALPN string "radius/1.0" is technically unnecessary, as it is largely equivalent to not sending any ALPN string.  However, that value is useful for RADIUS administrators.  A system which sends the ALPN string "radius/1.0" is explicitly signaling that it supports ALPN negotiation, but that it is not currently configured to support RADIUS/1.1.  That information can be used by administrators to determine which devices are capable of ALPN.

The use of the ALPN string "radius/1.0" also permits server implementations to send a TLS alert of "no_application_protocol" (120) when it cannot find a matching ALPN string.  Experiments with TLS library implementations suggest that in some cases it is possible to send that TLS alert when ALPN is not used.  However, such a scenario is not discussed on {{RFC7301}}, and is likely not universal.  As a result, ALPN as defined in {{RFC7301}} permits servers to send that TLS alert in situations where it would be otherwise forbidden, or perhaps unsupported.

Finally, defining ALPN strings for all known RADIUS versions will make it easier to support additional ALPN strings if that functionality is ever needed.

## Session Resumption

{{RFC7301, Section 3.1}} states that ALPN is negotiated on each connection, even if session resumption is used:

> When session resumption or session tickets {{?RFC5077}} are used, the previous contents of this extension are irrelevant, and only the values in the new handshake messages are considered.

In order to prevent down-bidding attacks, RADIUS systems which negotiate the "radius/1.1" protocol MUST associate that information with the session ticket, and enforce the use of "radius/1.1" on session resumption.  That is, if "radius/1.1" was negotiated for a session, both clients and servers MUST behave as if the RADIUS/1.1 flag was set to "require" for that session.

A client which is resuming a "radius/1.1" connection MUST advertise only the capability to do "radius/1.1" for the resumed session.  That is, even if the client configuration allows historic RADIUS/TLS for new connections, it MUST signal "radius/1.1" when resuming a session which had previously negotiated "radius/1.1".

Similarly, when a server does resumption for a session which had previously negotiated "radius/1.1".   If the client attempts to resume the sessions without signaling the use of RADIUS/1.1, the server MUST close the connection.  The server MUST send an appropriate TLS error, and also SHOULD log a descriptive message as described above.

In contrast, there is no requirement for a client or server to force the use of {{RFC6614}} RADIUS/TLS on session resumption.  Clients are free to signal support for "radius/1.1" on resumed sessions, even if the original session did not negotiate "radius/1.1".  Servers are free to accept this request, and to negotiate the use of "radius/1.1" for such sessions.

# RADIUS/1.1 Packet and Attribute Formats

This section describes the application-layer data which is sent inside of (D)TLS when using the RADIUS/1.1 protocol.  Unless otherwise discussed herein, the application-layer data is unchanged from traditional RADIUS.  This protocol is only used when "radius/1.1" has been negotiated by both ends of a connection.

## RADIUS/1.1 Packet Format

When RADIUS/1.1 is used, the RADIUS header is modified from standard RADIUS.  While the header has the same size, some fields have different meaning.  The Identifier and the Request / Response Authenticator fields are no longer used in RADIUS/1.1.  Any operations which depend on those fields MUST NOT be performed.  As packet signing and security are handled by the TLS layer, RADIUS-specific cryptographic primitives are no longer in RADIUS/1.1.

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
> The meaning of the Code field is unchanged from previous RADIUS specifications.

Reserved-1

> The Reserved-1 field is one octet.
>
> This field was previously used as the "Identifier" in historic RADIUS/TLS.  It is now unused, as the Token field replaces it both as the way to identify requests, and to associate responses with requests.
>
> When sending packets, the Reserved-1 field MUST be set to zero.  The Reserved-1 field MUST be ignored when receiving a packet.

Length

> The Length field is two octets.
>
> The meaning of the Length field is unchanged from previous RADIUS specifications.

Token

> The Token field is four octets, and aids in matching requests and
> replies, as a replacement for the Identifier field.  The RADIUS server can detect a duplicate request if it receives
> the same Token value for two packets on a particular connection.
>
> All values are possible for the Token field.  Implementations MUST treat the Token as an opaque blob when comparing Token values.
>
> Further requirements are given below in [](#sending-packets) for sending packets, and in [](#receiving-packets) for receiving packets.

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

### Sending Packets {#sending-packets}

The Token field MUST change for every new unique packet which is sent on the same connection. For DTLS transport, it is possible to retransmit duplicate packets, in which case the Token value MUST NOT be changed when a duplicate packet is (re)sent.  When the contents of a retransmitted packet change for any reason (such changing Acct-Delay-Time as discussed in {{RFC2866, Section 5.2}}), the Token value MUST be changed.  Note that on reliable transports, packets are never retransmitted, and therefore every new packet which is sent has a unique Token value.

We note that in previous RADIUS specifications, the Identifier field could have the same value for different packets on the same connection.  For example, Access-Request (Code 1) and Accounting-Request (Code 4) packets could both use ID 3, and still be treated as different packets.  This overlap required that RADIUS clients and servers track the Identifier field, not only on a per-connection basis, but also on a per-Code basis.  This behavior adds complexity to implementations.

In contrast, the Token values MUST be generated from a 32-bit counter which is unique to each connection.  Such a counter SHOULD be initialized to a random value, taken from a random number generator, whenever a new connection is opened.  The counter MUST then be incremented for every unique new packet which is sent by the client.  Retransmissions of the same packet MUST use the same unchanged Token value.  As the Token value is mandated to be unique per packet, a duplicate Token value is the only way that a server can detect duplicate transmissions.

This counter method ensures that the Tokens are unique, and are also independent of any Code value in the RADIUS packet header.  This method is mandated because any other method of generating unique and non-conflicting Token values is more complex, with no additional benefit and only the likelihood of increased bugs and interoperability issues.  Any other method for generating Token values would require substantially more resources to track outstanding Token values and their associated expiry times.  The chance that initial values are re-used across two connections is one in 2^32, which is acceptable.

The purpose for initializing the Token to a random counter is to aid administrators in debugging systems.  If the Token values always used the same sequence, then it would easier for a person to confuse different packets which have the same Token value.  By instead starting with a random value, those values are more evenly distributed across the set of allowed values, and are therefore more likely to be unique.

As there is no special meaning for the Token, there is no meaning when a counter "wraps" around from a high value back to zero.  The originating system can simply continue to increment the Token value without taking any special action in that situation.

Once a RADIUS response to a request has been received and there is no need to track the packet any longer, the Token value can be reused.  This reuse happens only when the counter "wraps around" after 2^32 packets have been sent over one connection.  This method of managing the counter automatically ensures a long delay (i.e. 2^32 packets) between multiple uses of the same Token value.   This large number of packets ensures that the only possible situation where there may be conflict is when a client sends billions of packets a second across one connection, or when a client sends billions of packets without receiving replies.  We suggest that such situations are vanishingly rare.  The best solution to those situations would be to limit the number out outstanding packets over one connection to a number much lower than billions.

If a RADIUS client has multiple independent subsystems which send packets to a server, each subsystem MAY open a new connection which is unique to that subsystem.  There is no requirement that all packets go over one particular connection.  That is, despite the use of a 32-bit Token field, RADIUS/1.1 clients are still permitted to open multiple source ports as discussed in {{RFC2865}} Section 2.5.

While multiple connections from client to server are allowed, We reiterate the suggestion of {{RFC3539, Section 3.3}} that a single connection is preferred to multiple connections.  The use of a single connection can improve throughput and latency, while simplifying the clients efforts to determine server status.

### Receiving Packets  {#receiving-packets}

A server which receives RADIUS/1.1 packets MUST perform packet deduplication for all situations where it is required by RADIUS.  Where RADIUS does not require deduplication (e.g. TLS transport), the server SHOULD NOT do deduplication.  However, DTLS transport is UDP-based, and therefore still requires deduplication.

When using RADIUS/1.1, implementations MUST do deduplication only on the Token field, and not on any other field or fields in the packet header. A server MUST treat the Token as being an opaque field with no intrinsic meaning.  This requirement makes the receiver behavior independent of the methods by which the Counter is generated.

Where Token deduplication is done, it MUST be done on a per-connection basis.  If two packets which are received on different connections contain the same Token value, then those packets MUST be treated as distinct (i.e. different) packets.  Systems performing deduplication MAY still track the packet Code, Length, and Attributes which is associated with a Token value.  If it determines that the sender is re-using Token values for distinct outstanding packets, then an error should be logged, and the connection MUST be closed.  There is no way to negotiate correct behavior in the protocol.  Either the parties both operate normally and can communicate, or one end misbehaves, and no communication is possible.

Once a reply has been sent, a system doing deduplication SHOULD cache the replies as discussed in {{RFC5080, Section 2.2.2}}:

> Each cache entry SHOULD be purged after a period of time.  This time
> SHOULD be no less than 5 seconds, and no more than 30 seconds.  After
> about 30 seconds, most RADIUS clients and end users will have given
> up on the authentication request.  Therefore, there is little value
> in having a larger cache timeout.

This change from RADIUS means that the Identifier field is no longer useful for RADIUS/1.1.  The Reserved-1 field (previously used as the Identifier) MUST be set to zero when encoding all RADIUS/1.1 packets.  Implementations of RADIUS/1.1 which receive packets MUST ignore this field.

# Attribute handling

Most attributes in RADIUS have no special encoding "on the wire", or any special meaning between client and server.  Unless discussed in this section, all RADIUS attributes are unchanged in this specification.  This requirement includes attributes which contain a tag, as defined in {{RFC2868}}.

## Obfuscated Attributes

Since the (D)TLS layer provides for connection authentication, integrity checks, and confidentiality, there is no need to hide the contents of an attribute on a hop-by-hop basis.  As a result, all attributes defined as being obfuscated via the shared secret no longer have the obfuscation step applied when RADIUS/1.1 is used.  Instead, those attributes MUST be encoded using the encoding for the underlying data type, with any encryption / obfuscation step omitted.  For example, the User-Password attribute is no longer obfuscated, and is instead sent as data type "text".

There are risks from sending passwords over the network, even when they are protected by TLS.  One such risk comes from the common practice of multi-hop RADIUS routing.  As all security in RADIUS is on a hop-by-hop basis, every proxy which receives a RADIUS packet can see (and modify) all of the information in the packet.  Sites wishing to avoid proxies SHOULD use dynamic peer discovery {{RFC7585}}, which permits clients to make connections directly to authoritative servers for a realm.

There are others ways to mitigate these risks.  The simplest is to follow the requirements of {{RFC6614, Section 2.4}} item (3) and {{RFC7360, Section 10.4}}, which mandates that RADIUS over TLS implementations validate the peer before sending any RADIUS traffic.

Another way to mitigate these risks is for the system being authenticated to use an authentication protocol which never sends passwords (e.g. EAP-pwd {{?RFC5931}}), or which sends passwords protected by a TLS tunnel (e.g. EAP-TTLS {{?RFC5281}}).  The processes to choose and configuring an authentication protocol are strongly site-dependent, so further discussion of these issues are outside of the scope of this document.  The goal here is to ensure that the reader has enough information to make an informed decision.

We note that as the RADIUS shared secret is no longer used in this specification, it is no longer possible or necessary for any attribute to be obfuscated on a hop-by-hop basis using the previous methods defined for RADIUS.

### User-Password

The User-Password attribute ({{RFC2865, Section 5.2}}) MUST be encoded the same as any other attribute of data type 'string' ({{RFC8044, Section 3.5}}).

The contents of the User-Password field MUST be at least one octet in length, and MUST NOT be more than 128 octets in length.  This limitation is maintained from {{RFC2865, Section 5.2}} for compatibility with historic transports.

Note that the User-Password attribute is not of data type 'text'.  The original reason in {{RFC2865}} was because the attribute was encoded as an opaque and obfuscated binary blob.  We maintain that data type here, even though the attribute is no longer obfuscated.  The contents of the User-Password attribute do not have to be printable text, or UTF-8 data as per the definition of the 'text' data type in {{RFC8044, Section 3.4}}.

However, implementations should be aware that passwords are often printable text, and where the passwords are printable text, it can be useful to store and display them as printable text.  Where implementations can process non-printable data in the 'text' data type, they MAY use the data type 'text' for User-Password.

### CHAP-Challenge

{{RFC2865, Section 5.3}} allows for the CHAP challenge to be taken from either the CHAP-Challenge attribute ({{RFC2865, Section 5.40}}), or the Request Authenticator field.  Since RADIUS/1.1 connections no longer use a Request Authenticator field, it is no longer possible to use the Request Authenticator field as the CHAP-Challenge when this transport profile is used.

Clients which send CHAP-Password attribute ({{RFC2865, Section 5.3}}) in an Access-Request packet over a RADIUS/1.1 connection MUST also include a CHAP-Challenge attribute ({{RFC2865, Section 5.40}}).

Proxies may need to receive Access-Request packets over a non-RADIUS/1.1 transport and then forward those packets over a RADIUS/1.1 connection.  In that case, if the received Access-Request packet contains a CHAP-Password attribute but no CHAP-Challenge attribute, the proxy MUST create a CHAP-Challenge attribute in the proxied packet using the contents from the incoming Request Authenticator of the received packet.

### Tunnel-Password

The Tunnel-Password attribute ({{RFC2868, Section 3.5}}) MUST be encoded the same as any other attribute of data type 'string' which contains a tag, such as Tunnel-Client-Endpoint ({{RFC2868, Section 3.3}}).  Since the attribute is no longer obfuscated in RADIUS/1.1, there is no need for a Salt field or Data-Length fields as described in {{RFC2868, Section 3.5}}, and the textual value of the password can simply be encoded as-is.

Note that the Tunnel-Password attribute is not of data type 'text'.  The original reason in {{RFC2868}} was because the attribute was encoded as an opaque and obfuscated binary blob.  We maintain that data type here, even though the attribute is no longer obfuscated.  The contents of the Tunnel-Password attribute do not have to be printable text, or UTF-8 data as per the definition of the 'text' data type in {{RFC8044, Section 3.4}}.

However, implementations should be aware that passwords are often printable text, and where the passwords are printable text, it can be useful to store and display them as printable text.  Where implementations can process non-printable data in the 'text' data type, they MAY use the data type 'text' for Tunnel-Password.

### Vendor-Specific Attributes

Any Vendor-Specific attribute which uses similar obfuscation MUST be encoded as per their base data type.  Specifically, the MS-MPPE-Send-Key and MS-MPPE-Recv-Key attributes ({{RFC2548, Section 2.4}}) MUST be encoded as any other attribute of data type 'string' ({{RFC8044, Section 3.4}}).

## Message-Authenticator

The Message-Authenticator attribute ({{RFC3579, Section 3.2}}) MUST NOT be sent over a RADIUS/1.1 connection.  That attribute is not used or needed in RADIUS/1.1.

If the Message-Authenticator attribute is received over a RADIUS/1.1 connection, the attribute MUST be silently discarded, or treated as an "invalid attribute", as defined in {{RFC6929, Section 2.8}}.  That is, the Message-Authenticator attribute is no longer used to sign packets for the RADIUS/1.1 transport.  Its existence (or not) in this transport is meaningless.

A system which receives a Message-Authenticator attribute in a packet MUST treat it as an "invalid attribute" as defined in {{RFC6929, Section 2.8}}.  That is, the packet can still be processed, even if the Message-Authenticator attribute is ignored.

For proxies, the Message-Authenticator attribute has always been defined as being created and consumed on a "hop by hop" basis.  That is, a proxy which received a Message-Authenticator attribute from a client would never forward that attribute as-is to another server.  Instead, the proxy would either suppress, or re-create, the Message-Authenticator attribute in the outgoing request.  This existing behavior is leveraged in RADIUS/1.1 to suppress the use of Message-Authenticator over a RADIUS/1.1 connection.

A proxy may receive an Access-Request packet over a RADIUS/1.1 connection, and then forward that packet over a RADIUS/UDP or a RADIUS/TCP connection.  In that situation, the proxy SHOULD add a Message-Authenticator attribute to every Access-Request packet which is sent over an insecure transport protocol.

The original text in {{RFC3579, Section 3.3}}, "Note 1" paragraph required that the Message-Authenticator attribute be present for certain Access-Request packets.  It also required the use of Message-Authenticator when the Access-Request packet contained an EAP-Message attribute.  Experience has shown that some RADIUS clients never use the Message-Authenticator, even for the situations where its use is suggested.

When the Message-Authenticator attribute is missing from Access-Request packets, it is often possible to trivially forge or replay those packets.  As such, this document RECOMMENDS that RADIUS clients always include Message-Authenticator in Access-Request packets when using UDP or TCP transport.  As the scope of this document is limited to defining RADIUS/1.1, we cannot mandate that behavior here.  Instead, we can note that there are no known negatives to this behavior, and there are definite positives, such as increased security.

## Message-Authentication-Code

Similarly, the Message-Authentication-Code attribute defined in {{RFC6218, Section 3.3}} MUST NOT be sent over a RADIUS/1.1 connection.  If it is received in a packet, it MUST be treated as "invalid attribute" as defined in {{RFC6929, Section 2.8}}.

As the Message-Authentication-Code attribute is no longer used in RADIUS/1.1, the related MAC-Randomizer attribute {{RFC6218, Section 3.2}} MUST NOT be sent over a RADIUS/1.1 connection.  If it is received in a packet, it MUST be treated as "invalid attribute" as defined in {{RFC6929, Section 2.8}}.

## CHAP, MS-CHAP, etc.

While some attributes such as CHAP-Password, etc. depend on insecure cryptographic primitives such as MD5, these attributes are treated as opaque blobs when sent between a RADIUS client and server.  The contents of the attributes are not obfuscated, and they do not depend on the RADIUS shared secret.  As a result, these attributes are unchanged in RADIUS/1.1.

A server implementing this specification can proxy CHAP, MS-CHAP, etc. without any issue.  A home server implementing this specification can authenticate CHAP, MS-CHAP, etc. without any issue.

## Original-Packet-Code

{{RFC7930, Section 4}} defines an Original-Packet-Code attribute.  This attribute is needed because otherwise it is impossible to correlate the Protocol-Error response packet with a particular request packet.  The definition in {{RFC7930, Section 4}} describes the reasoning behind this need:

> The Original-Packet-Code contains the code
> from the request that generated the protocol error so that clients
> can disambiguate requests with different codes and the same ID.

This attribute is no longer needed in RADIUS/1.1.  The Identifier field is unused, so it impossible for two requests to have the "same" ID.  Instead, the Token field permits clients and servers to correlate requests and responses, independent of the Code value being used.

Therefore, the Original-Packet-Code attribute ({{RFC7930, Section 4}}) MUST NOT be sent over a RADIUS/1.1 connection.  If it is received in a packet, it MUST be treated as "invalid attribute" as defined in {{RFC6929, Section 2.8}}.

# Other Considerations when using ALPN

Most of the differences between RADIUS and RADIUS/1.1 are in the packet header and attribute handling, as discussed above.  The remaining issues are a small set of unrelated topics, and are discussed here.

## Protocol-Error

There are a number of situations where a RADIUS server is unable to respond to a request.  One situation is where the server depends on a database, and the database is down.  While arguably the server should close all incoming connections when it is unable to do anything, this action is not always effective.  A client may aggressively try to open new connections, or send packets to an unconnected UDP destination where the server is not listening.  Another situation where the server is unable to respond is when the server is proxying packets, and the outbound connections are either full or failed.

In all RADIUS spercifications prior to this one, there is no way for the server to send a client the positive signal that it received a request, but is unable to send a response.  Instead, the server usually just discards the request, which to the client is indistinguishable from the situation where the server is down.  This failure case is made worse by the fact that perhaps some proxied packets succeed while others fail.  The client can only conclude then that the server is randomly dropping packets, and is unreliable.

It would be very useful for servers to signal to clients that they have received a request, but are unable to process it.  This specification uses the Protocol-Error packet {{RFC7930, Section 4}} as that signal.  The use of Protocol-Error allows for both hop-by-hop signaling in the case of proxy forwarding errors, and also for end-to-end signaling of server to client.  Such signaling should greatly improve the robustness of the RADIUS protocol.

When a RADIUS/1.1 server determines that it is unable to process an Access-Request or Accounting-Request packet, it MUST respond with a Protocol-Error packet containing an Error-Cause attribute.  A proxy which cannot forward the packet MUST respond with either 502 (Request Not Routable (Proxy)), or 505 (Other Proxy Processing Error).  This requirement is to help distinguish failures in the proxy chain from failures at the home server,

For a home server, if none of the Error-Cause values match the reason for the failure, then the value 506 (Resources Unavailable) MUST be used.

When a RADIUS proxy receives a Protocol-Error reply, it MUST examine the value of the Error-Cause attribute.  If there is no Error-Cause attribute, or its value is something other than 502 (Request Not Routable (Proxy)), 505 (Other Proxy Processing Error), or 506 (Resources Unavailable), the proxy MUST return the Protocol-Error response packet to the client, and include the Error-Cause attribute from the response it received.  This process allows for full "end to end" signaling of servers to clients.

In all situations other then outlined in the preceding paragraph, a client which receives a Protocol-Error reply MUST re-process the original outgoing packet through the client forwarding algorithm.  This requirement includes both clients which originate RADIUS traffic, and proxies which see an Error-Cause attribute of 502 (Request Not Routable (Proxy)), or 505 (Other Proxy Processing Error).

The expected result of this processing is that the client forwards the packet to a different server.  Clients MUST NOT forward the packet over the same connection, and SHOULD NOT forward it to over a different connection to the same server.

This process may continue over multiple connections and multiple servers, until the client either times out the request, or fails to find a forwarding destination for the packet.  A proxy which is unable to forward a packet MUST reply with a Protocol-Error packet containing Error-Cause, as defined above.  A client which originates packets MUST treat such a request as if it had received no response.

This behavior is intended to improve the stability of the RADIUS protocol by addressing issues first raised in {{RFC3539, Section 2.8}}.

## Status-Server

{{RFC6613, Section 2.6.5}}, and by extension {{RFC7360}}, suggest that the Identifier value zero (0) be reserved for use with Status-Server as an application-layer watchdog.  This practice MUST NOT be used for RADIUS/1.1, as the Identifier field is not used in this transport profile.

The rationale for reserving one value of the Identifier field was the limited number of Identifiers available (256), and the overlap in Identifiers between Access-Request packets and Status-Server packets.  If all 256 Identifier values had been used to send Access-Request packets, then there would be no Identifier value available for sending a Status-Server packet.

In contrast, the Token field allows for 2^32 outstanding packets on one RADIUS/1.1 connection.  If there is a need to send a Status-Server packet, it is nearly always possible to allocate a new value for the Token field.  If instead there are 2^32 outstanding packets for one connection, then it is likely that something has gone catastrophically wrong.  In that case, the safest way forward is likely to just close the connection.

## Proxies

A RADIUS proxy normally decodes and then re-encodes all attributes, included obfuscated ones.  A RADIUS proxy will not generally rewrite the content of the attributes it proxies (unless site-local policy requires such a rewrite).  While some attributes may be modified due to administrative or policy rules on the proxy, the proxy will generally not rewrite the contents of attributes such as User-Password, Tunnel-Password, CHAP-Password, MS-CHAP-Password, MS-MPPE keys, etc.  All attributes are therefore transported through a RADIUS/1.1 connection without changing their values or contents.

A proxy may negotiate RADIUS/1.1 (or not) with a particular client or clients, and it may negotiate RADIUS/1.1 (or not) with a server or servers it connect to, in any combination.  As a result, this specification is fully compatible with all past, present, and future RADIUS attributes.

# Other RADIUS Considerations

This section discusses issues in RADIUS which need to be addressed in order to support ALPN, but which aren't direcly part of the RADIUS/1.1 protocol.

## Crypto-Agility

The crypto-agility requirements of {{RFC6421}} are addressed in {{RFC6614, Appendix C}}, and in {{RFC7360, Section 10.1}}.  This specification makes no changes from, or additions to, those specifications.  The use of ALPN, and the removal of MD5 has no impact on security or privacy of the protocol.

RADIUS/TLS has been widely deployed in at least eduroam {{RFC7593}} and {{EDUROAM}} and in OpenRoaming {{OPENROAMING}}.  RADIUS/DTLS has seen less adoption, but it is known to be supported in many RADIUS clients and servers.

It is RECOMMENDED that all implementations of historic RADIUS/TLS be updated to support this specification.  Where a system already implements RADIUS over TLS, the additional effort to implement this specification is minimal.  Once implementations support it, administrators can gain the benefit of it with little or no configuration changes.  This specification is backwards compatible with {{RFC6614}} and {{RFC7360}}.  It is only potentially subject to down-bidding attacks if implementations do not enforce ALPN negotiation correctly on session resumption.

All crypto-agility needed or used by this specification is implemented in TLS.  This specification also removes all cryptographic primitives from the application-layer protocol (RADIUS) being transported by TLS.  As discussed in the following section, this specification also bans the development of all new cryptographic or crypto-agility methods in the RADIUS protocol.

## Error-Cause Attribute

The Error-Cause attribute is defined in {{RFC5176}}. The "Table of Attributes" section given in {{RFC5176, Section 3.6}} permits that attribute to appear in CoA-NAK and Disconnect-NAK packets.  As no other packet type is listed, the implication is that the Error-Cause attribute cannot appear in any other packet.  {{RFC7930}} also permits Error-Cause to appear in Protocol-Error packets.

However, {{?RFC5080, Section 2.6.1}} suggests that Error-Cause may appear in Access-Reject packets.  No explanation is given for this change from {{RFC5176}}.  There is not even an acknowledgment that this suggestion is a change from any previous specification.  We correct that issue here.

This specification updates {{RFC5176}} to allow the Error-Cause attribute to appear in Access-Reject packets.  It is RECOMMENDED that implementations include the Error-Cause attribute in Access-Reject packets where appropriate.

That is, the reason for sending the Access-Reject packet (or Protocol-Error packet) may match a defined Error-Cause value.  In that case, it is useful for implementations to send an Error-Cause attribute with that value.  This behavior can help RADIUS system administrators debug issues in complex proxy chains.

For example, a proxy may normally forward Access-Request packets which contain EAP-Message attributes.  The proxy can determine if the contents of the EAP-Message are invalid, for example if the first octet has value larger than 4.  In that case, there may be no benefit to forwarding the packet, as the home server will reject it.  It may then then possible for the proxy (with the knowledge and consent of involved parties) to immediately reply with an Access-Reject containing an Error-Cause attribute with value 202 for "Invalid EAP Packet (Ignored)".

Another possibility is that if a proxy is configured to forward packets for a particular realm, but it has determined that there are no available connections to the next hop for that realm.  In that case, it may be possible for the proxy (again with the knowledge and consent of involved parties) to reply with an Access-Reject containing an Error-Cause attribute with value 502 for "Request Not Routable (Proxy)"

These examples are given only for illustrative and informational purposes.  While it is useful to return an informative value for the Error-Cause attribute, proxies can only modify the traffic they forward with the explicit knowledge and consent of all involved parties.

## Future Standards

Future work may define new attributes, packet types, etc.  It is important to be able to do such work without requiring that every new standard mention RADIUS/1.1 explicitly.  This document defines RADIUS/1.1 as having functional overlap with legacy RADIUS: the protocol state machine is unchanged, the packet header Code field is unchanged, and the attribute format is largely unchanged.  As a result, any new packet Code or attribute defined for RADIUS is explicitly compatible with RADIUS/1.1: the field contents and meanings are identical.  The only difference between the two protocols is that obfuscated attributes in RADIUS are not obfuscated in RADIUS/1.1, and this document defines how that mapping is done.

Any future specification needs to mention RADIUS/1.1 only if it adds fields to the RADIUS/1.1 packet header.  Otherwise, transport considerations for RADIUS/1.1 are identical to RADIUS over (D)TLS.

We reiterate that this specification defines a new transport profile for RADIUS.  It does not define a completely new protocol.  Any future specification which defines a new attribute MUST define it for RADIUS/UDP first, after which those definitions can be applied to this transport profile.

New specifications MAY define new attributes which use the obfuscation methods for User-Password as defined in {{RFC2865, Section 5.2}}, or for Tunnel-Password as defined in {{RFC2868, Section 3.5}}.  There is no need for those specifications to describe how those new attributes are transported in RADIUS/1.1.  Since RADIUS/1.1 does not use MD5, any obfuscated attributes will by definition be transported as their underlying data type, "text" ({{RFC8044, Section 3.4}}) or "string" ({{RFC8044, Section 3.5}}).

New RADIUS specifications MUST NOT define attributes which can only be transported via RADIUS over TLS.  The RADIUS protocol has no way to signal the security requirements of individual attributes.  Any existing implementation will handle these new attributes as "invalid attributes" ({{RFC6929, Section 2.8}}), and could forward them over an insecure link.  As RADIUS security and signaling is hop-by-hop, there is no way for a RADIUS client or server to even know if such forwarding is taking place.  For these reasons and more, it is therefore inappropriate to define new attributes which are only secure if they use a secure transport layer.

The result is that specifications do not need to mention this transport profile, or make any special provisions for dealing with it.  This specification defines how RADIUS packet encoding, decoding, signing, and verification are performed when using RADIUS/1.1.  So long as any future specification uses the existing encoding, etc. schemes defined for RADIUS, no additional text in future documents is necessary in order to be compatible with RADIUS/1.1.

We note that it is theoretically possible for future standards to define new cryptographic primitives for use with RADIUS/UDP.  In that case, those documents would likely have to describe how to transport that data in RADIUS/1.1.  We believe that such standards are unlikely to be published, as other efforts in the RADEXT working group are forbidding such updates to RADIUS.

# Implementation Status

(This section to be removed by the RFC editor.)

This specification is being implemented (client and server) in the FreeRADIUS project which is hosted on GitHub at https://github.com/FreeRADIUS/freeradius-server/tree/v3.2.x  The code implementation "diff" is approximately 1,000 lines, including build system changes and changes to configuration parsers.

# Privacy Considerations

This specification requires secure transport for RADIUS.  RADIUS/1.1. has all of the privacy benefits of RADIUS/TLS {{RFC6614}} and RADIUS/DTLS {{RFC7360}}, and none of the privacy or security issues of RADIUS/UDP {{RFC2865}} or RADIUS/TCP {{RFC6613}}.

# Security Considerations

The primary focus of this document is addressing security considerations for RADIUS.  This specification relies on TLS and associated ALPN negotiation for much of its security.  We refer the reader to {{?RFC8446}} and {{RFC7360}} for discussions of the security of those protocols.  The discussion in this section is limited to issues unique to this specification.

Implementations should rely on the underlying TLS library to perform ALPN version negotiation.  That is, implementations should supply a list of permitted ALPN strings to the TLS library, and let it return the negotiated value.

There are few other opportunities for security issues.  If an implementation gets ALPN negotiation wrong, then the wrong application data will be transported inside of TLS.  While RADIUS/1.0 and RADIUS/1.1 share similar packet formats, the protocols are not mutually compatible.

RADIUS/1.0 requests sent over a RADIUS/1.1 connection may be accepted by the RADIUS/1.1 server, as the server will ignore the ID field, and try to use portions of the Request Authenticator as a Token.  However, the reply from the RADIUS/1.1 server will fail the Response Authenticator validation by the RADIUS/1.0 client.  The responses will therefore be dropped.  The client will generally log these failures, and an administrator will address the issue.

RADIUS/1.1 requests sent over a RADIUS/1.0 connection will generally be discarded the the RADIUS/1.0 server, as the packets will fail the Request Authenticator checks.  That is, all request packets such as Accounting-Request, CoA-Request, and Disconnect-Request will be discarded by the server.  For Access-Request packets containing EAP-Message, the packets will be missing Message-Authenticator, and will therefore be discarded by the server.  Other Access-Request packets contain obfuscated attributes such as User-Password will have those attributes decoded to nonsense, and will thus result in Access-Reject responses.

RADIUS/1.1 Access-Request packets containing non-obfuscated attributes such as CHAP-Password may be accepted by a RADIUS/1.0 server but the response will contain a Response Authenticator (i.e. MD5 hash), and not a Token which matches the Token in the request.  A similar analysis applies for Access-Request packets containing Service-Type = Authorize-Only.

In conclusion, any mismatch of versions between client and server will result in most request packets being discarded by the server, and all response packets being discarded by the client.  The two protocols are therefore incompatible, and safe from misconfigurations or erroneous implementations.

# IANA Considerations

IANA is requested to update the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry with two new entries:

~~~~
Protocol: RADIUS/1.0
Id. Sequence: 0x72 0x61 0x64 0x69 0x75 0x73 0x2f 0x31 0x2e 0x30
    ("radius/1.0")
Reference:  This document

Protocol: RADIUS/1.1
Id. Sequence: 0x72 0x61 0x64 0x69 0x75 0x73 0x2f 0x31 0x2e 0x31
    ("radius/1.1")
Reference:  This document
~~~~

# Acknowledgments

In hindsight, the decision to retain MD5 for historic RADIUS/TLS was likely wrong.  It was an easy decision to make in the short term, but it has caused ongoing problems which this document addresses.

Thanks to Bernard Aboba, Karri Huhtanen, Heikki Vatiainen, Alexander Clouter, Michael Richardson, Hannes Tschofenig, Matthew Newton, and Josh Howlett for reviews and feedback.

# Changelog

(This section to be removed by the RFC editor.)

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

draft-dekok-radext-radiusv11-04

>  Remove "radius/1.0" as it is unnecessary.
>
>  Update Introduction with more historical background, which motivates the rest of the section.
>
>  Change Identifier field to be reserved, as it is entirely unused.
>
>  Update discussion on clear text passwords.
>
> Clarify discussion of Status-Server, User-Password, and Tunnel-Password.
>
> Give high level summary of ALPN, clear up client / server roles, and remove "radius/1.0" as it is unnecessary.
>
> Add text on RFC6421.

draft-dekok-radext-radiusv11-05

> Clarify naming.  "radius/1.1" is the ALPN name.  "RADIUS/1.1" is the transport profile.
>
> Clarify that future specifications do not need to make provisions for dealing with this transport profile.

> Typos and word smithing.
>
> Define and use "RADIUS over TLS" instead of RADIUS/(D)TLS.
>
> Many cleanups and rework based on feedback from Matthew Newton.

draft-ietf-radext-radiusv11-00

> No changes from previous draft.

draft-ietf-radext-radiusv11-01

> Move to "experimental" based on WG feedback.
>
> Many cleanups based on review from Matthew Newton
>
> Removed requirement for supporting TLS-PSK.
>
> This document does not deprecate new cryptographic work in RADIUS.  The "deprecating insecure transports" document does that.

draft-ietf-radext-radiusv11-02

> Note that we also update RFC 7930
>
> Minor updates to text.
>
> Add text explaining why "allow" is the default, and how to upgrade to "require"
>
> Discuss the use of the TLS alert "no_application_protocol" (120), and its limitations.
>
> Suggest the use of Protocol-Error as an application signal when it is not possible to send a "no_application_protocol" TLS alert.
>
> Update discussion of Message-Authenticator, and suggest that RADIUS/1.1 proxies always add Message-Authenticator to Access-Request packets being sent over UDP or TCP.
>
> Add term "historic RADIUS/TLS" as it is simpler than more awkward "6614 or 7360".
>
> Re-add ALPN "radius/1.0" based on comments from Heikki.  It signals that the system is ALPN-capable, among other.

draft-ietf-radext-radiusv11-03

>  Rename to "RADIUS ALPN and removing MD5"
>
> Add a few things missed when re-adding "radius/1.0"
>
> Clarify wording in a number of places.

draft-ietf-radext-radiusv11-04

> Address github issues 3..9

draft-ietf-radext-radiusv11-05

> Add discussion of Protocol-Error as per-link signalling.

draft-ietf-radext-radiusv11-06

> Review from Paul Wouters
>
> Clarify client handling of Protocol-Error

draft-ietf-radext-radiusv11-07

> Clarifications as requested by Jan-Frederik

draft-ietf-radext-radiusv11-08

> Review from Claudio Allocchio

draft-ietf-radext-radiusv11-09

> Review from Barry Lieba

--- back
