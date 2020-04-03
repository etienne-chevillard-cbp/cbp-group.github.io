---
layout: post
title:  "Kube / Istio / External-Dns / Cert-Manager/ Let' Encrypt - Partie 1/2"
author: seb
categories: [ Kubernetes, Tutorial ]
tags: [Kubernetes, Istio, Let's Encrypt, External-Dns, Cert-Manager]
image: assets/images/kube-istio-externaldns-sert-manager-letsencrypt-part1/illustration.jpg
description: "Cet article en deux parties décrit comment utiliser Istio, External-Dns et Cert-Manager dans un cluster Kubernetes pour déployer automatiquement une application accessible en HTTPS et que les entrées DNS et le certificat Let's Encrypt soient créés automatiquement lors de ce déploiement."
featured: true
hidden: false
comments: false
toc: true
beforetoc: "Cet article en deux parties décrit comment utiliser Istio, External-Dns et Cert-Manager dans un cluster Kubernetes pour déployer automatiquement une application accessible en HTTPS et que les entrées DNS et le certificat Let's Encrypt soient créés automatiquement lors de ce déploiement. Dans cette première partie, nous allons décrire le contexte et l'installation d'Istio ainsi que d'External-Dns."
---

## Context

Le nom d’hôte utilisé dans ce document est `application.sample.com` cela suppose de possèder le nom de domaine `sample.com`.
### Logiciels utilisés

#### Kubernetes v1.15

[Kubernetes](https://kubernetes.io/){:target="_blank"} est un système open-source pour automatiser le déploiement, la mise à l'échelle et la gestion des applications conteneurisées.

#### Istio v1.5.0

[Istio](https://istio.io/){:target="_blank"} est un maillage de services (service mesh) qui assure la gestion du trafic, l'application des politiques de sécurité et la collecte de mesures.

#### External-DNS v0.7.0

[External-Dns](https://github.com/kubernetes-sigs/external-dns){:target="_blank"} synchronise les services et les entrées Kubernetes exposés avec les fournisseurs de DNS.

#### Cert-Manager v0.13.1

[Cert-Manager](https://cert-manager.io/){:target="_blank"} fournit des "certificats en tant que service" aux développeurs travaillant au sein du cluster Kubernetes. Il apporte notamment la prise en charge d'ACME (Let's Encrypt), de HashiCorp Vault, de Venafi, des autorités de certification autosignée et interne. Il est extensible pour prendre en charge les Autorités de Certification personnalisées, internes ou non prises en charge.

### Cluster

Le cluster Kubernetes est déployé sur AWS (EKS). 

Le cluster possède 3 points d’entrée exposés via des Classic Load Balancer AWS:
1. Un point d’entrée privé accessible uniquement via le réseau privé de l’entreprise. La terminaison SSL est réalisée sur le Load Balancer sur lequel un certificat wildcard est déployé (*.sample.com), ce certificat est généré par AWS ACM.
2. Un point d’entrée public avec terminaison SSL réalisée sur le Load Balancer sur lequel un certificat wildcard est déployé (*.sample.com), le même qu’utilisé sur le point d’entrée privé.
3. Un point d’entrée public avec terminaison ssl réalisée sur l’Ingress Gateway Istio avec des certificats SSL générés par Let’s Encrypt.

![image](/assets/images/kube-istio-externaldns-sert-manager-letsencrypt-part1/vue_generale.png){:class="img-responsive"}

## Installation et configuration

### Vue d’ensemble

![image](/assets/images/kube-istio-externaldns-sert-manager-letsencrypt-part1/vue_ensemble.png){:class="img-responsive"}

### Répartition par namespaces

![image](/assets/images/kube-istio-externaldns-sert-manager-letsencrypt-part1/namespaces.png){:class="img-responsive"}

### Pré-requis

#### Outils

Les outils suivants sont nécessaires:
- Istioctl: [https://istio.io/docs/setup/getting-started/](https://istio.io/docs/setup/getting-started/){:target="_blank"}
- Kubectl: [https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/){:target="_blank"}

#### Namespaces 

Les namespaces `Istio-system`, `cert-manager`, `external-dns` et `apps` sont nécessaires. Le namespace `apps` doit avoir le label istio-injection=enabled pour que l’injection des `sidecar` Istio se fasse automatiquement.

{% highlight shell %}
kubectl create namespace istio-system
kubectl create namespace cert-manager
kubectl create namespace external-dns
kubectl create namespace apps
kubectl label namespace apps istio-injection=enabled
{% endhighlight %}

### Istio

#### Fichier de configuration

On commence par générer un fichier manifest.yaml qui va contenir la description de toutes les ressources Istio nécessaires:

{% highlight shell %}
istioctl manifest generate \
--set "values.tracing.enabled"=true \
--set "values.grafana.enabled"=true \
--set "values.kiali.enabled"=true \
--set "values.kiali.dashboard.jaegerURL=http://jaeger-query:16686" \
--set "values.kiali.dashboard.grafanaURL=http://grafana:3000" \
--set "values.gateways.istio-ingressgateway.enabled=true" \
--set "values.gateways.istio-ingressgateway.sds.enabled=true" \
--set "values.global.k8sIngress.enabled=true" \
--set "values.global.k8sIngress.enableHttps=false" \
--set "values.global.k8sIngress.gatewayName=ingressgateway-ssl" > \
manifest.yaml
{% endhighlight %}

Les lignes importantes ici sont:
- `--set "values.gateways.istio-ingressgateway.sds.enabled=true"` qui va permettre d’injecter le certificat automatiquement sur les Istio Ingress Gateway en se basant sur le nom du secret contenant le certificat.
- `--set "values.global.k8sIngress.enabled=true"` qui va permettre à Istio de détecter les objets Ingress de Kubernetes et d’ajouter un routage vers ceux-ci. Cette configuration est nécessaire car Cert-Manager, pour effectuer le challenge HTTP, va créer un pod pour répondre au challenge et va exposer ce pod via un ingress Kubernetes.
- `--set "values.global.k8sIngress.gatewayName=ingressgateway-ssl"` qui permet de déterminer à partir de quelle Istio Ingress Gateway le trafic sera routé sur les Ingress Kubernetes détectés par Istio. Dans notre cas, on souhaite que cela soit par la gateway qui va gérer les certificats SSL de Let’s Encrypt. En effet, nous allons utiliser le challenge HTTP et c’est cette gateway qui sera appelée par Let’s Encrypt lors du challenge.

Les autres paramètres sont là pour déployer Kiali. Kiali est une console d'observabilité pour Istio avec des capacités de configuration de service mesh. Elle aide à comprendre la structure du service mesh en déduisant la topologie, et fournit également la santé de celui-ci. Kiali fournit des mesures détaillées, et une intégration Grafana de base est disponible pour les requêtes avancées. Le traçage distribué est fourni par l'intégration de Jaeger. C’est utile pour voir ce qui est déployé mais n’a pas d’utilité dans le déploiement de l'application.

#### Personnalisation du déploiement Istio

Avant de déployer Istio, nous allons enrichir le fichier de configuration avec les éléments correspondants à l’Istio Ingress Gateway qui va effectuer la terminaison SSL avec les certificats Let’s Encrypt.

### Public SSL Istio Ingress Gateway

![image](/assets/images/kube-istio-externaldns-sert-manager-letsencrypt-part1/detail_gateway.png){:class="img-responsive"}

Une `Ingress Gateway Istio` est composée d’un Service Kubernetes et d’un ou plusieurs Pods correspondants à un proxy Envoy qui sera en charge de router les requêtes.

Le service est ici de type `Load Balancer`. Ainsi, lorsque le service est créé, cela provisionne automatiquement un Load Balancer sur AWS qui sera utilisé pour exposer ce service à l’extérieur du cluster Kubernetes.

Le routage des requêtes est réalisé au niveau du Pod (qui n’est rien d’autre qu’un proxy Envoy). Les règles sont décrites dans deux types d’objets Istio :
- Les objets de type `Gateway` qui contiennent la configuration concernant les ports utilisés, les hostnames acceptés ainsi que la configuration TLS (SSL)
- Les objets de type `VirtualService` qui contiennent les informations de routage en fonction d’URIs, de noms d’hôtes ou de headers particulier notamment.

#### Fichier de configuration

Copier le contenu suivant dans le fichier `manifest.yaml`.

{% highlight yaml %}
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
 labels:
   app: istio-ingressgateway-ssl
   istio: ingressgateway-ssl
   release: istio
 name: istio-ingressgateway-ssl
 namespace: istio-system
spec:
 maxReplicas: 5
 metrics:
 - resource:
     name: cpu
     targetAverageUtilization: 80
   type: Resource
 minReplicas: 1
 scaleTargetRef:
   apiVersion: apps/v1
   kind: Deployment
   name: istio-ingressgateway-ssl
---
apiVersion: apps/v1
kind: Deployment
metadata:
 labels:
   app: istio-ingressgateway-ssl
   istio: ingressgateway-ssl
   release: istio
 name: istio-ingressgateway-ssl
 namespace: istio-system
spec:
 selector:
   matchLabels:
     app: istio-ingressgateway-ssl
     istio: ingressgateway-ssl
 strategy:
   rollingUpdate:
     maxSurge: 100%
     maxUnavailable: 25%
 template:
   metadata:
     annotations:
       sidecar.istio.io/inject: "false"
     labels:
       app: istio-ingressgateway-ssl
       chart: gateways
       heritage: Tiller
       istio: ingressgateway-ssl
       release: istio
       service.istio.io/canonical-name: istio-ingressgateway-ssl
       service.istio.io/canonical-revision: "1.5"
   spec:
     affinity:
       nodeAffinity:
         preferredDuringSchedulingIgnoredDuringExecution:
         - preference:
             matchExpressions:
             - key: beta.kubernetes.io/arch
               operator: In
               values:
               - amd64
           weight: 2
         - preference:
             matchExpressions:
             - key: beta.kubernetes.io/arch
               operator: In
               values:
               - ppc64le
           weight: 2
         - preference:
             matchExpressions:
             - key: beta.kubernetes.io/arch
               operator: In
               values:
               - s390x
           weight: 2
         requiredDuringSchedulingIgnoredDuringExecution:
           nodeSelectorTerms:
           - matchExpressions:
             - key: beta.kubernetes.io/arch
               operator: In
               values:
               - amd64
               - ppc64le
               - s390x
     containers:
     - args:
       - proxy
       - router
       - --domain
       - $(POD_NAMESPACE).svc.cluster.local
       - --proxyLogLevel=warning
       - --proxyComponentLogLevel=misc:error
       - --log_output_level=default:info
       - --drainDuration
       - 45s
       - --parentShutdownDuration
       - 1m0s
       - --connectTimeout
       - 10s
       - --serviceCluster
       - istio-ingressgateway-ssl
       - --zipkinAddress
       - zipkin.istio-system:9411
       - --proxyAdminPort
       - "15000"
       - --statusPort
       - "15020"
       - --controlPlaneAuthPolicy
       - NONE
       - --discoveryAddress
       - istio-pilot.istio-system.svc:15012
       - --trust-domain=cluster.local
       env:
       - name: JWT_POLICY
         value: third-party-jwt
       - name: PILOT_CERT_PROVIDER
         value: istiod
       - name: ISTIO_META_USER_SDS
         value: "true"
       - name: CA_ADDR
         value: istio-pilot.istio-system.svc:15012
       - name: NODE_NAME
         valueFrom:
           fieldRef:
             apiVersion: v1
             fieldPath: spec.nodeName
       - name: POD_NAME
         valueFrom:
           fieldRef:
             apiVersion: v1
             fieldPath: metadata.name
       - name: POD_NAMESPACE
         valueFrom:
           fieldRef:
             apiVersion: v1
             fieldPath: metadata.namespace
       - name: INSTANCE_IP
         valueFrom:
           fieldRef:
             apiVersion: v1
             fieldPath: status.podIP
       - name: HOST_IP
         valueFrom:
           fieldRef:
             apiVersion: v1
             fieldPath: status.hostIP
       - name: SERVICE_ACCOUNT
         valueFrom:
           fieldRef:
             fieldPath: spec.serviceAccountName
       - name: ISTIO_META_WORKLOAD_NAME
         value: istio-ingressgateway-ssl
       - name: ISTIO_META_OWNER
         value: kubernetes://apis/apps/v1/namespaces/istio-system/deployments/istio-ingressgateway-ssl
       - name: ISTIO_META_MESH_ID
         value: cluster.local
       - name: ISTIO_AUTO_MTLS_ENABLED
         value: "true"
       - name: ISTIO_META_POD_NAME
         valueFrom:
           fieldRef:
             apiVersion: v1
             fieldPath: metadata.name
       - name: ISTIO_META_CONFIG_NAMESPACE
         valueFrom:
           fieldRef:
             fieldPath: metadata.namespace
       - name: ISTIO_META_ROUTER_MODE
         value: sni-dnat
       - name: ISTIO_META_CLUSTER_ID
         value: Kubernetes
       image: docker.io/istio/proxyv2:1.5.0
       imagePullPolicy: IfNotPresent
       name: istio-proxy
       ports:
       - containerPort: 15020
       - containerPort: 80
       - containerPort: 443
       - containerPort: 15029
       - containerPort: 15030
       - containerPort: 15031
       - containerPort: 15032
       - containerPort: 15443
       - containerPort: 15011
       - containerPort: 8060
       - containerPort: 853
       - containerPort: 15090
         name: http-envoy-prom
         protocol: TCP
       readinessProbe:
         failureThreshold: 30
         httpGet:
           path: /healthz/ready
           port: 15020
           scheme: HTTP
         initialDelaySeconds: 1
         periodSeconds: 2
         successThreshold: 1
         timeoutSeconds: 1
       resources:
         limits:
           cpu: 2000m
           memory: 1024Mi
         requests:
           cpu: 100m
           memory: 128Mi
       volumeMounts:
       - mountPath: /var/run/secrets/istio
         name: istiod-ca-cert
       - mountPath: /var/run/secrets/tokens
         name: istio-token
         readOnly: true
       - mountPath: /var/run/ingress_gateway
         name: ingressgatewaysdsudspath
       - mountPath: /etc/istio/pod
         name: podinfo
       - mountPath: /etc/istio/ingressgateway-certs
         name: ingressgateway-certs
         readOnly: true
       - mountPath: /etc/istio/ingressgateway-ca-certs
         name: ingressgateway-ca-certs
         readOnly: true
     serviceAccountName: istio-ingressgateway-ssl-service-account
     volumes:
     - configMap:
         name: istio-ca-root-cert
       name: istiod-ca-cert
     - downwardAPI:
         items:
         - fieldRef:
             fieldPath: metadata.labels
           path: labels
         - fieldRef:
             fieldPath: metadata.annotations
           path: annotations
       name: podinfo
     - emptyDir: {}
       name: ingressgatewaysdsudspath
     - name: istio-token
       projected:
         sources:
         - serviceAccountToken:
             audience: istio-ca
             expirationSeconds: 43200
             path: istio-token
     - name: ingressgateway-certs
       secret:
         optional: true
         secretName: istio-ingressgateway-ssl-certs
     - name: ingressgateway-ca-certs
       secret:
         optional: true
         secretName: istio-ingressgateway-ssl-ca-certs
---
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
 name: ingressgateway-ssl
 namespace: istio-system
 labels:
   app: istio-ingressgateway-ssl
   istio: ingressgateway-ssl  
   release: istio
spec:
 minAvailable: 1
 selector:
   matchLabels:
     app: istio-ingressgateway-ssl
     istio: ingressgateway-ssl    
     release: istio
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
 name: istio-ingressgateway-sds
 namespace: istio-system
 labels:
   release: istio
rules:
- apiGroups: [""]
 resources: ["secrets"]
 verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
 name: istio-ingressgateway-ssl-sds
 namespace: istio-system
 labels:
   release: istio
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: Role
 name: istio-ingressgateway-sds
subjects:
- kind: ServiceAccount
 name: istio-ingressgateway-ssl-service-account
---
apiVersion: v1
kind: Service
metadata:
 name: istio-ingressgateway-ssl
 namespace: istio-system
 labels:
   app: istio-ingressgateway-ssl
   istio: ingressgateway-ssl   
   release: istio
spec:
 type: LoadBalancer
 selector:
   app: istio-ingressgateway-ssl
   istio: ingressgateway-ssl
 ports:
   - name: status-port
     port: 15020
     targetPort: 15020
   - name: http2
     port: 80
     targetPort: 80
   - name: https
     port: 443
   - name: kiali
     port: 15029
     targetPort: 15029
   - name: prometheus
     port: 15030
     targetPort: 15030
   - name: grafana
     port: 15031
     targetPort: 15031
   - name: tracing
     port: 15032
     targetPort: 15032
   - name: tls
     port: 15443
     targetPort: 15443
---
apiVersion: v1
kind: ServiceAccount
metadata:
 name: istio-ingressgateway-ssl-service-account
 namespace: istio-system
 labels:
   app: istio-ingressgateway-ssl
   istio: ingressgateway-ssl
   release: istio
{% endhighlight %}

#### Déploiement

Le déploiement s’effectue avec kubectl dans le namespace `istio-system`:
{% highlight shell %}
kubectl apply -f manifest.yaml -n istio-system
{% endhighlight %}

### External-Dns

#### Fonctionnement

![image](/assets/images/kube-istio-externaldns-sert-manager-letsencrypt-part1/external_dns.png){:class="img-responsive"}

External-Dns est à l’écoute des déploiements d’objets Gateway. Attention, Il ne s’agit pas des Istio Ingress Gateway mais bien des custom resources Gateway qui sont déployées avec les applications `(1)`.
Si un objet gateway porte l’annotation attendue par External-Dns et que le nom de domaine de l’hôte configuré dans l’objet Gateway est dans la liste de ceux gérés par External-Dns `(2)`, alors External-Dns va chercher sur l’Istio Ingress Gateway concernée par l’objet Gateway le nom DNS du point d’entrée correspondant (ici un Load Balancer AWS) `(3)`  et va créer un enregistrement dans Route 53 `(4)`.

#### Configuration

Copier la configuration suivante dans un fichier nommé `external-dns-deployment.yaml` :

{% highlight yaml %}
apiVersion: v1
kind: ServiceAccount
metadata:
 name: external-dns-ssl
 namespace: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
 name: external-dns-ssl
 namespace: external-dns
rules:
 - apiGroups: [""]
   resources: ["services"]
   verbs: ["get","watch","list"]
 - apiGroups: [""]
   resources: ["pods"]
   verbs: ["get","watch","list"]
 - apiGroups: ["extensions"]
   resources: ["ingresses"]
   verbs: ["get","watch","list"]
 - apiGroups: [""]
   resources: ["nodes"]
   verbs: ["list"]
 - apiGroups: ["networking.istio.io"]
   resources: ["gateways"]
   verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
 name: external-dns-viewer-ssl
 namespace: external-dns
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: ClusterRole
 name: external-dns-ssl
subjects:
 - kind: ServiceAccount
   name: external-dns-ssl
   namespace: external-dns
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
 name: external-dns-ssl
 namespace: external-dns
spec:
 strategy:
   type: Recreate
 template:
   metadata:
     labels:
       app: external-dns-ssl
   spec:
     serviceAccountName: external-dns-ssl
     containers:
       - name: external-dns-ssl
         image: eu.gcr.io/k8s-artifacts-prod/external-dns/external-dns:v0.7.0
         args:
           - --source=istio-gateway
           - --annotation-filter=externaldns=ssl
           - --domain-filter=sample.com
           - --provider=aws
           - --policy=upsert-only
           - --aws-zone-type=public
           - --registry=txt
           - --txt-owner-id=ser-test
           - --log-level=debug
{% endhighlight %}

Les arguments du conteneur sont les suivants:
- `--source=istio-gateway` indique que ce sont les hôtes configurés sur des Gateway Istio qui seront traités. Ici, ceux configurés sur des Ingress ou Services Kubernetes seront ignorés.
- `--annotation-filter=externaldns=ssl` indique que seul les objets Gateway avec l’annotation externaldns: ssl seront traités.
- `--domain-filter=sample.com` indique que seuls les noms d’hôtes appartenant au domaine `sample.com` seront pris en compte.
- `--provider=aws` indique que nous utilisons Route 53
- `--policy=upsert-only` indique que l’enregistrement ne sera pas supprimé si on supprime l’objet Gateway correspondant (valeurs possibles : `sync` ou `upsert-only`).
- `--aws-zone-type=public` indique que nous n’utilisons que les zones Publique de Route 53 (les valeurs possibles sont: `public`, `private` ou vide pour les deux ).
- `--registry=txt` et `--txt-owner-id=ser-test` indique que pour chaque enregistrement DNS créé, un enregistrement TXT sera créé pour le documenter. Il contiendra entre autre, la valeur `owner-id=ser-test`. C’est utile quand il y a plusieurs instances de Cert-Manager qui créent des enregistrements dans un même DNS.
- `--log-level=debug` indique que je débute avec External-Dns :)

#### Déploiement

Le déploiement s’effectue avec kubectl dans le namespace `default`:

{% highlight shell %}
kubectl apply -f external-dns-deployment.yaml -n external-dns
{% endhighlight %}

Nous terminons ici la première partie. La suite traitera de Cert-Manager et du déploiement de l'application.