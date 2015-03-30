---
layout: post
title: Using pip behind a corporate firewall
---

Yesterday I was going to try out [Fabric][1] framework as a possible alternative to our current capistrano setup at work. It turned out to be a long and complex yak shaving exercise.

There are several parts to the tale. The first order of business was to get `pip` installed on a CentOS 6.5 Minimal vm. `pip` is not included in the default yum repository, so I needed to add the Extra Packages for Enterprise Linux ([EPEL][2]) repository. The default configuration of the EPEL repository didn't work after installation. After having solved that problem, `pip` complained about SSL certificate problems when asked to install the fabric package. The issue turned out to be related to the EPEL issue: A corporate firewall that replaces the SSL certificate on all HTTPS traffic.

How SSL intercepting firewalls work
===================================
The root of the problem is pretty well explained in the answer to [this question on Stack Exchange][3]. Some companies have firewalls that replace the server certificates used in https communication with self signed variants. When an SSL or TSL handshake is being negotiated with a server, the proxy intercepts the server certificate and replaces it with a self signed certificate. This way, the communication between the client and the proxy is made with a server certificate that is owned by the proxy operator, while the communication between the proxy and the server is made using the original server certificate.
The proxy can decrypt the data sent from the client using the proxy certificate, and re-encrypts its own certificate before it is sent to the server. Likewise, the response can be decrypted by the proxy, reencrypted using its own certificate and then forwarded to the client. [ZDNet has a good article on the subject too][4].

Configuring EPEL
================

I needed to experiment with Fabric at work, and was blisfully unaware of the SSL issue I was about to run into. Since `pip` was not in the default `yum` repository, I needed to add the EPEL repository. The method I chose was to install it using yum.

```bash
yum install epel-release
```

The process was smooth, and I could immediately use the repository. But when I tried to install `pip` with the command `gem install pip`, I got an error message.

```
aded plugins: fastestmirror, protectbase, security
Loading mirror speeds from cached hostfile
Error: Cannot retrieve metalink for repository: epel. Please verify its path and try again
```


Updating certificates - take one
================================

A bit of digging suggested that the problem was outdated ssl root certificates, and that it could be fixed using the command `yum reinstall ca-certificates`. The command was successful, but I still got the same error message. I tried manually retrieving a new certificate bundle.

```bash
cp /etc/pki/tls/certs/ca-bundle.crt /etc/pki/tls/certs/ca-bundle.crt.bak
curl http://curl.haxx.se/ca/cacert.pem -o /etc/pki/tls/certs/ca-bundle.crt
```

This didn't help either. After a while I gave up, and fixed the yum repository paths to use http instead of https by changing the `mirrorlist` parameters in `/etc/yum.repos.d/epel.repo`. This solved the immediate problem by letting me install `pip`, but left a bad taste in my mouth.

Having installed `pip` it was jus a question of installing `fabric`. I entered the command `pip install fabric`, but quickly got another error message:

```
Downloading/unpacking fabric
  Could not fetch URL https://pypi.python.org/simple/fabric/: There was a problem confirming the ssl certificate: <urlopen error [Errno 1] _ssl.c:492: error:14090086:SSL routines:SSL3_GET_SERVER_CERTIFICATE:certificate verify failed>
  Will skip URL https://pypi.python.org/simple/fabric/ when looking for download links for fabric
  ...
No distributions at all found for fabric
Storing complete log in /root/.pip/pip.log
```

Which certificate is actually used?
===================================

Ok, so I really did need to figure out how to solve the certificate problem. After much digging, I found a way to check the certificate that was used in the SSL communication.

```
openssl s_client -connect pypi.python.org:443
```

On a machine outside the company network, the follwoing is returned:

