# Installation du module Distillerie

**Version documentée : 0.23.0** — Dolibarr 21 à 23 (production : 23.0.3), PHP 8.2 ou plus, MySQL/MariaDB.

## Prérequis

- Dolibarr 21 ou supérieur, PHP 8.2+
- Extensions PHP : `zip` (ZipArchive) et `xml`/`dom` (XMLReader) — **requises pour l'import Excel**
- Dossier `documents/` de Dolibarr accessible en écriture (PDF de distillation, fichiers temporaires d'import)
- Optionnel : compte Tuya Cloud (relevés du compteur électrique) — le module fonctionne sans

## Première installation

1. Copier le dossier `distillerie/` dans `htdocs/custom/`
2. Activer le module dans **Configuration → Modules/Applications**
3. Les tables, dictionnaires et droits sont créés automatiquement
4. Attribuer les droits (Utilisateurs & groupes → onglet Permissions, section Distillerie) : lecture/écriture par domaine (productions, distillations, registres), suppression, administration

## Mise à jour depuis une version antérieure

1. Remplacer le dossier `htdocs/custom/distillerie/` par la nouvelle version (le fichier `VERSION` à la racine indique la version installée)
2. Les changements de schéma n'exigent plus de désactiver/réactiver le module (depuis la 0.11.0) ; en revanche les **nouveaux menus** (ex. « Sites et Locaux ») n'apparaissent qu'après désactivation/réactivation du module. Ouvrir **Configuration → Distillerie → SQL Upgrade** : la page contrôle les tables et les colonnes manquantes et les corrige d'un clic
3. **Depuis la 0.14.0, désactiver puis réactiver le module après la mise à jour** : le hook d'audit des dictionnaires n'est enregistré qu'à l'activation (l'Autodiagnostic le signale s'il manque)
4. Certains correctifs sont aussi appliqués seuls à l'ouverture des pages concernées : colonne `date_line` des lignes de suivi (0.13.3), tables et colonnes Tuya (0.12.18 / 0.12.19), suivi des imports (0.12.3)
5. Vérifier **Configuration → Distillerie → Autodiagnostic** (version du fichier, tables, droits, classes, environnement PHP)

La désactivation / réactivation est sans danger (création idempotente) ; elle est nécessaire à la 0.14.0 pour les hooks, et utile si un droit ou un menu n'apparaît pas.

## Réglages PHP pour l'import Excel

L'import lit le fichier en flux (peu de mémoire) mais dépend des limites PHP :

| Réglage | Effet si trop bas |
|---|---|
| `upload_max_filesize` (défaut PHP 2 Mo) et `post_max_size` | l'envoi est refusé (message explicite depuis 0.13.1). Le module accepte 10 Mo au maximum |
| `max_input_vars` (défaut 1000) | la confirmation envoie 3 variables par production : au-delà d'environ 330 productions la page bloque l'import (sinon PHP couperait les données en silence). Relever à 5000 pour les gros fichiers |

Limites propres au module : 1 500 productions, 200 000 lignes, 5 000 colonnes par fichier ; feuille de 150 Mo décompressés maximum. Au-delà, scinder le fichier.

## Configuration de Tuya (optionnel)

1. **Configuration → Distillerie → Intégrations** : renseigner les identifiants Tuya Cloud puis utiliser « Tester la connexion »
2. **Configuration → Distillerie → Équipements Tuya** : déclarer chaque équipement et ses mesures (code DP, libellé, unité, échelle, conversion °F → °C si l'équipement renvoie des degrés Fahrenheit)
3. Les relevés sont pris à chaque ajout d'une ligne de suivi ; un échec réseau n'empêche jamais l'enregistrement de la ligne

## Vérification de l'installation

Après activation, vérifier que :

1. Le menu « Distillerie » apparaît dans le menu principal, avec Productions, Distillations, Registres, Déclarations mensuelles
2. L'onglet Configuration du module propose : Paramètres, Import productions, Autodiagnostic, Alambic, Intégrations, Équipements Tuya, Automatisations, Historique, SQL Upgrade, À propos
3. Les dictionnaires Fruits, États, Nettoyage, Broyage, Fûts et Types de registre sont accessibles via **Accueil → Configuration → Dictionnaires** (recherche « dist_dict »). Le dictionnaire des alambics se gère dans l'onglet **Alambic** du module
4. Créer une distillation, ajouter une ligne, générer le PDF : la date s'affiche sous l'heure et le pied de page indique les versions de Dolibarr et du module

## En cas d'erreur « Table doesn't exist » ou « Unknown column »

1. Ouvrir **SQL Upgrade** et appliquer les correctifs proposés
2. Sinon, désactiver puis réactiver le module
3. Si le problème persiste, consulter `documents/dolibarr.log` (le module y journalise les colonnes détectées/ajoutées et les erreurs SQL)

## Tables créées par le module

### Production et distillation
- `llx_dist_prod` — productions (fermentation/macération)
- `llx_dist_prod_step` — étapes de production (JSON)
- `llx_dist_distil` — distillations
- `llx_dist_distil_line` — lignes de suivi (heure, date `date_line`, coupe, volumes, relevés compteur)
- `llx_dist_distil_line_tuya` — relevés Tuya par ligne et par mesure
- `llx_dist_tuya_device`, `llx_dist_tuya_measure` — équipements et mesures Tuya
- `llx_dist_import_batch` — suivi des imports Excel (détection et suppression des imports interrompus)

### Registres douaniers
- `llx_dist_registry`, `llx_dist_mouvement`, `llx_dist_categorie_fiscale`, `llx_dist_stock_periode`, `llx_dist_inventaire`, `llx_dist_document_accompagnement`
- `llx_dist_site` (sites d'exploitation) et `llx_dist_local` (locaux : chai, atelier, entrepôt ; anciennement `llx_dist_chai`, renommée automatiquement en 0.17.1) — menu « Sites et Locaux »
- `llx_dist_notification_dgddi` — créée mais non encore utilisée par une interface

### Traçabilité
- `llx_dist_audit_log` — journal d'audit de toutes les écritures, y compris les dictionnaires natifs depuis la 0.14.0 (consultable dans Configuration → Historique)

### Dictionnaires
- `llx_dist_dict_fruits`, `llx_dist_dict_etats` (1-5), `llx_dist_dict_nettoyage`, `llx_dist_dict_broyage`, `llx_dist_dict_futs`, `llx_dist_dict_alambic`, `llx_dist_dict_typeregistre`

## Support

Pour toute question ou problème, consulter `ChangeLog.md` (dernières modifications) et `README.md`.
