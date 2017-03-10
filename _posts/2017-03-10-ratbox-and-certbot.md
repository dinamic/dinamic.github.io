---
layout: post
title:  "Ratbox and Letsencrypt/Certbot"
date:   2017-03-10 13:24:00 +0200
categories: ratbox letsencrypt certbot
---

A while back [Let's Encrypt][letsencrypt] was launched. It provides a free SSL/TLS certificates that can be used to powerup that new shiny website you've got, or, as in my case, to power up the encryption between the client and your irc server. Let's see how one can do this.

# Prerequisites

I will assume you already have the following installed and configured already:

- [ratbox][ratbox] ircd
- [certbot][certbot]

All examples will be using irc.salsaparty.bg as my irc server. Whenever you see that, be sure to replace it with the address of your own one.

# The SSL certificate

## Generating the SSL/TLS certificate

To obtain the SSL certificate issue:

```
certbot --debug certonly --standalone -d irc.salsaparty.bg
```

After the certificate is generated, certbot will let you know in which folder you can find it. The output looks something like this:

```
MPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /usr/local/etc/letsencrypt/live/irc.salsaparty.bg/fullchain.pem.
   Your cert will expire on 2017-06-08. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you lose your account credentials, you can recover through
   e-mails sent to nikola@petkanski.com.
 - Your account credentials have been saved in your Certbot
   configuration directory at /usr/local/etc/letsencrypt. You should
   make a secure backup of this folder now. This configuration
   directory will also contain certificates and private keys obtained
   by Certbot so making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

As you can see, in my case the folder is `/usr/local/etc/letsencrypt/live/irc.salsaparty.bg/`

Ratbox requires a Deffie-Helman key. You can generate one as so:

```
openssl dhparam -out dh.pem 4096
```

I would suggest moving this file to the folder the certificates are located in, so you are storing everything together.

```
mv dh.pem /usr/local/etc/letsencrypt/live/irc.salsaparty.bg/
```

## Renewing the certificate

Certificates issued by [Let's Encrypt][letsencrypt] usually last for 3 months. They should be renewed manually a week before expiring.

I will be looking at ways to automate the renewal and rehash the irc server to accomodate for the new certificate.

To be done.

# Configuring Ratbox

I have found out that on different systems the certificates tend to be stored at different locations. Be sure to modify the following lines as per the correct ones for your system.

For example:

- FreeBSD: /usr/local/etc/letsencrypt/
- Ubuntu Linux: /etc/letsencrypt

ircd.conf:

```
serverinfo {
  ...

  ssl_private_key = "/usr/local/etc/letsencrypt/live/irc.salsaparty.bg/privkey.pem";
  ssl_cert = "/usr/local/etc/letsencrypt/live/irc.salsaparty.bg/cert.pem";
  ssl_dh_params = "/usr/local/etc/letsencrypt/live/irc.salsaparty.bg/dh.pem";
  ssld_count = 1;
  sslport = 9999;
  use_sslonly = yes;
}
```

# Does it work?

There are various tools to verify your SSL encryption works as intended. If you have an IRC client at hand - use that. I don't, so I will be giving examples on how to verify it the other way around.

## Using gnutls-cli

```
# gnutls-cli --print-cert -p 9999 irc.salsaparty.bg --insecure

Processed 0 CA certificate(s).
Resolving 'irc.salsaparty.bg:9999'...
Connecting to '139.162.165.158:9999'...
- Certificate type: X.509
- Got a certificate list of 1 certificates.
- Certificate[0] info:
 - subject `CN=irc.salsaparty.bg', issuer `CN=Let's Encrypt Authority X3,O=Let's Encrypt,C=US', serial 0x039a0c5d20eaad0f954b560e801e8519b29a, RSA key 2048 bits, signed using RSA-SHA256, activated `2017-03-10 08:57:00 UTC', expires `2017-06-08 08:57:00 UTC', key-ID `sha256:9dcf013de2e7ed22630e44a9284dbc6cb797be213ee2f9bfb6a3b86357c18564'
	Public Key ID:
		sha1:b0f304f99fa2f5e4afec7f6bced6c5d26b89d755
		sha256:9dcf013de2e7ed22630e44a9284dbc6cb797be213ee2f9bfb6a3b86357c18564
	Public key's random art:
		+--[ RSA 2048]----+
		|                 |
		|       .         |
		|      +          |
		|       =        E|
		|      o S      o.|
		|       + . .  . =|
		|        + +   .o*|
		|       o *   o+=o|
		|      .  .*+o=*. |
		+-----------------+


-----BEGIN CERTIFICATE-----
MIIFBjCCA+6gAwIBAgISA5oMXSDqrQ+VS1YOgB6FGbKaMA0GCSqGSIb3DQEBCwUA
MEoxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MSMwIQYDVQQD
ExpMZXQncyBFbmNyeXB0IEF1dGhvcml0eSBYMzAeFw0xNzAzMTAwODU3MDBaFw0x
NzA2MDgwODU3MDBaMBwxGjAYBgNVBAMTEWlyYy5zYWxzYXBhcnR5LmJnMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzCcqJCcOX7p7h2pSAls6BrbXTnNX
WeC8sX7tNv/kpUSJQaz2L3xC6eMlOEbM1eiKfDsB/Z+8jCAgNNEEaluuHT0uRCGa
I6OtM04N8LRcFFm5BxZ+MjGyryYlIxG9OSADJjIrHl7NKlMgw1OJguDvv4Ditana
VZSmrIFU4968Gr+Ec2h7NYt3toKp0DKMJ2BgMBY4SFFEUtmb3dCV6DRoYtCLT3To
JjFX3uYbCRlo6hdJWqllZZ3wtecY5MQqg6WxCwWnx7Jit30WBl7HCheNNWO1Xbv/
ONWKD+/AXSCpWXz00PEpS9V06aG9fbSmLitLBHbEtPqiFZYQAt8VGjp8tQIDAQAB
o4ICEjCCAg4wDgYDVR0PAQH/BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggr
BgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQWBBSfrFW0M/fWNgEYSk/ThKCq
sUrIrzAfBgNVHSMEGDAWgBSoSmpjBH3duubRObemRWXv86jsoTBwBggrBgEFBQcB
AQRkMGIwLwYIKwYBBQUHMAGGI2h0dHA6Ly9vY3NwLmludC14My5sZXRzZW5jcnlw
dC5vcmcvMC8GCCsGAQUFBzAChiNodHRwOi8vY2VydC5pbnQteDMubGV0c2VuY3J5
cHQub3JnLzAcBgNVHREEFTATghFpcmMuc2Fsc2FwYXJ0eS5iZzCB/gYDVR0gBIH2
MIHzMAgGBmeBDAECATCB5gYLKwYBBAGC3xMBAQEwgdYwJgYIKwYBBQUHAgEWGmh0
dHA6Ly9jcHMubGV0c2VuY3J5cHQub3JnMIGrBggrBgEFBQcCAjCBngyBm1RoaXMg
Q2VydGlmaWNhdGUgbWF5IG9ubHkgYmUgcmVsaWVkIHVwb24gYnkgUmVseWluZyBQ
YXJ0aWVzIGFuZCBvbmx5IGluIGFjY29yZGFuY2Ugd2l0aCB0aGUgQ2VydGlmaWNh
dGUgUG9saWN5IGZvdW5kIGF0IGh0dHBzOi8vbGV0c2VuY3J5cHQub3JnL3JlcG9z
aXRvcnkvMA0GCSqGSIb3DQEBCwUAA4IBAQA8nGQsX0VuywSlRWMUBQxTOSgUnHJ+
CubuBVWGKCsdoQFCEJIiycl0IKF5zOwwrIwq2G373dXdtEy7tFycJfKimEy+13LT
O5Lb2q+r82cPKaNWkV/npfkJYnoe4wIU6cWC7p6sWCGxswpC2WJthK94HDZkd57g
1KSWZXvcXhxIBTCvZ//SDMtQKUtjYg5qk9BsNB+SlA5Jsfcv1i5/ROu0vVEUThI+
HSMayU07p2lwZTZtUgzb3nTqFUs3bAc+jay0yzK9uChmiJiqcw7j0hbrLtqqbi7K
Ta3/3odFTi14EWCZqL3FiHqBXyWkf5yMRZrbJEzo5+R9wH8zc2PRhBL2
-----END CERTIFICATE-----

- Status: The certificate is NOT trusted. The certificate issuer is unknown.
*** PKI verification of server certificate failed...
- Description: (TLS1.2)-(ECDHE-RSA-SECP256R1)-(AES-256-GCM)
- Session ID: 17:90:CF:A1:A8:ED:F6:50:89:33:C7:8A:63:F2:5F:C5:97:7E:EF:7B:2E:42:AD:0A:00:08:B5:ED:B6:DF:49:38
- Ephemeral EC Diffie-Hellman parameters
 - Using curve: SECP256R1
 - Curve size: 256 bits
- Version: TLS1.2
- Key Exchange: ECDHE-RSA
- Server Signature: RSA-SHA512
- Cipher: AES-256-GCM
- MAC: AEAD
- Compression: NULL
- Options: safe renegotiation,
- Handshake was completed

- Simple Client Mode:

NOTICE AUTH :*** Processing connection to irc.salsaparty.bg
NOTICE AUTH :*** Looking up your hostname...
NOTICE AUTH :*** Found your hostname
quit
ERROR :Closing Link: 127.0.0.1 (Client Quit)
- Peer has closed the GnuTLS connection
```


## Using openssl

```
# openssl s_client -connect irc.salsaparty.bg:9999

CONNECTED(00000003)
depth=0 /CN=irc.salsaparty.bg
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 /CN=irc.salsaparty.bg
verify error:num=27:certificate not trusted
verify return:1
depth=0 /CN=irc.salsaparty.bg
verify error:num=21:unable to verify the first certificate
verify return:1
---
Certificate chain
 0 s:/CN=irc.salsaparty.bg
   i:/C=US/O=Let's Encrypt/CN=Let's Encrypt Authority X3
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIFBjCCA+6gAwIBAgISA5oMXSDqrQ+VS1YOgB6FGbKaMA0GCSqGSIb3DQEBCwUA
MEoxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1MZXQncyBFbmNyeXB0MSMwIQYDVQQD
ExpMZXQncyBFbmNyeXB0IEF1dGhvcml0eSBYMzAeFw0xNzAzMTAwODU3MDBaFw0x
NzA2MDgwODU3MDBaMBwxGjAYBgNVBAMTEWlyYy5zYWxzYXBhcnR5LmJnMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzCcqJCcOX7p7h2pSAls6BrbXTnNX
WeC8sX7tNv/kpUSJQaz2L3xC6eMlOEbM1eiKfDsB/Z+8jCAgNNEEaluuHT0uRCGa
I6OtM04N8LRcFFm5BxZ+MjGyryYlIxG9OSADJjIrHl7NKlMgw1OJguDvv4Ditana
VZSmrIFU4968Gr+Ec2h7NYt3toKp0DKMJ2BgMBY4SFFEUtmb3dCV6DRoYtCLT3To
JjFX3uYbCRlo6hdJWqllZZ3wtecY5MQqg6WxCwWnx7Jit30WBl7HCheNNWO1Xbv/
ONWKD+/AXSCpWXz00PEpS9V06aG9fbSmLitLBHbEtPqiFZYQAt8VGjp8tQIDAQAB
o4ICEjCCAg4wDgYDVR0PAQH/BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggr
BgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQWBBSfrFW0M/fWNgEYSk/ThKCq
sUrIrzAfBgNVHSMEGDAWgBSoSmpjBH3duubRObemRWXv86jsoTBwBggrBgEFBQcB
AQRkMGIwLwYIKwYBBQUHMAGGI2h0dHA6Ly9vY3NwLmludC14My5sZXRzZW5jcnlw
dC5vcmcvMC8GCCsGAQUFBzAChiNodHRwOi8vY2VydC5pbnQteDMubGV0c2VuY3J5
cHQub3JnLzAcBgNVHREEFTATghFpcmMuc2Fsc2FwYXJ0eS5iZzCB/gYDVR0gBIH2
MIHzMAgGBmeBDAECATCB5gYLKwYBBAGC3xMBAQEwgdYwJgYIKwYBBQUHAgEWGmh0
dHA6Ly9jcHMubGV0c2VuY3J5cHQub3JnMIGrBggrBgEFBQcCAjCBngyBm1RoaXMg
Q2VydGlmaWNhdGUgbWF5IG9ubHkgYmUgcmVsaWVkIHVwb24gYnkgUmVseWluZyBQ
YXJ0aWVzIGFuZCBvbmx5IGluIGFjY29yZGFuY2Ugd2l0aCB0aGUgQ2VydGlmaWNh
dGUgUG9saWN5IGZvdW5kIGF0IGh0dHBzOi8vbGV0c2VuY3J5cHQub3JnL3JlcG9z
aXRvcnkvMA0GCSqGSIb3DQEBCwUAA4IBAQA8nGQsX0VuywSlRWMUBQxTOSgUnHJ+
CubuBVWGKCsdoQFCEJIiycl0IKF5zOwwrIwq2G373dXdtEy7tFycJfKimEy+13LT
O5Lb2q+r82cPKaNWkV/npfkJYnoe4wIU6cWC7p6sWCGxswpC2WJthK94HDZkd57g
1KSWZXvcXhxIBTCvZ//SDMtQKUtjYg5qk9BsNB+SlA5Jsfcv1i5/ROu0vVEUThI+
HSMayU07p2lwZTZtUgzb3nTqFUs3bAc+jay0yzK9uChmiJiqcw7j0hbrLtqqbi7K
Ta3/3odFTi14EWCZqL3FiHqBXyWkf5yMRZrbJEzo5+R9wH8zc2PRhBL2
-----END CERTIFICATE-----
subject=/CN=irc.salsaparty.bg
issuer=/C=US/O=Let's Encrypt/CN=Let's Encrypt Authority X3
---
No client certificate CA names sent
---
SSL handshake has read 2757 bytes and written 712 bytes
---
New, TLSv1/SSLv3, Cipher is DHE-RSA-AES256-SHA
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
SSL-Session:
    Protocol  : TLSv1
    Cipher    : DHE-RSA-AES256-SHA
    Session-ID: D6EB74216E5F4F4362A8DD56317851FA6504CF3280CA99B4756961D8F7B44E33
    Session-ID-ctx:
    Master-Key: 2B2040357F2BF0FD64A442FC97E4FE8CF1C6C57131FC2A0D5F3CF0D8288F197AE3227F79D8247B4C0485C4F356E855A4
    Key-Arg   : None
    Start Time: 1489143426
    Timeout   : 300 (sec)
    Verify return code: 21 (unable to verify the first certificate)
---
NOTICE AUTH :*** Processing connection to irc.salsaparty.bg
NOTICE AUTH :*** Looking up your hostname...
NOTICE AUTH :*** Found your hostname
quit
ERROR :Closing Link: 127.0.0.1 (Client Quit)
closed
```


[letsencrypt]: https://letsencrypt.org/
[ratbox]: http://www.ratbox.org/
[certbot]: https://certbot.eff.org/
