# Certifciate Management Within the pattern
Cert-Manager is used to provide certificates on request to applications that require certificates

The certificate pattern developed is multi-tiered with the hub cluster acting as a Root Certificate Authority (CA) that can generate intermediate certificates per application that are then distributed to the managed clusters, where they become a CA for that application on that cluster. Applications can then request certificates from that authority as needed.

![cert-chain](cert-chain.png "Certificate Chain Pattern in use")

# Caveats/Gotchas
- Certificates have a 90 day exirary by deafult, this can be changed.
- They are self signed, so will still give browser errors if used for external facing interfaces like routes
- They are stored as secrets within the clusters and nodes, so not entirely secret, however they are contained to specific namespaces.

# How It Works
Taking the vault deployment as an example the following steps and samples are required to setup a trust-chain, this all automated using the pattern described below.

The first 3 steps all take place on the Hub Cluster.

1. Cert-Manager installed and setup as Self-Signing CA (already part of the pattern)
(Performed once Per Hub Cluster)
```
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: self-signer
  namespace: cert-manager
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rootca
  namespace: cert-manager
spec:
  secretName: rootca
  issuerRef:
    name: self-signer
  commonName: "ca.self-signer.cert-manager"
  isCA: true
---  
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: local-issuer
  namespace: cert-manager
spec:
  ca:
    secretName: rootca  
```

2. Each application deployed via RHACM on the Hub requests a certificate that also is also a CA, which will become an intermediate CA, certificates are requested and issued for that application's namespace.
(Performed once Per Application to be deployed)

```
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: vault-intermediate-ca
  namespace: vault
spec:
  secretName: vault-intermediate-ca
  issuerRef:
    name: local-issuer
    kind: ClusterIssuer
    group: cert-manager.io
  commonName: "vault.local-issuer.cert-manager"
  isCA: true
```

3. Using Hive SelectorSyncSet, distribute the secret to child clusters using the clusterDeploymentSelector to determine which clusters, refer to the [Hive documentation] for more info.
The SelectorSyncSet below, will Sync a secret, in this case our certificates, with a secret in the child cluster called rootca in the namespace vault.
(Performed once Per Application to be deployed on hub cluster)

```
apiVersion: hive.openshift.io/v1
kind: SelectorSyncSet
metadata:
  name: vault-intermediate-ca-sync
spec:
  clusterDeploymentSelector:
    matchLabels:
      cluster.open-cluster-management.io/clusterset: azure-aws-clusterset
  secretMappings:
    - sourceRef:
        name: vault-intermediate-ca
        namespace: vault
      targetRef:
        name: rootca
        namespace: vault 
```
[Hive documentation]: https://github.com/openshift/hive/blob/master/docs/syncset.md


The following steps take place on the child cluster(s) (again automated through the pattern)

4. Create an certificate Issuer in the application namespace using the intermediate CA that was Synced buy the SelectorSyncSet in step 3.
(Performed once Per Application to be deployed on each child cluster)


```
---
# Create an Issuer that uses the above generated CA certificate to issue certs
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: vault-issuer
spec:
  ca:
    secretName: rootca
---
```

5. Request a certificate from the new namespace issuer for the given application, the certificate will be stored in the secret you specifiy, in this example 'vault-tls'
(Performed as needed per application per child cluster)

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: vault
spec:
  # Secret names are always required.
  secretName: vault-tls
  commonName: vault
  usages:
    - server auth
    - client auth  
  dnsNames:
  - vault
  - vault.vault.svc
  - vault.vault.svc.cluster.local
  issuerRef:
    name: vault-issuer
    kind: Issuer
    group: cert-manager.io
