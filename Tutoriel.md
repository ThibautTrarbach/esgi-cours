# Documentation Docker

## Description de l'infrastructure : 
5 Hote (2cpu, 4go ram, 1 disk system de 64go, 1 disk non formaté de 32go, Ubuntu 22.04) : 
- ESGI-1 | IP 10.100.1.200 | TYPE : Master
- ESGI-2 | IP 10.100.1.201 | TYPE : Master
- ESGI-3 | IP 10.100.1.202 | TYPE : Worker
- ESGI-4 | IP 10.100.1.203 | TYPE : Worker
- ESGI-5 | IP 10.100.1.204 | TYPE : Worker

1 ip virtuel : 10.100.1.5

## Préparation des machine : 

Mise à jour et instalation des packets nécessaire au projet : 

```apt update && apt upgrade -y && apt install qemu-guest-agent -y```

Changer les hostname par leur nom + ".local.lan"
exemple : 

```hostnamectl set-hostname ESGI-1.local.lan```

Définir dans le fichier host les ip des nodes : 

```nano /etc/hosts```

Ajouter les ligne suivante en modifiant les adresses ip par les vôtres :
```
10.100.1.200 ESGI-1.local.lan
10.100.1.201 ESGI-2.local.lan
10.100.1.202 ESGI-3.local.lan
10.100.1.203 ESGI-4.local.lan
10.100.1.204 ESGI-5.local.lan
```

Modifier la seconde ligne 127.0.1.1 par le hostname du serveur

Cela devrais donné un fichier correspondant (Dans l'exemple le serveur ESGI-1) : 
```
127.0.0.1 localhost
127.0.1.1 ESGI-1.local.lan
10.100.1.200 ESGI-1.local.lan
10.100.1.201 ESGI-2.local.lan
10.100.1.202 ESGI-3.local.lan
10.100.1.203 ESGI-4.local.lan
10.100.1.204 ESGI-5.local.lan

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters"
```


## Instalation des dependance : 
```sudo apt-get install ca-certificates curl gnupg -y```

## Instalation de docker: 
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update && apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

## Instalation de ceph uniquement sur le serveur 1 (ESGI-1) : 
```
CEPH_RELEASE=18.2.0
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x cephadm
./cephadm add-repo --release reef
apt update -y
./cephadm install
```
Initialisation du cluster : 
(Remplacer par l'ip de l'hôte)
```
cephadm bootstrap --mon-ip 10.100.1.200 --allow-overwrite --allow-fqdn-hostname
```

Suite à cette commande récuperer le nom d'utilisateur et le mot de passe indiqué en sortie de script

Sur le serveur ESGI-1, récuperer la clé ssh dans /etc/ceph/ceph.pub

```
cat  /etc/ceph/ceph.pub
```

Sur le reste des serveurs, ajouter cette clée SSH a l'utilisateur root
```
nano /root/.ssh/authorized_keys
```

Se connecter a https://10.200.1.200:8443 ( Remplacer par l'ip de votre node 1)

Changer le mot de passe

Cliquer sur "Expand cluster"

Ensuite cliquer sur Add node et rensegné le hostname et l'ip.
Recomencer pour l'ensemble des nodes, sauf le 1.

Au bout de 3 ou 4 minutes, l'ensemble des node devrait apparaître dans la liste.
Quand il sont tous présent, cliquer sur next jusqu'a avoir terminer la configuration.

Ensuite attendre 10 minutes que le cluster se contruise.

Après cela, allez dans le menu file system et en créez un que l'on nomme Data.
Un message d'erreur arrive, cela n'est pas un probleme, c'est le temps de la creation du file system.


## Montage des répertoires

Sur l'ensemble des nodes, installer le packet ceph-common
 ```apt install ceph-common -y ```

Recupérer sur le node 1 les fichier suivant :
- /etc/ceph/ceph.client.admin.keyring
- /etc/ceph/ceph.conf

Les placer au même endroit sur l'ensemble des nodes.

Sur l'ensemble des serveur : 

Modifier le fstab
```nano /etc/fstab```
```
10.100.1.200,10.100.1.201:/ /mnt/data ceph name=admin,noatime,_netdev 0 0
```

Crée le dossier /mnt/data
```mkdir /mnt/data```

monter le disk : 
```mount -a```

## Création du cluster Docker Swarm

Initialisation de docker swarm (ESGI-1) : 
``` docker swarm init```

A la sortie de la commande récuperer la commande qui apparait et l'éxecuter sur chacune des nodes.

Ensuite sur le node 1 éxecuter (changer le host-name par celui de votre serveur 2) : 
docker node promote ESGI-2.local.lan

## Mise en place 

Sur ESGI-1 éxecuter les commandes suivante en modifiant les ip: 
```
echo "modprobe ip_vs" >> /etc/modules
modprobe ip_vs
docker run -d --name keepalived --restart=always \
  --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
  -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['10.100.1.200', '10.100.1.201']" \
  -e KEEPALIVED_VIRTUAL_IPS=10.100.1.5 \
  -e KEEPALIVED_PRIORITY=200 \
  -e KEEPALIVED_INTERFACE=ens18 \
  -e KEEPALIVED_PASSWORD=sdnfoqmskdfrqsrlgq \
  osixia/keepalived:2.0.20
```

Sur ESGI-2 éxecuter les commandes suivante en modifiant les ip: 
```
echo "modprobe ip_vs" >> /etc/modules
modprobe ip_vs
docker run -d --name keepalived --restart=always \
  --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
  -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['10.100.1.200', '10.100.1.201']" \
  -e KEEPALIVED_VIRTUAL_IPS=10.100.1.5 \
  -e KEEPALIVED_PRIORITY=100 \
  -e KEEPALIVED_INTERFACE=ens18 \
  -e KEEPALIVED_PASSWORD=sdnfoqmskdfrqsrlgq \
  osixia/keepalived:2.0.20
```

# Déploiment de la stack Docker

Créer les dossier nécessaire à la stack
```
mkdir -p /mnt/data/wordpress/db
mkdir -p /mnt/data/wordpress/file
```

télécharger la stack : 
```
wget https://raw.githubusercontent.com/ThibautTrarbach/esgi-cours/dev/stack1.yaml
```
et coller la stack

Déployer la stack du projet : 
```
docker stack deploy -c stack1.yaml stack1
```


# Stack 2 
Supprimer la stack 1
```
docker stack rm stack1
```

Créer les dossier nécessaire à la stack
```
mkdir -p /mnt/data/wordpress2/db
mkdir -p /mnt/data/wordpress2/file
mkdir -p /mnt/data/proxy/
```

Télècharger la configuration traefik
```
wget https://raw.githubusercontent.com/ThibautTrarbach/esgi-cours/dev/traefik/traefik.yml
```

Déplacer la configuration traefik : 
```
mv traefik.yml /mnt/data/proxy/traefik.yml
```

Télècharger la stack 2 : 
```
wget https://raw.githubusercontent.com/ThibautTrarbach/esgi-cours/dev/stack2.yaml
```

Déployer la stack du projet : 
```
docker stack deploy -c stack2.yaml stack2
```
