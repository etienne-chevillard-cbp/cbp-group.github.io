---
layout: post
title:  "Ajouter un peu de résilience sur vos systèmes"
author: ebi
categories: [ Java, Developers, Resilience ]
tags: [Java, resilience4j, Circuit Breaker, Aspectj, Spring, Kibana, JMX]
image: assets/images/resilience/resilience.jpg
description: "Ajouter un peu de résilience sur vos systèmes"
featured: false
hidden: false
comments: false
---

Au risque de décevoir certaines personnes, nous n'allons pas parler ici du travail du neuropshychiatre [Boris Cyrulnik](https://fr.wikipedia.org/wiki/Boris_Cyrulnik){:target="_blank"} ni de son
travail sur l'aspect psychologique de la résilience. Le monde de l'informatique n'a pas fait exception aux autres domaines et lui aussi
s'est accaparé cette notion plutôt dans l'air du temps mais pas si nouvelle que ça...

## La résilience dans le SI

Dans un système applicatif, nous définirons la résilience comme la tolérance du système aux pannes. Nous pouvons préciser cette notion en ajoutant que le système soumis aux pannes doit pouvoir retrouver
le même fonctionnement qu'avant la perturbation. Cela reste très abstrait alors rentrons dans le détail avec du concret.

### Le Contexte

La problématique est la suivante : 

Imaginons un système publisher/subscriber qui gère les appels à un service tiers de manière asynchrone. Imaginons que les subscribers qui effectuent les appels aux services
externes soient très sollicités. La granularité de la gestion d'erreurs de ce système de topic permet de différencier les erreurs fonctionnelles des erreurs purement techniques.
En cas d'erreur fonctionnelles, les messages sont mis de côté pour être analysés. Par contre, en cas d'erreur purement technique, les messages sont "rejoués".

La conséquence de ce rejeu, c'est la très forte sollicitation du service externe, spécialement lorsque celui-ci est "en panne".

En cas de panne technique, la mise en place du système pub/sub est déjà un élément très important car il permet de "rejouer" les messages en erreur.
Notre problématique ici c'est principalement d'éviter d'effectuer un grand nombre d'appels à un service en peine.

### Resilience4j

Resilience4j est une bibliothèque de tolérance aux pannes, légère et facile à utiliser, inspirée de Netflix Hystrix. 
Elle fournit des fonctions d'ordre supérieur (décorateurs) pour améliorer toute interface fonctionnelle, expression lambda ou référence de méthode 
avec différents mécanismes s'inscrivant dans un système résilient :
- un disjoncteur
- un limiteur de débit
- une nouvelle tentative
- une cloison

Avec la possibilité d'empiler plusieurs décorateurs ou non.

### Le disjoncteur (ou CircuitBreaker)

Pour éviter que notre service sous-jacent "en panne" soit appelé et surcharge inutilement le traffic, nous allons mettre en place un disjoncteur.
Pour ceux familiarisés avec leurs installations électriques domestiques, l'image du disjoncteur est assez parlante. En cas de problème sur le réseau électrique,
le disjoncteur permet de couper le circuit et protéger l'installation/les hommes de tout dommage.

Dans notre contexte, le disjoncteur est implémenté via une machine à états finis avec trois états normaux : CLOSED, OPEN et HALF_OPEN et deux états spéciaux DISABLED et FORCED_OPEN.
Il est un peu plus sophistiqué qu'un simple disjoncteur électrique.

![image](/assets/images/resilience/state_machine.jpg){:class="img-responsive"}

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
fonctionnel avec du code "technique". Pour faire cela, on peut s'appuyer sur le paradygme AOP. La programmation orientée aspect va nous permettre
de mettre à part nos "préoccupations" techniques liées à la mise en place du CircuitBeaker.

Pour ce faire, on va simplement annoter notre méthode pour indiquer qu'elle bénéficie de l'outillage "coupe-circuit"

```java
    @DmsCircuitBreakerFunction(value = "CbpCircuitBreaker")
```

Par ailleurs, on définit un composant aspect avec notre annotation `@Around` qui sert de "proxy" à la méthode cible qui est censée faire nos appels externes.
Ce système de proxy utilise un supplier que l'on fournit à l'implémentation du CircuitBeaker qui se chargera de faire l'appel à notre méthode finale.

```java
    @Around("@annotation(CbpCircuitBreaker)")
    public Object doConcurrentOperation(ProceedingJoinPoint pjp) {
        // Ici on récupère le nom du circuit breaker issu du paramètre de l'annotation
        String cbName = ((MethodSignature) pjp.getSignature()).getMethod().getAnnotation(DmsCircuitBreakerFunction.class).value();
        Supplier<Object> supplier = () -> {
            try {
                return pjp.proceed();
            }
            catch (TechnicalException|FunctionalException e) {
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
Plus les appels échoueront, plus le circuit restera "ouvert" longtemps (configurable comme mentionné plus haut bien sûr)

### A propos des multi instances

Je vois déjà arriver le flot de questions concernant le Cloud Cbp ? 
Tout d'abord, le système publisher/subscriber fonctionne sur plusieurs instances Ec2 AWS grace à une consolidation BDD.
Mais comment le CicuitBreaker peut-il fonctionner à travers ces multiples instances ?
La réponse est simple, ça reste cloisonné par instance Ec2 et c'est pas grave !

Le mécanisme de CicuitBreaker est géré par la JVM, et à ce titre, les métriques ne sont valables que par instances. On peut donc
imaginer une machine Ec2 qui, à un instant donné, arrête de faire des appels au service sous-jacent en "ouvrant" son circuit alors 
qu'une autre machine continue un certain temps de faire les appels en échec jusqu'à ce qu'il "s'ouvre" naturellement lui aussi.

### Un suivi avec les bean JMX et l'intégration à Kibana

Pour terminer, parlons de la possibilité de sauvegarder l'ensemble des métriques du CircuitBreaker dans le bean JMX.
La finalité, c'est de pouvoir transmettre ces métriques à Kibana pour grapher ces informations. Et pourquoi pas cabler de l'alerting
pour être prévenu en cas d'ouverture de nos circuits !
Pour cela, on utilise la librairie io.micrometer

```
    MeterRegistry meterRegistry = new JmxMeterRegistry(s -> null, Clock.SYSTEM);
    TaggedCircuitBreakerMetrics.ofCircuitBreakerRegistry(circuitBreakerRegistry).bindTo(meterRegistry);
```

Ensuite, nous ajoutons, dans notre cas, la métrique failure rate qui nous intéresse dans le Bean JMX (`JMXLogger`)

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

Quelques liens utiles

Le projet [resilience4j](https://resilience4j.readme.io/docs/getting-started){:target="_blank"} 

Le projet [micrometer](https://micrometer.io/){:target="_blank"} 

L'article wikipedia sur les [publisher/subscripber](https://fr.wikipedia.org/wiki/Publish-subscribe){:target="_blank"} 
