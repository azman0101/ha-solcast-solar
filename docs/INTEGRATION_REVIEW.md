# Revue de l’intégration et reprise du fork

Date : 3 novembre 2026. Responsable du fork : [azman0101](https://github.com/azman0101).

## Périmètre et conclusion

La base examinée est le commit amont
[`685b0834041507320bf37b42a931a93c7f9ab7e7`](https://github.com/BJReplay/ha-solcast-solar/commit/685b0834041507320bf37b42a931a93c7f9ab7e7),
daté du 1er novembre 2026, repris sans divergence dans le fork au début de la revue.

Il s’agit d’une **intégration personnalisée Home Assistant**, de domaine
`solcast_solar`, avec flux de configuration, capteurs, actions, prévisions du
tableau Énergie, stockage local et appels au service cloud Solcast.
Ce n’est pas un module complémentaire Supervisor.

La structure permet de poursuivre la maintenance du fork sans changer le domaine
ou les identifiants existants. Cette PR règle l’identité et la distribution du
fork. Les défauts fonctionnels ci-dessous restent ouverts : cette revue ne vaut
pas validation d’une nouvelle release.

La lecture a porté sur le chargement/déchargement, la reconfiguration,
l’ordonnancement, les appels HTTP et quotas, les caches et migrations,
les diagnostics, les entités et le tableau Énergie, les métadonnées et la CI.
Les algorithmes de prévision/dampening n’ont pas fait l’objet d’une validation
numérique exhaustive.

## Problèmes de reprise corrigés dans cette PR

| Constat initial | Changement |
| --- | --- |
| Responsables de code et liens de réparation/support dirigés vers l’amont | `.github/CODEOWNERS`, manifeste et `MANUFACTURER` dirigés vers azman0101 ; suppression du `CODEOWNERS` racine redondant |
| README annonçant une installation par le catalogue par défaut de l’amont | Installation du fork comme dépôt personnalisé HACS et procédure de remplacement avec sauvegarde |
| `zip_release: true` alors que le fork n’a aucune release | Installation depuis les sources, sans exiger un ZIP inexistant |
| Badges et modèle de notes de release représentant l’amont | Badges utiles et modèle de release dirigés vers le fork ; retrait des badges de release/téléchargement sans release du fork |
| Attribution implicite et licence absente du dossier effectivement distribué | Notice de provenance, licence Apache 2.0 inchangée, copies de la licence et de la notice dans `custom_components/solcast_solar` pour HACS et le ZIP |
| Politique de sécurité héritée annonçant des versions 4.4.x | Politique du fork et consignes de signalement sans secrets |

Le nom d’intégration, le domaine, les identifiants d’entités, les schémas de
configuration et les formats de caches ne changent pas. Le champ fabricant
affiché dans les informations de l’appareil devient `azman0101`.
Le nom du dépôt dans HACS porte le suffixe `(azman0101)`.
La version du manifeste reste la base amont `4.6.3` ; aucune release n’est créée
par cette PR.

Les auteurs, le changelog historique et ses liens numérotés restent attribués
à l’amont. Les liens Sponsors continuent de soutenir BJReplay et autoSteve.
Le fichier MIT de `tests/aioresponses` est conservé.

## Constats fonctionnels ouverts

### P2 — Un changement d’affichage supprime le fichier de dampening automatique

Source :
[`async_update_options`, lignes 555–562](../custom_components/solcast_solar/__init__.py#L555).

La suppression du fichier vérifie uniquement l’ancienne valeur vraie de
`AUTO_DAMPEN`, sans vérifier que l’utilisateur vient de désactiver cette option.
Elle peut donc se produire lors d’un simple changement d’attribut affiché.

Reproduction : conserver le dampening automatique actif et un fichier de facteurs
existant ; modifier uniquement `attr_brk_hourly` (`BRK_HOURLY`). Le callback
supprime le fichier alors que `AUTO_DAMPEN` est toujours vrai.
Cette suppression a été reproduite en exécutant la fonction réelle extraite par
AST, avec un fichier temporaire et un coordinateur simulé.
L’impact établi est la suppression du fichier persistant ; les facteurs en
mémoire peuvent encore exister jusqu’au prochain rechargement/recalcul.

Correction recommandée : conditionner la suppression à la transition
`True → False`. Ajouter des tests HA pour un changement d’affichage sans
suppression, une désactivation avec suppression et un rechargement avec
conservation des facteurs. **Non corrigé dans cette PR.**

### P2 — L’annulation d’une requête HTTP est absorbée

Source :
[`Fetcher.fetch_data`, lignes 846–854](../custom_components/solcast_solar/fetcher.py#L846).

Le gestionnaire de `CancelledError` journalise l’annulation puis retourne
`None`. L’appelant reçoit un résultat ordinaire au lieu de l’annulation attendue
pendant un déchargement ou un rechargement.

Reproduction : faire lever `asyncio.CancelledError` par
`aiohttp_session.get` durant `fetch_data` ; l’appel retourne `None`.
Ce comportement a été reproduit sur la méthode réelle extraite par AST avec une
session simulée. Le maintien de tâches après un déchargement complet reste à
vérifier dans HA.

Correction recommandée : propager l’annulation après le nettoyage/journalisation
et vérifier le comportement des appelants. Ajouter un test pendant une requête
en cours et pendant le délai de retry, ainsi qu’un test de déchargement complet.
**Non corrigé dans cette PR.**

### P2 — Les caches sont réécrits en place

Sources :
[`SitesCache.serialise_data`, ligne 739](../custom_components/solcast_solar/sites_cache.py#L739),
[`serialise_usage`, ligne 780](../custom_components/solcast_solar/sites_cache.py#L780)
et les écritures du cache de sites dans ce même module.

L’ouverture en mode `w` tronque le fichier avant la fin de l’écriture.
Le verrou sérialise les écrivains, mais ne protège pas d’une interruption du
processus, d’une coupure ou d’une erreur disque. Un JSON partiel peut ainsi
remplacer le dernier cache valide. La gestion existante des sauvegardes et des
caches invalides atténue le risque sans rendre l’écriture atomique.

Constat par lecture du code ; aucune coupure réelle de Home Assistant n’a été
simulée. Correction recommandée : fichier temporaire dans le même dossier,
écriture complète puis remplacement atomique, avec nettoyage en cas d’erreur ;
adapter toutes les écritures et tester une interruption avant remplacement.
**Non corrigé dans cette PR.**

## Points de vigilance

- Les issues du fork sont désactivées au moment de la revue. Le propriétaire doit
  activer **Settings → General → Features → Issues** pour rendre utilisables les
  liens de support et `issue_tracker` de cette PR. Le connecteur disponible ne
  permet pas de modifier les paramètres du dépôt ; ce réglage reste à effectuer.
- `solcast_solar.get_options` retourne volontairement les clés API en clair
  ([`actions.py`, ligne 547](../custom_components/solcast_solar/actions.py#L547)).
  Les réponses et les éventuelles traces d’automatisation doivent être traitées
  comme sensibles. Ce constat n’établit pas un accès non authentifié.
  Un masquage par défaut devra faire l’objet d’un changement de contrat documenté.
- Les caches de sites contiennent les clés API ; en configuration multi-clé,
  les noms de certains fichiers les contiennent aussi. Ne pas partager ces caches.
  Les diagnostics utilisent une redaction ciblée sur `api_key`, qui ne constitue
  pas une garantie globale pour tous les logs et réponses d’actions.
- `ssl_verify` est vrai par défaut, et la session HTTP est fournie par HA.
  L’option avancée permettant de désactiver la vérification doit rester réservée
  à un environnement de développement maîtrisé.
- Le minimum HACS `2025.3` est hérité et reste **non vérifié** ici. La CI teste
  le Core actuel, sa branche de développement et éventuellement sa RC, pas cette
  borne minimale. Ajouter une vérification dédiée avant de promettre sa prise en
  charge ; ne pas augmenter arbitrairement cette borne sans essai.
- Le simulateur hors distribution utilise une syntaxe d’exceptions de Python
  3.14 dans `entities.py` et `guidance.py`. Sa compilation sous Python 3.13
  échoue ; la CI existante prévoit Python 3.14.
- Les déclarations `done` de `quality_scale.yaml` et l’objectif de couverture
  à 100 % sont hérités. Ils ne prouvent ni une certification Home Assistant du
  fork ni une couverture mesurée pendant cette revue.

## Validation et limites

Effectué localement :

- compilation Python 3.13 de `custom_components`, `tests` et
  `tests_solcast_sim` ;
- lecture JSON et YAML des métadonnées, traductions et workflows ;
- vérification de l’identité du fork, des références documentaires locales et des
  licences copiées ;
- contrôle du diff et de l’absence de modification des identifiants et schémas ;
- reproductions isolées des deux premiers défauts, sans requête Solcast réelle.

Les reproductions avec dépendances simulées ne remplacent pas des tests dans HA.
`python -m pytest`, `ruff check` et `ruff format --check` n’ont pas pu
s’exécuter : pytest, Home Assistant et Ruff ne sont pas installés dans cet
environnement. Le proxy réseau du shell est inaccessible, ce qui empêche d’y
installer ces dépendances ; l’accès GitHub a été effectué par le connecteur.
Aucun résultat de couverture du fork n’est donc revendiqué.

Les workflows existants pytest, hassfest et HACS sont conservés. Leur état sur la
branche de PR doit être consulté dans GitHub ; aucune exécution n’était présente
dans le fork au début de la revue.

Avant une première release, vérifier les contrôles CI et effectuer dans une
instance HA de test : installation HACS depuis ce fork, remplacement de l’amont,
rechargement/déchargement, réauthentification, quotas, changement de jour et DST,
diagnostics expurgés et alimentation du tableau Énergie. La compatibilité de
migration est attendue de la conservation du domaine et des schémas, mais n’a
pas été testée sur une instance réelle dans cette PR.
