---
layout: post
title:  "Ajouter un peu de résilience sur vos systèmes"
author: ebi
categories: [ Java, Developers, Resilience ]
tags: [Java, resilience4j, Circuit Breaker, Aspectj, Spring, Kibana, JMX]
image: assets/images/resilience/resilience.jpg
description: "Ajouter un peu de résilience sur vos systèmes"
featured: true
hidden: false
comments: false
---

Au risque de décevoir certaines personnes, nous n'allons pas parler ici du travail du neuropsychiatre [Boris Cyrulnik](https://fr.wikipedia.org/wiki/Boris_Cyrulnik){:target="_blank"} ni de son
travail sur l'aspect psychologique de la résilience. Le monde de l'informatique n'a pas fait exception aux autres domaines et lui aussi
s'est accaparé cette notion plutôt dans l'air du temps mais pas si nouvelle que ça...

## La résilience dans le SI

Dans un système applicatif, nous définirons la résilience comme la tolérance du système aux pannes. 
Nous pouvons préciser cette definition en y en ajoutant la notion d'auto-remédiation, en ajoutant que le système soumis aux pannes doit 
pouvoir retrouver le même fonctionnement qu'avant la perturbation, et enfin que notre système doit éviter de provoquer des nuisances collatérales potentielles. 
Cela reste très abstrait alors rentrons dans le détail avec du concret.

### Le Contexte

La problématique est la suivante : 

