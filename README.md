# Building durable VPNs with AWS Managed VPN

Blog post:

## Requisite
* AWS IAM role or user - with sufficient permissions
* AWS-CLI - installed and configured
* Node 9.0.0 or later (If Bauble is used) 

## Instructions

All the CloudFormation templates are located in `templates/` -directory. These templates can be used by uploading them to AWS CloudFormation via AWS Console, AWS CLI or with [Bauble](https://bauble.webscale.fi/).

## Building with Bauble

### Install dependencies

```
$ npm install
$ npm run version # to verify bauble installation
```

### Generate certificates

Easiest way to generate certificates for AWS Client VPN is by using [easy-rsa](https://github.com/OpenVPN/easy-rsa). For more information, please read AWS [documentation](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/authentication-authorization.html#mutual) about client auntehntication.

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

Import VPN certificate to AWS ACM. Please replace `{NAME_OF_CERTIFICATE}` and `{AWS_REGION}` with your values.
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

**Copy the outputed `CertificateArn` value and replace `data.vpn.ServerCertificateArn` - value with it in `stacks/config.yml` -file**

Note: ca.key and ca.crt files are used to sign clients' CSR. Please keep them in a safe and secure location.

### Customise the config -file

You can customise `stacks/config.yml` for your liking or use the default provided values. 

### Deploy

```
$ npm run launch
```

## VPN connection

To establish an VPN connection, the client needs to generate a Certificate Signing Request (CSR). To generate one, run the following command:
```
$ openssl req -new -newkey rsa:2048 -keyout client.key -nodes -out client.csr
```
A prompt will show up. Not all asked informations are required, but at least fill `Common Name` with a value that can identify you as an user, eg. e-mail address.

Keep `client.key` in a safe and secure location. Send `client.csr` to VPN administrator. Administrator will sign the CSR with the CA key and certificate and return the generated `client.crt` back to the client.
```
$ openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256
```

Download the client configuration -file from AWS console. It is located in VPC Dashboard -> Client VPN Endpoints.

Modify the downloaded .ovpn -file:
1. Add a random string in from of remote address.
```
remote foobar.cvpn-endpoint-0b013a9298effc710.prod.clientvpn.eu-west-1.amazonaws.com 443
```
2. Add absolute paths to your client crt and key at the bottom
```
cert /Users/heikki/vpn/client.crt
key /Users/heikki/vpn/client.key
```

Now open your favorite VPN Client software that supports OpenVPN with the .ovpn -file.

MacOS:
* [TunnelBlick](https://tunnelblick.net/)

Linux:
* Use OpenVPN client supported by your distro

Windows:
* [OpenVPN Connect](https://openvpn.net/client-connect-vpn-for-windows/)
