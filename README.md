# TP7 - Thibault FEUGERE

Pour ce TP 5 fichiers sont importants pour démarrer les containers :
- `docker-compose.yml` qui est à la racine du projet
- `maria1.cnf`
- `maria2.cnf`
- `maria3.cnf`
- `dumps/test1.sql`

Les trois fichiers de configuration `maria` sont dans le dossier `config` qui est à la racine du projet, au même niveau que le `docker-compose.yml` donc.

Voici mon `docker-compose.yml` qui instancie trois serveurs mariadb et qui va me permettre de répondre aux différents points du TP.

```yaml
version: "3.7"

services: 
  maria1:
    image: mariadb:10.4
    restart: on-failure
    environment:
      - MYSQL_ROOT_PASSWORD=password
    command: --wsrep-new-cluster
    volumes:
      - ./maria1:/var/lib/mysql
      - ./config/maria1.cnf:/etc/mysql/mariadb.conf.d/maria1.cnf
      - ./dumps:/dumps 
    expose: 
      - 3306
      - 4567
      - 4568
      - 4444
    networks:
      cluster:
        ipv4_address: 172.29.0.3
  maria2:
    image: mariadb:10.4
    restart: on-failure
    environment:
      - MYSQL_ROOT_PASSWORD=password
    volumes:
      - ./maria2:/var/lib/mysql
      - ./config/maria2.cnf:/etc/mysql/mariadb.conf.d/maria2.cnf
      - ./dumps:/dumps 
    expose: 
      - 3306
      - 4567
      - 4568
      - 4444
    links:
      - maria1
    networks:
      cluster:
        ipv4_address: 172.29.0.4
  maria3:
    image: mariadb:10.4
    restart: on-failure
    environment:
      - MYSQL_ROOT_PASSWORD=password
    volumes:
      - ./maria3:/var/lib/mysql
      - ./config/maria3.cnf:/etc/mysql/mariadb.conf.d/maria3.cnf
      - ./dumps:/dumps 
    expose: 
      - 3306
      - 4567
      - 4568
      - 4444
    links:
      - maria1
    networks:
      cluster:
        ipv4_address: 172.29.0.5

networks: 
  cluster:
    driver: bridge
    ipam:
     config:
       - subnet: 172.29.0.0/24
         gateway: 172.29.0.1
```

## 1. Instanciez 3 serveurs maria

Les trois services qui sont des serveurs `mariadb` sont nommés :
- `maria1`
- `maria2`
- `maria3`

Ils comportent tous les trois l'image : `image: mariadb:10.4`

## 2. Assurez vous que les serveurs puissent communiquer entre eux en ajoutant la configuration nécessaire dans les docker-compose

Pour qu'il puisse communiquer, au début j'avais mis un network internal afin qu'ils soient tous les trois dans le même réseau, cependant pour la suite, parfois les adresses IP changeaint ce qui n'était pas pratique du tout. De ce fait je me suis demandé si je pouvais définir un sous réseau avec trois IP fixes. Et c'est possible !

Pour ne pas avoir de conflit avec mes autres réseaux docker j'ai changé le nom de `internal` par `cluster`.

Ensuite je l'ai passé en accès par pont (`bridge`) et je l'ai mis dans le sous réseau `172.29.0.9/24` ce qui nous donne :

```yaml
networks: 
  cluster:
    driver: bridge
    ipam:
     config:
       - subnet: 172.29.0.0/24
         gateway: 172.29.0.1
```

Ensuite j'attribue le réseau `cluster` à mes trois services en spécifiant l'ip fixe souhaitée :

```yaml
networks:
  cluster:
    ipv4_address: 172.29.0.5
```

Ce qu'il faut retenir pour la suite :
- maria1 : 172.29.0.3
- maria2 : 172.29.0.4
- maria3 : 172.29.0.5

Je ne sais pas s'il y avait d'autres moyens mais j'aime bien travailler avec des IP fixes plutôt que des hostname (technique que j'avais testé avant).

## 3. Ajoutez la configuration nécessaire pour chaque node

Mes trois fichiers de configuration sont :

`maria1.cnf`

```
[mysqld]
# Load Galera Cluster

wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name='mariadb_cluster'

wsrep_node_address="172.29.0.3"
wsrep_node_name=node1
wsrep_cluster_address="gcomm://172.29.0.3,172.29.0.4,172.29.0.5"

binlog_format=ROW

default_storage_engine=InnoDB

innodb_autoinc_lock_mode=2
innodb_doublewrite=1

query_cache_size=0

wsrep_on=ON
```

`maria2.cnf`

```
[mysqld]
# Load Galera Cluster

wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name='mariadb_cluster'

wsrep_node_address="172.29.0.4"
wsrep_node_name=node2
wsrep_cluster_address="gcomm://172.29.0.3,172.29.0.4,172.29.0.5"

binlog_format=ROW

default_storage_engine=InnoDB

innodb_autoinc_lock_mode=2
innodb_doublewrite=1

query_cache_size=0

wsrep_on=ON
```

`maria3.cnf`

