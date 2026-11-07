# Module Distillerie — Dolibarr

Module externe Dolibarr (v21+, PHP 8.2+) dédié à la gestion de la production d'alcool artisanale (rhum, eau-de-vie, etc.) avec traçabilité complète des étapes de production, suivi des distillations, gestion des registres douaniers et import de données historiques.

---

## Fonctionnalités principales

### 🏭 Productions
- Création et gestion des lots de production (`TESTxxx`, `BATCHxxxxxx`)
- Suivi des étapes par lot avec données en JSON :
  - RECOLTE, NETTOYAGE, BROYAGE, DÉFÉCATION
  - FERMENTATION/MACÉRATION DEBUT, FERMENTATION/MACÉRATION FIN
  - DISTILLATION, DECHETS, RECYCLAGE, PREPARATION
- Étapes ordonnées par rang (10, 20, 30…) selon le processus de production
- Import en masse depuis fichier Excel (format Traçabilité) : prévisualisation, étapes détectées, **contrôle des valeurs hors norme** (erreurs / avertissements, « Importer quand même »), modes Ignorer / Mettre à jour
- Pagination de la liste des productions

### 🔥 Distillations
- Gestion des distillations liées aux productions
- Lignes de suivi (Tête, Cœur, Queue : volumes ml et TAV %)
- Capture automatique du compteur électrique atelier (Tuya Cloud : puissance, énergie, température) à chaque ligne et à la reprise : le compteur est un équipement Tuya comme les autres — voir `admin/tuya_devices.php` (identifiants du compte dans `admin/integration.php`)
- Dashboard distillation avec graphiques (Chart.js)
- **PDF du journal de distillation** : date par ligne, colonnes de relevés Tuya, paysage automatique, en-tête répété

### 📋 Registres
- Gestion des registres douaniers officiels
- Champs complets : identification entrepôt fiscal, SIRET, caution, adresses
- Informations réglementaires (RÈGLES 01, 05, 09, 10 DTI+)
- Type de registre lié au dictionnaire `dist_dict_typeregistre`
- Mouvements associés aux registres

