# Cephlab
Ce lab virtuel Vagrant contient un environnement pour installer et configurer un cluster Ceph en version octopus en quelques étapes accompagné de commentaires pour vous guider à chaque étape. ces informations ne sont pas une procédure d'installation automatisée pour une utilisation en production. Vous trouverez les différentes étapes configurer les services RBD, CEHPFS, S3, un Dashboard avec prometheus et grafana, ainsi que la mise en HA de 2 RGW.
## Installation de vagrant et libvirt sur votre poste Ubuntu
```
sudo apt-get install virt-manager
sudo apt install vagrant
# vérifier la liste des plugins
sudo vagrant list plugin 
```
## Clone cephlab
```
git clone https://github.com/sg4r/cephlab.git
cd cephlab
```
## Start lab
```
#start all Vms 
vagrant up
#status des Vms
vagrant status
```
## Préparation de la configuration des noeuds du cluster Ceph
```
#connexion au premier node
vagrant ssh cn1
#configuration de tous les nodes Ceph
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i dnf -y install -y podman; done
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i systemctl stop firewalld.service; done
[vagrant@cn1 ~]$ for i in {1..4}; do ssh root@cn$i systemctl disable firewalld.service; done
```

## ceph ports 

### Ceph Monitors listen on ports 3300 and 6789


### A Ceph Metadata Server or Ceph Manager listens on the first available port on the public network beginning at port 6800.

### By default, Ceph OSD Daemons bind to the first available ports on a Ceph Node beginning at port 6800.


### Each Ceph OSD Daemon on a Ceph Node may use up to four ports:

- One for talking to clients and monitors.
- One for sending data to other OSDs.
- Two for heartbeating on each interface.

When a daemon fails and restarts without letting go of the port, the restarted daemon will bind to a new port. You should open the entire 6800-7300 port range to handle this possibility.


## Etapes

Les différentes étapes :

* [Création du cluster Ceph](cephcreate.md)
* [Stockage RBD](cephrbd.md)
* [Stockage CephFS](cephfs.md)
* [Stockage S3](cephs3.md)
* [Ceph Dashboard](cephdashboard.md)
* [Mise a jour](upgrade.md)

## Fin de partie

Pour supprimer les vms ou tout simplement faire le ménage avant de refaire une configuration , utiliser vagrant pour détruire toutes les vms*
```
# suppressions de toutes les vms
vagrant destroy
# si vous voulez tout supprimer
cd ..
rm -fr ./ceph-octopus
```
## Evolutions
Les évolutions concernant ce lab sont :
- Pool Erasure code pour rgw
- Configuration HA des rgw S3
- Réplication multi-sites S3
- [Convertir un cluster Ceph existant](upgrade.nautilus2octopus.md)
- [Exemple d'installation avec ceph-ansible](cephansible.md)