```

## Outputs
What will be generated on the child cluster will be a Secret Object, with 3 Secrets,
- ca.crt - contains the original Root CA certificate
- tls.crt - contains both the intermediate and application certificates
- tls.key - contains the singing key for the application certificate

### ca.crt
```
-----BEGIN CERTIFICATE-----
MIIDGTCCAgGgAwIBAgIRAMCJDBZpzqgTtO3jJRWvOQcwDQYJKoZIhvcNAQELBQAw
JjEkMCIGA1UEAxMbY2Euc2VsZi1zaWduZXIuY2VydC1tYW5hZ2VyMB4XDTIyMDEy
NDA5MjAzNVoXDTIyMDQyNDA5MjAzNVowJjEkMCIGA1UEAxMbY2Euc2VsZi1zaWdu
ZXIuY2VydC1tYW5hZ2VyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA
vwy4EaL5BZcadfYV+rZSD1uE+lcFRIYkjGQ7Snhwag2PbKx0XRDUiMMNeWvNb+lz
Q0T8WkMG0Odm9u2qMUDuPXoWhkv5+Pv5veW9WDq+ZZ1Fjz9PfGfjfhDncA+aXaUV
o17QZq61VzUKEHjfS+Pj3qCgcdmF79dTo8w3Y3SZzTrV9zhCOL33i+r0ccMe+E9F
SKH5ijfffJapk6N421SarBLWW1ijlTZzd9AaRYXw2j7heMTJBimRJQgRU+AGtohu
kTPqQn0Ztv/FaTIom9AFYQIDAQABo0IwQDAOBgNVHQ8BAf8EBAMCAqQwDwYDVR0T
CKqh28NRAn+HNkYI5bpGCLTalaQHT/cNTx7Qih/6hr15TQ62k4wmwS0ofmBOyNQj
geunCuD/pH6t8BZinuLGofhUeTwysAr4Uq9ji5WjQ1yRCUmR8/DWVEs3gACCZSe1
IqOGd05gyVYWzF2ehsPtO6B+TRMyXpzVC7yh6Qr79lrHb4azc1dNoVuarzx5RoDe
0eUnxJe3r/DtOjlE9TFsDjDgBekPOcpLqIq7b88=
-----END CERTIFICATE-----
```
### tls.crt
Note the 2 certificates
```
-----BEGIN CERTIFICATE-----
MIIG4DCCBcigAwIBAgIQUX6HKg0awZB2fwZTeyW+tDANBgkqhkiG9w0BAQsFADAq
MSgwJgYDVQQDEx92YXVsdC5sb2NhbC1pc3N1ZXIuY2VydC1tYW5hZ2VyMB4XDTIy
MDEyNDA5NTA0NloXDTIyMDQyNDA5NTA0NlowEjEQMA4GA1UEAxMHdmF1bHQtMjCC
ASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALJeY1//qoFDMGN1lO6SvzFR
CIcEJbabJJeKwwNiIJh8UHXdexL8KViwC6lT710+qDpuF2+WVIhOBu8+3XfM1PuH
UZVicPs/tCyyTU6tU++/J/VW5Jp9tgXKJgIHP9JEkAZbKBi9qq1OYDAzObFnhzkk
yTwmLjNqQtatV1LMX64QAbHGaM1/FQTKeSqosYTkbNfhJwkWjztJCGrDyCVYE/6x
F2b99+3GnvDal8EZvMz7yTf0ANYE4rkDjemSKI5xKB+Ruet6GNKteOK+O7QQsKhw
LWNsdXN0ZXItc2hhcmVkLTIudmF1bHQtaW50ZXJuYWwudmF1bHQuc3ZjLmNsdXN0
ZXJzZXQubG9jYWyCRnZhdWx0LTIuYXdzLWNsdXN0ZXItc2hhcmVkLTIudmF1bHQt
aW50ZXJuYWwudmF1bHQuc3ZjLmNsdXN0ZXJzZXQubG9jYWyCIHZhdWx0LTAudmF1
bHQtaW50ZXJuYWwudmF1bHQuc3Zjgi52YXVsdC0wLnZhdWx0LWludGVybmFsLnZh
dWx0LnN2Yy5jbHVzdGVyLmxvY2FsgiB2YXVsdC0xLnZhdWx0LWludGVybmFsLnZh
dWx0LnN2Y4IudmF1bHQtMS52YXVsdC1pbnRlcm5hbC52YXVsdC5zdmMuY2x1c3Rl
ci5sb2NhbIIgdmF1bHQtMi52YXVsdC1pbnRlcm5hbC52YXVsdC5zdmOCLnZhdWx0
LTIudmF1bHQtaW50ZXJuYWwudmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWyCBXZhdWx0
gg92YXVsdC52YXVsdC5zdmOCHXZhdWx0LnZhdWx0LnN2Yy5jbHVzdGVyLmxvY2Fs
MA0GCSqGSIb3DQEBCwUAA4IBAQBPsuDp/I2dSJwTv1Vs7QE2Gbho91jXnT2wUlCx
DTkwoPJtEVBx2dVw5Mfvhc7UlW9Jiwj3VvKI/APAID01+xz9+lHWjuYy2lm0zSdc
WAqdDaJgGA+Gd5diLMh+ISZPtlxzsE/NCvyqwEKvR9PD7pcjPySjGbIaC/bhQSmw
9y0CNLFILPJYz/SVUVrWcJUEe09B1TNJ25G0S3hB4enmeNgv7isGBqeLTvCmryY1
RFUjd2mj3BcHGvLyjHssYWRkzCzS5jFcHLfxhbKThY7kzIjyxL8TPRCul5TVm+P4
94qrElHNdUv26AX2Xi8nrzL0EiFeOHYAfnh3yG4xYTEaXDVI
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDPTCCAiWgAwIBAgIQCb7jZavAIEy842mkEcwo0TANBgkqhkiG9w0BAQsFADAm
MSQwIgYDVQQDExtjYS5zZWxmLXNpZ25lci5jZXJ0LW1hbmFnZXIwHhcNMjIwMTI0
MDkzMDA0WhcNMjIwNDI0MDkzMDA0WjAqMSgwJgYDVQQDEx92YXVsdC5sb2NhbC1p
c3N1ZXIuY2VydC1tYW5hZ2VyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
AQEAobKgUDevDkmgoRfEAxc+xBEhfF86A/PbfSvMIRJAQp8UAL07TQFmde6pgS43
H4bRfOT21jWGzbDmyaVcpqxovkvI37n68xoKEc7QxIC82RCoZlsDTHjB72vRSkyh
74ALnQdRqesxpfPMAC8je/yF/fZ9/w2HLvhoL6fTOOAwS0Gzinjp2mywWWIkIujl
ANrsihDjb4fdUzGxUWJZvqVFyZjhDOPqf/yWpadOSFHykxRniAdrHNBX4Bq4ZYmk
v7TXp/urW8Ebm8dnGxJWbvtujA6prgUjeYuT3vLK8hBoubp/aDbsUH4jv8oXE89P
YGK+iJNuj0Ta3y8ewwFcLR4=
-----END CERTIFICATE-----
```
### tls.key
```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAsl5jX/+qgUMwY3WU7pK/MVEIhwQltpskl4rDA2IgmHxQdd17
EvwpWLALqVPvXT6oOm4Xb5ZUiE4G7z7dd8zU+4dRlWJw+z+0LLJNTq1T778n9Vbk
mn22BcomAgc/0kSQBlsoGL2qrU5gMDM5sWeHOSTJPCYuM2pC1q1XUsxfrhABscZo
zX8VBMp5KqixhORs1+EnCRaPO0kIasPIJVgT/rEXZv337cae8NqXwRm8zPvJN/QA
1gTiuQON6ZIojnEoH5G563oY0q144r47tBCwqHDe+lNMiIRqJ8326OoYtygF1j5v
Hwshruhmiuqp0s12b9kxaPKgiP8EK8X7Dm1TkQIDAQABAoIBAQCUw+yWMKKA1Q0S
0WmpxzG0Qd1HPsuIIIYXqOqGoqSYJmm7G37tH6L+FZ2UaXNaL9GU8DHyAFv7n5FN
fl2P0S1TuOloLO1NU+sZLKlU8z7y2elataW1G+rpAoGAdv1N+TSFh65xKkYiHvvV
TD/rBT6yuoo4PusyTOQp6c8A2AslY5kmNC4L0kvoECytOAnRPf67aYKIc2oJl0PY
LgAEqxG9Eju3FI7n1KVHqYZJaiEnsOvHG1oAncibLvHxRf1kipuSagZnij62tBjg
swJhi7tYqu2iHqck6I98cikCgYEAozkNkx5HuSg7APobNu8kwNkOA3lGmic4I0JC
UJost4EcmuiLJ53gWtS2KnQC2k2l8MIubXfl0CL6SQlYUubHRL/WNj1z4I4Dz47c
J7AFubgI7hmsheDZ8BSKiQo990lqg7kRLNYnzoK2WJE+cE+GWRW2G2vJXi7jzrow
bbx4BwECgYAFuS6bYdORzBhAUoyS/n4tkZE1cETEihP2w4Yjm3v97zpawbq5Z6+E
BURS8oYF8dS73AekWE2khhYxF7CX3WzVVohBFrPi/iS0Bh8v/a+MtR0Ta1mASwlO
WzSZ73+7yAiBAqFwxcQn71eo6uKa4A01/rKT7vnvrRj3xiL4UIGVQw==
-----END RSA PRIVATE KEY-----
```
