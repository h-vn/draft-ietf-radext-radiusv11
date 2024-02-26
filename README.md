# RADIUS/(D)TLS Version 1.1.

This document defines RADIUS version 1.1.  It is a transport profile for RADIUS which requires TLS, and does not use MD5 for packet signing or encrypting attributes.

Its is fully compatible with normal RADIUS, and can transport all RADIUS attributes, including CHAP, MS-CHAP, etc.

This document is for the IETF RADEXT WG
http://datatracker.ietf.org/wg/radext

The IETF datatracker page for this document is https://datatracker.ietf.org/doc/draft-dekok-radext-radiusv11/

## RADIUS and FIPS-140

Historical RADIUS over UDP is not FIPS compatible.  This version can be used in a FIPS compatible environment.

## Interoperability

GPLv2 code for this implementation is available at https://github.com/FreeRADIUS/freeradius-server/tree/v3.2.x

You will need to pass an extra option when building it:

```
./configure --with-radiusv11
```

In order to use it, the following configurations have to be updated:

```
client foo {
	...
	radiusv1_1 = allow
}
...

home_server bar {
	...
	tls {
		...
		radiusv1_1 = allow
	}
}
...

listen {
	...
	tls {
		...
		radiusv1_1 = allow
	}
}
```
