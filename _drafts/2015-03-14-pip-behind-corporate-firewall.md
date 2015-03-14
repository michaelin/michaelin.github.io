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

Installing `pip`
================

I needed to experiment with Fabric
This can only work if the client trusts the certificate the proxy uses to encrypt the




[1]: http://www.fabfile.org/
[2]: https://fedoraproject.org/wiki/EPEL
[3]: http://security.stackexchange.com/questions/3778/getting-self-signed-ssl-certificates-for-all-https-connections-made-from-some-pr
[4]: http://www.zdnet.com/article/how-the-nsa-and-your-boss-can-intercept-and-break-ssl/