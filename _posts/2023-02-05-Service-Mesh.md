---
layout: post
title: "Un Service Mesh c'est quoi ?"
tags: ["Service Mesh", "101"]
---

Un `Service Mesh` ou `maillage de services` est une couche d'infrastructure dédiée qui ajoute de manière totalement transparente des fonctionnalités comme l'observabilité, le management du trafic réseaux ou encore des briques de sécurité sans que nous ayons besoin de l'ajouter a notre propre code.  
Il s'occupe de router toutes les communications inter-service au travers d'un proxy afin de pouvoir gérer les communications réseaux entre micro-services.  

J'espère que cette suite de billets sera utile pour votre voyage au pays des services mesh.

- __Un Service Mesh c'est quoi ?__
- Istio (A venir)

## Ce qu'apporte le service mesh

- Load balancing (répartition de charge)
- Observabilité
- Service discovery au travers des différents environements
- Encryption mTLS
- Retry automatique des requêtes réseaux
- Granularité dans le routing du trafic pour les protocoles HTTP et gRPC
- RBAC (Droits basés sur des rôles)
- Authentification & Autorisation

## Architecture

### ![](/assets/images/2023-02-05-servicemesh.png)
[Source: nginx service mesh architecture](https://www.nginx.com/wp-content/uploads/2019/02/service-mesh-generic-topology.png)

Un service mesh est composé de 2 composants clés:
- Le control plane qui contient toute la configuration
- Le data plane qui pour chaque service contient une instance (LB, routing, service discovery, encryption, auth, etc..) et une instance du proxy dite sidecar

## Inconvénients

- Augmentation du nombre de pods et de l'utilisation des ressources du cluster
- Tous les appels à un service doivent d'abord passer par le proxy
- Les services mesh ne s'intégrent pas avec les autres systèmes et services
- Une montés en compétences sur les services mesh est nécessaires
- Ajout de complexité dans l'architecture réseaux

## Les service mesh ne sont pas forcément nécessaire

Vous n'avez pas besoin de service mesh si:
- Vous n'avez pas une architecture en microservice
- Vous n'avez pas besoin de toutes les fonctionnalités offertes par un service mesh  
Par exemple, si vous voulez uniquement l'observabilité, vous pouvez installer uniquement les outils nécessaires et ainsi éviter l'ajout de complexité.
- Vous n'avez ni le temps ni l'équipe pour monter en compétences sur le sujet

## Le marché du service mesh

- [Linkerd](https://linkerd.io/docs/)
- [Istio](https://istio.io/latest/docs/)
- [Consul Connect](https://developer.hashicorp.com/consul/docs/connect)
- [Open Service Mesh (OSM)](https://docs.openservicemesh.io/)
- [Gloo Mesh](https://docs.solo.io/gloo-mesh-enterprise/latest/)
- etc...