# ICAD SISTEMI PKI

GitHub Pages repository for ICAD Sistemi Public Key Infrastructure

## Contents

* [Purpose](#purpose)
* [Prerequisites](#prerequisites)
* [Installing](#installing)
* [Running the tests](#running-the-tests)
  * [Get a certificate with a CRL](#get-a-certificate-with-a-crl)
  * [Getting the certificate chain](#getting-the-certificate-chain)
  * [Combining the CRL and the Chain](#combining-the-crl-and-the-chain)
  * [OpenSSL Verify](#openssl-verify)
* [Contributing](#contributing)
* [Authors](#authors)

## Purpose

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes.
See deployment instruction under "**[Running the tests](#running-the-tests)**" section for notes on how to deploy the project on a live system.

## Prerequisites

* **[OpenSSL](https://github.com/openssl/openssl)**

## Installing

There is no need to install any component because all check can be done from your local command line.
You can use OpenSSL in order to execute all necessary step.

## Running the tests

* [Get a certificate with a CRL](#get-a-certificate-with-a-crl)
* [Getting the certificate chain](#getting-the-certificate-chain)
* [Combining the CRL and the Chain](#combining-the-crl-and-the-chain)
* [OpenSSL Verify](#openssl-verify)

### Get a certificate with a CRL

Note that I'll be using `<host_ip>` e `<host_port>` in order to indicate the remote server and port that you want to test during the entire procedure.

First we will need the server certificate from our remote TLS server.
Save this output to a file, for example, `server.pem`.
We can retreive this certificate with the following OpenSSL command:

```{r, engine='bash', count_lines}
openssl s_client -connect <host_ip>:<host_port> 2>&1 < /dev/null | \
  sed -n '/-----BEGIN/,/-----END/p' > server.pem
```

Now, check if this certificate has a CRL URI:

```{r, engine='bash', count_lines}
openssl x509 -noout -text -in server.pem | \
  grep -A 4 'X509v3 CRL Distribution Points'
```

Download the CRL:

```{r, engine='bash', count_lines}
wget -O crl.pem "http://crl.icadsistemi.com/intermediate.crl.pem"
```

Note that our CRL downloaded is already in `PEM` format (*base64 encoded DER*) so you don't need to do any conversion.
In case the CRL will be in `DER` format (*binary*) you need to run this OpenSSL command:

```
openssl crl -inform DER -in crl.der -outform PEM -out crl.pem
```

### Getting the certificate chain

Save all certificates in the order OpenSSL sends them (as in, first the one which directly issued your server certificate,
then the one that issues that certificate and so on, with the root or most-root at the end of the file) to a file, named `chain.pem`.

To do so you can use the following command:

```{r, engine='bash', count_lines}
OLDIFS=$IFS; \
  IFS=':' certificates=$(openssl s_client -connect <host_ip>:<host_port>  \
  -showcerts -tlsextdebug -tls1_2 2>&1 </dev/null |  \
  sed -n '/-----BEGIN/,/-----END/ {/-----BEGIN/ s/^/:/; p}');  \
  for certificate in ${certificates#:}; do echo $certificate |  \
  tee -a chain.pem ; done; IFS=$OLDIFS 
```

### Combining the CRL and the Chain

The Openssl command needs both the certificate chain and the CRL, in PEM format concatenated together for the validation to work.
You can omit the CRL, but then the CRL check will not work, it will just validate the certificate against the chain.
To concatenate `chain.pem` and `crl.pem` you can use the following command in order to obtain the concatenated `crl_chain.pem`:

```{r, engine='bash', count_lines}
cat chain.pem crl.pem > crl_chain.pem
```
### OpenSSL Verify

We now have all the data we need in order validate the certificate using this command:

```{r, engine='bash', count_lines}
openssl verify -crl_check -CAfile crl_chain.pem server.pem 
```

If output will return you OK than it shows a good certificate status.

You can test this procedure using the certificate and chain on the 
[Verisign revoked certificate test page](https://test-sspev.verisign.com:2443/test-SSPEV-revoked-verisign.html).

## Contributing

This is an internal project, so this repository will not accept external PR.
For any change or issue please discuss via "[issues](https://github.com/icadsistemi/icadsistemipki/issues)",
[email](mailto:github@icadsistemi.com) or any other method with the owners of this repository instead of open a PR.

## Authors

* **Trimarchi Manuele** - [mtrimarchi](https://github.com/mtrimarchi)
