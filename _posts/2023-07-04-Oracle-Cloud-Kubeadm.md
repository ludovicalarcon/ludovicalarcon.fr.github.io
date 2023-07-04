---
layout: post
title: Oracle Cloud + Kubeadm
img: kubeadm.png
tags: [kubernetes, containerd, cloud]
---

## Introduction

Oracle cloud a une offre gratuite plutôt sympa, avec du compute ARM.
> Des cœurs Ampere A1 basés sur Arm 4 OCPUs et 24 Go de mémoire utilisables comme 1 VM ou jusqu'à 4 VM avec 3 000 heures d'OCPU et 18 000 Go-heures par mois.

La série de [zwindler](https://blog.zwindler.fr/2023/04/24/cluster-kubernetes-gratuit-part1/) sur comment avoir un cluster kubernetes gratuit avec k3s sur Oracle Cloud m'a inspiré pour cet article.

Vu que les VMs qu'on peut avoir sont vraiment pas mal (surtout pour du gratuit), je voulais déployer un cluster kubernetes complet avec `Kubeadm` et non pas la version légère qu'est k3s.

Je suis donc partie sur `Kubeadm` pour setup le cluster, `Containerd` pour le container runtime et `Flannel` pour la partie network.
Notre cluster sera donc composé comme suit:
- controlplane: 2 OCPUs et 12 GB de RAM
- worker-01: 1 OCPU et 6 GB de RAM
- worker-02: 1 OCPU et 6 GB de RAM

#### ![](/assets/images/2023-07-04-instances.png)

À vrai dire, à la base, je ne voulais pas utiliser `Flannel`, mais `Cilium`. 
Sauf que j'ai eu des soucis network sur les workers. 
Je suis ensuite passé a `Calico`, mais pareil... 
Donc je suis partie pour le moment sur `Flannel`, mais je n'abandonne pas mon idée principale d'avoir `Cilium`.

## Création des ressources sur Oracle Cloud

Cet article n'a pas vocation à être un guide complet sur Oracle Cloud, je vais donc monter rapidement les ressources à créer.

Tout d'abord, on va créer un `Virtual Cloud Networks VCN` (Réseau Cloud Virtuel).
> Un réseau cloud virtuel est un réseau privé virtuel que vous configurez dans des centres de données Oracle. Il ressemble grandement à un réseau traditionnel : il comporte des règles de pare-feu et des types de passerelle de communication spécifiques que vous pouvez choisir d'utiliser.

Il faut aller dans Networking -> Virtual Cloud Networks -> Start VCN Wizard 
Il suffit d'ajouter un nom et ensuite de garder les valeurs par défaut.
#### ![](/assets/images/2023-07-04-vcn.png)

Maintenant, il nous faut créer nos VMs instances.
Ça se passe dans Compute -> Instances -> Create Instance

- Donner un nom à l'instance
- Modifier la section correspondant à l'image et la section "shape"
- image: Ubuntu 22.04
- shape: Ampere
- 2 OCPUs & 12 GB RAM pour le control plane
- 1 OCPU & 6 GB RAM pour chaque worker
- Modifier la partie réseau
- Ça va automatiquement remplir les valeurs avec notre VCN
- Générer (ou upload) des clés SSH
- Sans cette étape, nous ne serons pas capables de nous connecter en SSH à nos instances.

#### ![](/assets/images/2023-07-04-configInstance.png)

Il nous reste encore une chose à faire avant de commencer à setup notre cluster. 
Nous allons ouvrir les connectivités sur le Firewall. 
En gros par défaut nous avons uniquement le port 22 pour les connections SSH d'ouvert. 
Comme notre cluster est un cluster de test et pas de production, nous allons simplement ouvrir tous les ports en TCP et UDP pour les IPs internes à notre cluster (10.0.0.0/16).
Dans un contexte de production, je ne recommande pas de suivre cette approche mais plutôt d'ouvrir uniquement les [ports requis](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) 
Nous allons également ouvrir le port 80 pour tout les hosts (0.0.0.0/0) afin de pouvoir exposer notre trafic http.

Pour cela, nous allons aller dans Networking -> Virtual Cloud Networks -> your VCN -> Subnets -> public-subnet
#### ![](/assets/images/2023-07-04-subnets.png)
TPuis dans Security Lists -> Default Security List
#### ![](/assets/images/2023-07-04-securityLists.png)
Et enfin sur Ingress Rules -> Add Ingress Rules
#### ![](/assets/images/2023-07-04-ingressrules.png)

## Setup des instances

### Configuration commune

Cette configuration doit être exécuté sur __tout les nodes__ (controlplane, worker-01, worker-02). 
Pour commencer, nous allons nous connecter en SSH sur les machines.
```sh
ssh -i PATH_TO_YOUR_SSH_KEY ubuntu@PUBLIC_IP_OF_VM
```

#### iptables

J'ai été agréablement surpris par la configuration par défaut d'iptables fournit par Oracle. 
Elle est super stricte et drop quasiment tout. 
Leurs règles de sécurité sont plutôt bien, je trouve. 
Tout d'abord, nous allons éditer la configuration d'iptables afin d'ouvrir les ports dont nous avons besoin. 
```sh
sudo vi /etc/iptables/rules.v4
```
Nous allons ajouter les règles ci-dessous juste après la règle pour SSH port 22.
`-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT`
```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 6443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2379 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2380 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10250 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10251 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10252 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --match multiport --dports 30000:32767 -j ACCEPT
```
Et nous allons supprimer ces deux lignes.
```
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
```
Il nous faut maintenant exécuter la commande ci-dessous afin de charger notre nouvelle configuration d'iptables.
```sh
sudo iptables-restore < /etc/iptables/rules.v4
```

#### Containerd

Maintenant, que nous avons ouvert tous les ports dont nous avons besoin, nous allons installer et configurer `Containerd`.

La `Swap` doit être désactivé afin que Kubelet fonctionne correctement.
Depuis la version 1.8 de Kubernetes, le flag de kubelet `fail-on-swap` a été mis par défaut à `true`, ce qui veut dire que part défaut la swap n'est pas supportée.
Plus d'informations disponibles dans cet [article](https://leizhilong.github.io/post/why-swap-should-be-disabled-on-kubernetes/)

```sh
# désactiver la swap
sudo sed -i "/ swap / s/^/#/" /etc/fstab
sudo swapoff -a
```

Nous avons besoin également d'autoriser iptables à voir les bridged traffic
```sh
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure IPTables pour bridged traffic
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s-cri-containerd.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Appliquer les changements sans reboot
sudo sysctl --system
```

Nous allons tout d'abord installer `Containerd en tant que service`
```sh
# Installation de containerd
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.2/containerd-1.7.2-linux-arm64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.2-linux-arm64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo cp containerd.service /lib/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
# Installation de runc
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.7/runc.arm64
sudo install -m 755 runc.arm64 /usr/local/sbin/runc
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-arm64-v1.3.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-arm64-v1.3.0.tgz
```
Nous devons maintenant générer la configuration par défaut de containerd et activer les `Cgroup`
```sh
# containerd configuration
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Activation de Cgroup
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

#### Kubeadm, Kubelet and Kubectl

Ensuite, nous allons installer `Kubeadm`, `Kubelet` et `Kubectl` dans leurs versions `1.27.3`.
```sh
# Ajout des sources for kubectl, kubeadm and kubelet
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# Installation des outils
KUBERNETES_VERSION="1.27.3-00"
sudo apt-get install -y kubelet=$KUBERNETES_VERSION \
                        kubeadm=$KUBERNETES_VERSION \
                        kubectl=$KUBERNETES_VERSION
sudo apt-mark hold kubelet=$KUBERNETES_VERSION kubeadm=$KUBERNETES_VERSION kubectl=$KUBERNETES_VERSION
```

## Control plane

Nous allons exécuter la commande `kubeadm init` sur le controlplane.
Nous utiliserons le CIDR suivant `10.244.0.0/16` pour la partie network des pods qui est nécessaire pour Flannel.
```sh
CONTROL_PLANE_IP="10.0.0.XX" # IP privée du control plane
POD_CIDR="10.244.0.0/16" # Flannel défaut CICD
sudo kubeadm init --pod-network-cidr=$POD_CIDR --apiserver-advertise-address=$CONTROL_PLANE_IP --apiserver-cert-extra-sans=$CONTROL_PLANE_IP --node-name=$(hostname -s)
```

Mettons de côté la commande `kubeadm join` fournit par l'output de la commande `kubeadm init`. 
Nous allons l'utiliser juste après.

## Worker node

Maintenant nous allons exécuter la commande `kubeadm join` que nous avons récupérée juste avant sur nos worker node.
```sh
kubeadm join ...
```

## Control plane suite

Tout d'abord, récupérons le fichier `kubeconfig` afin de pouvoir interagir avec notre cluster.
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Si nous exécutons la commande `get node` nous pouvons voir que nos nodes sont dans un état `NotReady`. 
Pas de panique, c'est tout à fait normal, nous devons installer notre CNI.
```sh
kubectl get no
```

Il est temps d'installer `Flannel`
```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
Une fois que tout les pods dans le namespace `kube-flannel` sont en train de tourner, nos nodes vont passer en état `Ready`.  
```sh
kubectl get no
NAME           STATUS   ROLES           AGE     VERSION
controlplane   Ready    control-plane   1h      v1.27.3
worker-01      Ready    worker          1h      v1.27.3
worker-02      Ready    worker          1h      v1.27.3
```

Afin de s'assurer que tout fonctionne correctement, nous allons créer un pod `dnsutils`.
```sh
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
```
Une fois que le pod est en train de tourner, nous pouvons exécuter la commande suivante.
```sh
kubectl exec -it dnsutils -- nslookup kubernetes.default
```

Ok, nous allons maintenant installer `metrics-server`.  
```sh
curl -OL https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
sed -i "/--metric-resolution/a\        - --kubelet-insecure-tls" components.yaml
kubectl apply -f components.yaml
rm components.yaml
```
Nous pouvons utiliser la commande suivante et nous assurer que `metrics-server` fonctionne correctement.
```sh
kubectl top no
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
controlplane   70m          3%     1874Mi          15%
worker-01      28m          2%     1612Mi          27%
worker-02      21m          2%     1463Mi          25%
```

Nous avons maintenant un cluster kubernetes sur nos VMs ARM sur Oracle Cloud, mais nous ne pouvons pas exposer de trafic http...
C'est plutôt décevant... 
Nous allons maintenant corriger ça!  

## Expose traffic

### MetalLB

D'après leur documentation [documentation](https://metallb.universe.tf)

> MetalLB est une implémentation de load-balancer pour les cluster kubernetes qui utilise les protocoles de routing standard.

MetalLB est capable de gérer le `BGP` ainsi que le `Layer2`. 
Nous allons configurer un protocole Layer2 avec IPv4.
```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

Nous devons ensuite configurer notre `IPAddressPool` pour `MetalLB`.
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.240-10.0.0.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
```

### Nginx Ingress Controller

Nous allons utiliser le Ingress Controller `Nginx`.  
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```
Grâce à metalLB, nous obtenons une ip externe pour notre service nginx de type load-balancer correspond à l'IP`10.0.0.240`.
```sh
kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.111.77.181   10.0.0.240    80:30325/TCP,443:32653/TCP   2h
ingress-nginx-controller-admission   ClusterIP      10.99.16.76     <none>        443/TCP                      2h
```

Nous allons déployer une application de test afin de vérifier que tout fonctionne correctement.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami-deployment
spec:
  selector:
    matchLabels:
      app: whoami
  replicas: 2
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami
        image: containous/whoami
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: whoami
  name: whoami
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: whoami
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /whoami
        pathType: Prefix
        backend:
          service:
            name: whoami
            port:
              number: 80
```
Et maintenant nous devrions être capables de faire un curl depuis un des nodes du cluster.
```sh
curl 10.0.0.240/whoami
```

Parfait ! Le seul problème est que c'est une ip privé... et donc à part depuis le cluster on peut pas faire grand chose...
On va donc installer `nginx` sur chaque node en mode reverse proxy et rediriger les call http sur l'ip de notre Ingress Controller. 
Grâce à ça, nous seront capable d'effectuer des requêtes depuis n'importe où.
Il est également possible de faire des routes NAT avec iptables, mais je préfère passer par un reverse proxy pour plus de modularité par la suite.
```sh
sudo apt install nginx
sudo vi /etc/nginx/sites-enabled/default
```
Remplacer le bloque `location /` par le contenu suivant:
```
location / {
    proxy_pass http://10.0.0.240:80/;
}
```
Il nous faut ensuite reload nginx.
```sh
sudo systemctl reload nginx
```
Nous pouvons maintenant appeler notre `whoami` depuis une des ips publiques de nos VMs.

## Load Balancer Oracle

L'offre gratuite comprend également un `Load Balancer`, nous allons donc en configurer un pour pointer sur l'ensemble de nos nodes.  
Je me sers de notre `whoami` comme prob de health check pour le LB.
Nous allons allez dans Networking -> Load Balancers -> Create a Load Balancers
- Dans la page de détails
    - donner un nom au LB
    - Ajouter le VCN et le subnet public
- Page des backends
    - Ajouts de tout nos nodes en backend
    - Dans la partie correspondant au path du health check utiliser `/whoami`
- Configuration du Listener
    - Si vous avez des certificats utiliser du HTTPS sinon du HTTP

Après quelques minutes, notre LB est fin prêt.
#### ![](/assets/images/2023-07-04-LB.png)

## Conclusion

Nous avons maintenant un cluster Kubernertes `up & running` en utilisant uniquement le free tier d'Oracle.
Ça nous fait quand même un jolie playground pour s'amuser sur tout ce qui touche à kube.

J'espère que cet article vous a été utile !