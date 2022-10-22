---
layout: post
title: Wireshark TLS Decryption
categories: [Cryptography, Networking, Wireshark]
---

The first step in many desktop network troubleshooting situations is to open a tool such as [Wireshark](https://www.wireshark.org/) to view live or recorded network traffic. Wireshark is traditionally most useful for lower level protocols, or those that send data without encryption - in 'plain text.'

Many higher-level protocols are moving towards more and more encryption, however. With the rise of [zero-trust security models](https://www.cloudflare.com/learning/security/glossary/what-is-zero-trust/), many companies are moving towards encryption schemes such as TLS to secure connections towards their network resources. Inspecting and troubleshooting traffic used in modern infrastructures is becoming more difficult, due to the design and security model of user-facing applications.

Even traditional protocols such as DNS aren't necessarily easily inspected. Browsers such as [Mozilla Firefox are starting to roll out DoH, DNS over HTTPS](https://support.mozilla.org/en-US/kb/firefox-dns-over-https), as their main method of resolving network addresses. With the assumption that many devices will use the DNS server assigned to them by their ISP or company, using DoH to a supported external service such as [Google's Public DNS](https://developers.google.com/speed/public-dns) or [Cloudflare's 1.1.1.1 DNS resolver](https://www.cloudflare.com/learning/dns/what-is-1.1.1.1/) can bring easy confidentiality and integrity benefits to users both inside and outside their organization.

While this is great for users, it can be a nightmare for network professionals trying to get down to the root cause of an issue. With encryption comes both privacy *and* opacity. Luckily, applications embedding Chromium or curl (most of them) can make it easy for someone to decrypt their local TLS-encrypted traffic.

There are a few main steps:
1. Collecting decryption keys
2. Collecting network traffic
3. Viewing network traffic

# Collecting decryption keys

Set an environment variable `SSLKEYLOGFILE` to the location of a writable file. This file can be empty, existing, or even open by multiple applications at the same time.

It's that simple. For cooperating programs.

For a Windows-based example, this value could be set to `%USERPROFILE%\.sslkey.log` to put the keys in your user's home folder. This could be set within the main environment variable editor, the same one commonly used to adjust the system path, or for a specific application session through a command line.

In Powershell, for example, to log a new Chrome session's keys to the Desktop:

```ps1
$env:SSLKEYLOGFILE=$env:USERPROFILE+"\Desktop\ssl_keys.log"
& 'C:\Program Files\Google\Chrome\Application\chrome.exe'
```

## Windows
I found that at least the following programs write to this file when the environment variable is set (Tested on Windows 21H1 x64 Build 19044, versions below are the ones I tested with).

|Vendor|Application|Tested Version|
|-|-|-|
|Microsoft|Microsoft Edge|v99.0.1150.39 x64|
|Google|Google Chrome|v99.0.4844.51 x64|
|Mozilla|Firefox*|v98.0.1 x64|
|Spotify|Spotify for Windows|1.1.80.699.gc3dac750|
|Discord|Discord|Stable 118205 (081ceba) Host 1.0.9004 Windows 10 64-Bit (10.0.19044)|
|Valve|Steam**|Built: 2022-14-03 at 12:51:31, API: v020, Package versions: 1647446817|
|Cisco|Jabber|v12.9.3.54813 Build 304813|
|Microsoft|Visual Studio Code|1.67.0-insider (6f26fa1) with Chromium 98.0.4758.141|

\* Firefox's networking library also [supports many more environment](https://firefox-source-docs.mozilla.org/security/nss/legacy/reference/nss_environment_variables/index.html) variables that can be used to tune it's TLS systems, including being able to use a predicable PRNG seed.

\*\* May only be the website portions, as it embeds Chromium?

There's a theme here: **Programs using Chromium or libcurl for network traffic can dump their TLS session information for later decryption.**

## Linux, macOS

Neither have been tested by the author. Any application embeddeding Chromium or libcurl should be capable of this.

## Java

Many Java applications do not use Chromium or libcurl, and instead use the built-in [Java Secure Socket Extension (JSSE)](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/JSSERefGuide.html) library. It [has other ways](https://docs.oracle.com/javase/7/docs/technotes/guides/security/jsse/ReadDebug.html) to debug traffic. 

Users have [published open-source tools](https://github.com/neykov/extract-tls-secrets) to have the JVM dump variables to a key log file, however the JVM must be started with an explicit hook, rather than simply reading an environment variable.

## Disclaimers

- *Make sure this file is only readable by you, otherwise other users capturing packets on the system could decrypt your private network traffic! This could include plaintext passwords for banking, etc!*

- *Similarly, make sure you are aware this file is being constantly written to, and could be retrievable at any point by any system administrators, possibly without your knowledge.* **Network traffic could be captured at any point along the path, not just at your host, and combined with these keys later.**

- *Simply deleting files does not erase the data on many systems. It is notoriously hard to permanently delete data from a disk after it has been written for many reasons. Once both the keys and packet capture have been written to disk, the data should be considered available to any administrators of the machine, **forever**.*
  - One way to mitigate this may be to use a RAM-based disk, after disabling the system's swap/page file.

## libcurl

libcurl is a common network library for managing HTTP/HTTPS connections. Since it is used so commonly, this TLS inspection technique works for many programs. Not all.

For libcurl applications, [curl's manpage](https://curl.se/docs/manpage.html) has this to say about the variable:

> If you set this environment variable to a file name, curl will store TLS secrets from its connections in that file when invoked to enable you to analyze the TLS traffic in real time using network analyzing tools such as Wireshark. This works with the following TLS backends: OpenSSL, libressl, BoringSSL, GnuTLS, NSS and wolfSSL.

That last sentence tells use that most builds of `curl`/`libcurl` you see on a desktop computer would be configured with `SSLKEYLOGFILE` support. Some Windows builds of curl is built using Windows' built-in 'schannel' TLS library which does not support this feature.

# Collecting traffic

Any tool that can output a standard packet capture should work for Wireshark. Specifically, this could be done headlessly using `tcpdump` on a server, whose traffic could later be inspected in a desktop environment.

The easiest way would be to startup Wireshark before or after you startup your applications. If Wireshark is pointed to the `SSLKEYLOGFILE` before capture, it may live-update with decrypted packet dissections, as other applications update the file.

**Decryption can only occur when the whole TLS connection up to a specific packet is available. Notably, the TLS Handshake and all previous packets must be included in the packet catpure, up to the decryption target.**

# Viewing traffic

You can load in the `SSLKEYLOGFILE` within Wireshark's TLS Preferences. Put the file path within the `(Pre)-Master-Secret log filename` field.

![Wireshark > Preferences > Protocols > TLS](/images/2022-tls-wireshark-preferences.png)

After setting that up and collecting some traffic, it will automatically decode it for you within the dissector.

![Wireshark dissector showing decrypted HTTP/2 traffic to example.org](/images/2022-tls-wireshark-dissector.png)

# Additional References:

- [Wireshark's own documentation on using the (Pre)-Master-Secret](https://wiki.wireshark.org/TLS#using-the-pre-master-secret)
- [`SSLKEYLOGFILE` from `everything.curl.dev` by Daniel Stenberg, the creator and maintainer of curl. This could be considered `libcurl` documentation.](https://everything.curl.dev/usingcurl/tls/sslkeylogfile)
- [Article from Daniel Stenberg, the creator and maintainer of curl, on this topic (including historical support from the libcurl).](https://daniel.haxx.se/blog/2018/01/15/inspect-curls-tls-traffic/)
- [Excellent article about how Antivirus' like Avast have used this environment variable to MITM user traffic. Think about the "trusted" or "malicious" site indicators added to search engines.](https://textslashplain.com/2019/08/11/spying-on-https)
- [Description of the file format from Mozilla](https://firefox-source-docs.mozilla.org/security/nss/legacy/key_log_format/index.html)
- [Excellent slideshow-based tutorial by Peter Wu from SharFest '19 Europe. Includes a survey of applications/environments providing this method (slide 19), as well as alternate methods to inspect traffic.](https://lekensteyn.nl/files/wireshark-tls-debugging-sharkfest19eu.pdf)

# Additional Notes

## libcurl security
Setting `SSLKEYLOGFILE` to point to a single file, system-wide, would be a terrible idea. Not everyone may be able to read/write it, and everyone with packet-capturing permissions (by default: everyone on the machine) could decode anyone else's encrypted traffic - passwords and all.

The [libcurl security page](https://curl.se/libcurl/security.html) has this to say about running curl (or applications embedding libcurl) as root privileges with `SSLKEYLOGFILE` set:

> Giving setuid powers to the application means that libcurl can save files using those new rights (if for example the `SSLKEYLOGFILE` environment variable is set). Also: if the application wants these powers to read or manage secrets that the user is otherwise not able to view (like credentials for a login etc), it should be noted that libcurl still might understand proxy environment variables that allow the user to redirect libcurl operations to use a proxy controlled by the user.

## QUIC

libcurl also supports dumping encryption keys used for [QUIC](https://en.wikipedia.org/wiki/QUIC), a relatively newer protocol used for HTTP/3. This uses a separate environment variable as the protocol discards the traditional encrypted-TCP model, and instead uses UDP-based connections that multiplex different data streams across it. [Daniel Stenberg's blog](https://daniel.haxx.se/blog/2020/05/07/qlog-with-curl/) has more information on logging this.
