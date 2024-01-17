# Documentation Docker

## Pré-requis :
5 VM composé de :
- 2 Coeur
- 4 Go de RAM
- 1 disques principale de 64 Go
- 1 disque non formaté de 32 Go
- Ubuntu serveur 22.04 

## Description de l'infrastructure : 
5 Hote (2cpu, 4go ram, 1 disk system de 64go, 1 disk non formaté de 32go, Ubuntu Serveur 22.04) : 
- ESGI-CEPH01 | IP 10.100.1.200 | TYPE : CEPH
- ESGI-CEPH02 | IP 10.100.1.201 | TYPE : CEPH
- ESGI-CEPH03 | IP 10.100.1.202 | TYPE : CEPH
- ESGI-CEPH04 | IP 10.100.1.203 | TYPE : CEPH
- ESGI-CEPH05 | IP 10.100.1.204 | TYPE : CEPH
- ESGI-DOCKER01 | IP 10.100.1.210 | TYPE : MANAGER
- ESGI-DOCKER02 | IP 10.100.1.211 | TYPE : MANAGER
- ESGI-DOCKER03 | IP 10.100.1.212 | TYPE : WORKER
- ESGI-DOCKER04 | IP 10.100.1.213 | TYPE : WORKER
- ESGI-DOCKER05 | IP 10.100.1.214 | TYPE : WORKER
- ESGI-DOCKER06 | IP 10.100.1.215 | TYPE : WORKER
- ESGI-DOCKER07 | IP 10.100.1.216 | TYPE : WORKER
- KeepAlived | IP 10.100.1.220 | TYPE : IP Virtuel

## Preparations OS (CEPH | MANAGER | WORKER) : 

Mise à jour et instalation des packets nécessaire sur l'infrastructure : 

```apt update && apt upgrade -y && apt install qemu-guest-agent ca-certificates curl gnupg -y```

Changer les hostname par leur nom + ".ent-alpha.lan"
exemple : 

```hostnamectl set-hostname ESGI-CEPH01.ent-alpha.lan```

Définir dans le fichier host les ip des nodes : 

```nano /etc/hosts```

Ajouter les ligne suivante en modifiant les adresses ip par les vôtres :
```
10.100.1.200 ESGI-CEPH01.ent-alpha.lan
10.100.1.201 ESGI-CEPH02.ent-alpha.lan
10.100.1.202 ESGI-CEPH03.ent-alpha.lan
10.100.1.203 ESGI-CEPH04.ent-alpha.lan
10.100.1.204 ESGI-CEPH05.ent-alpha.lan
10.100.1.210 ESGI-DOCKER01.ent-alpha.lan
10.100.1.211 ESGI-DOCKER02.ent-alpha.lan
10.100.1.212 ESGI-DOCKER03.ent-alpha.lan
10.100.1.213 ESGI-DOCKER04.ent-alpha.lan
10.100.1.214 ESGI-DOCKER05.ent-alpha.lan
10.100.1.215 ESGI-DOCKER06.ent-alpha.lan
10.100.1.216 ESGI-DOCKER07.ent-alpha.lan
```

Modifier la seconde ligne 127.0.1.1 par le hostname du serveur

