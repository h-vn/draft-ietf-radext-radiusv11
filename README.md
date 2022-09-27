# Secure RADIUS

This document defines Secure RADIUS (SRADIUS).  It is a transport profile for RADIUS which requires TLS, and does not use MD5.

This document is for the IETF RADEXT WG
http://datatracker.ietf.org/wg/radext

## RADIUS and FIPS-140

RADIUS is not FIPS compatible.  This document defines a variant of RADIUS - SRADIUS, which can be used in a FIPS compatible environment.