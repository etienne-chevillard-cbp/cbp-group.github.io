---
layout: post
title:  "mTLS / Istio : Sécuriser un service exposé à l'exterieur du service mesh"
author: seb
categories: [ Tutorial, Sécurité, Istio]
tags: [mTLS, istio, sécurité]
image: assets/images/mtls-istio/mtls-istio.jpg
description: "Cet article décrit comment exposer un service à l'extérieur d'Istio en le sécurisant avec du mutual TLS."
featured: true
hidden: false
comments: false
toc: true
beforetoc: "Cet article décrit comment exposer un service à l'extérieur d'Istio en le sécurisant avec du mutual TLS."
---

Cet article décrit comment exposer un service à l'extérieur d'Istio en le sécurisant avec du mutual TLS.

## Context

Tout le monde connait ou a entendu parler de TLS (anciennement SSL). Ce protocole permet de chiffrer les communications et de s'assurer que le client discute avec le bon serveur, ce dernier s'authentifiant auprès du client en envoyant un certificat.

Le mTLS (ou mutual TLS) est une version étendue du protocole TLS qui, toujours en permettant de chiffrer les communications, permet non seulement au client de s'assurer que le serveur auquel il s'addresse est le bon, mais permet aussi au serveur d'être sûr que le client est bien autorisé à l'appeler.

Dans cette article, nous allons voir comment configurer Istio pour exposer un service à l'extérieur du service mesh en le sécurisant avec le protocole mTLS.

Ce modèle avec authentification mutuelle apporte évidemment un meilleur niveau de sécurité (Merci à notre équipe cybersécurité :))
## Quelques rappels sur Istio

L'exposition d'un service au sein d'un service mesh géré avec Istio ce décompose ainsi :
- Une exposition du cluster via un Loadbalancer.
- Un point d’entrée dans le cluster avec terminaison ssl réalisée par l’Ingress Gateway Istio.
- Le certificat est stocké dans un secret dans le cluster.
- La configuration du certificat TLS est réalisée dans la configuration de l'Ingress Gateway Istio.

![image](/assets/images/mtls-istio/acces-application.png){:class="img-responsive"}

La configuration de l'Ingress Gateway Istio est réalisée avec :
- Une ressource custom `Gateway`
- Une ressource custom `VirtualService`

![image](/assets/images/mtls-istio/istio-ingress-gateway.png){:class="img-responsive"}

## De quoi avons-nous besoin ?

