---
layout: post
title: "Vaultwarden - storage of secrets"
date: 2024-12-21 09:00:00 +0200
tags: vaultwarden secret password bitwarden manager
published: true
---
## Preface
Vaultwarden is alternative implementation of the Bitwarden (commercial) server, which can be used as container to store your passwords or any other secret/sensitive data. 


## HTTPS
For proper operation of vaultwarden, enabling HTTPS is required nowadays, since the Bitwarden web vault uses web crypto APIs that most browsers only make available in HTTPS contexts.
There are a few ways you can enable HTTPS:
- put vaultwarden behind a reverse proxy that handles HTTPS connections on behalf of vaultwarden.
- enable the HTTPS functionality built into vaultwarden (via the Rocket web framework). Rocket's HTTPS implementation is relatively immature and limited.

To use built in (Rocket's HTTP implementation) need to generate key and certificate, as going forward vaultwarden application will be used on phone it is required properly to generate CA certificate and then use it to generate certificate for vaultwarden.

Example how to generate CA key and certificate (for 10 years), it is highly recommended to secure ca-key by password:
```
openssl genpkey -algorithm RSA -aes128 -out ca.key -outform PEM -pkeyopt rsa_keygen_bits:2048

openssl req -x509 -new -nodes -sha256 -days 3650 -key ca.key -out ca.crt
```

Prepare vaultwarden extension file
```
cat > vw.ext
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = vw
DNS.2 = vw.lan
DNS.3 = vaultwarden
DNS.4 = vaultwarden.lan
```

Generate vaultwarden key and issue certificate which will be signed by CA-certificate
```
openssl genpkey -algorithm RSA -out vw.key -outform PEM -pkeyopt rsa_keygen_bits:2048
.............+++++
.......+++++

openssl genpkey -algorithm RSA -out vw.key -outform PEM -pkeyopt rsa_keygen_bits:2048
.............+++++
.......+++++
root@p5:~# openssl req -new -key vw.key -out vw.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:LT
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:Vilnius
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:vaultwarden
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:


openssl x509 -req -in vw.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out vw.crt -days 730 -sha256 -extfile vw.ext
Signature ok
subject=C = LT, ST = Some-State, L = Vilnius, O = Internet Widgits Pty Ltd, CN = vaultwarden
Getting CA Private Key
Enter pass phrase for ca.key:
```

After issuing certificate for vaultwarden it is important to move CA-key to some offline storage, it will be needed only to renew certificate for vaultwarden (in 2 years).

## Installation with Helm
It’s recommended to have persistent volume claim created before as there will be stored main database (db.sqlite3), in example below nfs-client storageclass is used for that. LoadBalancer service type is used based on Metallb functionality.


```
cat > vaultwarden-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: "vaultwarden"
  name: "vaultwarden-pvc"
  annotations:
    volume.beta.kubernetes.io/storage-class: nfs-client
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: "1Gi"
  
cat > vaultwarden-web.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    metallb.universe.tf/loadBalancerIPs: YOUR_VW_IP_ADDRESS
  name: vaultwarden-web
  namespace: vaultwarden
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: vaultwarden
    app.kubernetes.io/instance: vaultwarden
  ports:
    - port: 443
      targetPort: 80

cat > vaultwarden-values.yaml
env:
  DATA_FOLDER: "config"
  ROCKET_TLS: '{certs="/config/vw.crt",key="/config/vw.key"}'
persistence:
  config:
    enabled: true
    existingClaim: "vaultwarden-pvc"

kubectl create namespace vaultwarden

helm repo add k8s-at-home https://k8s-at-home.com/charts

helm install -n vaultwarden--values vaultwarden-values.yaml vaultwarden k8s-at-home/vaultwarden
```

After initial deployment it will be needed to terminate pod to copy vw.key and vw.crt files then into newly created persistent volume (under config-folder).

Then to start pod again and it should be accessible on https://YOUR_VW_IP_ADDRESS url.

