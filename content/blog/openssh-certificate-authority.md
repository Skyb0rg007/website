---
title: 'Creating a Certificate Authority with OpenSSL'
date: 2025-01-10T19:08:39-06:00
draft: true
---

# Background

Setting up an X.509 certificate authority can be a great way to authenticate
traffic with your homelab.
Good uses are:

1. Mutual TLS (mTLS). Distribute a signed certificate to client devices
like your laptop and phone, and have your server verify the client before
any data is transfered.

2. Using HTTPS on internal domains.
If you don't own a domain name, or want to visit `https://pi.hole`,
you can't use Let's Encrypt to get the required certificates.
After installing your root certificate, a device's browser will
display the lock like any other website.
If you do own a domain, consider whether Let's Encrypt and `*.in.your.tld`
domains suits your purposes before continuing.

3. Multiple TLS identities on the same device.
Let's Encrypt certificates certify that the domain is owned by you, but nothing
more.
If you want to have `auth.example.com:9091` and `auth.example.com:9000` use
different certificates, you need a CA as Let's Encrypt will only give you
one per domain. <!-- Determine truth -->

4. You develop software for Windows and want to sign your code.
Using your own CA will still cause a popup for users who have not installed
your root certificate into the trusted store, but the name of the cert
will appear on the prompt.
X.509 certificates can also be used with WDAC policies to lock down a desktop
while maintaining Administrator priviledges, or for signing directories
for whitelisting software installations.

Think twice if all you want to use your CA to do is the following:

