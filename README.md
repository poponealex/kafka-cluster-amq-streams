# Introduction

L'objectif de ce projet est de déployer un cluster Kafka dans un cluster OpenShift via l'opérateur AMQ Streams et les outils/composants sous-jacents
- un cluster Kafka, composé de trois nœuds ZooKeeper et de trois brokers, avec un mode d'authentification SCRAM-SHA-512 et des certificats SSL
- des utilisateurs et des topics 
- une interface graphique permettant de visualiser le cluster en mode authentifié
- un connecteur Debezium pour détecter les changements effectués dans la base de données, 
- un registre de schéma pour valider le contenu des évènements générés et reçus
- des producteurs et des des consommateurs utilisant le schéma de registre ou non écrits en java /spring boot ou en quarkus
- le tout déployé via ArgoCD


NB: ce projet n'est pas compatible avec d'autres distribution Kubernetes vanilla ou Rancher etc... car il utilise par exemple les routes OpenShift 

on suppose par ailleurs que l'opérateur "Strimzi" ou "AMQ Streams", et "Apicurio Registry Operator" sont déjà installés

# Détails
## Flux 
Le flux du jeu d'essai consiste à déployer le cluster Kafka et les objets nécessaires à sa configuration, tel que les utilisateurs et les topics puis de déployer des producteurs et des consommateurs, de sorte que: 

- un producteur écrit avec Spring Boot produira des évènements du type "personne" qui ne seront pas validés par le registre de schéma, 
- un consommateur avec Spring Boot consommera à ces évènements du type "personne" pour les insérer dans une base de données

- un producteur écrit avec Spring Boot produira des évènements du type "voiture" qui seront validés par le registre de schéma, 
- un consommateur écrit avec Spring Boot consommera à ces évènements du type "voiture" pour les insérer dans une base de données

- un producteur écrit avec Quarkus produira des évènements du type "livre" qui ne seront pas validés par le registre de schéma, 
- un consommateur avec Quarkus consommera à ces évènements du type "livre" pour les insérer dans une base de données

- un producteur écrit avec Quarkus produira des évènements du type "film" qui seront  validés par le registre de schéma, 
- un consommateur écrit avec Quarkus consommera à ces évènements du type "film" pour les insérer dans une base de données

## Ordonnancement
L'ordonnancement se fait ainsi 
1. le producteur produit un évènement dans un topic dédié, 
2. l'évènement est consommé par le consommateur dédié, 
3. le consommateur insère cet évènement dans une base de données MySQL, 
4. L'insertion dans la base de données est détectée par le connecteur Debezium 
5. Debezium génère également un évènement au moment de la détection du changement de données dans la base de données (change data capture)


## Production d'évènements
Les producteurs exposent un endpoint "/send" avec un paramètre non obligatoire "count" qui envoie automatiquement 1 évènement de type "voiture" ou "personne" générés automatiquement et avec des valeurs aléatoires, si le paramètre count n'est pas spécifié ou 'count' évènements. 
Pour générer les évènements facilement:

```
for pod in $(oc get pod -o name | grep producer); do oc exec -it $pod -- curl localhost:8080/send?count=1 ; done 
```

## Code et liens

Le code des consommateurs et des producteur se trouvent également disponible à ces adresses:
### Spring Boot, sans registre
- pour le producteur : https://github.com/olivierits4u/kafka-producer-springboot
- pour le consommateur: https://github.com/olivierits4u/kafka-consumer-springboot


### Spring Boot, avec registre
- pour le producteur : https://github.com/olivierits4u/kafka-producer-springboot-registry
- pour le consommateur: https://github.com/olivierits4u/kafka-consumer-springboot-registry

### Quarkus, sans registre
- pour le producteur : https://github.com/olivierits4u/kafka-producer-quarkus
- pour le consommateur: https://github.com/olivierits4u/kafka-consumer-quarkus

### Quarkus, avec registre
- pour le producteur : https://github.com/olivierits4u/kafka-producer-quarkus-registry
- pour le consommateur: https://github.com/olivierits4u/kafka-consumer-quarkus-registry
-registry
 


Le code ArgoCD est disponible ici: https://github.com/olivierits4u/kafka-argocd


NB: il est utile de créer un secret nommé regcred qui permet d'accéder au registre docker qui héberge les images


pour cela: 

```
kubectl create secret generic regcred  --from-file=.dockerconfigjson=</.docker/config.json> --type=kubernetes.io/dockerconfigjson
```
