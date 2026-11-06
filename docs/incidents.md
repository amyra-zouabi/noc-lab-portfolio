# Rapports d'incidents

Chaque incident de ce laboratoire a été provoqué volontairement ou rencontré en cours de
déploiement, puis diagnostiqué et documenté selon un format de rapport d'incident standard :
contexte, détection, diagnostic, cause racine, résolution, enseignement.

---

## INC-01 — Panne du lien WAN sur le pare-feu de bordure

**Date :** Phase 5
**Sévérité :** Critique
**Équipement concerné :** FW-HQ (VyOS, pare-feu de bordure)
**Statut :** Clos — résolu par refonte architecturale (voir [Module 2 — OOB](#module-2))

### Contexte

Simulation d'une panne du lien opérateur : désactivation de l'interface `eth0` (WAN) sur FW-HQ
pendant que la supervision Zabbix était active sur l'ensemble du parc.

**Comportement attendu :** Zabbix déclenche le trigger `FW-HQ: WAN interface down - CRITICAL`
et signale précisément l'interface en panne, tout en continuant à superviser l'équipement.

### Détection

Le trigger spécifique à l'interface WAN **ne s'est pas déclenché comme prévu**. À la place,
Zabbix a basculé l'hôte FW-HQ entier en état *Unreachable*, puis l'a marqué comme indisponible —
faisant disparaître toute visibilité sur l'équipement.

```
show interfaces
eth0    -               -       -     1500   A/D    (interface désactivée, sans adresse)
eth1    10.0.0.1/24     -       -     1500   u/u
```

Depuis le serveur de supervision :
```
ping: Network is unreachable
```

### Diagnostic

Analyse du chemin emprunté par le trafic de supervision :

1. Le serveur Zabbix (`192.168.122.200`) atteignait le lab via le réseau WAN simulé
2. Le trafic entrait par `eth0` de FW-HQ, qui le routait vers le LAN interne
3. **Le trafic de retour empruntait le même chemin**, donc dépendait de `eth0`

Quand `eth0` tombe, le chemin de retour disparaît : les réponses des équipements internes
ne peuvent plus atteindre le superviseur. Zabbix interprète cette absence de réponse comme
une indisponibilité générale de l'hôte, et non comme la panne d'une interface spécifique.

### Cause racine

**Dépendance architecturale du chemin de supervision au lien supervisé.**

La supervision était en *in-band* : elle utilisait l'infrastructure de production pour atteindre
les équipements. Conséquence directe — l'outil censé signaler une panne devient lui-même victime
de cette panne, précisément au moment où il est le plus nécessaire.

### Résolution

Construction d'un **réseau de management out-of-band** (`192.168.200.0/24`), détaillé dans le
Module 2 du README :

- Segment réseau isolé, sans route vers Internet (`forward mode='none'`)
- Interface dédiée côté superviseur (`enp7s0`) et côté routeur cœur (R-HQ `eth4`)
- Préfixe annoncé en OSPF avec `passive-interface` pour que les routeurs distants sachent
  répondre sans former d'adjacence sur le segment de management
- Route de retour statique sur FW-HQ vers le préfixe OOB

### Validation

Incident rejoué à l'identique après mise en place de l'OOB :

| Indicateur | Avant OOB | Après OOB |
|---|---|---|
| Hôtes disponibles | Dégradé (FW-HQ unreachable) | `7 Available / 0 Not available` |
| Problème remonté | Hôte entier indisponible | `FW-HQ: WAN interface down - CRITICAL` uniquement |
| Diagnostic possible | Non — perte de visibilité | Oui — cause précisément identifiée |

### Enseignement

Un système de supervision ne doit jamais dépendre du chemin qu'il surveille. Cette règle,
évidente en théorie, ne se révèle qu'au moment où la panne arrive — d'où l'intérêt de simuler
les pannes plutôt que de les attendre.

---

## INC-02 — Perte d'un routeur distant et délai de détection excessif

**Date :** Phase 5
**Sévérité :** Majeure
**Équipement concerné :** R-B1 (VyOS, routeur agence 1)
**Statut :** Clos — résolu par dimensionnement des pollers

### Contexte

Arrêt brutal du routeur R-B1 pour observer la reconvergence OSPF et mesurer le délai réel de
détection par la supervision.

### Détection

La reconvergence OSPF s'est effectuée correctement et rapidement côté réseau. En revanche,
le basculement de l'état de disponibilité dans Zabbix a pris **entre 40 et 60 minutes** —
un délai inacceptable pour un contexte NOC où l'objectif se mesure en minutes.

### Diagnostic

Investigation multi-couches pour localiser la lenteur :

**1. Vérification des processus de polling**
```bash
ps -ef | grep "unreachable poller"
```
Un seul processus de poller pour équipements injoignables était actif.

**2. Interrogation directe de la base Zabbix**
```sql
SELECT h.host, i.available, i.errors_from, i.error
FROM interface i JOIN hosts h ON h.hostid = i.hostid;
```
Confirmation que l'état `available` ne basculait pas malgré l'échec confirmé des checks.

**3. Analyse de la file d'attente**

Avec **77 items SNMP** sur l'hôte et un `Timeout` de 4 secondes par item, un balayage complet
de l'équipement injoignable monopolisait l'unique poller pendant 5 à 9 minutes. Plusieurs cycles
étant nécessaires avant le basculement, le délai total atteignait l'heure.

### Cause racine

**Sous-dimensionnement des pollers pour équipements injoignables**, combiné à un timeout par
item trop élevé. La configuration par défaut de Zabbix est calibrée pour de petits parcs
réactifs, pas pour un objectif de détection rapide sur des hôtes multi-items.

### Résolution

Ajustement dans `/etc/zabbix/zabbix_server.conf` :

| Paramètre | Avant | Après | Effet |
|---|---|---|---|
| `Timeout` | 4 | 2 | Échec constaté deux fois plus vite par item |
| `StartPollersUnreachable` | 1 | 3 | Parallélisation du balayage des hôtes injoignables |
| `UnreachablePeriod` | 45 | 30 | Fenêtre de confirmation raccourcie |
| `UnreachableDelay` | 15 | 10 | Nouvelle tentative plus rapprochée |
| `UnavailableDelay` | 60 | 30 | Reprise plus rapide après indisponibilité |

Redémarrage du service puis nouvelle mesure : **détection ramenée sous la minute**.

### Limitation connue

Le switch SW-B1 n'a jamais vu sa colonne `errors_from` renseignée malgré un échec confirmé
en ICMP et SNMP. Les pistes de maintenance planifiée, statut d'hôte, items non supportés et
saturation de file ont été écartées par requêtes SQL directes. Comportement documenté comme
limitation plutôt que poursuivi, l'effort de diagnostic devenant disproportionné au regard
de l'impact.

### Enseignement

Un trigger correctement écrit ne garantit pas un délai de détection acceptable. La chaîne
complète doit être dimensionnée : nombre de pollers, timeout par item, fenêtres de confirmation.
C'est la différence entre « la supervision fonctionne » et « la supervision tient un SLA ».

---

## INC-03 — Dégradation de lien par injection de latence

**Date :** Phase 5
**Sévérité :** Mineure (test de calibration)
**Lien concerné :** R-HQ ↔ R-B2
**Statut :** Clos — comportement nominal validé

### Contexte

Contrairement aux incidents 1 et 2 qui simulaient des pannes franches, celui-ci teste la
capacité de la supervision à détecter une **dégradation progressive de qualité** sans basculer
l'équipement en panne — cas de figure fréquent en production (lien saturé, opérateur dégradé).

### Mise en œuvre

Injection de 300 ms de latence via la policy VyOS `network-emulator`, qui s'appuie sur le module
`netem` du noyau Linux :

```
set qos policy network-emulator LATENCY-TEST delay 300
set qos policy network-emulator LATENCY-TEST bandwidth 10mbit
set qos interface eth2 egress LATENCY-TEST
```

**Deux points de syntaxe relevés :**
- `delay` attend une valeur numérique brute en millisecondes — `300ms` est rejeté, `300` accepté
- `delay` ne peut pas être appliqué seul : `bandwidth` est obligatoire dans la même policy

Vérification côté noyau (la commande VyOS `show qos interface` n'existe pas) :
```bash
tc qdisc show dev eth2
```

### Résultat observé

| Indicateur | Avant injection | Après injection |
|---|---|---|
| RTT moyen R-HQ → R-B2 | ~2 ms | ~302 ms |
| Trigger latence | Inactif | **Déclenché** |
| État de l'hôte dans Zabbix | Available | **Available** (inchangé) |

Le trigger `avg(/R-B2/icmppingsec,5m)` s'est déclenché comme attendu, et l'effet s'est
propagé en cascade sur SW-B2 (situé derrière R-B2).

**Point clé :** l'hôte est resté *Available* pendant toute la durée du test. C'est précisément
le comportement recherché — une dégradation de qualité est signalée sans être confondue avec
une panne, ce qui permet à un opérateur NOC de prioriser correctement.

### Retour à la normale

```
delete qos interface eth2 egress
delete qos policy network-emulator LATENCY-TEST
```
RTT revenu à sa valeur nominale, trigger résolu automatiquement.

### Enseignement

Une supervision bien calibrée distingue trois états, pas deux : nominal, dégradé, en panne.
Tester explicitement l'état intermédiaire évite de découvrir en production qu'une dégradation
passe inaperçue ou déclenche une fausse alerte critique.

---

## INC-04 — Timeouts SNMPv3 intermittents causés par un engine-id dupliqué

**Date :** Module 3 (migration SNMPv3)
**Sévérité :** Majeure
**Équipements concernés :** SW-HQ, SW-B1 (MikroTik CHR)
**Statut :** Clos

### Contexte

Après migration du parc en SNMPv3, SW-HQ apparaissait en timeout intermittent dans Zabbix,
alors que les autres équipements fonctionnaient normalement.

### Détection

Le symptôme était contradictoire :

```bash
time snmpwalk -v3 -u nocadmin -l authPriv -a SHA -A '***' -x AES -X '***' 10.0.99.2 .1.3.6.1.2.1.1
# Réponse complète en 0.186 s
```

Le test manuel répondait instantanément, mais Zabbix signalait
`Cannot connect to "10.0.99.2:161": Timeout`.

Analyse des logs serveur : alternance continue entre échec et récupération.
```
SNMP agent item "..." on host "SW-B1" failed: first network error, wait for 10 seconds
resuming SNMP agent checks on host "SW-B1": connection restored
```

### Diagnostic — pistes écartées

| Hypothèse | Vérification | Résultat |
|---|---|---|
| Mauvais identifiants | Comparaison champ par champ Zabbix / équipement | Identiques |
| Perte de paquets réseau | `ping -c 50` | 0 % de perte |
| Saturation CPU/RAM de l'hôte | `top`, `free -h` | 86 % CPU idle, RAM disponible |
| IP source forcée | `grep SourceIP zabbix_server.conf` | Non configuré |
| Pollers SNMP saturés | Queue overview Zabbix | File vide (0 partout) |
| Timeout trop court | Mesure du temps de réponse réel | 0,186 s — très en deçà du seuil |

### Cause racine

Comparaison de la configuration SNMP des deux switches :

```
[SW-HQ] /snmp print
         engine-id: 80003a8c04

[SW-B1] /snmp print
         engine-id: 80003a8c04     ← identique
```

**Les deux switches partageaient le même `engine-id`**, hérité du clonage des machines
virtuelles CHR depuis une image de base commune, sans régénération de l'identité système.

En SNMPv3/USM, l'engine-id constitue l'identité cryptographique de l'agent et sert de clé
pour la synchronisation temporelle (compteurs *engine boots* et *engine time*). Zabbix met
en cache cet état **par engine-id**. Deux équipements partageant le même identifiant font
entrer leurs états en conflit dans ce cache : les compteurs de l'un invalident les requêtes
vers l'autre, produisant des rejets intermittents.

Le test manuel ne révélait rien parce que chaque invocation de `snmpwalk` effectue une
découverte fraîche et indépendante, sans cache partagé.

### Résolution

Attribution d'un suffixe d'engine-id unique sur chaque switch :

```
[SW-HQ] /snmp set engine-id-suffix="hq01"
[SW-B1] /snmp set engine-id-suffix="b101"
[SW-B2] /snmp set engine-id-suffix="b201"
```

Vérification que les trois valeurs diffèrent, puis retest : disponibilité stable sur les
trois équipements.

### Enseignement

Le clonage de machines virtuelles duplique des identités supposées uniques (engine-id SNMP,
UUID machine, clés SSH hôte, adresses MAC). Ces doublons restent invisibles tant qu'aucun
protocole ne s'appuie dessus — puis provoquent des pannes dont le symptôme ne pointe jamais
vers la vraie cause.

À retenir également : un écart de comportement entre un test manuel et un outil automatisé
oriente le diagnostic vers l'état interne de l'outil (cache, session, pool de connexions),
pas vers le réseau.

---

## INC-05 — « Authentication failure » trompeuse après automatisation Ansible

**Date :** Module 4 (automatisation)
**Sévérité :** Majeure
**Équipements concernés :** les 4 routeurs VyOS
**Statut :** Clos

### Contexte

Après exécution réussie du playbook `rotate_snmpv3.yml` (rotation des identifiants SNMPv3 sur
les 7 équipements), les 4 routeurs VyOS sont passés simultanément en erreur dans Zabbix :

```
Cannot connect to "10.0.0.254:161": Authentication failure (incorrect password, community or key).
```

### Détection

Le message désignait explicitement un problème d'identifiants, mais les vérifications le
contredisaient :

- `snmpwalk -v3` manuel avec les nouveaux identifiants : **succès immédiat** sur les 4 routeurs
- Comparaison visuelle champ par champ des fiches Zabbix : identifiants, protocoles et niveau
  de sécurité strictement identiques à ceux validés en CLI
- Les 3 switches MikroTik, pourtant modifiés par le même playbook, fonctionnaient normalement

### Diagnostic

Indice déterminant trouvé dans la sortie `snmpwalk` d'un routeur :

```
iso.3.6.1.2.1.1.8.0 = Timeticks: (34) 0:00:00.34
```

Cette valeur très faible indiquait que le sous-système SNMP venait de redémarrer. Or chaque
`commit` VyOS effectué par Ansible provoque un rechargement du démon SNMP — ce qui **incrémente
le compteur *engine boots*** de l'agent.

Zabbix conservait en cache mémoire la valeur précédente de ce compteur. Les réponses des
routeurs, pourtant parfaitement authentiques, tombaient hors de la fenêtre de validation
temporelle USM et étaient rejetées. Le message d'erreur générique « Authentication failure »
ne distingue pas un mauvais mot de passe d'une désynchronisation de compteur.

Les switches MikroTik n'étaient pas affectés : le playbook y applique des commandes `/snmp
community set` qui ne redémarrent pas le service SNMP.

### Résolution

```bash
sudo systemctl restart zabbix-server
```

Le cache USM étant maintenu en mémoire, le redémarrage force une redécouverte complète des
paramètres d'engine de chaque agent. Les 7 équipements sont repassés disponibles en moins
de deux minutes.

### Enseignement

Un message d'erreur peut désigner avec assurance une cause qui n'est pas la bonne.
« Authentication failure » décrivait le symptôme observé par le client SNMP, pas la cause réelle.

Ce cas illustre aussi un effet de bord propre à l'automatisation : une opération en masse
(ici, 4 commits quasi simultanés) peut invalider l'état d'un système tiers qui n'était pas
dans le périmètre du playbook. L'automatisation accélère l'exécution — et propage donc aussi
les effets de bord plus vite.

---

## Enseignements transverses

**1. Distinguer le symptôme de la cause.**
Trois des cinq incidents présentaient un message d'erreur qui désignait une mauvaise piste.
La vérification systématique de chaque hypothèse — plutôt que l'action sur la première
intuition — a été la méthode la plus rentable.

**2. Lire le message d'erreur littéralement avant d'escalader.**
Une confusion de syntaxe (`ping <cible> count <n>` en VyOS contre `ping -c <n> <cible>` en Linux)
a provoqué une heure de diagnostic réseau inutile, alors que le message
`ping: count: Temporary failure in name resolution` nommait précisément le problème.

**3. Valider le nouveau chemin avant de supprimer l'ancien.**
Appliqué systématiquement lors des migrations (OOB, SNMPv3, rotation des identifiants) :
ajouter → tester → basculer → confirmer → seulement ensuite retirer l'ancien. Cette discipline
a évité toute perte de visibilité pendant les migrations.

**4. Un écart entre test manuel et outil automatisé oriente vers l'état interne de l'outil.**
Deux incidents (INC-04, INC-05) présentaient ce schéma. Dans les deux cas, la cause se trouvait
dans un cache applicatif, jamais dans le réseau.
