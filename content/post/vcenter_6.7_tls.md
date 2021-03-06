---
authors:
- cunnie
categories:
- vSphere
date: 2018-05-09T23:16:22Z
draft: false
short: |
    We install a Transport Layer Security (TLS) certificate issued by a
    commercial Certificate Authority (CA) on a VMware VCSA 6.7 while avoiding
    several pitfalls.
title: How to Install a TLS Certificate on vCenter Server Appliance (VCSA) 6.7
---

We want a green padlock on our VCSA's web client; when we use our browser to
navigate to our VCSA, we'd like it too look like the following:

{{< responsive-figure src="https://user-images.githubusercontent.com/1020675/39831510-9630f47e-5379-11e8-9edf-b4ec6ca8c397.png" >}}

This blog posts describes the steps to follow in order to install a TLS
certificate on a VCSA 6.7.

*Author's note: we use our VCSA's fully qualified domain name
([FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name))
"vcenter-67.nono.io" throughout this document, with the understanding that you
should substitute your VCSA's FQDN accordingly.*

Note that this blog post is narrowly scoped: we are only replacing one of the
many certificates that the VCSA's services use, we are only replacing the
certificate that the human operator sees. Large enterprises may be more
interested in replacing _all_ the certificates, in which case they should refer
to this [VMware Knowledge Base
article](https://kb.vmware.com/s/article/2111219).

## 0. Pre-requisites

Before we begin, we need a TLS key, a chained certificate, and a root
certificate. Also, we need to enable ssh access to our VCSA, for when
certificates go horribly wrong the web interface may be unusable, and the only
mechanism to recover (other than reinstalling) is to ssh onto the VCSA and reset
the certificates.

### 0.0 Enable OpenSSH

Enable ssh by browsing to the [Appliance Management User
Interface](https://blogs.vmware.com/vsphere/2015/09/web-based-management-for-the-vcsa-is-back.html)
(Appliance MUI, formerly known as VAMI), https://vcenter-67.nono.io:5480, and
navigating to *Access → Access Settings → Edit → Edit Access Settings → Enable
SSH Login → toggle enabled → OK*.

### 0.1 Acquire Certificate from a Commercial CA

We won't discuss how to acquire a certificate, but we will point out that Heroku has a fairly succinct (and vendor-agnostic) [description](https://devcenter.heroku.com/articles/acquiring-an-ssl-certificate) of the process.

Note: VCSA, like Heroku, doesn't support [elliptic
curve](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography) keys and only
supports RSA keys (even though, in our estimation, elliptic curves are better if
for no other reason than they are so much more [terse than
RSA](https://www.globalsign.com/en/blog/elliptic-curve-cryptography/) keys).
This means that if you use a modern tool such as
[cfssl](https://github.com/cloudflare/cfssl), you'll need to override its
defaults to force it to generate RSA keys (e.g. `cfssl print-defaults csr | jq
-r '.key = {"algo":"rsa","size":2048}'`).

After the process is complete, we should have three files:

- [`vcenter-67.nono.io.chain.pem`](https://gist.github.com/cunnie/1119b9b59c7a50391987987310c76e6d#file-vcenter-67-nono-io-chain-pem) - this is the chained certificate; your
  server's certificate should be at the top, followed by any intermediate
  certificates.
- [`vcenter-67.nono.io.key`](https://gist.github.com/cunnie/1119b9b59c7a50391987987310c76e6d#file-vcenter-67-nono-io-key) - this is the RSA private key. You can see our
  key; however, we have taken the precaution of removing several
  lines
<sup><a href="#not_what_you_think" class="alert-link">[not-what-you-think]</a></sup>
.
- [`addtrustexternalcaroot.crt`](https://support.comodo.com/index.php?/Knowledgebase/Article/View/854/75/root-addtrustexternalcaroot) — This is the root certificate
<sup><a href="#hand_wavy" class="alert-link">[hand-wavy]</a></sup>
.

## 1. Install the Certificate

### 1.0 Install the Certificate via Web Interface

- Browse to the VCSA's web client <https://vcenter-67.nono.io>
- Select "Launch vSphere Client (HTML5)" *(we use the HTML interface; Flash™ is too much for us)*
- Log in
- Menu → Administration
- Certificates → Certificate Management
- Manage Certificates of vCenter
  - Server IP/FQDN: **vcenter-67.nono.io** *(don't use "localhost")*
  - Username: **administrator@vsphere.local**
  - Password: **_WhateverYourPasswordIs_**
- Click "Log in and manage certificates"
- Trusted Root Certificates
  - Click Add
  - Certificate Chain → Browse (browse to your root certificate & upload). Click Add.
- Page should now show two Trusted Root Certificates
- Find "\_\_MACHINE\_CERT"
- Actions → Replace
  - Certificate Chain → Browse (browse to your certificate & upload)
  - Private Key → Browse (browse to your certificate & upload)
- Click Replace, success should resemble image below

{{< responsive-figure src="https://user-images.githubusercontent.com/1020675/39830647-e237c6c0-5376-11e8-9549-600f05b9c3b2.png" >}}

Reboot the vCenter for the new certificates to take effect:
Browse to the VCSA Management Interface (VAMI),
https://vcenter-67.nono.io:5480, and navigate to *Actions → Reboot →
Reboot the system? → Yes*.

### 1.1 Install the Certificate via CLI

For those who would prefer to bypass the GUI and install the certificate via the
CLI, we offer this alternative. First, ssh into our VCSA and start a shell:

```
ssh root@vcenter-67.nono.io

    * List APIs: "help api list"
    * List Plugins: "help pi list"
    * Launch BASH: "shell"

Command> shell
```

Next we copy our certificates and keys from our workstation, `tara.nono.io`:

```bash
scp cunnie@tara.nono.io:{vcenter-67.nono.io.chain.pem,vcenter-67.nono.io.key,addtrustexternalcaroot.crt} .
```

Launch the certificate manager and install the keys and certificates:

```
/usr/lib/vmware-vmca/bin/certificate-manager
  1. Replace Machine SSL certificate with Custom Certificate
    2. Import custom certificate(s) and key(s) to replace existing Machine SSL certificate
      Please provide valid custom certificate for Machine SSL. vcenter-67.nono.io.chain.pem
      Please provide valid custom key for Machine SSL. vcenter-67.nono.io.key
      Please provide the signing certificate of the Machine SSL certificate. addtrustexternalcaroot.crt
```

Unlike the GUI, which returns immediately, the CLI spends several minutes replacing certificates. When it has finished, reboot:

```
shutdown -r now
```

### 1.2 Check: Refresh Your Browsers

Browse to the VCSA web client and confirm the certificate is installed (green
padlock). You may need to refresh the page. As a bonus, the Appliance MUI (VAMI)
has the new certificate, too!

## 2. Gotchas

If you don't upload the root certificate but update the certificate & key, after rebooting you will see the following in a red banner at the top:

> 503 Service Unavailable (Failed to connect to endpoint: [N7Vmacore4Http20NamedPipeServiceSpecE:0x00007f843400bef0] \_serverNamespace = / action = Allow \_pipeName =/var/run/vmware/vpxd-webserver-pipe)

If you browse to port 5480 and see an odd "0 -" it means you need to refresh
your browser (on macOS, ⌘-R). It may also mean that your vCenter is still
booting and that you should wait a few more minutes.

If you use an elliptic curve key, you will not be able to upload your key &
certificate; you'll see an error similar to the following:

{{< responsive-figure src="https://user-images.githubusercontent.com/1020675/39839080-e62ccdac-538f-11e8-9b8a-c62acb55bfb1.png" >}}

If things go really bad, you won't get a login screen; instead, you'll see:

> [400] An error occurred while sending an authentication request to the vCenter
Single Sign-On server - An error occurred when processing the metadata during
vCenter Single Sign-On setup - javax.net.ssl.SSLHandshakeException:
sun.security.validator.ValidatorException: PKIX path building failed:
sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid
certification path to requested target.

You'll need to reset your certificates; see the next section.

### 2.0 Resetting the Certificates when things go wrong

If the web client is unusable, you'll need to ssh in and use
`certificate-manager` to reset the certificates.

```
ssh root@vcenter-67.nono.io
/usr/lib/vmware-vmca/bin/certificate-manager
```

Choose option 8, "Reset all Certificates". Watch the screen scroll by as it
replaces the certificates, then reboot (`shutdown -r now`).

### References

- VMware Knowledge Base article, ["Replacing a vSphere 6.x Machine SSL certificate
with a Custom Certificate Authority Signed Certificate"](https://kb.vmware.com/s/article/2112277?other.KM_Utility.getArticleLanguage=1&r=1&other.KM_Utility.getArticleData=1&other.KM_Utility.getArticle=1&ui-comm-runtime-components-aura-components-siteforce-qb.Quarterback.validateRoute=1&other.KM_Utility.getGUser=1).
- We followed [these
  instructions](https://github.com/cunnie/docs/blob/7a53bb45d3a062ae40a2e3fb21a577d76d44ec3f/cfssl.md)
  when generating our Certificate Signing Request (CSR) with `cfssl`.
- [This gist](https://gist.github.com/cunnie/1119b9b59c7a50391987987310c76e6d)
  contains our chained certificate, our root certificate, and our redacted key
  (our key with several lines removed).

## Footnotes

<a name="not_what_you_think"><sup>[not-what-you-think]</sup></a>
We're not publishing our private key, but not for the reasons you think. You probably think it's about security, about preventing man-in-the-middle (MITM) attacks. It's not. Our server is behind a firewall and can only be accessed from our internal network. If you were in a position to execute an MITM attack, it would mean that our network was grossly compromised, and the jig would be up (an MITM attack would be the least of our concerns).

No, security is not the reason. Revocation is the reason. The last time that we published a private key, our certificate was revoked in a rather [spectacular manner](https://news.ycombinator.com/item?id=10184866), and the CA refused to issue us another one unless we promised not to publish it. We agreed, and we hew to our agreement, for we are men of our word.

<a name="hand_wavy"><sup>[hand-wavy]</sup></a>
This is the most hand-wavy part of the blog post, obtaining the root
certificate. On one hand, root certificates are everywhere — every one of the
billions of browsers has a copy of the [approximately
160](https://wiki.mozilla.org/CA/Included_Certificates) root certificates.

On the other hand, you may have to do some digging. Whichever CA you choose
should publish their root certificate. Comodo, for example, publishes their root
certificate
[here](https://support.comodo.com/index.php?/Knowledgebase/Article/View/854/75/root-addtrustexternalcaroot).

If that's a dead-end, you may want to check [Mozilla's list](https://wiki.mozilla.org/CA/Included_Certificates) of root certificates.

To determine if the certificate you have is a root certificate, confirm the
subject is the same as the issuer. In the following example, we use two common
TLS command line tools (`cfssl` and `openssl`) to examine the Comodo root
certificate `addtrustexternalcaroot.crt` and ensure the issuer is the same as
the subject:

First we use `cfssl`, whose output is JSON, which we pipe to `jq` to extract the
Common Name of the issuer and subject:

```bash
cfssl certinfo -cert addtrustexternalcaroot.crt |
  jq -r '[ .issuer.common_name, .subject.common_name ]'
```
produces:
```
[
  "AddTrust External CA Root",
  "AddTrust External CA Root"
]
```
With `openssl` we can use a simple `grep` to extract the information we need.
```bash
openssl x509 -in addtrustexternalcaroot.crt -noout -text |
  egrep "Issuer:|Subject:"
```

Now, when faced with the above output, a reasonable might ask, "why on earth is
Comodo using Addtrust's root certificate? Why don't they use their own
certificate?" One can only speculate.