```
CONNECTED(00000003)
depth=1 /C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert SHA2 Extended Validation Server CA
verify error:num=20:unable to get local issuer certificate
verify return:0
---
Certificate chain
 0 s:/businessCategory=Private Organization/1.3.6.1.4.1.311.60.2.1.3=US/1.3.6.1.4.1.311.60.2.1.2=Delaware/serialNumber=3359300/street=16 Allen Rd/postalCode=03894-4801/C=US/ST=NH/L=Wolfeboro,/O=Python Software Foundation/CN=www.python.org
   i:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert SHA2 Extended Validation Server CA
 1 s:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert SHA2 Extended Validation Server CA
   i:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert High Assurance EV Root CA
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIG2jCCBcKgAwIBAgIQAbtvABIrF382yrSc6otrJjANBgkqhkiG9w0BAQsFA...
...HmpqlkA2AvjdvSvwnODux3QPbMucIaJXrUUwf
-----END CERTIFICATE-----
subject=/businessCategory=Private Organization/1.3.6.1.4.1.311.60.2.1.3=US/1.3.6.1.4.1.311.60.2.1.2=Delaware/serialNumber=3359300/street=16 Allen Rd/postalCode=03894-4801/C=US/ST=NH/L=Wolfeboro,/O=Python Software Foundation/CN=www.python.org
issuer=/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert SHA2 Extended Validation Server CA
---
No client certificate CA names sent
---
SSL handshake has read 3140 bytes and written 456 bytes
---
New, TLSv1/SSLv3, Cipher is AES128-SHA
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
SSL-Session:
    Protocol  : TLSv1
    Cipher    : AES128-SHA
    Session-ID: BC01C2354CE725DF4D6AEF5F1E484B2F2F6EC349C082225D8D059C7158250ADA
    Session-ID-ctx:
    Master-Key: 3DEDD6577224BB55D4EAF14A544261F7904BEF5C31E4E7132876101A32F6F15341F5B1EB6A9057E7CD484ED5D9095201
    Key-Arg   : None
    Start Time: 1426456062
    Timeout   : 300 (sec)
    Verify return code: 0 (ok)
---
```

This long blurp tells us that the server identifies itself with a certificate owned by Python Software Foundation, and that the certificate is issued by DigiCert (issuer=/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert SHA2 Extended Validation Server CA).

But when I ran it from my VM at work, the result was a bit different:

```
Server certificate
subject=/businessCategory=Private Organization/1.3.6.1.4.1.311.60.2.1.3=US/1.3.6.1.4.1.311.60.2.1.2=Delaware/serialNumber=3359300/street=16 Allen Rd/postalCode=03894-4801/C=US/ST=NH/L=Wolfeboro,/O=Python Software Foundation/CN=www.python.org
issuer=/CN=company ca
```

This snippet of the output tells me that the Python Software Foundation certifiate is one issued by my workplace (issuer=/CN=company ca), not the original one. A bit more Googling and I had the explenation: The companys SSL intercepting firewall.

I now had a reasonable idea of what I needed to do to fix the problem: I had to extract the fake certificate, and all certificates in its certificate chain. I then needed to add them to the trusted certificate bundle for the server.

The first part is simple, since all the certificates are included in `-----BEGIN CERTIFICATE-----` `-----END CERTIFICATE-----` sections in the openssl command output. Just copy each section to a text file and give it an appropriate name. The certificate file for the original text file would contain the following:

