# Cephadmin
cephadm est un outil pour déployer et gérer un cluster Ceph par connexion aux hôtes à partir du démon gestionnaire (mgr) via SSH. Il permet d'ajouter, supprimer ou mettre à jour des conteneurs de démons Ceph. Il ne repose pas sur des outils de configuration ou d'orchestration externes comme Ansible, Rook ou Salt.

cephadm est le nouvel outil de gestion d'un cluster Ceph fourni à partir de la version Octopus v15.2.0 et ne prend pas en charge les anciennes versions de Ceph. Il est possible de migrer les anciennes instances vers cephadmin. cephadmin remplace ceph-deploy utilisé dans les anciennes version de Ceph.
## Installation du cluster Ceph avec cephadm
```
[vagrant@cn1 ~]$ curl --silent --remote-name --location https://github.com/ceph/ceph/raw/pacific/src/cephadm/cephadm
[vagrant@cn1 ~]$ chmod +x cephadm
[vagrant@cn1 ~]$ sudo ./cephadm add-repo --release pacific
sudo ./cephadm install
# Le boostrap créé un daemon monitor qui doit écouter sur une IP celle de la machine.
[vagrant@cn1 ~]$ sudo cephadm bootstrap --mon-ip 192.168.0.11
# La .11 est l'IP de l'ethernet 1 de la machine

Fetching dashboard port number...
Ceph Dashboard is now available at:
....
sudo /sbin/cephadm shell --fsid 5d6cc7a0-e316-11eb-8cc9-525400bb19d6 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring


For more information see:

	https://docs.ceph.com/docs/master/mgr/telemetry/

Bootstrap complete.

# Votre cluster Ceph est créé. Notez bien la valeur du Password par défaut. il sera utilisé plus tard pour accéder au dashboard.
# 

# Ajout ssh ceph.pub key sur les nodes 2 à 4
[vagrant@cn1 ~]$ for i in {2..4}; do ssh-copy-id -f -i /etc/ceph/ceph.pub root@cn$i; done

# Connexion au cluster ceph
[vagrant@cn1 ~]$ sudo cephadm shell 
Inferring fsid 2e90db8c-541a-11eb-bb6e-525400ae1f18
Inferring config /var/lib/ceph/2e90db8c-541a-11eb-bb6e-525400ae1f18/mon.cn1/config
Using recent ceph image docker.io/ceph/ceph:v15
```
# Remarque  : on passe dans le conteneur. Attention au changement de shell en [ceph: root@cn1 /]


Quel est le status du cluster ?
```
[ceph: root@cn1 /]#  ceph -s  = "ceph status"
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum cn1 (age 5m)
    mgr: cn1.pnzyvw(active, since 5m)
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:  
```

Quel est l'état du cluster ?

```
ceph health detail 
```

Du process monitor? de l'osd 
```
ceph: root@cn1 /]# ceph mon dump
epoch 1
fsid 5d6cc7a0-e316-11eb-8cc9-525400bb19d6
last_changed 2021-07-12T13:40:03.600961+0000
created 2021-07-12T13:40:03.600961+0000
min_mon_release 16 (pacific)
election_strategy: 1
0: [v2:192.168.0.11:3300/0,v1:192.168.0.11:6789/0] mon.cn1
dumped monmap epoch 1
[ceph: root@cn1 /]# 
```
Quel est l'espace utilisé par le cluster:
```
[ceph: root@cn1 /]# ceph df
--- RAW STORAGE ---
CLASS  SIZE  AVAIL  USED  RAW USED  %RAW USED
TOTAL   0 B    0 B   0 B       0 B          0
 
--- POOLS ---
POOL  ID  PGS  STORED  OBJECTS  USED  %USED  MAX AVAIL
[ceph: root@cn1 /]# 

```


