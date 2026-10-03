# Oculox Cluster OpenSearch

Ce depot deploie le cluster OpenSearch Oculox a trois noeuds conteneurises sur
une VM dediee. Tous les scripts et configurations sont strictement identiques
au depot Gitea source.

## 1. Prerequis

- VM Debian/Ubuntu, compte avec `sudo`, heure synchronisee ;
- IP stable attribuee a une interface locale ;
- profil production : au moins 4 CPU, 14 Gio de RAM et 200 Gio libres ;
- port `9200/TCP` autorise depuis Core et Collecteurs ;
- port `8404/TCP` reserve a la supervision ;
- ne jamais exposer `9300/TCP` hors du reseau Docker.

## 2. Cloner Le Depot Cluster

```bash
git clone <URL_DEPOT_OCULOX_CLUSTER> ~/oculox-cluster
cd ~/oculox-cluster
git status --short
```

La derniere commande ne doit rien afficher. Utilisez la meme version ou le meme
tag Oculox sur les trois VM.

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

## 5. Creer Les Bundles

```bash
mkdir -p ~/oculox-bundles
./oculox cluster client-bundle core ~/oculox-bundles/core
./oculox cluster client-bundle hedgehog ~/oculox-bundles/hedgehog
(cd ~/oculox-bundles/core && sha256sum -c SHA256SUMS)
(cd ~/oculox-bundles/hedgehog && sha256sum -c SHA256SUMS)
```

- `core` contient les comptes necessaires a Logstash, Dashboards, Arkime, API ;
- `hedgehog` contient uniquement les acces necessaires au Collecteur ;
- les bundles contiennent des secrets et restent hors Git.

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

Documentation complementaire :

- `dev/docs/Opensearch/guide_installation_vm_neuves.md` ;
- `dev/scripts/opensearch-cluster/README.md` ;
- `dev/compose/opensearch-cluster/README.md` ;
- `docs/UPSTREAM_README.md`.
