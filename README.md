# Building durable VPN with AWS Managed VPN

Blog post:

## Generate certificates

https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/authentication-authorization.html#mutual

```
$ git clone https://github.com/OpenVPN/easy-rsa.git
$ cd easy-rsa/easyrsa3
$ ./easyrsa init-pki

# Build root CA (Certificate authority) key and certificate
$ ./easyrsa build-ca nopass

# Build certificate and key for managed VPN
# $ ./easyrsa build-server-full {NAME_OF_CERTIFICATE} nopass
$ ./easyrsa build-server-full demo-managed-vpn nopass
```

Move certificates and keys to one location before importing to AWS ACM
```
$ mkdir temp/
$ mv pki/issued/* temp/
$ mv pki/ca.crt temp/
$ mv pki/private/* temp/
$ cd temp/
```

Import VPN certificate to AWS ACM. Please change `{NAME_OF_CERTIFICATE}` and `{AWS_REGION}`.
```
$ aws acm import-certificate \
--certificate file://{NAME_OF_CERTIFICATE}.crt \
--private-key file://{NAME_OF_CERTIFICATE}.key \
--certificate-chain file://ca.crt \
--region {AWS_REGION}

# e.g.
$ aws acm import-certificate \
--certificate file://demo-managed-vpn.crt \
--private-key file://demo-managed-vpn.key \
--certificate-chain file://ca.crt \
--region eu-west-1
```

__Copy the outputed `CertificateArn` value.__ 