```
-----BEGIN CERTIFICATE-----
MIIG2jCCBcKgAwIBAgIQAbtvABIrF382yrSc6otrJjANBgkqhkiG9w0BAQsFADB1
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMTQwMgYDVQQDEytEaWdpQ2VydCBTSEEyIEV4dGVuZGVk
IFZhbGlkYXRpb24gU2VydmVyIENBMB4XDTE0MDkwNTAwMDAwMFoXDTE2MDkwOTEy
MDAwMFowgfkxHTAbBgNVBA8TFFByaXZhdGUgT3JnYW5pemF0aW9uMRMwEQYLKwYB
BAGCNzwCAQMTAlVTMRkwFwYLKwYBBAGCNzwCAQITCERlbGF3YXJlMRAwDgYDVQQF
EwczMzU5MzAwMRQwEgYDVQQJEwsxNiBBbGxlbiBSZDETMBEGA1UEERMKMDM4OTQt
NDgwMTELMAkGA1UEBhMCVVMxCzAJBgNVBAgTAk5IMRMwEQYDVQQHEwpXb2xmZWJv
cm8sMSMwIQYDVQQKExpQeXRob24gU29mdHdhcmUgRm91bmRhdGlvbjEXMBUGA1UE
AxMOd3d3LnB5dGhvbi5vcmcwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIB
AQCtUnfHpOteoIqZxGsaR/2tIenj0+pBtNBiWT6PlYLLXC6MNRjFwtnhRzEVanAm
GEEOEQwUokYZHw8kCL2SIZ1DFI5IIFyhTFql1dqiKtoQse0LAZlUHscVxn9OZyWM
DA4JZ6A4c3/j5SA9hGO3+KyTc95GfiEXqkSkmjH3aBtY2flr+H1fvatQA8AIAD5k
weQLFbbqi33Uvf4sJ3OhY63Kf1ZWteXSeCT+FRMlFTaYbauo86AmU9X2/b85wold
naUO3VjcGjTSoSuaxtWuHFRxpOTBG7bqPbtWk+X5l+rjsIoGJ6ZrRFbAtHqG+S3v
luEG9FtgGAo+3hKm99U8UKKVAgMBAAGjggLfMIIC2zAfBgNVHSMEGDAWgBQ901Cl
1qCt7vNKYApl0yHU+PjWDzAdBgNVHQ4EFgQUTWfmKThuIBhkZX4B3yNf+DpBqokw
ggEUBgNVHREEggELMIIBB4IOd3d3LnB5dGhvbi5vcmeCCnB5dGhvbi5vcmeCD3B5
cGkucHl0aG9uLm9yZ4IPZG9jcy5weXRob24ub3JnghN0ZXN0cHlwaS5weXRob24u
b3Jngg9idWdzLnB5dGhvbi5vcmeCD3dpa2kucHl0aG9uLm9yZ4INaGcucHl0aG9u
Lm9yZ4IPbWFpbC5weXRob24ub3JnghRwYWNrYWdpbmcucHl0aG9uLm9yZ4IQcHl0
aG9uaG9zdGVkLm9yZ4IUd3d3LnB5dGhvbmhvc3RlZC5vcmeCFXRlc3QucHl0aG9u
aG9zdGVkLm9yZ4IMdXMucHljb24ub3Jngg1pZC5weXRob24ub3JnMA4GA1UdDwEB
/wQEAwIFoDAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwdQYDVR0fBG4w
bDA0oDKgMIYuaHR0cDovL2NybDMuZGlnaWNlcnQuY29tL3NoYTItZXYtc2VydmVy
LWcxLmNybDA0oDKgMIYuaHR0cDovL2NybDQuZGlnaWNlcnQuY29tL3NoYTItZXYt
c2VydmVyLWcxLmNybDBCBgNVHSAEOzA5MDcGCWCGSAGG/WwCATAqMCgGCCsGAQUF
BwIBFhxodHRwczovL3d3dy5kaWdpY2VydC5jb20vQ1BTMIGIBggrBgEFBQcBAQR8
MHowJAYIKwYBBQUHMAGGGGh0dHA6Ly9vY3NwLmRpZ2ljZXJ0LmNvbTBSBggrBgEF
BQcwAoZGaHR0cDovL2NhY2VydHMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0U0hBMkV4
dGVuZGVkVmFsaWRhdGlvblNlcnZlckNBLmNydDAMBgNVHRMBAf8EAjAAMA0GCSqG
SIb3DQEBCwUAA4IBAQBsTgMOFUP8wHVpgCzm/fQTrKp4nxcb9m9gkTW1aRKuhlAY
g/CUQ8DC0Ii1XqOolTmGi6NIyX2Xf+RWqh7UzK+Q30Y2RGGb/47uZaif9WaIlKGn
40D1mzzyGjrfTMSSFlrtwyg/3yM8KN800Cz5HgXnHD2qIuYcYqXRRS6E7PEHB1Dm
h72iCAHYwUTgfcfqUWVEZ26EQhP4Lk4+hs2UJsAUnMWj7/bnk8LR/KZumLuuv3RK
lmR1Qg+9AChafiCCFra1UxfgznvF5ocJzr6nNmYc6k1ImaipRq7c/OuwUTTqNqR2
FceHmpqlkA2AvjdvSvwnODux3QPbMucIaJXrUUwf
-----END CERTIFICATE-----
```

Updating the trusted certificate stores
=======================================

Now that I had the certificates, I needed to add them to the machines list of trusted certificates. It turned out that there are two of these: One for the operating system, and a separate one for `pip`.

To add the certificate to the CentOS SSL CA certificate bundle, you need to modify `/etc/pki/tls/certs/ca-bundle.crt`.
It can be done by hand, but a more secure way to do it is to store the individual certificates in `/etc/pki/ca-trust/source/anchors/` and then run

```
update-ca-trust enable
update-ca-trust extract
```

[The process is different][5] depending on which operating system your are using.



[1]: http://www.fabfile.org/
[2]: https://fedoraproject.org/wiki/EPEL
[3]: http://security.stackexchange.com/questions/3778/getting-self-signed-ssl-certificates-for-all-https-connections-made-from-some-pr
[4]: http://www.zdnet.com/article/how-the-nsa-and-your-boss-can-intercept-and-break-ssl/
[5]: http://kb.kerio.com/product/kerio-connect/server-configuration/ssl-certificates/adding-trusted-root-certificates-to-the-server-1605.html