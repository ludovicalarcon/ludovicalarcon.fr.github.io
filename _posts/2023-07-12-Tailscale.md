---
layout: post
title: Accéder à votre cluster kubernetes depuis votre machine avec tailscale
img: k8s-logo.png
tags: [kubernetes, tailscale]
---

## Tailscale

Nous allons utiliser `tailscale` afin de créer un réseau VPN entre notre `control plane` et notre machine locale.  
Tout d'abord, nous allons installer `tailscale` sur les deux machines, le control plane et notre machine.  
Pour tout autre système d'exploitation qu'Ubuntu, vous pouvez vous référer à la [documentation officielle](https://tailscale.com/kb/installation/).
```sh
# Ajout des repos & des clés
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/focal.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
# Installation de tailscale
sudo apt-get update
sudo apt-get install tailscale
```
Nous devons maintenant nous authentifier afin de connecter notre machine au réseau tailscale.
```sh
sudo tailscale up
```
Depuis le console d'administrateur, il est possible de désactiver `l'expiration de la clé`.  
Cela permet de ne pas avoir à se reconnecter régulièrement pour se re-authentifier.

__L'installation de tailscale doit se faire sur les deux machines.__

## Connectivité

Tout d'abord, nous allons ouvrir notre `firewall` sur Orcale Cloud.  
Mais avant de commencer, nous devons récupérer l'adresse IP de notre machine dans le réseau tailscale.  
Exécuter la commande suivante:
```sh
tailscale ip -4
100.X.Y.Z
```
Nous allons ouvrir le port `6433` en TCP uniquement pour notre machine.  
Pour cela, sur le portail d'Oracle Cloud, nous allons aller dans Cloud Networks public-subnet  
#### ![](/assets/images/2023-07-04-subnets.png)
Puis dans Security Lists -> Default Security List
#### ![](/assets/images/2023-07-04-securityLists.png)
Et enfin sur Ingress Rules -> Add Ingress Rules
#### ![](/assets/images/2023-07-12-rules.png)

## KubeConfig

Nous allons devoir nous connecter en ssh sur le control plane afin d'y récupérer le fichier `KubeConfig` situé `.kube/config`.  
Nous devons également récupérer l'adresse IP du control plane dans le réseau tailscale.   
```sh
tailscale ip -4
100.X.Y.Z
```
Nous pouvons maintenant créer le fichier de configuration pour nos cluster sur notre machine locale.  
Nous allons aussi remplacer la line `server: https://10.0.0.X:6443` par l'adresse IP que nous avons obtenue précédemment.  
```sh
# Sur la machine locale
mkdir -p ~/.kube
# Coller le contenue du fichier obtenu sur le control plane et remplacer l'IP
vi ~/.kube/oracle-config
# Utilisation de ce kube config
export KUBECONFIG=~/.kube/oracle-config
```
Si nous essayons d'interagir avec notre cluster, nous obtenons une erreur de certificat.
```
Unable to connect to the server: tls: failed to verify certificate: x509: certificate is valid for 10.0.0.W, not 100.X.Y.Z
```

## Ajout de SAN

Nous devons ajouter notre adresse IP pour le certificat.  
Pour cela, nous allons d'abord supprimer les certificats de l'API server existant afin de pouvoir par la suite les régénérer.
```sh
sudo rm /etc/kubernetes/pki/apiserver.{crt,key}
```
Ensuite, nous devons récupérer le fichier de configuration kubeadm de notre cluster.
```sh
kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm.yaml
```
Nous devons maintenant le modifier pour y ajouter la configuration SAN.  
Remplacer par vos adresses IPs.
```yaml
apiServer:
certSANs:
- "PRIVATE_IP_ADDR_OF_CONTROLPLANE"
- "kubernetes.default"
- "TAILSCALE_IP_ADDR_OF_CONTROLPLANE"
extraArgs:
authorization-mode: Node,RBAC
timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
...
...
...
```
Pour finir nous allons régénérer les certificats de l'API server avec notre nouvelle configuration.
```sh
sudo kubeadm init phase certs apiserver --config kubeadm.yaml
```

## Conclusion

Nous sommes maintenant en mesure d'accéder a notre cluster kubernetes depuis notre machine locale grâce à tailscale.  

J'espère que cet article vous a été utile !