# Remarque : Le cluster est en warring au début, car il n'y a pas encore d'osd ou plusieurs mon, mais on va rêgler cela rapidement
```
[ceph: root@cn1 /]# cat /etc/redhat-release 
CentOS Linux release 8.3.2011
# Remarque : l'OS du conteneur est basé sur CentOS Linux release 8.3.2011

# Ajouter les noeudes supplémentaires au cluster Ceph
for 
[ceph: root@cn1 /]# ceph orch host add cn2
Added host 'cn2'
[ceph: root@cn1 /]# ceph orch host add cn3
Added host 'cn3'
[ceph: root@cn1 /]# ceph orch host add cn4
Added host 'cn4'

# liste des noeudes dans le cluster. On retrouve bien les 4 noeudes
[ceph: root@cn1 /]# ceph orch host ls
HOST  ADDR  LABELS  STATUS  
cn1   cn1                   
cn2   cn2                   
cn3   cn3                   
cn4   cn4                   

# l'ajout des noeudes va augmenter automatiquement le nombre de process dans le cluster.
[ceph: root@cn1 /]# ceph orch ls
NAME           RUNNING  REFRESHED  AGE  PLACEMENT  IMAGE NAME                            IMAGE ID      
alertmanager       1/1  51s ago    7m   count:1    docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash              1/4  51s ago    8m   *          docker.io/ceph/ceph:v15               5553b0cb212c  
grafana            1/1  51s ago    7m   count:1    docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mgr                1/2  51s ago    8m   count:2    docker.io/ceph/ceph:v15               5553b0cb212c  
mon                1/5  51s ago    8m   count:5    docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter      1/4  51s ago    7m   *          docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
prometheus         1/1  51s ago    7m   count:1    docker.io/prom/prometheus:v2.18.1     de242295e225  

# Surveillons les modifications au niveau du cluster 
[ceph: root@cn1 /]# ceph -w
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_WARN
            failed to probe daemons or devices
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 1 daemons, quorum cn1 (age 10m)
    mgr: cn1.pnzyvw(active, since 10m), standbys: cn2.rkgnmp
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
 

2021-01-11T14:49:03.762707+0000 mon.cn1 [INF] mon.cn1 calling monitor election
2021-01-11T14:49:05.680296+0000 mon.cn3 [INF] mon.cn3 calling monitor election
2021-01-11T14:49:08.820774+0000 mon.cn1 [INF] mon.cn1 is new leader, mons cn1,cn3 in quorum (ranks 0,1)
2021-01-11T14:49:08.897971+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN failed to probe daemons or devices; OSD count 0 < osd_pool_default_size 3
2021-01-11T14:49:08.898046+0000 mon.cn1 [WRN] [WRN] CEPHADM_REFRESH_FAILED: failed to probe daemons or devices
2021-01-11T14:49:08.898062+0000 mon.cn1 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
2021-01-11T14:49:09.118427+0000 mon.cn1 [INF] mon.cn1 calling monitor election
2021-01-11T14:49:09.140783+0000 mon.cn3 [INF] mon.cn3 calling monitor election
2021-01-11T14:49:11.104706+0000 mon.cn2 [INF] mon.cn2 calling monitor election
2021-01-11T14:49:14.161148+0000 mon.cn1 [INF] mon.cn1 is new leader, mons cn1,cn3,cn2 in quorum (ranks 0,1,2)
2021-01-11T14:49:14.234202+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN failed to probe daemons or devices; OSD count 0 < osd_pool_default_size 3
2021-01-11T14:49:14.234224+0000 mon.cn1 [WRN] [WRN] CEPHADM_REFRESH_FAILED: failed to probe daemons or devices
2021-01-11T14:49:14.234229+0000 mon.cn1 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
2021-01-11T14:49:30.890447+0000 mon.cn1 [INF] Health check cleared: CEPHADM_REFRESH_FAILED (was: failed to probe daemons or devices)
2021-01-11T14:50:00.004693+0000 mon.cn1 [WRN] Health detail: HEALTH_WARN OSD count 0 < osd_pool_default_size 3
2021-01-11T14:50:00.004792+0000 mon.cn1 [WRN] [WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3


# au bout de 3 minutes, on retrouve bien 4/4 service crash, 3/5 mon, 4/4 node-exporter 
[ceph: root@cn1 /]# ceph orch ls
NAME           RUNNING  REFRESHED  AGE  PLACEMENT  IMAGE NAME                            IMAGE ID      
alertmanager       1/1  63s ago    13m  count:1    docker.io/prom/alertmanager:v0.20.0   0881eb8f169f  
crash              4/4  65s ago    13m  *          docker.io/ceph/ceph:v15               5553b0cb212c  
grafana            1/1  63s ago    13m  count:1    docker.io/ceph/ceph-grafana:6.6.2     a0dce381714a  
mgr                2/2  64s ago    13m  count:2    docker.io/ceph/ceph:v15               5553b0cb212c  
mon                3/5  65s ago    13m  count:5    docker.io/ceph/ceph:v15               5553b0cb212c  
node-exporter      4/4  65s ago    13m  *          docker.io/prom/node-exporter:v0.18.1  e5a616e4b9cf  
prometheus         1/1  63s ago    13m  count:1    docker.io/prom/prometheus:v2.18.1     de242295e225  

On peut aussi utiliser cephadm
[vagrant@cn1 ~]$ sudo cephadm shell -- ceph orch host ls
Inferring fsid 5d6cc7a0-e316-11eb-8cc9-525400bb19d6
Inferring config /var/lib/ceph/5d6cc7a0-e316-11eb-8cc9-525400bb19d6/mon.cn1/config
Using recent ceph image docker.io/ceph/ceph@sha256:829ebf54704f2d827de00913b171e5da741aad9b53c1f35ad59251524790eceb
HOST  ADDR          LABELS  STATUS  
cn1   192.168.0.11  _admin          
cn2   192.168.0.12                  
cn3   192.168.0.13                  
cn4   192.168.0.14    
[vagrant@cn1 ~]$ sudo cephadm ls|jq .[].name
"mon.cn1"
"mgr.cn1.esnqup"
"alertmanager.cn1"
"crash.cn1"
"grafana.cn1"
"node-exporter.cn1"
"prometheus.cn1"
"osd.3"
"osd.7"

[vagrant@cn1 ~]$ sudo cephadm shell -- ceph orch device ls
Inferring fsid 5d6cc7a0-e316-11eb-8cc9-525400bb19d6
Inferring config /var/lib/ceph/5d6cc7a0-e316-11eb-8cc9-525400bb19d6/mon.cn1/config
Using recent ceph image docker.io/ceph/ceph@sha256:829ebf54704f2d827de00913b171e5da741aad9b53c1f35ad59251524790eceb
WARNING: The same type, major and minor should not be used for multiple devices.
WARNING: The same type, major and minor should not be used for multiple devices.
Hostname  Path      Type  Serial  Size   Health   Ident  Fault  Available  
cn1       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    No         
cn1       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    No         
cn2       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    No         
cn2       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    No         
cn3       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    No         
cn3       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    No         
cn4       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    No         
cn4       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    No         



# le cluster Ceph est toujours en warring, mais il a maintenant 3 moniteurs et 2 mgr
[ceph: root@cn1 /]# ceph -s
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3
 
  services:
    mon: 3 daemons, quorum cn1,cn3,cn2 (age 2m)
    mgr: cn1.pnzyvw(active, since 14m), standbys: cn2.rkgnmp
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     
         
# ajoutons des disques au cluster, mais avant cela listons l'ensemble des disques libres existants
[ceph: root@cn1 /]# ceph orch device ls
Hostname  Path      Type  Serial  Size   Health   Ident  Fault  Available  
cn1       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    Yes        
cn1       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    Yes        
cn2       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    Yes        
cn2       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    Yes        
cn3       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    Yes        
cn3       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    Yes        
cn4       /dev/vdb  hdd           42.9G  Unknown  N/A    N/A    Yes        
cn4       /dev/vdc  hdd           53.6G  Unknown  N/A    N/A    Yes        

# une commande pour ajouter tous les disques libres, c'est pratique ;)
[ceph: root@cn1 /]# ceph orch apply osd --all-available-devices
Scheduled osd.all-available-devices update...

# surveillons les modifications du Cluster
[ceph: root@cn1 /]# ceph -w
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_WARN
            failed to probe daemons or devices
 
  services:
    mon: 3 daemons, quorum cn1,cn3,cn2 (age 6m)
    mgr: cn1.pnzyvw(active, since 18m), standbys: cn2.rkgnmp
    osd: 8 osds: 0 up, 0 in
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:     100.000% pgs unknown
             1 unknown
 

2021-01-11T14:55:42.462132+0000 mon.cn1 [INF] osd.1 [v2:192.168.0.13:6800/3357115903,v1:192.168.0.13:6801/3357115903] boot
2021-01-11T14:55:42.462189+0000 mon.cn1 [INF] osd.0 [v2:192.168.0.14:6800/3301640803,v1:192.168.0.14:6801/3301640803] boot
2021-01-11T14:55:43.489327+0000 mon.cn1 [INF] osd.2 [v2:192.168.0.12:6800/1513505846,v1:192.168.0.12:6801/1513505846] boot
2021-01-11T14:55:44.516120+0000 mon.cn1 [INF] osd.3 [v2:192.168.0.11:6802/475127529,v1:192.168.0.11:6803/475127529] boot
2021-01-11T14:55:45.552359+0000 mon.cn1 [INF] osd.4 [v2:192.168.0.14:6808/2752264528,v1:192.168.0.14:6809/2752264528] boot
2021-01-11T14:55:46.573267+0000 mon.cn1 [INF] osd.6 [v2:192.168.0.13:6808/3030442383,v1:192.168.0.13:6809/3030442383] boot
2021-01-11T14:55:47.608346+0000 mon.cn1 [INF] osd.5 [v2:192.168.0.12:6808/3117922813,v1:192.168.0.12:6809/3117922813] boot
2021-01-11T14:55:48.695563+0000 mon.cn1 [INF] osd.7 [v2:192.168.0.11:6810/4205497031,v1:192.168.0.11:6811/4205497031] boot
2021-01-11T14:55:54.989466+0000 mon.cn1 [INF] Health check cleared: CEPHADM_REFRESH_FAILED (was: failed to probe daemons or devices)
2021-01-11T14:55:54.989528+0000 mon.cn1 [INF] Cluster is now healthy


# vérification des disques utilisés par le cluster
[ceph: root@cn1 /]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME      STATUS  REWEIGHT  PRI-AFF
-1         0.35156  root default                            
-9         0.08789      host cn1                            
 3    hdd  0.03909          osd.3      up   1.00000  1.00000
 7    hdd  0.04880          osd.7      up   1.00000  1.00000
-7         0.08789      host cn2                            
 2    hdd  0.03909          osd.2      up   1.00000  1.00000
 5    hdd  0.04880          osd.5      up   1.00000  1.00000
-3         0.08789      host cn3                            
 1    hdd  0.03909          osd.1      up   1.00000  1.00000
 6    hdd  0.04880          osd.6      up   1.00000  1.00000
-5         0.08789      host cn4                            
 0    hdd  0.03909          osd.0      up   1.00000  1.00000
 4    hdd  0.04880          osd.4      up   1.00000  1.00000

# vérifions le status du cluster. C'est ok, on a bien 8 osds dans le cluster avec 4 nodes.
[ceph: root@cn1 /]# ceph -s
# = ceph health status
  cluster:
    id:     2e90db8c-541a-11eb-bb6e-525400ae1f18
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum cn1,cn3,cn2 (age 7m)
    mgr: cn1.pnzyvw(active, since 19m), standbys: cn2.rkgnmp
    osd: 8 osds: 8 up (since 71s), 8 in (since 71s)
 
  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   8.0 GiB used, 352 GiB / 360 GiB avail
    pgs:     1 active+clean




--- RAW STORAGE ---
CLASS     SIZE    AVAIL    USED  RAW USED  %RAW USED
hdd    360 GiB  360 GiB  41 MiB    41 MiB       0.01
TOTAL  360 GiB  360 GiB  41 MiB    41 MiB       0.01
 
--- POOLS ---
POOL                   ID  PGS  STORED  OBJECTS  USED  %USED  MAX AVAIL
device_health_metrics   1    1     0 B        0   0 B      0    114 GiB

# On peut relier le stockage à une localisation géographique
ceph osd crush add-bucket rack1 rack
ceph osd crush add-bucket rack2 rack
ceph osd crush add-bucket rack3 rack
ceph osd crush add-bucket rack4 rack
ceph osd crush set osd.4 1.0 datacenter=datacenter1 root=default rack=rack4


```
# Remarque : Le cluster est maintenant ok, et prêt pour la configuration des pools
```
## connexion au dashboard du cluster Ceph
retrouvez vos informations de connexions fournies lors de l'initialisation du cluster , et utilisez un navigateur pour vous connecter à l'adresse https://192.168.0.11:8443
saisissez le compte admin et le mot de passe associé. Il faudra changer le mot de passe par défaut à la première connexion.
Voici une copie d'écran du dashboard de ceph octopus
![ceph-dashboard](ceph-dashboard.png)