1. HTTPS with your public domain.
[Let's Encrypt](https://letsencrypt.org) is a free service that provides
certificates which are already trusted in all major browsers.
You can run the acme client once a month to update the certificate,
which makes this even more secure than what I will be showing
because I'm assuming you're not going to setup your own acme server
and just run the `openssl` command once a year.

2. SSH Certificates.
I would highly recommend setting up SSH certificate-based authentication,
especially for authenticating hosts
(no more "The authenticity of host '…' can't be established." prompts!).
But OpenSSH doesn't use X.509, it has its own separate mechanism.
There are builds that Roumen Petrov publishes that have X.509 support
(found [here](https://roumenpetrov.info/secsh/index.html)),
but I can't recommend using a less-used SSH server, especially if you're
exposing it to the internet.

3. S/MIME email encryption.
Companies such as [Castle Cloud][castle-cloud]
and [Actalis][actalis] offer free S/MIME certificates.
Your certificate authority will be even less trustworthy than these services.
Uploading a PGP key to [OpenPGP][openpgp] can serve a similar
purpose, though I would recommend S/MIME if that's an option.
If you want to sign Git commits, I would recommend SSH keys over S/MIME and PGP.

4. Trusted timestamping.
There are free trusted timestamping websites such as [freeTSA][freetsa].
Maybe there's a usecase for performing trusted timestamps locally,
but you can't beat the guarentees of a third party.

There are a bunch of tutorials on how to use `openssl` to create certificates
and sign them, but none seem to be updated for the modern versions of
OpenSSL (> 3.0).
As a result there are way more commands to run than necessary:
in general every step is exactly one command.

As you will see, X.509 certificates are extremely customizable,
and as a result running OpenSSL can be complicated with long command lines.
One method for making these shorter is to utilize a configuration file,
where the options are specified to apply to different subcommands.
This tutorial will avoid using the configuration file when possible as using it
has two main issues: the specification of the options is convoluted, and which
sections apply for which commands is also complicated.
By using command line arguments, everything inside the code block is necessary.

To guide our choices, we will be following the [CA/Browser forum][caforum]
baseline requirements for certificate authorities.
The documents can be found [here][caforum-baseline] (version as of writing is 3.9.0),
and the relevant sections will be referenced out in this tutorial.
Some of the requirements are marked as `MAY`: guidance will either be
explained inline or justified through comparison with Google and Let's Encrypt.

# Authority Topology

Certificates work on a chain of trust.
If a client trusts a public key, it will treat documents signed by
the corresponding private key as trustworthy as well.
In this case, the documents will be other public keys along with metadata
that describes how this public key can be utilized.

For commercial certificate authorities, the baseline requirement is that
the certificates are 3 levels.
There is the root authority, whose public key is shared to everyone but
whose private key is rarely accessed.
The root authority signs the public keys of the intermediate authorities,
what the CA/Browser forum calls a *Subordinate Certificate Authority*.
These authorities' capabilities can be limited to only signing
certain kinds of documents, and have a method for revocation.
These subordinare certificate authorities are who will sign the certificates
for servers and clients.

#### Why this setup?

You need to have the private key in hand to renew the certificate below.
Certificates are re-issued relatively often (multiple times per year),
which could potentially cause the intermediate CA's private key to be compromised.
If this happens, one can use the root to create a new intermediate,
and add the old intermediate CA to a revocation list.
The trusted stores of devices do not need to be updated, though any
certificates issued to those devices will need to be.

# Running OpenSSL

Note: the `.key` files produced in these steps must be kept secure,
especially the root and intermediate authorities.
When you want to run these scripts yourself, I would recommend
disconnecting from the internet and doing the work on a flash drive.
Leave the secret files there and keep the drive unplugged.

#### A Note on file names

The key files and certificate files are both encoded using PEM:
a text-based encoding of ASN.1 (the data structure used by both).
One naming strategy is `root_ca_key.pem`, `root_ca_cert.pem`.
Another is `root_ca.key`, `root_ca.crt`.
This tutorial uses the second, though OpenSSL manpages prefer the first.

## Customizing my steps

- Organization name: <input type="text" placeholder="Homelab" id="homelab-org" size="10">
- Domain: <input type="text" placeholder="homelab.lan" id="homelab-domain" size="10">
- <input type="checkbox" name="crl" id="homelab-crl" checked> Support CRLs (Certificate Revocation Lists)
- <input type="checkbox" name="ocsp" id="homelab-ocsp"> Support OCSP (Online Certificate Status Protocol)
- <input type="radio" name="alg" value="rsa" checked> RSA <input type="radio" name="alg" value="ecdsa"> ECDSA

Recommendation: Support CRLs, don't support OCSP, use ECDSA

<span class="homelab-crl" data-on="CRL is on!" data-off="CRL is off!"></span>
<span class="homelab-ocsp" data-on="OCSP is on!" data-off="OCSP is off!"></span>

<span class="homelab-crl-on">CRL is on!</span>
<span class="homelab-crl-off" style="display:none;">CRL is off!</span>
<!-- <span class="homelab-ocsp" data-on="OCSP is on!" data-off="OCSP is off!"></span> -->
<script>
const dynamicText = label => {
    const input = document.getElementById(label);
    const def = input.placeholder;
    const results = document.getElementsByClassName(label);
    const update = () => {
        for (let span of results) {
            span.innerText = input.value || def;
        }
    };
    input.addEventListener('input', update);
    input.addEventListener('propertychange', update);
    document.addEventListener('DOMContentLoaded', update);
};
dynamicText('homelab-org');
dynamicText('homelab-domain');

const dynamicCheckbox = label => {
    const input = document.getElementById(label);
    const results = document.getElementsByClassName(label);
    const resultsOn = document.getElementsByClassName(label + "-on");
    const resultsOff = document.getElementsByClassName(label + "-off");
    const update = () => {
        const on = input.checked;
        for (let span of results) {
            span.innerText = on ? span.dataset.on : span.dataset.off;
        }
        for (let span of resultsOn) {
            span.style.display = on ? "block" : "none";
        }
        for (let span of resultsOff) {
            span.style.display = on ? "none" : "block";
        }
    };
    input.addEventListener('input', update);
    input.addEventListener('propertychange', update);
    document.addEventListener('DOMContentLoaded', update);
};
dynamicCheckbox('homelab-crl');
dynamicCheckbox('homelab-ocsp');
</script>

## Step 1: Create the root authority

Files produced: `root_ca.key`, `root_ca.crt`

<!-- -newkey EC -pkeyopt ec_paramgen_curve:P-384 -->
<div class="highlight">
<pre>
<code class="language-bash custom-codeblock">
$ openssl req -x509 -newkey rsa:4096 -keyout root_ca.key -out root_ca.crt \
    -days 9132 \
    -subj "/C=US/CN=<span class="homelab-org">Homelab</span> Root/O=Homelab" \
    -addext "basicConstraints=critical, cA:TRUE" \
    -addext "keyUsage=critical,digitalSignature,keyCertSign,cRLSign" \
    -addext "authorityKeyIdentifier = keyid, issuer:always"
</code>
</pre>
</div>

You will be prompted for a password.
This is used to encrypt the `root_ca.key` file.
Be sure to choose something strong (preferably randomly generated).
You can use `openssl rand -base64 72` for some random ASCII for this purpose.

This command creates a public/private key pair.
It then creates a certificate signing request with the metadata
from the second line onwards.
Finally, `openssl` signs the signing request with the private key itself,
resulting in the self-signed certificate `root_ca.crt`.

### Command Breakdown and Explanation

#### `openssl req -x509`

The `req` subcommand is for creating *certificate signing requests*.
It supports creating keys and signing the request at the same time,
which massively simplifies file management.
We are using the X.509 standard for certificates, which lets us specify
the metadata in the other lines of the command.

<!-- -newkey EC -pkeyopt ec_paramgen_curve:P-384 -->
#### `-newkey rsa:4096`

Create the public/private key pair using RSA with a 4096 bit secret.
The baseline requirement is 2048, but using a higher number for the
root is not that much of a speed issue because the root-subordinate
check will be cached.

#### `-keyout root_ca.key -out root_ca.crt`

These are the filenames for the private key (`root_ca.key`)
and the self-signed root certificate (`root_ca.crt`).
Keep the key secure and distribute the root everywhere.

#### `-days 9132`

This is how long the certificate will be valid for.
While some people will just choose a big number, 9,132 days (~25 years)
is the maximum length allowed by the CA forum.
The minimum is 2,922 days (~8 years).
(7.1.2.1.1)

#### `-subj "/C=US/CN=Homelab Root/O=Homelab"`

The subject: what is the name of your root certificate?
`C` is country, which should be the 2-letter country code.
`CN` is "common name", how you might refer to the root certificate in writing.
`O` is the organization.
The values are all arbitrary but can be viewed in the browser.
There are other fields that are allowed (such as "State or Province"),
but keeping the metadata short is valuable since this is sent with each
session.

#### `-addext "basicConstraints = critical, CA:true"`

The `critical` means clients that don't understand the extension must
reject the certificate.
`CA:true` marks the certificate as a certificate authority: ie. it can
sign certificates.

#### `-addext "keyUsage = critical, digitalSignature, keyCertSign, cRLSign"`

`keyCertSign` lets the CA sign other certs, and `cRLSign` lets the CA
revoke certificates (discussed later).
The `digitalSignature` usage is optional and you can remove it:
see the discussion on OCSP in the CRL section.

#### `-addext "authorityKeyIdentifier = keyid:always, issuer:alway`

The `authorityKeyIdentifier` extension is a the signing key.
This is enabled by default in OpenSSL for certificates that are
not self-signed, but needs to be specified because CA forum recommends
root certificates include it too.
This is skippable if you want to.

### Checking that this works

```bash
$ # Verify that the certificate is valid using itself (it's self-signed)
$ openssl verify -trusted root_ca.cert root_ca.crt
root_ca.crt: OK
$ # Print the certificate in text form
$ openssl x509 -text -in root_ca.crt -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            12:c8:9e:a7:....
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, CN = Homelab Root, O = Homelab
        Validity
            Not Before: Jan 10 16:36:01 2025 GMT
            Not After : Jan 11 16:36:01 2050 GMT
        Subject: C = US, CN = Homelab Root, O = Homelab
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:ce:0f:1c:...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                C0:4A:B6:3E:...
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Authority Key Identifier:
                DirName:/C=US/CN=Homelab Root/O=Homelab
                serial:12:C8:9E:A7:...
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        26:77:92:35:...
```

## Step 2: Create intermediate certificate authorities

<!-- -newkey EC -pkeyopt ec_paramgen_curve:P-384 -->
```bash
$ openssl req -x509 -newkey rsa:2048 -keyout int_ca.key -out int_ca.crt \
    -CA root_ca.crt -CAkey root_ca.key \
    -days 9132 \
    -subj "/C=US/CN=Homelab Intermediate/O=Homelab" \
    -addext "basicConstraints = critical, CA:true, pathlen:0" \
    -addext "keyUsage=critical,digitalSignature,keyCertSign,cRLSign" \
    -addext "extendedKeyUsage = serverAuth, clientAuth" \
    -addext "authorityInfoAccess = caIssuers;URI:http://example.com/ca.cer" \
    -addext "crlDistributionPoints = URI:http://example.com/myca.crl"
```

<!-- -newkey EC -pkeyopt ec_paramgen_curve:P-384 -->
#### `openssl req -x509 -newkey rsa:2048 -keyout int_ca.key -out int_ca.crt`

Same as before: create a private key (`int_ca.key`) and a signed
certificate (`int_ca.crt`).
Using 2048 bits for this intermediate CA is okay.

#### `-CA root_ca.crt -CAkey root_ca.key`

This time, don't self-sign the certificate. Use the root authority
to sign the generated `int_ca.crt`.

#### `-days 9132`

There aren't any requirements on the validity other than being shorter
than the root validity. Choose a number that works for you.
If you setup revocation (discussed later), using a long validity period
is probably fine.

#### `-subj "/C=US/CN=Homelab Intermediate/O=Homelab"`

Same requirements as the root CA.
Be sure the subject is different and recognizable; no need to use
the same country or organization.

#### `-addext "basicConstraints = critical, CA:true, pathlen:0"`

This is a certificate authority like the root,
but the `pathlen:0` constraint prevents it from being used to verify
certificate authorities.

#### `-addext "keyUsage = critical, digitalSignature, keyCertSign, cRLSign"`

Same note as the root CA.

#### `-addext "extendedKeyUsage = serverAuth, clientAuth"`

This certificate can be used by servers and clients (but not code signing,
S/MIME, OCSP, or others).
The `serverAuth` is required, but `clientAuth` usage can be removed if the
certificate authority isn't signing client certificates.

#### `-addext "authorityInfoAccess = caIssuers;URI:http://homelab.lan/root_ca.crt"`

This extension isn't required - skip this if you want.
Includes a link to download the issuing certificate (in this case `root_ca.crt`).
This is used during the revocation process.
If you implement OCSP, you need `OCSP;URI:http://homelab.lan/ocsp`.

#### `-addext "crlDistributionPoints = URI:http://homelab.lan/int_ca.crl"`

Includes a link to download the certificate revocation list.
You will need to keep this up-to-date, as CRLs' maximum validity is 30 days.

## Step 3: Create Subscriber Certificates

You're almost done! Now you can issue certificates for user devices and
servers.

<!-- -newkey EC -pkeyopt ec_paramgen_curve:P-384 -->
```bash
$ # Create a certificate for a domain
$ openssl req -x509 -newkey rsa:2048 -keyout photos.homelab.lan.key -out photos.homelab.lan.crt \
    -CA int_ca.crt -CAkey int_ca.key \
    -days 397 \
    -subj "/" \
    -addext "basicConstraints = critical, CA:false" \
    -addext "crlDistributionPoints = URI:http://example.com/sub.crl" \
    -addext "keyUsage = critical, digitalSignature" \
    -addext "extendedKeyUsage = serverAuth, clientAuth" \
    -addext "authorityInfoAccess = caIssuers;URI:http://homelab.lan/int_ca.crt" \
    -addext "certificatePolicies = 2.23.140.1.2.1" \
    -addext "subjectAltName = critical, DNS:photos.homelab.lan"
```

<!-- -newkey EC -pkeyopt ec_paramgen_curve:P-384 -->
```bash
$ # Create a certificate for a client
$ openssl req -x509 -newkey rsa:2048 -keyout photos.homelab.lan.key -out photos.homelab.lan.crt \
    -CA int_ca.crt -CAkey int_ca.key \
    -days 397 \
    -subj "/C=US/ST=Illinois/L=Chicago/SN=Doe/GN=John" \
    -addext "basicConstraints = critical, CA:false" \
    -addext "crlDistributionPoints = URI:http://example.com/sub.crl" \
    -addext "keyUsage = critical, digitalSignature" \
    -addext "extendedKeyUsage = serverAuth, clientAuth" \
    -addext "authorityInfoAccess = caIssuers;URI:http://homelab.lan/int_ca.crt" \
    -addext "certificatePolicies = 2.23.140.1.2.3" \
    -addext "subjectAltName = DNS:photos.homelab.lan"
```

## Step 4: Create the Certificate Revocation List

This is the one step that requires an option to be set which can only be
done through the OpenSSL configuration file.
I will get around it but the workaround may look more unusual.

```bash
$ # Create the revocation index file (empty for now)
$ # This will hold all the revoked keys with the time and reason of revocation
$ touch int_crl_index.txt
$ # Create the revocation number file
$ echo "01" > crlnumber
$ # Create a minimal config file
$ cat > crl.cnf <<EOF
database = int_crl_index.txt
crlnumber = crlnumber
crl_extensions = crl_ext

[ crl_ext ]
authorityKeyIdentifier = keyid:always
EOF
$ openssl ca -gencrl -cert int_crt.pem -keyfile int_key.pem -out int.crl \
    -md sha256 \
    -crldays 10 \
    -config crl.cnf -section default
```

#### `openssl ca -gencrl -cert int_crt.pem -keyfile int_key.pem -out int.crl`

We're generating a CRL using the intermediate certificate authority.
The resulting file will be int.crl.

#### `-md sha256`

This sets the message digest to SHA256. There are many options to choose from.

#### `-crldays 10`

This is how long the CRL is valid before it needs to be renewed.
The CA forum requires this to be at most 10 days for CRLs that are given
to Subjects (servers, clients, ie. not CAs).
It is up to you whether this is something you want to manage -- otherwise
you can set `-crldays 365` or something else longer.

#### `-config <(echo "database = int_crl_index.txt") -section default`

This is a trick to get around the need for a config file.
Essentially this is the equivalent of a fake option `-database int_crl_index.txt`.
The database file is an unencrypted list of revoked certificates.
As we will see in next step, revoking a certificate will just add
an entry to this file.

# Step 4a: distributing the CRL

If you're doing server authentication, you should host the CRL on your
local website, and have it updated every week.

If you're doing client authentication, you can have your CRL live much
longer, and manually update it when needed.
Example Nginx config:

```nginx
server {
  # Could maybe use `ssl_trusted_certificate` instead: see docs
  ssl_client_certificate /etc/nginx/client_cert.pem;
  ssl_crl /etc/nginx/client_crl.pem;
  # Could also use `ssl_verify_client: optional` along with $ssl_client_verify
  # This makes it possible to only deny certain paths.
  ssl_verify_client on;
  ssl_verify_depth 1;
}
```

# Step 4b: Revoking a certificate

```bash
$ openssl ca -revoke bob_crt.pem -cert int_crt.pem -keyfile int_key.pem -out int.crl \
    -md sha256 \
    -crldays 30 \
    -crl_compromise 20250113142354 \
    -config crl.cnf -section default
```

This revokes the certificate `bob_crt.pem`.

#### `-crl_compromise 20250113142354`

When revoking a certificate, you attach the reason for revocation.
The primary reasons are "The certificate was compromised" and
"The Intermediate CA was compromised".
These are the `-crl_compromise <timestamp>` and `-crl_CA_compromise <timestamp>`
options. The timestamp is YYYYMMDDHHMMSS, and should be the time the
certificate was compromised.
You are also supposed to update this timestamp if you discover the compromise
occurred earlier.

If the certificate wasn't compromised you use `-crl_reason <reason>`.
This attaches a timestamp, but uses the current date.
`affiliationChanged` means that the metadata changed so the cert is reissued.
For example if you change the domain that a given service is hosted at.
The reason is only applicable if there is no evidence of compromise: if
you do discover evidence of compromise,
you should update the revocation reason to keyCompromise.
The other reasons are irrelevant for a self-hosted CA.

# Revocation management options

Computers use certificates to determine that data comes from who the device
says it does.
Certificates have a limited lifetime to help ensure that guarentee is up-to-date,
but what if the certificate is lost or taken out of service before the
renewal date?

This is where revocation comes in.
Because certificates are entirely offline, there's no way to validate if
something has changed between the issuing of the cert and now.
CRLs and OCSP servers are methods for a CA to serve information about a
certificate's validity via HTTP, and a validator can query these online
servers to have more updated knowledge of the cert status.

This works quite well for client certificates.
Let's say you protect a subdomain (ex. `auth.example.com`) via mTLS,
and you require clients to provide a signed certificate along with
their requests.
If you lose your phone, you can issue a new certificate for your new phone
and should also prevent that lost certificate from being considered valid.
You add the public key of the lost certificate to OpenSSL's certificate
database with the revocation mark.
Then you have the intermediate CA sign that database and distribute it to
your webservers.
The signed database is the CRL.
Then whenever a client tries to authenticate, the webserver ensures that the
certificate is signed and valid, and then also that it does not appear on this
revocation list.

It also works well for other X.509 certificate uses such as
code signing certificates.
An attacker signing malicious code requires a compromise of the certificate,
and the attacker intercepting web traffic is a different attack vector.

Problems arise when it comes to validating server certificates.
In the client example, the revocation list exists on the same machine
as the validator, so it will always be up-to-date.
In the code-signing example, the revocation list is queried using a different
method than the malicious executable is.
In the server example, the revocation list is queried using the same
underlying protocol as the website data itself.
The browser or other application will send an HTTP request to the URL
listed in the certificate, and use the response as authoritative.
HTTP requests failing is not usually a serious error condition,
and browsers implement *soft-failure*: if a CRL is not available the
connection is considered valid.

The reason we use server certificates in HTTPS is for server validation.
If you connect to a café Wi-Fi and visit `google.com`, this ensures that
you are actually talking to Google and not someone else.
An attempt to redirect the `google.com` domain somewhere else would fail
at certificate validation, as the other location will not have a valid cert.

Let's say Google's domain certificate key gets leaked online.
Google adds the public key to their CRL at http://c.pki.goog/path/to/revokes.crl,
and re-issues their certificate.
The attacker then sets up their own fake `google.com` using this leaked
certificate and has the Wi-Fi's DNS records point to this fake website.
They also point http://c.pki.goog (and http://o.pki.goog) to their own IP
address that just doesn't answer any requests.

You connect to the fake website, get the revoked cert, and then send a request
to validate it.
Since the attacker controls the Wi-Fi and DNS your CRL (or OCSP) request
will go unanswered, and due to soft-failure be accepted.
HTTPS averted.
This problem discussed in [this blog post][revchecking-blog] which I highly
recommend reading.

The best solution is to just use short-lived certificates.
CA/Browser forum even has a definition:
A short-lived subscriber certificate is one that is valid for 7 days or less.
When the validity is that short, the CA is allowed to omit CRL and OCSP links.

If you don't want to run an ACME server, you can run a CA server
with your constrained intermediate certificate and use something
as basic as a password to regenerate the certificates.
Here's a simple Bash script that does that:

```bash

declare -A passwords
passwords[app1.home.arpa]=""
passwords[app2.home.arpa]=""

case "$1" in
    generate)
        domain="$2"
        pass="$(openssl rand -base64 80)"
        digest="$(openssl kdf -keylen 32 -kdfopt salt:"$domain" \
            -kdfopt memcost:9216 -kdfopt iter:4 -kdfopt lanes:1 \
            -kdfopt pass:"$pass" ARGON2ID)"
        echo "Password: $pass"
        echo "Digest: $digest"
        ;;
esac

```

Something you do want to worry about a little bit is your intermediate CA
becoming compromised.
If you don't want to restart from scratch and create a new root CA,
an attacker with control of your network -- ex. a café public Wi-Fi --
can potentially spoof *other* websites such as `google.com`.
This is why the instructions above recommend setting up a
*Technically Constrained TLS Subordinate Certificate Authority*, so that
even if your intermediate certificate is lost the worst that can happen
is `google.lan` can be spoofed.
If you're using an unroutable tld like `.home.arpa` this only degrades to
"as insecure as http" which I assume you've been using beforehand just fine.

# What am I leaving out?

Precertificates and the Signed Certificate Timestamp List:
These are to support Certificate Transparency, a way for third parties
to maintain a log of all issued certificates for auditing.
Not relevant for self-hosted setups.

OCSP: This form of revocation becomes better than CRL when the revocation list
gets large.
This will not happen if you are self-hosting.
It also requires server logic compared to the static file option of CRLs,
and the privacy concerns of OCSP have caused Let's Encrypt to
[deprecate their OCSP offering][letsencrypt-ocsp].

Partitioned CRL: This allows a server to maintain many small CRLs, where
the full list of revoked certificates is their concatenation.
Again, this is only relevant when you have a lot of revoked certificates.

[caforum]: https://cabforum.org
[caforum-baseline]: https://cabforum.org/working-groups/code-signing/documents/
[letsencrypt-ocsp]: https://letsencrypt.org/2024/07/23/replacing-ocsp-with-crls/
[castle-cloud]: https://acme.castle.cloud
[actalis]: https://www.actalis.com
[openpgp]: https://keys.openpgp.org
[freetsa]: https://freeTSA.org
[revchecking-blog]: https://www.imperialviolet.org/2014/04/19/revchecking.html