```
[mysqld]
# Load Galera Cluster

wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name='mariadb_cluster'

wsrep_node_address="172.29.0.5"
wsrep_node_name=node3
wsrep_cluster_address="gcomm://172.29.0.3,172.29.0.4,172.29.0.5"

binlog_format=ROW

default_storage_engine=InnoDB

innodb_autoinc_lock_mode=2
innodb_doublewrite=1

query_cache_size=0

wsrep_on=ON
```

Il faut définir le chemin vers la librairie Galera avec la ligne : `wsrep_provider = /usr/lib/galera/libgalera_smm.so`

Pour qu'ils appartiennent tous au même cluster je leur donne tous le nom : `wsrep_cluster_name='mariadb_cluster'`.

On active la réplication : `wsrep_on=ON`

On définit l'adresse IP du noeud (node) en question avec la ligne : `wsrep_node_address="172.29.0.5"` ainsi que son nom `wsrep_node_name=node3`.

Ensuite je défini l'adresse de tous les cluster : `wsrep_cluster_address="gcomm://172.29.0.3,172.29.0.4,172.29.0.5"`.

J'ai mis les IP en dur mais je me demande si avec la définit de l'IP du node ainsi que son nom cela ne me permet pas de travailler avec les noms de nodes par la suite. Je n'ai pas plus creusé ce sujet.

Ces fichiers de configuration sont associés avec la commande : la ligne dans `volumes` : - `./config/maria1.cnf:/etc/mysql/mariadb.conf.d/maria1.cnf`.

## 4. Importez un dump quelconque sur un des nodes et vérifiez que celui ci est bien présent sur les autres nodes

Dans mon dossier `dumps` j'ai un fichier test1.sql que je vais importer sur `maria1` afin de vérifier que la réplication s'effectue sur `maria2` et `maria3`.

J'effectue la commande : `mysql -u root --password=password < /dumps/test1.sql` qui va créer une base de données `teams`.

Sur `maria1` :

```
MariaDB [teams]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| teams              |
+--------------------+
4 rows in set (0.000 sec)
```

Sur `maria2` :

```
MariaDB [teams]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| teams              |
+--------------------+
4 rows in set (0.000 sec)
```

Sur `maria3` :

```
MariaDB [teams]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| teams              |
+--------------------+
4 rows in set (0.000 sec)
```

On peut regarder dans les logs aussi et on voit que Wsrep fonctionne correctement. Avec le début de la replication :

```
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: Start replication
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: Connecting with bootstrap option: 0
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: Setting GCS initial position to 00000000-0000-0000-0000-000000000000:-1
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: protonet asio version 0
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: Using CRC-32C for message checksums.
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: backend: asio
```

La librairie est bien chargée :

```
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: Loading provider /usr/lib/galera/libgalera_smm.so initial position: 00000000-0000-0000-0000-000000000000:-1
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: wsrep_load(): loading provider library '/usr/lib/galera/libgalera_smm.so'
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: wsrep_load(): Galera 26.4.5(rb3764ab6) by Codership Oy <info@codership.com> loaded successfully.
maria3_1  | 2020-10-25 10:40:10 0 [Note] WSREP: CRC-32C: using hardware acceleration.
```

On peut voir que les différents serveurs communiquent entre-eux :

`[Note] WSREP: async IST sender starting to serve tcp://172.29.0.4:4568 sending 2-11, preload starts from 2` (on remarque l'importance d'ouvrir le port 4568)

## 5. Éteignez toutes les nodes et trouvez celui depuis lequel vous pouvez bootstrapper le cluster au redémarrage

Pour stopper, on fait : `docker-compose stop`  :

```
Stopping tp7_maria3_1 ... done
Stopping tp7_maria2_1 ... done
Stopping tp7_maria1_1 ... done
```

Pour boostraper nous pouvons regarder le fichier intitulé : `grastate.dat`. 

Avant modification le contenu était :

```
root@maria1:/# cat /var/lib/mysql/grastate.dat 
# GALERA saved state
version: 2.1
uuid:    a0186291-16af-11eb-b941-eb9ac8040905
seqno:   -1
safe_to_bootstrap: 1
```

```
root@maria2:/# cat /var/lib/mysql/grastate.dat 
# GALERA saved state
version: 2.1
uuid:    a0186291-16af-11eb-b941-eb9ac8040905
seqno:   -1
safe_to_bootstrap: 0
```

```
# GALERA saved state
version: 2.1
uuid:    a0186291-16af-11eb-b941-eb9ac8040905
seqno:   -1
safe_to_bootstrap: 0
```

Celui que je peux boostraper est donc maria1. Je pensais devoir la passer à 1 manuellement mais j'ai l'impression que mariadb s'en est chargé.

## 6. Redémarrez les nodes et vérifiez le bon fonctionnement du cluster

Pour redémarrer les nodes : `docker-compose start`

```
Starting maria1 ... done
Starting maria2 ... done
Starting maria3 ... done
```

Rien de semble anormal. La réplication est toujours effective.

## Arborescence du projet

Comme d'habitude petite visualisation finale de l'arborescence :

```
➜  tp7 tree
.
├── config
│   ├── maria1.cnf
│   ├── maria2.cnf
│   └── maria3.cnf
├── docker-compose.yml
├── dumps
│   └── test1.sql
├── FEUGERE_Thibault_tp7.md
├── maria1/
├── maria2/
└── maria3/

11 directories, 34 files
```