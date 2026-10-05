# Oculox Cluster OpenSearch

Ce depot deploie le cluster OpenSearch Oculox a trois noeuds conteneurises sur
une VM dediee. Tous les scripts et configurations sont strictement identiques
au depot Gitea source.

## 1. Prerequis

- VM Debian/Ubuntu, compte avec `sudo`, heure synchronisee ;
- IP stable attribuee a une interface locale ;
- profil production : minimum controle de 4 CPU, 12 Gio de RAM et 100 Gio
  libres ; 14 Gio ou plus sont recommandes pour la recette ;
- port `9200/TCP` autorise depuis Core et Collecteurs ;
- port `8404/TCP` reserve a la supervision ;
- ne jamais exposer `9300/TCP` hors du reseau Docker.

Le profil `lab` exige au moins 4 CPU, 6 Gio de RAM et 25 Gio libres, avec un
heap maximal de `1g` par noeud. Le profil `production` exige un heap minimal de
`2g` par noeud. Les minima disque sont des controles d'installation, pas un
dimensionnement de retention.

```bash
sudo apt update
sudo apt install -y git curl ca-certificates
hostname -I
timedatectl status
free -h
df -h /
nproc
```

## 2. Cloner Le Depot Cluster

```bash
git clone https://github.com/Vieux-Mbaye/oculox-cluster.git ~/oculox-cluster
cd ~/oculox-cluster
git status --short
```

La derniere commande ne doit rien afficher. Le depot est prive : configurez
l'authentification GitHub de la VM avant le clone (cle SSH ou identifiant Git
avec jeton de lecture). Ne placez jamais le jeton dans l'URL clonee.

## 3. Configurer Le Cluster

```bash
cd ~/oculox-cluster
cp dev/config/opensearch-cluster/cluster.yml.example ~/oculox-cluster.yml
nano ~/oculox-cluster.yml
```

Reglages essentiels :

| Champ | Role |
|---|---|
| `cluster.profile` | `production` pour les VM de recette |
| `cluster.name` | nom immuable du cluster |
| `cluster.heap_per_node` | tas Java de chacun des trois noeuds |
| `endpoint.ip` | IP locale stable de la VM Cluster |
| `endpoint.port` | endpoint HTTPS, normalement `9200` |
| `endpoint.monitoring_port` | supervision HAProxy, normalement `8404` |
| `storage.primary_shards` | shards principaux des nouveaux index |
| `storage.replicas` | copies, normalement `1` avec trois noeuds |
| `watermarks` | seuils d'occupation disque |
| `delete_enabled` | laisser `false` tant que la retention n'est pas validee |

Contraintes validees automatiquement : `profile` vaut `lab` ou `production`,
le heap utilise `2g` ou `2048m`, `replicas` vaut `1` ou `2`, les ports sont
distincts et `low < high < flood_stage`. `snapshots.enabled` doit rester
`false` car cette fonction n'est pas encore disponible dans ce parcours.

Verifier sans demarrer :

```bash
./oculox install cluster --config ~/oculox-cluster.yml --check
```

`--check` controle la configuration, les ressources, l'IP, les ports et le
Compose rendu. Installer ensuite :

```bash
./oculox install cluster --config ~/oculox-cluster.yml
```

Alternative laboratoire :

```bash
./oculox install cluster --endpoint-ip <IP_CLUSTER>
```

L'installation est reexecutable apres interruption et ne remplace pas une PKI
existante. Elle prepare Docker, genere la PKI OpenSearch, les comptes techniques,
HAProxy et OpenSearch Security, puis demarre les trois noeuds.

Elle attend ensuite un cluster vert, applique les politiques et cree les deux
bundles. Ne pas interrompre les redemarrages progressifs. Une trace Java
`ClosedSelectorException` apres Security Admin correspond a la fermeture de
son client HTTP ; elle est sans gravite uniquement si
`SECURITY_INITIALIZATION_RESULT=PASS` est affiche et que la commande termine.

## 4. Verifier

```bash
./oculox cluster status
./oculox cluster validate
./oculox cluster logs opensearch-1
```

Resultat attendu : trois noeuds, etat `green`, zero shard non affecte.

```bash
curl --cacert dev/generated/opensearch-cluster/pki/client-trust/oculox-opensearch-ca.crt \
  -u oculox_platform_admin \
  https://<IP_CLUSTER>:9200/_cluster/health?pretty
```

`curl` demande le mot de passe sans l'inscrire dans l'historique. Les comptes
et mots de passe techniques sont conserves dans
`dev/generated/opensearch-cluster/security/accounts.env` en mode `600`.

