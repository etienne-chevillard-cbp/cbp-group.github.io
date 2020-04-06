---
layout: post
title:  "Kube / Istio / External-Dns / Cert-Manager/ Let's Encrypt - Partie 2/2"
author: seb
categories: [ Kubernetes, Tutorial ]
tags: [Kubernetes, Istio, Let's Encrypt, External-Dns, Cert-Manager]
image: assets/images/kube-istio-externaldns-cert-manager-letsencrypt-part2/illustration.jpg
description: "Cet article en deux parties décrit comment utiliser Istio, External-Dns et Cert-Manager dans un cluster Kubernetes pour déployer automatiquement une application accessible en HTTPS et que les entrées DNS et le certificat Let's Encrypt soient créés automatiquement lors de ce déploiement."
featured: true
hidden: false
comments: false
toc: true
beforetoc: "Deuxième partie de l'article qui décrit comment utiliser Istio, External-Dns et Cert-Manager dans un cluster Kubernetes pour déployer automatiquement une application accessible en HTTPS et que les entrées DNS et le certificat Let's Encrypt soient créés automatiquement lors de ce déploiement. Dans cette deuxième partie, nous allons aborder Cert-manager et l'installation de l'application."
---

La première partie de l'arcticle est disponible [ici]({% post_url 2020-04-3-kube-istio-externaldns-cert-manager-letsencrypt-part1 %}).

# Cert-Manager

## Composants

### Cert-Manager

Cert-manager vient sous la forme de trois Pods :

{% highlight shell %}
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5655447474-pfkb8              1/1     Running   1          21d
cert-manager-cainjector-59c9dfd4f7-8kwr4   1/1     Running   0          5d22h
cert-manager-webhook-865b8fb666-77phk      1/1     Running   0          5d22h
{% endhighlight %}

### Issuer

Les Issuers et Cluster Issuers sont des ressources Kubernetes qui représentent des autorités de certification (CA) capables de générer des certificats signés en honorant les demandes de signature de certificats. Tous les certificats des gestionnaires de certificats nécessitent un émetteur référencé qui est en état de répondre à la demande.

Celui utilisé ici est configuré pour Let’s Encrypt en mode staging:

{% highlight yaml %}
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
 name: letsencrypt-staging
 namespace: cert-manager
spec:
 acme:
   # The ACME server URL
   server: https://acme-staging-v02.api.letsencrypt.org/directory
   # Email address used for ACME registration
   email: firstname.lastname@xxxxxxx.com
   # Name of a secret used to store the ACME account private key
   privateKeySecretRef:
     name: letsencrypt-staging
   # Enable the HTTP-01 challenge provider
   solvers:
     - selector: {}
       http01:
           ingress:
             class: istio
{% endhighlight %}

### Certificate

Cert-manager ajoute la Custom Resource Definition (CRD) `Certificat` qui définit un certificat x509 souhaité et qui sera renouvelé et tenu à jour. Un certificat est une ressource lié à un namespace qui fait référence à un Issuer ou Cluster Issuer qui détermine ce qui honorera la demande de certificat.

Lorsqu'une CRD `Certificat` est créé, une CRD `CertificateRequest` correspondante est créée par le gestionnaire de certificats, contenant la demande de certificat x509 encodée, la référence de l'émetteur et d'autres options basées sur la spécification de la CRD `Certificat`.

Ci dessous, le certificat utilisé dans notre cas:

{% highlight yaml %}
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
 annotations:
   kubectl.kubernetes.io/last-applied-configuration: |
     {"apiVersion":"cert-manager.io/v1alpha2","kind":"Certificate","metadata":{"annotations":{},"name":"application.sample.com","namespace":"istio-system"},"spec":{"commonName":"application.sample.com","dnsNames":["application.sample.com"],"issuerRef":{"kind":"ClusterIssuer","name":"letsencrypt-staging"},"secretName":"application.sample.com"}}
 creationTimestamp: "2020-03-24T10:27:33Z"
 generation: 1
 name: application.sample.com
 namespace: istio-system
 resourceVersion: "7238959"
 selfLink: /apis/cert-manager.io/v1alpha2/namespaces/istio-system/certificates/application.sample.com
 uid: 4ad2e950-2bd9-48ae-ba3e-778a1093cd19
spec:
 commonName: application.sample.com
 dnsNames:
 - application.sample.com
 issuerRef:
   kind: ClusterIssuer
   name: letsencrypt-staging
 secretName: application.sample.com
status:
 conditions:
 - lastTransitionTime: "2020-03-24T10:27:33Z"
   message: Waiting for CertificateRequest "application.sample.com-1891847550"
     to complete
   reason: InProgress
   status: "False"
   type: Ready
{% endhighlight %}

### Challenge

Les Challenges sont utilisées par l'émetteur d'ACME pour gérer le cycle de vie d'un `défi` ACME qui doit être complété afin de compléter une `autorisation` pour un nom/identifiant DNS unique.

Un Challenge est créé pour chaque nom DNS qui est autorisé par le serveur ACME.

En tant qu'utilisateur final, vous n'aurez jamais besoin de créer manuellement une ressource Challenge.

{% highlight yaml %}
apiVersion: acme.cert-manager.io/v1alpha2
kind: Challenge
metadata:
 creationTimestamp: "2020-03-24T10:27:36Z"
 finalizers:
 - finalizer.acme.cert-manager.io
 generation: 1
 name: application.sample.com-1891847550-1229735898-2462782750
 namespace: istio-system
 ownerReferences:
 - apiVersion: acme.cert-manager.io/v1alpha2
   blockOwnerDeletion: true
   controller: true
   kind: Order
   name: application.sample.com-1891847550-1229735898
   uid: 78505800-ca0f-49eb-a63e-50bc099ade4f
 resourceVersion: "7238986"
 selfLink: /apis/acme.cert-manager.io/v1alpha2/namespaces/istio-system/challenges/application.sample.com-1891847550-1229735898-2462782750
 uid: 7dd6b485-394a-43d6-b5cb-9612bf6c9360
spec:
 authzURL: https://acme-staging-v02.api.letsencrypt.org/acme/authz-v3/45257162
 dnsName: application.sample.com
 issuerRef:
   kind: ClusterIssuer
   name: letsencrypt-staging
 key: fbosTcirMUkf2ZGbD7E6HGgWM2aXchRvBKiJvbuLWDc.gsu3fJ7fFYXPjtJbN45VPBcP5URGdzSzO2Wmy37EFG0
 solver:
   http01:
     ingress:
       class: istio
 token: fbosTcirMUkf2ZGbD7E6HGgWM2aXchRvBKiJvbuLWDc
 type: http-01
 url: https://acme-staging-v02.api.letsencrypt.org/acme/chall-v3/45257162/7t5yQw
 wildcard: false
status:
 presented: true
 processing: true
 reason: 'Waiting for http-01 challenge propagation: failed to perform self check
   GET request ''http://application.sample.com/.well-known/acme-challenge/fbosTcirMUkf2ZGbD7E6HGgWM2aXchRvBKiJvbuLWDc'':
   Get http://application.sample.com/.well-known/acme-challenge/fbosTcirMUkf2ZGbD7E6HGgWM2aXchRvBKiJvbuLWDc:
   dial tcp: lookup application.sample.com on 172.20.0.10:53: no such host'
 state: pending
{% endhighlight %}

### Resolver

#### Pod

Lorsqu’un certificat est demandé via le protocole ACME en mode HTTP, un Pod contenant le Token généré par l’autorité de certification permettant de résoudre le challenge  est créé par cert-manager. Ce Pod disparaît lorsque le challenge est résolu et que le certificat a été généré.

Exemple de Pod:

{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
 annotations:
   kubernetes.io/psp: eks.privileged
   sidecar.istio.io/inject: "false"
 creationTimestamp: "2020-03-24T10:27:37Z"
 generateName: cm-acme-http-solver-
 labels:
   acme.cert-manager.io/http-domain: "2167933383"
   acme.cert-manager.io/http-token: "1225395914"
   acme.cert-manager.io/http01-solver: "true"
 name: cm-acme-http-solver-5zppf
 namespace: istio-system
 ownerReferences:
 - apiVersion: acme.cert-manager.io/v1alpha2
   blockOwnerDeletion: true
   controller: true
   kind: Challenge
   name: application.sample.com-1891847550-1229735898-2462782750
   uid: 7dd6b485-394a-43d6-b5cb-9612bf6c9360
 resourceVersion: "7238995"
 selfLink: /api/v1/namespaces/istio-system/pods/cm-acme-http-solver-5zppf
 uid: ca42dda5-70c9-454a-a58c-4f2523f00f0e
spec:
 containers:
 - args:
   - --listen-port=8089
   - --domain=application.sample.com
   - --token=fbosTcirMUkf2ZGbD7E6HGgWM2aXchRvBKiJvbuLWDc
   - --key=fbosTcirMUkf2ZGbD7E6HGgWM2aXchRvBKiJvbuLWDc.gsu3fJ7fFYXPjtJbN45VPBcP5URGdzSzO2Wmy37EFG0
   image: quay.io/jetstack/cert-manager-acmesolver:v0.13.1
   imagePullPolicy: IfNotPresent
   name: acmesolver
   ports:
   - containerPort: 8089
     name: http
     protocol: TCP
   resources:
     limits:
       cpu: 100m
       memory: 64Mi
     requests:
       cpu: 10m
       memory: 64Mi
   terminationMessagePath: /dev/termination-log
   terminationMessagePolicy: File
   volumeMounts:
   - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
     name: default-token-x6gr7
     readOnly: true
 dnsPolicy: ClusterFirst
 enableServiceLinks: true
 nodeName: ip-10-96-147-38.eu-west-3.compute.internal
 priority: 0
 restartPolicy: OnFailure
 schedulerName: default-scheduler
 securityContext: {}
 serviceAccount: default
 serviceAccountName: default
 terminationGracePeriodSeconds: 30
 tolerations:
 - effect: NoExecute
   key: node.kubernetes.io/not-ready
   operator: Exists
   tolerationSeconds: 300
 - effect: NoExecute
   key: node.kubernetes.io/unreachable
   operator: Exists
   tolerationSeconds: 300
 volumes:
 - name: default-token-x6gr7
   secret:
     defaultMode: 420
     secretName: default-token-x6gr7
status:
 conditions:
 - lastProbeTime: null
   lastTransitionTime: "2020-03-24T10:27:37Z"
   status: "True"
   type: Initialized
 - lastProbeTime: null
   lastTransitionTime: "2020-03-24T10:27:38Z"
   status: "True"
   type: Ready
 - lastProbeTime: null
   lastTransitionTime: "2020-03-24T10:27:38Z"
   status: "True"
   type: ContainersReady
 - lastProbeTime: null
   lastTransitionTime: "2020-03-24T10:27:37Z"
   status: "True"
   type: PodScheduled
 containerStatuses:
 - containerID: docker://55cbdc878a0285c615622fd51a696d1f2a7d65fbb9dac44031654e9400452f12
   image: quay.io/jetstack/cert-manager-acmesolver:v0.13.1
   imageID: docker-pullable://quay.io/jetstack/cert-manager-acmesolver@sha256:de550b673cf29876e8107dfb93d134281add9af8c1d6b84132c1812a71986bc4
   lastState: {}
   name: acmesolver
   ready: true
   restartCount: 0
   state:
     running:
       startedAt: "2020-03-24T10:27:38Z"
 hostIP: 10.96.147.38
 phase: Running
 podIP: 10.96.146.198
 qosClass: Burstable
 startTime: "2020-03-24T10:27:37Z"
{% endhighlight %}

#### Service

Un service est créé par cert-manager pour exposer le pod contenant la réponse au challenge (le Token). Il correspond à la description ci dessous.

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  annotations:
    auth.istio.io/8089: NONE
  creationTimestamp: "2020-03-24T10:27:37Z"
  generateName: cm-acme-http-solver-
  labels:
    acme.cert-manager.io/http-domain: "2167933383"
    acme.cert-manager.io/http-token: "1225395914"
    acme.cert-manager.io/http01-solver: "true"
  name: cm-acme-http-solver-qqw7l
  namespace: istio-system
  ownerReferences:
  - apiVersion: acme.cert-manager.io/v1alpha2
    blockOwnerDeletion: true
    controller: true
    kind: Challenge
    name: application.sample.com-1891847550-1229735898-2462782750
    uid: 7dd6b485-394a-43d6-b5cb-9612bf6c9360
  resourceVersion: "7238982"
  selfLink: /api/v1/namespaces/istio-system/services/cm-acme-http-solver-qqw7l
  uid: aa34aebd-31c0-4312-8c87-cf07ed617cca
spec:
  clusterIP: 172.20.103.179
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 32221
    port: 8089
    protocol: TCP
    targetPort: 8089
  selector:
    acme.cert-manager.io/http-domain: "2167933383"
    acme.cert-manager.io/http-token: "1225395914"
    acme.cert-manager.io/http01-solver: "true"
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
{% endhighlight %}

#### Ingress

En plus du Pod et du service, un Ingress Kubernetes est créé par cert-manager pour accéder au service. Dans notre cas, cet Ingress est détecté par Istio qui va ajouter à l’Ingress Gateway une route vers le service exposant le Pod contenant la réponse au challenge (le Token). Cette route est supprimée lorsque le challenge est résolu.

{% highlight yaml %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 annotations:
   kubernetes.io/ingress.class: istio
   nginx.ingress.kubernetes.io/whitelist-source-range: 0.0.0.0/0,::/0
 creationTimestamp: "2020-03-24T10:27:37Z"
 generateName: cm-acme-http-solver-
 generation: 1
 labels:
   acme.cert-manager.io/http-domain: "2167933383"
   acme.cert-manager.io/http-token: "1225395914"
   acme.cert-manager.io/http01-solver: "true"
 name: cm-acme-http-solver-rqgd4
 namespace: istio-system
 ownerReferences:
 - apiVersion: acme.cert-manager.io/v1alpha2
   blockOwnerDeletion: true
   controller: true
   kind: Challenge
   name: application.sample.com-1891847550-1229735898-2462782750
   uid: 7dd6b485-394a-43d6-b5cb-9612bf6c9360
 resourceVersion: "7238983"
 selfLink: /apis/extensions/v1beta1/namespaces/istio-system/ingresses/cm-acme-http-solver-rqgd4
 uid: 851fa1af-2b4f-46ab-b007-7a2706ed32cc
spec:
 rules:
 - host: application.sample.com
   http:
     paths:
     - backend:
         serviceName: cm-acme-http-solver-qqw7l
         servicePort: 8089
       path: /.well-known/acme-challenge/fbosTcirMUkf2ZGbD7E6HGgWM2aXchRvBKiJvbuLWDc
status:
 loadBalancer: {}
{% endhighlight %}

## Fonctionnement

![image](/assets/images/kube-istio-externaldns-cert-manager-letsencrypt-part2/certmanager.png){:class="img-responsive"}

1. Un objet `Certificat` est déployé sur le cluster. Cet objet contient notamment les informations suivantes:
    1. Le nom DNS pour lequel on souhaite un certificat.
    2. Le type et le nom de l’Issuer à utiliser.
    3. Le nom du Secret qui contiendra le certificat. Ce nom doit être le même que celui indiqué dans le paramétrage de la Gateway.
2. Cert-Manager récupère les informations du Certificat.
3. Cert-Manager récupère les informations de configuration de l’Issuer indiqué dans le Certificat.
4. Cert-Manager va créer un challenge (non représenté ici) ainsi que le nécessaire pour résoudre le challenge.
5. L’ingress Gateway ajoute une route correspondant au service exposant le pod contenant la réponse au challenge (un Token).
6. Cert-Manager demande un certificat au serveur configuré dans l’Issuer (ici Let’s Encrypt staging).
7. Let’s Encrypt va résoudre le nom. C’est pour cela que External-Dns est nécessaire.
8. Let’s Encrypt va appeler le Load Balancer exposant l’Ingress Gateway à l'extérieur du Cluster et qui expose donc l’application et le resolver.
9. Le Pod contenant la réponse au challenge (le Token) est appelé et le Token renvoyé à Let’s Encrypt qui va générer le certificat.
10. Cert-Manager récupère le certificat et le stocke dans un Secret Kubernetes.
11. L’Istio Ingress Gateway récupere le certificat dans secret via le Secret Discovery Service (sds) d’Istio.

> Une fois ceci réalisé, l’Ingress, le Service et le Pod du resolver sont supprimés.

## Configuration

Copier la configuration de l’Issuer dans un fichier `letsencrypt-staging-ClusterIssuer.yaml`.

{% highlight yaml %}
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
 name: letsencrypt-staging
 namespace: cert-manager
spec:
 acme:
   # The ACME server URL
   server: https://acme-staging-v02.api.letsencrypt.org/directory
   # Email address used for ACME registration
   email: sebastien.errien@cbp-group.com
   # Name of a secret used to store the ACME account private key
   privateKeySecretRef:
     name: letsencrypt-staging
   # Enable the HTTP-01 challenge provider
   solvers:
     - selector: {}
       http01:
           ingress:
             class: istio
{% endhighlight %}

## Déploiement

### Cert-Manager

{% highlight yaml %}
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.13.1/cert-manager.yaml
{% endhighlight %}

### Issuers

{% highlight yaml %}
kubectl apply -f letsencrypt-staging-ClusterIssuer.yaml -n cert-manager
{% endhighlight %}

# Application

## Composants

![image](/assets/images/kube-istio-externaldns-cert-manager-letsencrypt-part2/application.png){:class="img-responsive"}

Le déploiement d’une application comprend plusieurs objets :
- Le ou les Pods où tourne l’application.
- Le service (de type ClusterIp) permettant d'accéder aux Pods.
- Une Gateway (Custom Resource Définition fournie par Istio) contenant la configuration concernant les ports. utilisés, les hostnames acceptés ainsi que la configuration TLS (SSL).
- Le Virtual Service permettant de gérer plus finement les règles d'accès et de routage.
- A CRD `Certificat` pour l’obtention du certificat.

## Configuration

### Application

L’application utilisé est `httpbin`. Elle est présente dans les exemples fournis par istio (istio-1.5.0/samples/httpbin).

La configuration est dans un fichier httpbin.yaml: 

{% highlight yaml %}
apiVersion: v1
kind: ServiceAccount
metadata:
 name: httpbin
---
apiVersion: v1
kind: Service
metadata:
 name: httpbin
 labels:
   app: httpbin
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

### Gateway

La Gateway est configurée pour paramétrer l’Istio Ingress Gateway ayant le label istio:  `ingressgateway-ssl`. 
Elle possède l’annotation `externaldns: ssl` pour être prise en compte par External-Dns.
Elle accepte les requêtes sur le nom d’hôte `application.sample.com` uniquement.
Elle écoute sur les ports 80 et 443. Les requêtes sur le port 80 sont redirigées automatiquement sur le port 443. 
Le nom du secret contenant le certificat pour le port 443 est `application.sample.com`.

Copier le contenu suivant dans un fichier `gateway-application.yaml`:

{% highlight yaml %}
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
 name: application-gateway
 namespace: istio-system
 labels:
   release: istio
 annotations:
   externaldns: ssl
spec:
 selector:
   istio: ingressgateway-ssl # use istio default controller
 servers:
 - port:
     number: 443
     name: https
     protocol: HTTPS
   hosts:
   - "application.sample.com"
   tls:
     credentialName: application.sample.com
     mode: SIMPLE # enables HTTPS on this port
    
 - port:
     number: 80
     name: http
     protocol: HTTP
   hosts:
   - "application.sample.com"
   tls:
     httpsRedirect: true # sends 301 redirect for http requests
{% endhighlight %}

### Virtual Service

Le Virtual Service est configuré pour être rattaché à la Gateway nommée `application-gateway` dans le namespace `istio-system`.

Copier le contenu suivant dans un fichier `virtualservice-application.yaml`:

{% highlight yaml %}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
 name: bookinfo
 namespace: apps
spec:
 hosts:
 - "application.sample.com"
 gateways:
 - istio-system/application-gateway
 http:
 - match:
   - uri:
       exact: /productpage
   - uri:
       prefix: /static
   - uri:
       exact: /login
   - uri:
       exact: /logout
   - uri:
       prefix: /api/v1/products
   route:
   - destination:
       host: productpage
       port:
         number: 9080
{% endhighlight %}

### Certificate

Le certificat est configuré pour utiliser l’Issuer `letsencrypt-staging`, et pour obtenir un certificat pour le nom `application.sample.com`.

Copier le contenu suivant dans un fichier `certificate-application.yaml`:

{% highlight yaml %}
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
 name: application.sample.com
 namespace: istio-system
spec:
 commonName: application.sample.com
 dnsNames:
 - application.sample.com
 issuerRef:
   kind: ClusterIssuer
   name: letsencrypt-staging
 secretName: application.sample.com
{% endhighlight %}

Déploiement
Le déploiement consiste à appliquer les fichiers yaml avec kubectl. Il est possible de ne faire qu’un seul fichier contenant toute la configuration pour n’avoir qu’un commande à exécuter.

{% highlight shell %}
kubectl apply -f httpbin.yaml -n apps
kubectl apply -f virtualservice-application.yaml -n apps
kubectl apply -f gateway-application.yaml -n istio-system
kubectl apply -f certificate-application.yaml -n istio-system
{% endhighlight %}

## Tests

### Redirection

{% highlight yaml %}
curl -i http://application.sample.com                                       

HTTP/1.1 301 Moved Permanently
location: https://application.sample.com/
date: Tue, 24 Mar 2020 17:51:18 GMT
server: istio-envoy
content-length: 0
{% endhighlight %}

### HTTPS

{% highlight yaml %}
curl -i https://application.sample.com                                   

HTTP/2 200
server: istio-envoy
date: Tue, 24 Mar 2020 17:51:45 GMT
content-type: text/html; charset=utf-8
content-length: 9593
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 3

<!DOCTYPE html>
<html lang="en">
...
</html>
{% endhighlight %}

Et voilà !