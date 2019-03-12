## Procedure to use k8s cert-manager & let's encrypt

Some cool infos are available here :
- Blog article : https://www.nearform.com/blog/how-to-run-a-public-docker-registry-in-kubernetes/
- Cert-manager project: https://docs.cert-manager.io/

###### Table of Content

   * [How to install the k8s cert-manager](#how-to-install-the-k8s-cert-manager)
   * [Getting a TLS certificate using Let's encrypt staging and HTTP-01](#getting-a-tls-certificate-using-lets-encrypt-staging-and-http-01)
   * [All in one](#all-in-one)
   * [Cleanup](#cleanup)
   * [Using the lego go tool to create a certificate with GoDaddy](#using-the-lego-go-tool-to-create-a-certificate-with-godaddy)

## How to install the k8s cert-manager

```bash
oc new-project cert-manager
oc label namespace cert-manager certmanager.k8s.io/disable-validation=true

oc apply -f https://raw.githubusercontent.com/jetstack/cert-manager/v0.6.2/deploy/manifests/00-crds.yaml --validate=false
oc apply -f https://raw.githubusercontent.com/jetstack/cert-manager/v0.6.2/deploy/manifests/cert-manager.yaml --validate=false
```

- Grant `cluster admin` role to the `cert-manager` SA
```bash
oc adm policy add-cluster-role-to-user cluster-admin -z cert-manager -n cert-manager
```

## Getting a TLS certificate using Let's encrypt staging and HTTP-01

- Create an `issuer CRD` for `letsencrypt` which represents the CA authority able to generate a certificate
```
oc apply -f http01/letsencrypt-staging.yml
```

**Remark**: The previous command will generate a Private Key Secret which is referenced by the secret `snowdrop-issuert-key`

- Verify what has been created
```bash
oc describe issuer.certmanager.k8s.io/letsencrypt-issuer
Name:         letsencrypt-issuer
Namespace:    cert-manager
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"certmanager.k8s.io/v1alpha1","kind":"Issuer","metadata":{"annotations":{},"name":"letsencrypt-issuer","namespace":"cert-manager"},"spec"...
API Version:  certmanager.k8s.io/v1alpha1
Kind:         Issuer
Metadata:
  Creation Timestamp:  2019-03-06T16:47:08Z
  Generation:          1
  Resource Version:    18944555
  Self Link:           /apis/certmanager.k8s.io/v1alpha1/namespaces/cert-manager/issuers/letsencrypt-issuer
  UID:                 7ad67f3f-402f-11e9-bce0-107b44b03540
Spec:
  Acme:
    Email:  cmoulliard@redhat.com
    Http 01:
    Private Key Secret Ref:
      Key:   
      Name:  snowdrop-issuer-key
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
Status:
  Acme:
    Uri:  https://acme-staging-v02.api.letsencrypt.org/acme/acct/8467053
  Conditions:
    Last Transition Time:  2019-03-06T16:47:08Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

- Next, send a `Certificate` request to `letscrypt` using this `Certificate CRD`

```bash
oc apply -f http01/certificate.yml
```

- Verify what has been created
```bash
oc describe certificate.certmanager.k8s.io/snowdrop-me
Name:         snowdrop-me
Namespace:    cert-manager
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"certmanager.k8s.io/v1alpha1","kind":"Certificate","metadata":{"annotations":{},"name":"snowdrop-me","namespace":"cert-manager"},"spec":{...
API Version:  certmanager.k8s.io/v1alpha1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2019-03-06T16:49:16Z
  Generation:          1
  Resource Version:    18944873
  Self Link:           /apis/certmanager.k8s.io/v1alpha1/namespaces/cert-manager/certificates/snowdrop-me
  UID:                 c74ddab5-402f-11e9-bce0-107b44b03540
Spec:
  Acme:
    Config:
      Domains:
        snowdrop.me
        www.snowdrop.me
      Http 01:
        Ingress:        
        Ingress Class:  nginx
  Common Name:          snowdrop.me
  Dns Names:
    snowdrop.me
    www.snowdrop.me
  Issuer Ref:
    Name:       letsencrypt-issuer
  Secret Name:  snowdrop-me-tls
Status:
  Conditions:
    Last Transition Time:  2019-03-06T16:49:16Z
    Message:               Certificate does not exist
    Reason:                NotFound
    Status:                False
    Type:                  Ready
Events:
  Type    Reason        Age   From          Message
  ----    ------        ----  ----          -------
  Normal  Generated     9s    cert-manager  Generated new private key
  Normal  OrderCreated  9s    cert-manager  Created Order resource "snowdrop-me-593892605"
```

- We will see now if the order has been created
```bash
oc describe order/snowdrop-me-593892605
Name:         snowdrop-me-593892605
Namespace:    cert-manager
Labels:       acme.cert-manager.io/certificate-name=snowdrop-me
Annotations:  <none>
API Version:  certmanager.k8s.io/v1alpha1
Kind:         Order
Metadata:
  Creation Timestamp:  2019-03-06T16:49:16Z
  Generation:          1
  Owner References:
    API Version:           certmanager.k8s.io/v1alpha1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Certificate
    Name:                  snowdrop-me
    UID:                   c74ddab5-402f-11e9-bce0-107b44b03540
  Resource Version:        18944878
  Self Link:               /apis/certmanager.k8s.io/v1alpha1/namespaces/cert-manager/orders/snowdrop-me-593892605
  UID:                     c7761023-402f-11e9-bce0-107b44b03540
Spec:
  Common Name:  snowdrop.me
  Config:
    Domains:
      snowdrop.me
      www.snowdrop.me
    Http 01:
      Ingress:        
      Ingress Class:  nginx
  Csr:                MIICrDCCAZQCAQAwLTEVMBMGA1UEChMMY2VydC1tYW5hZ2VyMRQwEgYDVQQDEwtzbm93ZHJvcC5tZTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAKZO2DFH7mCLxmm1+rts5laIeGkA9Dg9kFrfkJ8Wl1IEZngA3Gc/bH8rQTuBtvn2ZrWLp7ewfpl3qiO1kiF/rIbm4raaU91SYTpWDrMgXu1kvz/5gyjAwWjL+ms2MNah0/fedZKJNrM760Xq4xvw/917GPMW1nCyk/BOQGD9HkTYtsVscN93kzrDgpCmJCDKZkz5ubHrMO3JM/QYJgyFckDUXCClj64cy3htTE1qF5is0Em7sSrnOpReUbRG6kHLnE48NC3xKS1BRGvUULVaEPopHAZYGA8q5oCAeTuGVB1j8gRDeqimdc1Cd84J0NyNq63VWIQtxu7YMBkECRrZz2kCAwEAAaA6MDgGCSqGSIb3DQEJDjErMCkwJwYDVR0RBCAwHoILc25vd2Ryb3AubWWCD3d3dy5zbm93ZHJvcC5tZTANBgkqhkiG9w0BAQsFAAOCAQEAWx2bMlJ4/MODiHhudrWxRYgKOJN9kzzi2me2Pvw2bFX3Oz2OYHi7DZLvYKaEaxkc/voUBDV8egJexlGeLvFqaRRVJmJ5gz3a9XPFn1wy36q6CJeGFHHPXkQKHRsbEujfOB1b7IPoTcYtQzEQNtTEgdWC6hgd7XNmmwhxeUbhGzFqLT0LN39EXBzy+/BiUu+YXaS4GWlvSb26Pj28Gacj/bqZ7/9NrJtz7W56tclKSYrJL4nchqbS2hzoRQMvWbStBIV0e9cvwgTeb1gdXbjL0bU6Ecl14ZIAt0HST/xeq3Wx3n2a4qAO41IMuRwNXGmdziERrIOHPxn/+voZcHfIBA==
  Dns Names:
    snowdrop.me
    www.snowdrop.me
  Issuer Ref:
    Name:  letsencrypt-issuer
Status:
  Certificate:  <nil>
  Challenges:
    Authz URL:  https://acme-staging-v02.api.letsencrypt.org/acme/authz/PwfeomKelHPeK7QwyLYOmb_wF2bqn50rd6VRQDS-u_s
    Config:
      Http 01:
        Ingress:        
        Ingress Class:  nginx
    Dns Name:           snowdrop.me
    Issuer Ref:
      Name:     letsencrypt-issuer
    Key:        EgVNaxRAAP_F-GLNfGHAUqA4bnfwTw2QdbZ4L6pHcGo.lnZIc09JepuXAg_lzElR-60rYKHwoAUuqna24O74OI0
    Token:      EgVNaxRAAP_F-GLNfGHAUqA4bnfwTw2QdbZ4L6pHcGo
    Type:       http-01
    URL:        https://acme-staging-v02.api.letsencrypt.org/acme/challenge/PwfeomKelHPeK7QwyLYOmb_wF2bqn50rd6VRQDS-u_s/262900554
    Wildcard:   false
    Authz URL:  https://acme-staging-v02.api.letsencrypt.org/acme/authz/9o4FT5P6UrGpSbvxEkYPoQn_Ka6Qb7fw-iQ14Ly07ZA
    Config:
      Http 01:
        Ingress:        
        Ingress Class:  nginx
    Dns Name:           www.snowdrop.me
    Issuer Ref:
      Name:      letsencrypt-issuer
    Key:         4pi6dfrmu-PTdbCl9R5XN9_-rE3GjQ8cZKHUYGg_TAA.lnZIc09JepuXAg_lzElR-60rYKHwoAUuqna24O74OI0
    Token:       4pi6dfrmu-PTdbCl9R5XN9_-rE3GjQ8cZKHUYGg_TAA
    Type:        http-01
    URL:         https://acme-staging-v02.api.letsencrypt.org/acme/challenge/9o4FT5P6UrGpSbvxEkYPoQn_Ka6Qb7fw-iQ14Ly07ZA/262900556
    Wildcard:    false
  Finalize URL:  https://acme-staging-v02.api.letsencrypt.org/acme/finalize/8467053/25552951
  Reason:        
  State:         pending
  URL:           https://acme-staging-v02.api.letsencrypt.org/acme/order/8467053/25552951
Events:
  Type    Reason   Age   From          Message
  ----    ------   ----  ----          -------
  Normal  Created  1m    cert-manager  Created Challenge resource "snowdrop-me-593892605-1" for domain "www.snowdrop.me"
  Normal  Created  1m    cert-manager  Created Challenge resource "snowdrop-me-593892605-0" for domain "snowdrop.me"
```

- You can then go on to run `oc describe challenge snowdrop-me-593892605-0` to further debug the progress of the Order.

**WARNING**: This process will fail as we must use a DNS-01 provider but unfortunately GoDday is not yet there. A PR will be created and submitted to `cert-manager` in order to support it too

## All in one 

```bash
oc apply -f http01/letsencrypt-staging.yml
```

## Cleanup

```bash
oc delete issuer letsencrypt-issuer
oc delete certificate snowdrop-me
oc delete secret snowdrop-issuer-key
oc delete secret snowdrop-me-tls
```

## Using the lego go tool to create a certificate with GoDaddy

- Install the letsencrypt go tool

```bash
go get -u github.com/xenolf/lego/cmd/lego
```
- To get help about dns provider and display the list available
```bash
lego dnshelp
godaddy: GODADDY_POLLING_INTERVAL, GODADDY_PROPAGATION_TIMEOUT, GODADDY_TTL, GODADDY_HTTP_TIMEOUT, GODADDY_SEQUENCE_INTERVAL
```

- Create a GoDaddy API Key / Secret [here](https://developer.godaddy.com/keys#)

- Generate a certificate using the Goddady DNS provider
```bash
DOMAINS="snowdrop.dev"
EMAIL="cmoulliard@redhat.com"
API_KEY=GODADDY_API_KEY
API_SECRET=GODADDY_API_SECRET
GODADDY_API_KEY=${API_KEY} GODADDY_API_SECRET=${API_SECRET} lego -a -k rsa2048 --pem --dns godaddy --email=${EMAIL} --domains=${DOMAINS} --cert.timeout 200 run
```

**Remarks**: We added the options `--pem` to generate a .pem file by concatenating the .key and .crt files together, `-a` to indicate that we accept the current Let's Encrypt terms of service

- Rename the `.lego` folder using as suffix the domain name
```bash
mkdir -p snowdrop-dev
mv .lego snowdrop-dev
```
- Check certificate content created
```bash
openssl x509 -in snowdrop-dev/.lego/certificates/snowdrop.dev.crt -text -noout 
```

- To list the certificate created
```bash
lego --path snowdrop-dev/.lego list        
Found the following certs:
  Certificate Name: snowdrop.dev
    Domains: snowdrop.dev
    Expiry Date: 2019-06-05 06:49:18 +0000 UTC
    Certificate Path: snowdrop-dev/.lego/certificates/snowdrop.dev.crt
```

- Next, create an `Openshift Edge TLS` route where you will copy/paste the Cert, CA Cert parts from the pem file.
```bash
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    service: snowdrop-site-angular
  name: snowdrop-dev
spec:
  host: snowdrop.dev
  port:
    targetPort: '8080'
  tls:
    caCertificate: |-
      -----BEGIN CERTIFICATE-----
      COPY 2nd CERTIFICATE from the file snowdrop.dev.pem
      -----END CERTIFICATE-----
    certificate: |-
      -----BEGIN CERTIFICATE-----
      COPY 1st CERTIFICATE from the file snowdrop.dev.pem
      -----END CERTIFICATE-----
    insecureEdgeTerminationPolicy: Allow
    key: |-
      -----BEGIN RSA PRIVATE KEY-----
      COPY private key  from the file snowdrop.dev.pem
      -----END RSA PRIVATE KEY-----
    termination: edge
  to:
    kind: Service
    name: snowdrop-dev-site
    weight: 100
  wildcardPolicy: None
```

- To renew the certificate
```bash
GODADDY_API_KEY=GODADDY_API_KEY GODADDY_API_SECRET=GODADDY_API_SECRET lego --path snowdrop-dev/.lego --email="cmoulliard@redhat.com" --domains="snowdrop.dev" --dns godaddy renew --days 360
```
