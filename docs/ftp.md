# FTP

The FTP server implementation supports [RFC 959](https://datatracker.ietf.org/doc/html/rfc959){:target="_blank"}.

Both password and client certificate authentication are supported.

The following extensions are implemented:

- [AUTH](https://tools.ietf.org/html/rfc2228#page-6){:target="_blank"} - Control session protection
- [AUTH TLS](https://tools.ietf.org/html/rfc4217#section-4.1){:target="_blank"} - TLS session
- [PROT](https://tools.ietf.org/html/rfc2228#page-8){:target="_blank"} - Transfer protection
- [EPRT/EPSV](https://tools.ietf.org/html/rfc2428){:target="_blank"} - IPv6 support
- [MDTM](https://tools.ietf.org/html/rfc3659#page-8){:target="_blank"} - File Modification Time
- [SIZE](https://tools.ietf.org/html/rfc3659#page-11){:target="_blank"} - Size of a file
- [REST](https://tools.ietf.org/html/rfc3659#page-13){:target="_blank"} - Restart of interrupted transfer
- [MLST](https://tools.ietf.org/html/rfc3659#page-23){:target="_blank"} - Simple file listing for machine processing
- [MLSD](https://tools.ietf.org/html/rfc3659#page-23){:target="_blank"} - Directory listing for machine processing
- [HASH](https://tools.ietf.org/html/draft-bryan-ftpext-hash-02){:target="_blank"} - Hashing of files
- [AVLB](https://tools.ietf.org/html/draft-peterson-streamlined-ftp-command-extensions-10#section-4){:target="_blank"} - Available space
- [COMB](htps://help.globalscape.com/help/archive/eft6-4/mergedprojects/eft/allowingmultiparttransferscomb_command.htm){:target="_blank"} - Combine files
