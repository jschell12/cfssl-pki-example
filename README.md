# Simple PKI Example with `cfssl`, `cfssljson`, and `multirootca`
This example was inspired by [Building a Secure Public Key Infrastructure for Kubernetes](https://www.mikenewswanger.com/posts/2018/kubernetes-pki/) by **Mike Newswanger**.

## Installation:

See the official `cfssl` [README](https://github.com/cloudflare/cfssl/blob/master/README.md#installation).

For `multirootca` installation:

```sh
go get -u github.com/cloudflare/cfssl/cmd/multirootca
```

## Generating the Certificate Authorities and Associated Certificate Signing Server

Generate `root` certificate authority certificate:

```sh
$ cfssl gencert -initca root-csr.json | cfssljson -bare root-ca

2019/04/06 11:58:11 [INFO] generating a new CA key and certificate from CSR
2019/04/06 11:58:11 [INFO] generate received request
2019/04/06 11:58:11 [INFO] received CSR
2019/04/06 11:58:11 [INFO] generating key: rsa-2048
2019/04/06 11:58:12 [INFO] encoded CSR
2019/04/06 11:58:12 [INFO] signed certificate with serial number 167891568625587002661900040634378932042675797874
```

Generate an `intermediary` certificate authority certificate:

```sh
$ cfssl gencert -ca=root-ca.pem -ca-key=root-ca-key.pem -config=intermediary-config.json -hostname=0.0.0.0 -profile=default intermediary-csr.json | cfssljson -bare intermediary-ca

2019/04/06 11:58:18 [INFO] generate received request
2019/04/06 11:58:18 [INFO] received CSR
2019/04/06 11:58:18 [INFO] generating key: rsa-2048
2019/04/06 11:58:18 [INFO] encoded CSR
2019/04/06 11:58:18 [INFO] signed certificate with serial number 202678780292982818036866864115695594468867817862
```

Generate a `server` leaf certificate:
(For secure connections to `multirootca` server)

```sh
$ cfssl gencert -ca=intermediary-ca.pem -ca-key=intermediary-ca-key.pem -config=server-config.json -hostname=0.0.0.0 -profile=default server-csr.json | cfssljson -bare server

2019/04/06 11:58:25 [INFO] generate received request
2019/04/06 11:58:25 [INFO] received CSR
2019/04/06 11:58:25 [INFO] generating key: rsa-2048
2019/04/06 11:58:25 [INFO] encoded CSR
2019/04/06 11:58:25 [INFO] signed certificate with serial number 267055182799407713312172136708586130721128410514
```

Start the `multirootca` server:

```sh
/usr/local/bin/multirootca \
    -a 0.0.0.0:8888 \
    -l default \
    -roots multiroot-profile.ini \
    -tls-cert server.pem \
    -tls-key server-key.pem

2019/04/06 12:26:48 [INFO] loaded signer default
2019/04/06 12:26:48 [INFO] Now listening on https:// 0.0.0.0:8888
```

Request a `client` leaf certificate from `multirootca`:

```sh
$ cfssl gencert -config=request-profile.json -hostname=127.0.0.1 -tls-remote-ca server.pem -profile=default client-requester-csr.json | cfssljson -bare client-requester

2019/04/06 12:03:57 [INFO] generate received request
2019/04/06 12:03:57 [INFO] received CSR
2019/04/06 12:03:57 [INFO] generating key: rsa-2048
2019/04/06 12:03:58 [INFO] encoded CSR
2019/04/06 12:03:58 [INFO] Using trusted CA from tls-remote-ca: server.pem
```

 Observe certificate contents:

```sh
openssl x509 -in root-ca.pem -text -noout
openssl x509 -in intermediary-ca.pem -text -noout
openssl x509 -in server.pem -text -noout
openssl x509 -in client-requester.pem -text -noout
```