### 🏢 Sites et Locaux
- Rubrique de menu « **Sites et Locaux** » (au niveau de « Registres ») : sites d'exploitation rattachés à un registre, avec commune, siège de la comptabilité matières, autorisation DGDDI
- **Locaux** d'un site : chai, atelier, entrepôt ou autre ; rôle principal / secondaire ; superficie et capacité
- **Entrepôt** : régime douanier **sous douane** (accises non acquittées) ou **hors douane** (accises acquittées) ; lien facultatif vers un **entrepôt Dolibarr** (module Stock)
- Écritures auditées ; suppression d'un site refusée tant qu'il contient des locaux
- **Mouvements rattachés aux locaux** (0.18.0) : local d'origine / de destination, régime douanier figé à la validation, contrôle de cohérence (avertissements, mise à la consommation signalée)
- **Stock par local** (0.19.0) : stock comptable en litres et en LAP à partir des mouvements validés rattachés, situation à une date, taux d'occupation vs capacité
- **Inventaires par local** (0.19.1) : arrêté annuel et inventaires ponctuels, comptage physique, écart avec le stock comptable, justification obligatoire, validation qui fige, mouvements de régularisation en brouillon, PDF
- **Stock Dolibarr** (0.19.1) : comparaison en lecture seule avec l'entrepôt Dolibarr lié (estimation en litres)
- **Alertes de capacité par e-mail** (0.19.1) : une alerte par franchissement du seuil, tâche planifiée
- **Détail par produit** (0.20.0) : stock et inventaire ventilés par catégorie fiscale et nature de produit
- **Régime douanier** (0.22.0) : règles bloquantes configurables, mise à la consommation dans la déclaration mensuelle
- **Non-apurement** (0.21.0) : documents d'accompagnement DAE/DAA/DSA, apurement, relevé mensuel et PDF
- **Rapprochement Dolibarr** (0.20.0) : propositions de mouvements de registre depuis les mouvements de stock Dolibarr (brouillons à relire), correspondance produits, transferts internes, anti-doublon
- *À venir* : blocages issus du régime douanier, détail par produit, rapprochement actif avec le stock Dolibarr (voir EVOLUTIONS.md #35, #36, #31)

### 📊 Dashboards
- Dashboard global (productions + distillations + registres)
- Dashboard productions avec graphiques par fruit et évolution mensuelle
- Dashboard distillations avec statistiques TAV
- Dashboard registres avec mouvements

### 📤 Déclarations mensuelles
- Génération PDF des déclarations récapitulatives mensuelles (DRM)
- Conforme RÈGLE 06 CGI / DTI+ v1.0.23

---

## Installation

1. Copier le dossier `distillerie/` dans `htdocs/custom/`
2. Activer le module dans **Configuration → Modules/Applications**
3. À l'activation : toutes les tables et données par défaut sont créées automatiquement
4. À la désactivation : les tables dictionnaires sont supprimées

> ⚠️ Si vous mettez à jour depuis une version < 0.7.3, exécuter le script `sql/migrate_typeregistre_0_7_3.sql` pour migrer le champ `typeregistre` de INTEGER vers VARCHAR(64).

---

## Structure des fichiers

```
distillerie/
├── admin/
│   ├── about.php               — Page À propos
│   ├── alambic.php             — Gestion alambics
│   ├── automation.php          — Automatisations
│   ├── dict_broyage.php        — Dictionnaire broyage
│   ├── dict_etats.php          — Dictionnaire états
│   ├── dict_fruits.php         — Dictionnaire fruits
│   ├── dict_futs.php           — Dictionnaire fûts
│   ├── dict_nettoyage.php      — Dictionnaire nettoyage
│   ├── autodiag.php            — Autodiagnostic runtime du module
│   ├── import_productions.php  — Import masse depuis Excel
│   ├── integration.php         — Intégrations externes
│   ├── sql_upgrade.php         — Diagnostic et correction du schéma SQL
│   └── setup.php               — Configuration module
├── class/
│   ├── distaudit.class.php            — Journalisation audit
│   ├── distautomation.class.php       — Automatisations
│   ├── distcategoriefiscale.class.php — Catégories fiscales
│   ├── distdeclaration.class.php      — Déclarations mensuelles
│   ├── distdicttyperegistre.class.php — Dictionnaire type registre
│   ├── distdistil.class.php           — Distillations
│   ├── distdistilline.class.php       — Lignes de distillation
│   ├── distexportexcel.class.php      — Export Excel
│   ├── distimport.class.php           — Import productions Excel
│   ├── distintegration.class.php      — Intégrations
│   ├── distmouvement.class.php        — Mouvements registre
│   ├── distprod.class.php             — Productions
│   ├── distprodstep.class.php         — Étapes de production
│   └── distregistry.class.php         — Registres
├── core/modules/
│   ├── modDistillerie.class.php       — Descripteur module
│   └── distillerie/doc/
│       └── pdf_declaration_mensuelle.modules.php
├── includes/
│   ├── SimpleXLSX.php          — Librairie lecture XLSX autonome
│   └── SimpleXLSXWriter.php    — Librairie écriture XLSX autonome
├── langs/
│   ├── fr_FR/distillerie.lang  — Traductions français
│   └── en_US/distillerie.lang  — Traductions anglais
├── lib/
│   └── distillerie.lib.php     — Fonctions partagées + définition étapes
├── sql/
│   ├── llx_dist_prod.sql
│   ├── llx_dist_prod_step.sql
│   ├── llx_dist_distil.sql
│   ├── llx_dist_distil_line.sql
│   ├── llx_dist_registry.sql
│   ├── llx_dist_mouvement.sql
│   ├── llx_dist_categorie_fiscale.sql
│   ├── llx_dist_dict_fruits.sql
│   ├── llx_dist_dict_etats.sql
│   ├── llx_dist_dict_nettoyage.sql
│   ├── llx_dist_dict_broyage.sql
│   ├── llx_dist_dict_futs.sql
│   ├── llx_dist_dict_alambic.sql
│   └── llx_dist_dict_typeregistre.sql
├── distdashboard.php
├── distprod_list.php
├── distprod_card.php
├── distprod_dashboard.php
├── distdistil_list.php
├── distdistil_card.php
├── distdistil_dashboard.php
├── distregistry_list.php
├── distregistry_card.php
├── distreg_dashboard.php
├── distmouvement_list.php
├── distmouvement_card.php
├── distdeclaration_generate.php
├── production_step.php
├── production_step_edit.php
└── test/
    └── agent_controle.php       — Agent de contrôle qualité statique (CLI)
```

---

## Contrôle qualité (agent statique)

```bash
php test/agent_controle.php [chemin_module]
```

18 familles de contrôles : structure fichiers, descripteur ModuleBuilder (IDs, plage BARNECKI 250000-250019, dictionnaires), bootstrap `main.inc.php`, sécurité (CSRF/GETPOST/escape), pagination, pièges TCPDF, inclusions, redirections POST, fichiers `.lang`, classes `CommonObject`, pattern SQL sécurisé, cohérence dictionnaires (`init()`/`remove()`), piège `information_schema` (`DATABASE()` vs `database_name`), tableaux imbriqués dans `multiselectarray()`, boutons d'action avec libellé traduit, migration de colonnes ajoutées après installation, anti-régression noms de colonnes.

Score cible : **100%**, à exécuter avant chaque nouvelle version.

Complémentaire : `admin/autodiag.php` (Configuration → Autodiagnostic) pour les contrôles runtime en conditions réelles (tables, droits, classes chargées), et `admin/sql_upgrade.php` (Configuration → SQL Upgrade) pour vérifier et corriger en un clic l'état du schéma SQL en production, sans désactiver/réactiver le module.

---

## Base de données

### Tables principales

| Table | Description |
|-------|-------------|
| `llx_dist_prod` | Productions (ref, date_prod, type_prod, fk_fruit, quantite_totale, lieu_recolte) |
| `llx_dist_prod_step` | Étapes production (fk_prod, step_code, step_label, rang, date_event, json_data) |
| `llx_dist_distil` | Distillations (fk_prod, date_distillation) |
| `llx_dist_distil_line` | Lignes distillation (fk_distil, cut_type, volume_ml, tav_pct) |
| `llx_dist_registry` | Registres (label, typeregistre VARCHAR(64), numero_entrepot, siret…) |
| `llx_dist_mouvement` | Mouvements registre |
| `llx_dist_categorie_fiscale` | Catégories fiscales |

### Dictionnaires (créés à l'activation, supprimés à la désactivation)

| Table | Description | Valeurs par défaut |
|-------|-------------|-------------------|
| `llx_dist_dict_fruits` | Fruits utilisés | — |
| `llx_dist_dict_etats` | États qualité 1-5 | — |
| `llx_dist_dict_nettoyage` | Types nettoyage | RINCAGE, RINCAGE_EPLUCHAGE, VINAIGRE_BLANC_10, VINAIGRE_BLANC_6 |
| `llx_dist_dict_broyage` | Types broyage | — |
| `llx_dist_dict_futs` | Fûts | — |
| `llx_dist_dict_alambic` | Alambics | — |
| `llx_dist_dict_typeregistre` | Types de registres | PRODUCTION, DISTILLATION, STOCK, MOUVEMENTS, PERTES, COMPTE_MATIERES |

---

## Permissions

| Permission | ID | Description |
|-----------|----|-------------|
| read | 25000401 | Lecture globale |
| write | 25000402 | Écriture globale |
| delete | 25000403 | Suppression globale |
| distillation.read | 25000411 | Lecture distillations |
| distillation.write | 25000412 | Écriture distillations |
| production.read | 25000421 | Lecture productions |
| production.write | 25000422 | Écriture productions |
| registre.read | 25000431 | Lecture registres |
| registre.write | 25000432 | Écriture registres |
| admin | 25000499 | Administration module |

> IDs au pattern `$this->numero . sprintf("%02d", ...)` — module ID officiel **250004** (plage BARNECKI 250000-250019).

---

## Menus

```
Distillerie (top menu)
├── Dashboard
├── Productions
│   ├── Liste des productions
│   └── Nouvelle production
├── Distillations
│   ├── Liste des distillations
│   └── Nouvelle distillation
├── Registres
│   ├── Liste des registres
│   └── Nouveau registre
├── Déclarations mensuelles
└── Configuration
    ├── Paramètres (FormSetup : entrepôt fiscal, unités, taux)
    ├── Import productions
    ├── Autodiagnostic (contrôles runtime)
    ├── SQL Upgrade (diagnostic/correction schéma SQL)
    └── À propos
```

---

## Import Excel (Traçabilité)

Format attendu : fichier `.xlsx` avec :
- **Ligne 2** col A = Étape, col B = Champ, col C+ = Noms des productions
- Noms productions : format `TESTxxx` ou `BATCHxxxxxxxx`
- Librairie embarquée : `includes/SimpleXLSX.php` (sans dépendance Composer/PhpSpreadsheet)
- Étapes créées avec rang automatique (RECOLTE=10, NETTOYAGE=20, BROYAGE=30…)
- Fruits inconnus créés automatiquement dans `dist_dict_fruits`
- Toutes les données d'étapes stockées en JSON dans `json_data`
- Références déjà existantes : ignorées (défaut) ou mises à jour (mode UPDATE, lignes cochées ; production brouillon, distillation encore « Importée »)

---

## Librairies incluses

| Librairie | Usage | Chemin |
|-----------|-------|--------|
| SimpleXLSX | Lecture fichiers .xlsx (ZipArchive natif PHP) | `includes/SimpleXLSX.php` |
| SimpleXLSXWriter | Écriture fichiers .xlsx (ZipArchive natif PHP, styles de base) | `includes/SimpleXLSXWriter.php` |
| Chart.js v3.9.1 | Graphiques dashboards | CDN |

---

## Versions

Voir [ChangeLog.md](ChangeLog.md) pour l'historique complet.

**Version actuelle : 0.22.0**