Le JSON doit indiquer `status: green` et `unassigned_shards: 0`. Des lignes
`401 No Authorization header` dans les logs peuvent provenir d'une sonde non
authentifiee ; la sante des conteneurs et la requete authentifiee font foi.

## 5. Utiliser Les Bundles Crees Par L'installation

L'installation du Cluster cree automatiquement les deux bundles dans :

```bash
dev/generated/opensearch-cluster/client-bundles/core
dev/generated/opensearch-cluster/client-bundles/hedgehog
```

- `core` contient les comptes necessaires a Logstash, Dashboards, Arkime, API ;
- `hedgehog` contient uniquement les acces necessaires au Collecteur ;
- les bundles contiennent des secrets et restent hors Git.

Verifier les bundles automatiques avant leur transfert :

```bash
(cd dev/generated/opensearch-cluster/client-bundles/core && sha256sum -c SHA256SUMS)
(cd dev/generated/opensearch-cluster/client-bundles/hedgehog && sha256sum -c SHA256SUMS)
```

La commande `client-bundle` est optionnelle. Elle sert seulement a reexporter
un bundle vers un autre emplacement, ou a le recreer si le bundle automatique
a ete supprime. Le repertoire de sortie ne doit pas deja exister :

```bash
mkdir -p ~/oculox-bundles
./oculox cluster client-bundle core ~/oculox-bundles/core
./oculox cluster client-bundle hedgehog ~/oculox-bundles/hedgehog
```

## 6. Configurer OIDC Apres Keycloak

Apres `./oculox keycloak provision` sur le Core, recevoir sa CA publique dans
`/tmp/oculox-web-ca.crt`, puis executer :

```bash
./oculox cluster configure-oidc \
  --keycloak-auth-url https://<IP_CORE_OU_DNS>/keycloak \
  --realm oculox \
  --keycloak-ca /tmp/oculox-web-ca.crt
./oculox cluster validate
```

Arguments :

- `--keycloak-auth-url` : URL HTTPS publique de Keycloak, sans URL HTTP ;
- `--realm` : realm humain Oculox, normalement `oculox` ;
- `--keycloak-ca` : CA publique qui permet aux noeuds de verifier le Core.

L'argument optionnel `--client-id` vaut par defaut `oculox-dashboards`. La
commande peut recreer progressivement les noeuds pour installer la CA, puis
applique OIDC. Attendre le message final avant d'activer Dashboards sur le Core.

La methode Basic de secours OpenSearch reste presente pendant l'activation
OIDC. Une mauvaise configuration Keycloak ne doit donc pas supprimer le chemin
d'administration technique.

## 7. Exploitation

```bash
./oculox cluster start
./oculox cluster stop
./oculox cluster restart
./oculox cluster status
./oculox cluster logs [service]
./oculox cluster validate
./oculox cluster config
```

Pour appliquer heap, watermarks ou politiques sans changer l'endpoint :

```bash
./oculox cluster apply --config ~/oculox-cluster.yml
```

Ne jamais utiliser `docker compose down -v` : les volumes contiennent les
index. Sauvegarder hors de la VM les bundles et secrets selon la politique de
l'organisation.

Apres un redemarrage de VM :

```bash
cd ~/oculox-cluster
./oculox cluster start
./oculox cluster status
./oculox cluster validate
```

`cluster apply` accepte heap, watermarks et politiques, mais refuse de changer
le nom ou l'endpoint d'un cluster existant. Ces identites exigent une migration
planifiee.

## 8. Depannage Et Recette Finale

| Symptome | Cause probable | Action |
|---|---|---|
| `cluster status` echoue | mauvais repertoire | `cd ~/oculox-cluster` |
| IP refusee | IP absente de la VM | verifier `ip -br address` |
| port occupe | service sur 9200/8404 | verifier avec `ss -ltnp` |
| cluster jaune/rouge persistant | noeud, disque ou replicas | consulter logs et watermarks |
| checksum bundle invalide | copie incomplete | recopier le bundle automatique |
| OIDC refuse Keycloak | CA ou URL incorrecte | verifier certificat, dates et URL |

Le Cluster est accepte lorsque le precontrole passe, les quatre conteneurs sont
sains, l'API retourne `green` avec zero shard non affecte, `cluster validate`
passe, les deux bundles sont intacts et un redemarrage complet revient au vert
sans perte d'index. Ne jamais utiliser `docker volume prune` sur cette VM.

Documentation complementaire :

- `dev/docs/Opensearch/guide_installation_vm_neuves.md` ;
- `dev/scripts/opensearch-cluster/README.md` ;
- `dev/compose/opensearch-cluster/README.md` ;
- `docs/UPSTREAM_README.md`.