Nous avons besoin :
En plus d'un cluster Kubernetes avec istio d'installé (il est possible d'utiliser [microK8s](https://microk8s.io/){:target="_blank"} et son plugin istio pour tester. N'ayant dans ce cas pas de Load Balancer, on accédera à la gateway istio en l'exposant en local avec la commande `mk8sctl port-forward -n istio-system service/istio-ingressgateway 8080:80`)
- d'une autorité de certification pour valider les certificats lors de la négociation TLS.
- d'un "certificat serveur" pour que le serveur s'identifie auprès du client.
- d'un "certificat client" pour que le client s'identifie auprès du server.
- d'une clé privée pour le client pour chiffrer le premier échange avec le serveur.
- d'une application de test. Nous utiliserons l'appli httpbin fournie comme example avec Istio.

## Génération des certificats

Pour réaliser ces tâches, nous allons utiliser OpenSSL.

### Génération de l'autorité de certification

{% highlight shell %}
openssl req \
  -new \
  -x509 \
  -nodes \
  -days 365 \
  -subj '/CN=my-ca' \
  -keyout ca.key \
  -out ca.crt
{% endhighlight %}

On obtient le fichier `ca.crt` qui sera `à fournir au client et au serveur` pour que les 2 puissent valider les certificats qui vont être générés ci-après.
On obtient aussi le fichier `ca.key` contenant la clé privée permettant de signer des certificats avec cette autorité.

Inspection de l'autorité de certification

{% highlight shell %}
openssl x509 \
  -in ca.crt \
  -text \
  -noout

Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 11572271673436068397 (0xa098f7c64378da2d)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=my-ca
        Validity
            Not Before: Nov 16 15:56:24 2021 GMT
            Not After : Nov 16 15:56:24 2022 GMT
        Subject: CN=my-ca
{% endhighlight %}

On voit bien que les champs Issuer et Subject sont les mêmes. Nous avons donc notre autorité de certification.

### Génération de la clé et du certificat pour le serveur

#### Génération de la clé

{% highlight shell %}
openssl genrsa \
  -out server.key 4096
{% endhighlight %}

#### Création de la Certificate Signing Request (CSR)

{% highlight shell %}
openssl req \
  -new \
  -key server.key \
  -subj '/CN=<DOMAINE CONCERNE PAR LE CERTIFICAT  (ex. test.mondomaine.com)>' \
  -out server.csr
{% endhighlight %}

#### Signature du certificat serveur

{% highlight shell %}
openssl x509 \
  -req \
  -in server.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -days 365 \
  -out server.crt
{% endhighlight %}

### Génération de la clé et du certificat pour le client

#### Génération de la clé

{% highlight shell %}
openssl genrsa \
  -out client.key 4096
{% endhighlight %}

#### Création de la Certificate Signing Request (CSR)

{% highlight shell %}
openssl req \
  -new \
  -key client.key \
  -subj '/CN=my-client' \
  -out client.csr
{% endhighlight %}

#### Signature du certificat client

Dans notre cas, nous maîtrisons "à la fois" le client et le serveur. `Dans le cas d'une configuration pour appeler un service fourni par un tiers`, celui-ci ne fournira que l'équivalent de notre ca.crt et ne diffusera jamais sa clé privée (le ca.key). Ce certificat seul permet de vérifier un autre certificat mais ne permet pas d'en signer. `Il faudra donc lui envoyer la CSR` afin qu'il nous renvoie le certificat client signé.

Pour cet article, nous allons signer notre certificat client nous-même.

{% highlight shell %}
openssl x509 \
  -req \
  -in client.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -days 365 \
  -out client.crt
{% endhighlight %}

## Configuration d'Istio

### Création du secret

Nous allons créer un secret contenant le certificat serveur et sa clé privée ainsi que le certificat de l'autorité de certification.

Le secret est de type `generic` mais il est possible de créer un secret de type `tls`. Seule la syntaxe des clés dans le secret change. Voir la [documentation d'Istio](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/#key-formats)

L'application déployée ici étant `httpbin`, le secret est nommé `httpbin-mtls` mais le nom importe peu.

{% highlight shell %}
kubectl create -n istio-system secret generic httpbin-mtls  \
--from-file=key=server.key \
--from-file=cert=server.crt \
--from-file=cacert=ca.crt
{% endhighlight %}

### Déploiement de l'application

Contenu du fichier `httpbin.yaml` contenant la description du déploiement :

{% highlight yaml %}
# Copyright Istio Authors
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

##################################################################################################
# httpbin service
##################################################################################################
apiVersion: v1
kind: ServiceAccount
metadata:
  name: httpbin
  namespace: httpbin
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    app: httpbin
    service: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  namespace: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      serviceAccountName: httpbin
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 80
{% endhighlight %}

Déploiement
{% highlight shell %}
kubectl apply -f httpbin.yaml
{% endhighlight %}

### Configuration de l'Ingress Gateway Istio

Nous allons déployer deux ressources custom d'Istio pour configurer l'Ingress Gateway :
- une ressource de type `VirtualService` pour router les requêtes vers notre application.
- une ressource de type `Gateway` pour configurer les `hosts` traités par l'Ingress Gateway Istio ainsi que la partie `TLS`.

Contenu du fichier `httpbin-gw-and-vs.yaml` contenant la description du déploiement :

{% highlight yaml %}
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  namespace: istio-system
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - hosts:
        - "<URL DE MON APPLICATION AVEC LE MEME DOMAINE QUE LE CERTIFICAT (ex. test.mondomaine.com)>"
      port:
        number: 80
        name: http-httpbin
        protocol: HTTP
      tls:
        httpsRedirect: true
    - hosts:
        - "<URL DE MON APPLICATION AVEC LE MEME DOMAINE QUE LE CERTIFICAT (ex. test.mondomaine.com)>"
      port:
        number: 443
        name: https-httpbin
        protocol: HTTPS
      tls:
        mode: MUTUAL
        credentialName: "httpbin-mtls"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - "<URL DE MON APPLICATION AVEC LE MEME DOMAINE QUE LE CERTIFICAT (ex. test.mondomaine.com)>"
  gateways:
    - istio-system/httpbin-gateway
  http:
    - route:
        - destination:
            host: httpbin
            port:
              number: 8000
{% endhighlight %}

La configuration du mutual TLS tient en ces lignes :

{% highlight yaml %}
      tls:
        mode: MUTUAL
        credentialName: "httpbin-mtls"
{% endhighlight %}

On spécifie le mode `MUTUAL` et le nom du secret créé précédemment pour la valeur de `credentialName`.


{% highlight shell %}
kubectl apply -f httpbin-gw-and-vs.yaml
{% endhighlight %}

## Tests

### Avec le l'autorité de certification  mais sans utiliser le certificat client
{% highlight shell %}
curl -I \
  --cacert ca.crt \
  https://test.mondomaine.com

curl: (35) error:1401E410:SSL routines:CONNECT_CR_FINISHED:sslv3 alert handshake failure
{% endhighlight %}

### Sans utiliser le certificat client ni l'autorité de certification 
{% highlight shell %}
curl -I https://test.mondomaine.com 

curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
{% endhighlight %}

### Avec le certificat client mais sans utiliser l'autorité de certification 

{% highlight shell %}
curl -I \
  --key client.key \
  --cert client.crt \
  https://https://test.mondomaine.com 

curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
{% endhighlight %}

### Avec le certificat client et l'autorité de certification 
{% highlight shell %}
curl -I \
  --cacert ca.crt \
  --key client.key \
  --cert client.crt \
  https://test.mondomaine.com 

HTTP/2 200
server: istio-envoy
date: Fri, 03 Dec 2021 15:51:03 GMT
content-type: text/html; charset=utf-8
content-length: 9593
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 1
{% endhighlight %}
