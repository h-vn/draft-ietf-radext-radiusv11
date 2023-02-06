# RADIUS/(D)TLS Version 1.1.

This document defines RADIUS version 1.1.  It is a transport profile for RADIUS which requires TLS, and does not use MD5 for packet signing or encrypting attributes.

Its is fully compatible with normal RADIUS, and can transport all RADIUS attributes, including CHAP, MS-CHAP, etc.

This document is for the IETF RADEXT WG
http://datatracker.ietf.org/wg/radext

## RADIUS and FIPS-140

Historical RADIUS is not FIPS compatible.  This version can be used in a FIPS compatible environment.
