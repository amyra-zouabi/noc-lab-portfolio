# NetBox — Déploiement et modélisation

Déploiement de NetBox (IPAM / DCIM) en conteneurs sur le serveur de supervision, et modélisation
complète de l'infrastructure du laboratoire.

---

## Pourquoi NetBox dans ce projet

Avant NetBox, la documentation du plan d'adressage était répartie entre plusieurs sources :
notes, inventaire Ansible, fiches Zabbix. Gérable sur 7 équipements, intenable à l'échelle d'un
réseau d'entreprise.

NetBox centralise cette information dans une base structurée et interrogeable :

- Taux d'occupation par sous-réseau, calculé automatiquement
- Détection des conflits d'adressage avant qu'ils ne surviennent
- Relation préfixe ↔ adresses IP reconstituée sans saisie manuelle
- Source de vérité exploitable par d'autres outils (inventaire dynamique Ansible via API)

---

## Prérequis

- Docker et Docker Compose installés
- Environ 2 Go de RAM disponibles
- Port `8000` libre sur l'hôte

Dans ce laboratoire, NetBox est hébergé sur la VM de supervision (déjà raccordée au réseau OOB),
dont l'allocation mémoire a été portée de 1,9 à 4 Go pour l'occasion :

```bash
sudo virsh shutdown zabbix-server
sudo virsh setmaxmem zabbix-server 4G --config
sudo virsh setmem zabbix-server 4G --config
sudo virsh start zabbix-server
```

---

## Déploiement

### 1. Installer Docker

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 git
sudo usermod -aG docker $USER
```

Déconnexion / reconnexion requise pour que l'appartenance au groupe `docker` prenne effet.

### 2. Cloner le dépôt officiel

```bash
cd ~
git clone -b release https://github.com/netbox-community/netbox-docker.git
cd netbox-docker
```

La branche `release` correspond à la dernière version stable testée par le projet.

### 3. Fixer le port d'écoute

Par défaut, `netbox-docker` expose un port aléatoire. Surcharge pour fixer le port `8000` :

```bash
cat > docker-compose.override.yml << 'EOF'
services:
  netbox:
    ports:
      - "8000:8080"
EOF
```

### 4. Démarrer

```bash
docker compose pull
docker compose up -d
```

> **Premier démarrage :** l'initialisation de la base PostgreSQL (plusieurs centaines de
> migrations Django) prend 5 à 10 minutes. Le conteneur peut apparaître `unhealthy` pendant
> cette période — c'est normal, les migrations se poursuivent et reprennent où elles s'étaient
> arrêtées à chaque redémarrage.

Suivre la progression :
```bash
docker compose logs -f netbox
```

Le service est prêt lorsque les logs affichent `Listening at: http://0.0.0.0:8080`.

### 5. Créer le compte administrateur

Aucun compte n'est créé automatiquement — il faut l'initialiser :

```bash
docker compose exec netbox /opt/netbox/netbox/manage.py createsuperuser
```

Renseigner nom d'utilisateur, adresse e-mail et mot de passe.

### 6. Accéder à l'interface

```
http://<IP_DU_SERVEUR>:8000
```

---

## Modélisation réalisée

La hiérarchie NetBox se construit du général vers le particulier. L'ordre de création compte :
chaque niveau dépend du précédent.

### Organisation

| Objet | Valeurs |
|---|---|
| Région | `NOC-LAB` |
| Sites | `HQ` (siège), `B1` (agence 1), `B2` (agence 2) |

### Types d'équipement

| Objet | Valeurs |
|---|---|
| Fabricants | `VyOS`, `MikroTik` |
| Modèles | `VyOS Rolling`, `CHR` |
| Rôles | `Router`, `Firewall`, `Switch` |

### Équipements

| Nom | Rôle | Modèle | Site |
|---|---|---|---|
| FW-HQ | Firewall | VyOS Rolling | HQ |
| R-HQ | Router | VyOS Rolling | HQ |
| SW-HQ | Switch | CHR | HQ |
| R-B1 | Router | VyOS Rolling | B1 |
| SW-B1 | Switch | CHR | B1 |
| R-B2 | Router | VyOS Rolling | B2 |
| SW-B2 | Switch | CHR | B2 |

### Interfaces et adressage

Chaque équipement porte ses interfaces réelles, avec les adresses IP assignées :

| Équipement | Interfaces |
|---|---|
| FW-HQ | `eth0` (192.168.122.60/24), `eth1` (10.0.0.1/24) |
| R-HQ | `eth0` (10.0.0.254/24), `eth1` (10.1.1.1/30), `eth2` (10.2.1.1/30), `eth3` (10.0.99.1/24), `eth4` (192.168.200.1/24) |
| SW-HQ | `ether1` (10.0.99.2/24) |
| R-B1 | `eth0` (10.1.1.2/30), `eth1` (10.1.0.1/24) |
| SW-B1 | `ether1` (10.1.0.2/24) |
| R-B2 | `eth0` (10.2.1.2/30), `eth1` (10.2.0.1/24) |
| SW-B2 | `ether1` (10.2.0.2/24) |

### Préfixes IPAM

Rôles créés : `LAN`, `WAN`, `OOB-Management`, `Interconnexion`.

| Préfixe | Rôle | Site |
|---|---|---|
| `10.0.0.0/8` | — | — (agrégat parent) |
| `192.168.122.0/24` | WAN | HQ |
| `192.168.200.0/24` | OOB-Management | HQ |
| `10.0.0.0/24` | LAN | HQ |
| `10.0.99.0/24` | LAN | HQ |
| `10.1.1.0/30` | Interconnexion | B1 |
| `10.1.0.0/24` | LAN | B1 |
| `10.2.1.0/30` | Interconnexion | B2 |
| `10.2.0.0/24` | LAN | B2 |

> **Note de modélisation :** la case « Est une plage d'adresses » (`is_pool`) a été laissée
> décochée pour tous les préfixes. Cochée, elle indiquerait que l'adresse réseau et l'adresse
> de diffusion sont utilisables — ce qui serait incorrect pour des sous-réseaux IP classiques.

---

## Intégration API

Un token API a été généré (**admin → API tokens**) avec droits d'écriture, en vue d'une
intégration NetBox ↔ Ansible en **inventaire dynamique** : Ansible interrogerait NetBox pour
obtenir la liste des équipements à piloter, au lieu de s'appuyer sur un fichier `inventory.yml`
maintenu à la main.

Le token est stocké chiffré dans le coffre Ansible (`group_vars/all/vault.yml`) sous la
variable `vault_netbox_token`.

---

## Exploitation courante

### Arrêter / démarrer

```bash
cd ~/netbox-docker
docker compose stop      # arrêt
docker compose up -d     # démarrage
docker compose ps        # état des conteneurs
```

### Consulter les logs

```bash
docker compose logs --tail=50 netbox
```

### Sauvegarder la base

```bash
docker compose exec postgres pg_dump -U netbox netbox > netbox-backup-$(date +%F).sql
```

---

## Limitation connue

La création de **câbles** entre interfaces via l'interface web n'a pas pu être finalisée : le
menu déroulant de sélection d'interface du formulaire câble ne se peuple pas dans cette version,
alors que les interfaces existent bien sur les fiches d'équipement. Contournement possible via
l'API REST de NetBox, non mis en œuvre — la documentation des câbles n'étant pas critique pour
les objectifs du laboratoire.