Cela devrais donné un fichier correspondant (Dans l'exemple le serveur ESGI-CEPH01) : 
```
127.0.0.1 localhost
127.0.1.1 ESGI-CEPH01.ent-alpha.lan
10.100.1.200 ESGI-CEPH01.ent-alpha.lan
10.100.1.201 ESGI-CEPH02.ent-alpha.lan
10.100.1.202 ESGI-CEPH03.ent-alpha.lan
10.100.1.203 ESGI-CEPH04.ent-alpha.lan
10.100.1.204 ESGI-CEPH05.ent-alpha.lan
10.100.1.210 ESGI-DOCKER01.ent-alpha.lan
10.100.1.211 ESGI-DOCKER02.ent-alpha.lan
10.100.1.212 ESGI-DOCKER03.ent-alpha.lan
10.100.1.213 ESGI-DOCKER04.ent-alpha.lan
10.100.1.214 ESGI-DOCKER05.ent-alpha.lan
10.100.1.215 ESGI-DOCKER06.ent-alpha.lan
10.100.1.216 ESGI-DOCKER07.ent-alpha.lan

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters"
```

## Instalation de docker: 
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh test-docker.sh
```

# Instalations du cluster CEPH

## Instalation de ceph ADM sur ESGI-CEPH01 : 

```
CEPH_RELEASE=18.2.0
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x cephadm
./cephadm add-repo --release reef
apt update -y
./cephadm install
```

Initialisation du cluster : 
```
cephadm bootstrap --mon-ip 10.100.1.200 --allow-overwrite --allow-fqdn-hostname
```

Suite à cette commande récuperer le nom d'utilisateur et le mot de passe indiqué en sortie de script

Sur le serveur ESGI-CEPH01, récuperer la clé ssh dans /etc/ceph/ceph.pub

```
cat  /etc/ceph/ceph.pub
```

Sur l'ensemble des serveur CEPH, ajouter cette clée SSH a l'utilisateur root
```
nano /root/.ssh/authorized_keys
```

Se connecter a https://10.200.1.200:8443

Changer le mot de passe

Cliquer sur "Expand cluster"

Ensuite cliquer sur Add node et rensegné le hostname et l'ip.
Recomencer pour l'ensemble des nodes CEPH, sauf ESGI-CEPH01.

Au bout de 3 ou 4 minutes, l'ensemble des node devrait apparaître dans la liste.
Quand il sont tous présent, cliquer sur next jusqu'a avoir terminer la configuration.

Ensuite attendre 10 minutes que le cluster se contruise.

Après cela, allez dans le menu file system et en créez un que l'on nomme Data.
Un message d'erreur arrive, cela n'est pas un probleme, c'est le temps de la creation du file system.

# Instalation des hotes Docker 

## Montage du system de fichier CEPH

Sur l'ensemble des nodes, installer le packet ceph-common
 ```apt install ceph-common -y ```

Recupérer sur ESGI-CEPH01 les fichier suivant :
- /etc/ceph/ceph.client.admin.keyring
- /etc/ceph/ceph.conf

Les placer au même endroit sur l'ensemble des WORKER et MANAGER.

Sur l'ensemble des serveur WORKER et MANAGER : 

Modifier le fstab avec vos adresses ip :
```nano /etc/fstab```
```
ESGI-CEPH01.ent-alpha.lan,ESGI-CEPH02.ent-alpha.lan,ESGI-CEPH03.ent-alpha.lan,ESGI-CEPH04.ent-alpha.lan,ESGI-CEPH05.ent-alpha.lan:/ /mnt/data ceph name=admin,noatime,_netdev 0 0
```

Crée le dossier /mnt/data
```mkdir /mnt/data```

monter le disk : 
```mount -a```

## Création du cluster Docker Swarm

Initialisation de docker swarm (ESGI-1) : 
``` docker swarm init```

A la sortie de la commande récuperer la commande qui apparait et l'éxecuter sur chacune des nodes.

Ensuite sur ESGI-DOCKER01 éxecuter : 
docker node promote ESGI-DOCKER02.ent-alpha.lan

## Mise en place 

Sur ESGI-DOCKER01 éxecuter les commandes suivante en modifiant les ip: 
```
echo "modprobe ip_vs" >> /etc/modules
modprobe ip_vs
docker run -d --name keepalived --restart=always \
  --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
  -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['10.100.1.210', '10.100.1.211']" \
  -e KEEPALIVED_VIRTUAL_IPS=10.100.1.220 \
  -e KEEPALIVED_PRIORITY=200 \
  -e KEEPALIVED_INTERFACE=ens18 \
  -e KEEPALIVED_PASSWORD=sdnfoqmskdfrqsrlgq \
  osixia/keepalived:2.0.20
```

Sur ESGI-DOCKER02 (ou l'ensemble des MANAGER restant) éxecuter les commandes suivante en modifiant les ip: 
```
echo "modprobe ip_vs" >> /etc/modules
modprobe ip_vs
docker run -d --name keepalived --restart=always \
  --cap-add=NET_ADMIN --cap-add=NET_BROADCAST --cap-add=NET_RAW --net=host \
  -e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['10.100.1.210', '10.100.1.211']" \
  -e KEEPALIVED_VIRTUAL_IPS=10.100.1.220 \
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