Imaginons un système publisher/subscriber<sup>[1](#pubsub)</sup> qui gère les appels à un service tiers de manière asynchrone. Imaginons que les subscribers qui effectuent les appels aux services
externes soient très sollicités. La granularité de la gestion d'erreurs de ce système de topic permet de différencier les erreurs fonctionnelles des erreurs purement techniques.
En cas d'erreur fonctionnelle, les messages sont mis de côté pour être analysés. Par contre, en cas d'erreur purement technique, les messages sont "rejoués".

La conséquence de ce rejeu, c'est la très forte sollicitation du service externe, spécialement lorsque celui-ci est "en panne".

En cas de panne technique, la mise en place du système pub/sub est déjà un élément très important car il permet de "rejouer" les messages en erreur.
Notre problématique ici c'est principalement d'éviter d'effectuer un grand nombre d'appels à un service en panne.

### Resilience4j

Resilience4j<sup>[2](#resilience4j)</sup> est une bibliothèque de tolérance aux pannes, légère et facile à utiliser, inspirée de Netflix Hystrix<sup>[3](#hystrix)</sup>. 
Elle fournit des fonctions d'ordre supérieur (décorateurs) pour améliorer toute interface fonctionnelle, expression lambda ou référence de méthode 
avec différents mécanismes s'inscrivant dans un système résilient :
- un disjoncteur ([CircuitBreaker](https://resilience4j.readme.io/docs/circuitbreaker){:target="_blank"})
- un limiteur de débit ([Rate limiter](https://resilience4j.readme.io/docs/ratelimiter){:target="_blank"})
- une nouvelle tentative ([Automatic retrying](https://resilience4j.readme.io/docs/retry){:target="_blank"})
- une cloison ([Bulkhead](https://resilience4j.readme.io/docs/bulkhead){:target="_blank"})

Avec la possibilité d'empiler plusieurs décorateurs ou non.

### Le disjoncteur (ou CircuitBreaker)

Pour éviter que notre service sous-jacent "en panne" soit appelé et que l'on surcharge inutilement le trafic, nous allons mettre en place un disjoncteur.
Pour ceux familiarisés avec leurs installations électriques domestiques, l'image du disjoncteur est assez parlante. En cas de problème sur le réseau électrique,
le disjoncteur permet de couper le circuit et protéger l'installation/les hommes de tout dommage.

Dans notre contexte, le disjoncteur est implémenté via une machine à états finis avec trois états normaux : CLOSED, OPEN et HALF_OPEN 
et deux états spéciaux DISABLED et FORCED_OPEN.
Il est un peu plus sophistiqué qu'un simple disjoncteur électrique.

![image](/assets/images/resilience/state_machine.jpg){:class="img-responsive"}

- CLOSED : Mon circuit fonctionne normalement et mes appels externes fonctionnent
- OPEN : Mon circuit est coupé, dysfonctionnement constaté et appels externes non faits
- HALF_OPEN : Après une temporisation, le circuit OPEN passe en HALF_OPEN pour tester si un problème persiste
- DISABLED : Circuit désactivé, pas de coupure, les appels externes sont systématiquement faits
- FORCED_OPEN : Ouverture forcée du circuit, aucun appel externe

Il stocke et agrège les résultats des appels sur une période de temps coulissant avec la possibilité de choisir entre une période glissante basée sur le nombre des N derniers appels 
ou une période glissante temporelle qui agrège le résultat des appels des N dernières secondes.

#### La configuration

Le CircuitBreaker est configurable sur beaucoup de paramètres
(seuil de déclenchement, nombres d'appels minimums avant ouverture, durée avant ouverture, temps de la fenêtre glissante etc.)
Sans rentrer dans les détails, l'implémentation en Java pourrait ressembler à ça :

```java
public CircuitBreakerConfig createConfiguration() {
        return CircuitBreakerConfig.custom()
                                   .failureRateThreshold(failureRateThreshold)
                                   .slowCallRateThreshold(slowCallRateThreshold)
                                   .waitDurationInOpenState(Duration.ofMillis(waitDurationInOpenState))
                                   .slowCallDurationThreshold(Duration.ofSeconds(slowCallDurationThreshold))
                                   .permittedNumberOfCallsInHalfOpenState(permittedNumberOfCallsInHalfOpenState)
                                   .minimumNumberOfCalls(minimumNumberOfCalls)
                                   .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.TIME_BASED)
                                   .slidingWindowSize(slidingWindowSize)
                                   /*
                                    *  Circuit breaker must not open on functional, retry is useless.
                                    */
                                   .ignoreExceptions(FunctionalException.class)
                                   .build();
    }
```

### Implémentation avec AspectJ

La mise en place d'un circuit breaker s'inscrit dans une logique technique. A ce titre, il serait intéressant d'éviter de "polluer" le code
fonctionnel avec du code "technique". Pour faire cela, on peut s'appuyer sur le paradigme AOP<sup>[5](#aop)</sup>. La programmation orientée aspect va nous permettre
de mettre à part nos "préoccupations" techniques liées à la mise en place du CircuitBeaker. 
La mise en place des concepts AOP se font par l'intermédiaire d'AspectJ<sup>[6](#aspectj)</sup>, une extension AOP pour Java.

Pour ce faire, on va simplement annoter notre méthode pour indiquer qu'elle bénéficie de l'outillage "coupe-circuit"

```java
    @CircuitBreakerFunction(value = "CbpCircuitBreaker")
```

Par ailleurs, on définit un composant aspect avec notre annotation `@Around`<sup>[7](#aroundaop)</sup> qui sert de "proxy" à la méthode cible qui est censée faire nos appels externes.
Ce système de proxy utilise un supplier que l'on fournit à l'implémentation du CircuitBeaker qui se chargera de faire l'appel à notre méthode finale.

```java
    @Around("@annotation(CircuitBreakerFunction)")
    public Object doConcurrentOperation(ProceedingJoinPoint pjp) {
        // Ici on récupère le nom du circuit breaker issu du paramètre de l'annotation
        String cbName = ((MethodSignature) pjp.getSignature()).getMethod().getAnnotation(CircuitBreakerFunction.class).value();
        Supplier<Object> supplier = () -> {
            try {
                return pjp.proceed();
            }
            catch (FunctionalException e) {
                throw e;
            }
            catch (Throwable throwable) {
                throw new CircuitBreakerException(throwable);
            }
        };
        // Ici on on execute la methode a travers le mecanisme de circuit breaker
        return getCircuitBreaker(cbName).executeSupplier(supplier);
    }
```


Et voilà.
À partir d'ici, si notre service sous-jacent n'est pas disponible pour des raisons techniques, notre circuit "s'ouvre" et
les appels ne sont pas faits. Selon le paramétrage, le CircuitBreaker fera ponctuellement des appels pour savoir s'il doit à nouveau fermer son circuit.
Plus les appels échoueront, plus le circuit restera "ouvert" longtemps (configurable comme mentionné plus haut bien sûr).

### A propos des multi instances

Mais comment notre CicuitBreaker peut-il fonctionner à travers de multiples instances déployées sur le Cloud ?
Le système publisher/subscriber fonctionne sur plusieurs instances Ec2 AWS grace à une consolidation BDD. Les différentes instances ne doivent pas
consommer plusieurs fois le même message. Pour ce qui est du fonctionnement du CicrcuitBreaker multi instances, la réponse est simple, 
ça reste cloisonné par instance Ec2 et c'est pas grave !

Le mécanisme de CicuitBreaker est géré par la JVM, et à ce titre, les métriques ne sont valables que par instance. On peut donc
imaginer une machine Ec2 qui, à un instant donné, arrête de faire des appels au service sous-jacent en "ouvrant" son circuit alors 
qu'une autre machine continue un certain temps de faire les appels en échec jusqu'à ce qu'il "s'ouvre" naturellement lui aussi.

### Un suivi avec les bean JMX et l'intégration à Kibana

Pour terminer, parlons de la possibilité de sauvegarder l'ensemble des métriques du CircuitBreaker dans le bean JMX.
La finalité, c'est de pouvoir transmettre ces métriques à Kibana pour grapher ces informations. Et pourquoi pas cabler de l'alerting
pour être prévenu en cas d'ouverture de nos circuits !
Pour cela, on utilise la librairie io.micrometer<sup>[4](#micrometer)</sup>

```
    MeterRegistry meterRegistry = new JmxMeterRegistry(s -> null, Clock.SYSTEM);
    TaggedCircuitBreakerMetrics.ofCircuitBreakerRegistry(circuitBreakerRegistry).bindTo(meterRegistry);
```

Ensuite, nous pouvons ajouter la métrique failure rate qui nous intéresse dans le Bean JMX à l'aide 
d'une classe custom `JMXLogger`. Cette classe nous permet de valuer nos log Marker en fonction des attributs JMX désirés.

```
                <bean class="com.cbp.csp.dms.monitoring.MonitoringPoint">
                    <property name="objectName" value="metrics:name=resilience4jCircuitbreakerFailureRate.name.ReferentialSocleCircuitBreaker" />
                    <property name="alias" value="circuitbreaker.referentialsocle.failure.rate" />
                    <property name="attributes">
                        <list>
                            <value>Value</value>
                        </list>
                    </property>
                </bean>
```
                
Et voilà, il ne reste plus qu'à exploiter notre index `circuitbreaker.referentialsocle.failure.rate` dans kibana pour le grapher et/ou le coupler
à l'alerting.

#### References externes

<a name="pubsub">[1]</a> L'article wikipedia sur les [publisher/subscripber](https://fr.wikipedia.org/wiki/Publish-subscribe){:target="_blank"} 

<a name="resilience4j">[2]</a> Le projet [resilience4j](https://resilience4j.readme.io/docs/getting-started){:target="_blank"} 

<a name="hystrix">[3]</a> Le projet [Hystrix](https://github.com/Netflix/Hystrix){:target="_blank"} 

<a name="micrometer">[4]</a> Le projet [micrometer](https://micrometer.io/){:target="_blank"} 

<a name="aop">[5]</a> Concepts de la Programmation Orientée Aspect [AOP](https://fr.wikipedia.org/wiki/Programmation_orient%C3%A9e_aspect){:target="_blank"} 

<a name="aspectj">[6]</a> Java [AspectJ](https://fr.wikipedia.org/wiki/AspectJ){:target="_blank"} 

<a name="aroundaop">[7]</a> Around advice [SpringAOP](https://docs.spring.io/spring-framework/docs/4.3.15.RELEASE/spring-framework-reference/html/aop.html#aop-ataspectj-around-advice){:target="_blank"} 

Un article intéressant de Martin Fowler sur le [CicuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html){:target="_blank"} 
