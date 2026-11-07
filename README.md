# Module Distillerie — Dolibarr

Module externe Dolibarr (v21 à 23, PHP 8.2+, MySQL/MariaDB) dédié à une **distillerie artisanale** (rhum, eau-de-vie…) : traçabilité des productions et distillations, **comptabilité matières** conforme au CGI (registres, mouvements, déclarations mensuelles, non-apurement, inventaires), gestion des **sites et locaux** avec leur régime douanier, et rapprochement avec le stock Dolibarr.

Production : Dolibarr 23.0.3 — BARNECKI SPIRITS. Module ID `250004` (plage 250000-250019).

---

## Sommaire

1. [Fonctionnalités](#fonctionnalités)
2. [Installation et mise à jour](#installation-et-mise-à-jour)
3. [Menus](#menus)
4. [Configuration](#configuration)
5. [Permissions](#permissions)
6. [Structure des fichiers](#structure-des-fichiers)
7. [Base de données](#base-de-données)
8. [Tâches planifiées](#tâches-planifiées)
9. [Import Excel (Traçabilité)](#import-excel-traçabilité)
10. [Tests et contrôle qualité](#tests-et-contrôle-qualité)
11. [Librairies incluses](#librairies-incluses)
12. [Documentation et versions](#documentation-et-versions)

---

## Fonctionnalités

### 🏭 Productions
- Lots de production (`TESTxxx`, `BATCHxxxxxx`) avec fruit, quantité, lieu de récolte.
- **Étapes** ordonnées par rang (10, 20, 30…), données en JSON : RECOLTE, NETTOYAGE, BROYAGE, DÉFÉCATION, FERMENTATION/MACÉRATION DÉBUT et FIN, DISTILLATION, DECHETS, RECYCLAGE, PREPARATION.
- **Import en masse depuis Excel** (format Traçabilité) : prévisualisation, étapes détectées, **contrôle des valeurs hors norme** (erreurs / avertissements, « Importer quand même »), modes Ignorer / Mettre à jour, suivi par lot d'import (signalement et suppression d'un import interrompu).
- Liste paginée, dictionnaires (fruits, états qualité, nettoyage, broyage, fûts).

### 🔥 Distillations
- Distillations liées aux productions : alambic (dictionnaire), volume et TAV de départ, relevés EDF.
- **Lignes de suivi** : Tête / Cœur / Queue (volume ml, TAV %), température, thermostat.
- **Compteur électrique et équipements Tuya Cloud** : puissance, énergie, température capturées à chaque ligne et à la reprise (équipements et mesures configurables, voir Configuration > Équipements Tuya).
- **Dashboard** distillations (Chart.js) et **PDF du journal de distillation** (paysage automatique, colonnes Tuya, en-tête répété), archivé dans les documents du module.

### 📋 Registres et comptabilité matières (CGI 50-00)
- **Registres** douaniers : identification de l'entrepôt fiscal, SIRET, caution, adresses, clôture d'exercice, taux de pertes / déchets, type de registre (dictionnaire).
- **Mouvements** (production, transformation, entrée, sortie, détention, correction) : numérotation **séquentielle inviolable** et horodatée (RÈGLE 04), ventilation par catégorie fiscale / nature de produit, volume et alcool pur, document d'accompagnement (DAE, DSA, DRM, DRA), brouillon puis **validation qui fige**, audit de chaque écriture.
- **Déclarations mensuelles** (DRM, RÈGLE 06 / DTI+) : stocks début / fin de période, mouvements du mois, **mise à la consommation** (sous douane → hors douane), export PDF et Excel.
- **Documents d'accompagnement et relevé de non-apurement** (0.21.0, RÈGLE 07) : suivi des DAE / DAA / DSA émis, apurement et désapurement, délai configurable, relevé mensuel (écran + PDF), import des DAE XML dans le registre choisi (0.21.1).
- Catégories fiscales, exports DTI+ et FEC ; table de notifications DGDDI (taux de pertes / déchets) prévue au schéma, sans écran dédié à ce jour.

### 🏢 Sites et Locaux (RÈGLE 02)
- **Sites d'exploitation** rattachés à un registre (commune, siège de la comptabilité matières, autorisation DGDDI) ; suppression refusée tant qu'un site contient des locaux.
- **Locaux** d'un site : chai, atelier, entrepôt ou autre ; rôle principal / secondaire ; superficie et capacité. L'**entrepôt** porte un régime douanier **sous douane** ou **hors douane** et peut être lié à un **entrepôt Dolibarr**.
- **Mouvements rattachés aux locaux** (0.18.0) : local d'origine / destination, régime figé à la validation, contrôles de cohérence en avertissements ; **règles du régime douanier rendues bloquantes** à la demande (0.22.0).
- **Stock par local** (0.19.0) : stock comptable en litres et alcool pur à une date, ventilé par catégorie fiscale et nature de produit (0.20.0), taux d'occupation vs capacité.
- **Stock Dolibarr** (0.19.1) : comparaison en **lecture seule** avec l'entrepôt Dolibarr lié (estimation en litres).
- **Rapprochement actif Dolibarr** (0.20.0) : propositions de mouvements de registre (brouillons à relire) à partir des mouvements de stock Dolibarr, correspondance produit → catégorie fiscale / nature / TAV, transferts internes, anti-doublon, mouvements ignorés restaurables.
- **Alertes de capacité par e-mail** (0.19.1) : une alerte par franchissement du seuil, réarmée au retour sous le seuil.

### 📦 Inventaires (RÈGLE 09, CGI 50-00 G)
- Types **annuel** (un par registre et par année), **ponctuel** et **ouverture** (0.23.0, un par registre).
- Stock théorique figé à la date d'inventaire, **comptage physique par local et par produit** (volume + TAV → alcool pur), écart en L, en alcool pur et en %, **justification obligatoire** au-delà d'une tolérance configurable (sauf ouverture), suggestions (évaporation, fuite, saisie).
- Date limite de l'arrêté (viticole : 10 septembre ; sinon le 10 du deuxième mois après la clôture), validation qui fige, **mouvements de régularisation en brouillon**, **PDF** avec bloc de signatures.
- **PDF signé archivé** (0.23.0) : dépôt sur un inventaire validé, nom horodaté, ni remplacement ni suppression.
- **Relances e-mail** (0.23.0) : échéance à 30 jours puis hebdomadaire en retard, un e-mail regroupé ; création automatique de l'arrêté annuel 2 mois avant l'échéance.
- **Inventaire d'ouverture assisté** (0.23.0) : bilan (compté, locaux vides, stock Dolibarr non rattaché) ; les excédents deviennent des mouvements « détention ».

### 📊 Dashboards
- Global (productions + distillations + registres), productions (par fruit, évolution mensuelle), distillations (statistiques TAV), registres (mouvements).

### 🔧 Administration et conformité
- **Journal d'audit** (RÈGLE 04) consultable (Configuration > Historique) : création, modification, suppression, validation, exports, changements de configuration.
- **Intégrations** : import DAE XML, export DTI+ (DRM) et FEC, test de connexion Tuya.
- **Automatisations** : calcul des stocks mensuels, alertes d'échéance, génération des inventaires annuels, alertes de capacité, relances d'inventaires, tolérance d'inventaire, délai d'apurement, règles du régime bloquantes.
- **Autodiagnostic** (contrôles runtime) et **SQL Upgrade** (diagnostic et correction du schéma sans désactiver le module).
- Hook `dictionaryadmin` : audit des dictionnaires natifs modifiés par l'administrateur.

---

## Installation et mise à jour

Voir aussi [INSTALL.md](INSTALL.md).

1. Prérequis : Dolibarr 21+, PHP 8.2+, extensions `zip` et `xml`/`dom`, dossier `documents/` accessible en écriture.
2. Copier le dossier `distillerie/` dans `htdocs/custom/`.
3. Activer le module dans **Configuration → Modules/Applications** : les tables, dictionnaires et données par défaut sont créés. Les tâches planifiées sont créées **désactivées**.
4. **Mise à jour** : remplacer le dossier, puis ouvrir **Configuration → Distillerie → SQL Upgrade** (ou désactiver/réactiver le module). Les tables et colonnes ajoutées après coup (locaux, inventaires, non-apurement, rapprochement, relances…) sont aussi **créées automatiquement à la première utilisation** de la page concernée.
5. Si la mise à jour part d'une version < 0.7.3 : exécuter `sql/migrate_typeregistre_0_7_3.sql` (champ `typeregistre` INTEGER → VARCHAR(64)).
6. Si le module était déjà activé avant l'ajout d'une tâche planifiée : désactiver/réactiver le module ou créer la tâche à la main (le descripteur ne recrée pas les tâches existantes).

> À la désactivation, les tables dictionnaires sont supprimées ; les tables métier sont conservées.

---

## Menus

Menu principal **Distillerie** (droit `read`). Menu de gauche :

```
Distillerie
├── Tableau de bord                          (production.read)
├── Productions                              (production.read)  → dashboard productions
│   ├── Liste Productions
│   └── Nouvelle production                  (production.write)
├── Distillations                            (distillation.read) → dashboard distillations
│   ├── Liste des distillations
│   └── Nouvelle distillation                (distillation.write)
├── Registres                                (registre.read)    → dashboard registres
│   ├── Liste des registres
│   ├── Nouveau registre                     (registre.write)
│   ├── Déclarations mensuelles              → DRM, mise à la consommation, relevé de non-apurement
│   └── Documents d’accompagnement           → DAE / DAA / DSA, apurement
├── Sites et Locaux                          (registre.read)
│   ├── Sites d’exploitation (+ Nouveau site)
│   ├── Locaux (+ Nouveau local)
│   ├── Stock par local
│   ├── Rapprochement stock Dolibarr
│   └── Inventaires                          → annuel / ponctuel / ouverture
└── Configuration                            (admin)
    └── À propos
```

Les **mouvements** s'ouvrent depuis la fiche d'un registre (boutons « Liste des mouvements » et « Nouveau mouvement »).

Onglets de **Configuration** (`admin/setup.php`) : Paramètres · Import productions · Autodiagnostic · Alambics · Intégrations · Équipements Tuya · Automatisations · Correspondance produits (rapprochement) · Historique (audit) · SQL Upgrade · Champs complémentaires · À propos.

---

## Configuration

Constantes enregistrées par l'écran de configuration :

| Constante | Écran | Rôle |
|-----------|-------|------|
| `DISTILLERIE_NUMERO_ENTREPOT`, `DISTILLERIE_SIRET`, `DISTILLERIE_RAISON_SOCIALE` | Paramètres | Identification de l'entrepôt fiscal |
| `DISTILLERIE_UNITE_VOLUME`, `DISTILLERIE_TAUX_PERTES_DEFAUT`, `DISTILLERIE_IMPORT_AUTO_FRUITS` | Paramètres | Production et import |
| `DISTILLERIE_TUYA_CLIENT_ID`, `_CLIENT_SECRET`, `_ENDPOINT`, `_DEVICE_ID` | Intégrations | Compte Tuya Cloud (le module fonctionne sans) |
| `DISTILLERIE_ALERT_CAPACITE_SEUIL`, `DISTILLERIE_ALERT_CAPACITE_EMAILS` | Automatisations | Seuil (100 % par défaut) et destinataires des alertes de capacité |
| `DISTILLERIE_ALERT_INVENTAIRE_EMAILS` | Automatisations | Destinataires des relances d'inventaire (à défaut : ceux des alertes de capacité) |
| `DISTILLERIE_INVENTAIRE_TOLERANCE_PCT` | Automatisations | Tolérance d'écart d'inventaire (1 % par défaut) |
| `DISTILLERIE_APUREMENT_DELAI_JOURS` | Automatisations | Délai avant « en retard » d'un document non apuré (30 j par défaut) |
| `DISTILLERIE_REGIME_BLOCK` | Automatisations | Règles du régime douanier rendues bloquantes (aucune par défaut) |

---

## Permissions

| Permission | ID | Description |
|-----------|----|-------------|
| read | 25000401 | Lecture globale |
| write | 25000402 | Écriture globale |
| delete | 25000403 | Suppression globale |
| distillation.read / write | 25000411 / 25000412 | Distillations |
| production.read / write | 25000421 / 25000422 | Productions |
| registre.read / write | 25000431 / 25000432 | Registres, mouvements, sites, locaux, stock, inventaires, documents d'accompagnement |
| admin | 25000499 | Administration du module |

IDs au pattern `$this->numero . sprintf("%02d", ...)`. Des droits dédiés Sites / Locaux sont envisagés (EVOLUTIONS #37).

---

## Structure des fichiers

```
distillerie/
├── admin/                       — Configuration (onglets)
│   ├── setup.php                — Paramètres (FormSetup)
│   ├── import_productions.php   — Import masse depuis Excel
│   ├── autodiag.php             — Autodiagnostic runtime
│   ├── alambic.php              — Dictionnaire alambics
│   ├── dict_*.php               — Dictionnaires fruits, états, nettoyage, broyage, fûts
│   ├── integration.php          — Import DAE XML, export DTI+/FEC, Tuya
│   ├── tuya_devices.php         — Équipements et mesures Tuya
│   ├── automation.php           — Automatisations, alertes, relances, tolérance, régime
│   ├── product_map.php          — Correspondance produits Dolibarr → registre
│   ├── historique.php           — Journal d'audit
│   ├── sql_upgrade.php          — Diagnostic et correction du schéma SQL
│   └── about.php
├── class/
│   ├── Objets métier            — distprod, distprodstep, distdistil, distdistilline, distregistry,
│   │                              distmouvement, distsite, distlocal, distinventaire,
│   │                              distcategoriefiscale, distdicttyperegistre
│   ├── Comptabilité matières    — diststock (stock par local), distdeclaration (DRM, mise à la
│   │                              consommation), distdocaccomp (DAE/DAA/DSA, relevé)
│   ├── Rapprochement Dolibarr   — distdolistock (lecture seule), distrecon (propositions)
│   ├── Alertes et relances      — distautomation, distlocalalert, distinventairerelance
│   ├── Import / export          — distimport, distexportexcel, distintegration, tuyaclient
│   ├── Audit                    — distaudit, distauditedtrait
│   └── actions_distillerie.class.php — hook dictionaryadmin
├── core/modules/
│   ├── modDistillerie.class.php — Descripteur (menus, droits, dictionnaire, tâches planifiées)
│   └── distillerie/doc/pdf_declaration_mensuelle.modules.php
├── lib/
│   ├── distillerie.lib.php      — Fonctions partagées, onglets admin, migrations de schéma (ensure_*)
│   ├── distilpdf.lib.php        — PDF journal de distillation
│   ├── inventairepdf.lib.php    — PDF d'inventaire
│   ├── inventairedocs.lib.php   — Archivage du PDF signé
│   └── relevepdf.lib.php        — PDF relevé de non-apurement
├── sql/                         — 1 fichier par table (llx_dist_*.sql), clés, migrations historiques
├── langs/fr_FR, langs/en_US     — distillerie.lang
├── includes/                    — SimpleXLSX.php, SimpleXLSXWriter.php
├── img/                         — Pictogrammes
├── doc/                         — Conformité ModuleBuilder, import, plan, règles CGI, tests dashboards
├── build/makepack-distillerie.conf
├── test/
│   ├── agent_controle.php       — Agent de contrôle statique (CLI)
│   └── harness/                 — Harnais d'exécution (run.php, tests/t_01 … t_22, SQLite)
├── Pages : distdashboard, distprod_(list|card|dashboard), distdistil_(list|card|dashboard),
│           distregistry_(list|card), distreg_dashboard, distmouvement_(list|card),
│           distdeclaration_generate, distdocaccomp_list, distsite_(list|card),
│           distlocal_(list|card|stock), diststock_recon, distinventaire_(list|card),
│           production_step(_edit), distillerieindex, check_tables
└── README.md · INSTALL.md · ChangeLog.md · EVOLUTIONS.md · CLAUDE.md · VERSION
```

---

## Base de données

Préfixe `llx_`, colonne `entity` partout. Les colonnes ajoutées après coup sont placées en fin de table ; les tables récentes sont créées ou migrées automatiquement (`distillerie_ensure_*`).

### Production et distillation

| Table | Description |
|-------|-------------|
| `dist_prod` | Productions (ref, date_prod, type_prod, fk_fruit, quantite_totale, lieu_recolte, fk_import_batch) |
| `dist_prod_step` | Étapes de production (fk_prod, step_code, rang, date_event, json_data) |
| `dist_distil` | Distillations (fk_prod, date_start, TAV / volume de départ, fk_alambic, EDF) |
| `dist_distil_line` | Lignes de suivi (cut_type, volume_ml, tav_pct, puissance, énergie, température) |
| `dist_distil_line_tuya` | Relevés des équipements Tuya rattachés à une ligne |
| `dist_tuya_device` / `dist_tuya_measure` | Équipements Tuya du compte et mesures suivies |
| `dist_import_batch` | Suivi des imports Excel |

### Registres et comptabilité matières

| Table | Description |
|-------|-------------|
| `dist_registry` | Registres (type, entrepôt, SIRET, caution, clôture, taux pertes / déchets) |
| `dist_mouvement` | Mouvements (séquence, nature, document, catégorie fiscale, volume, alcool pur, local origine / destination, régimes figés) |
| `dist_categorie_fiscale` | Catégories fiscales |
| `dist_stock_periode` | Stocks début / fin par mois, catégorie et nature (déclaration mensuelle) |
| `dist_document_accompagnement` | DAE / DAA / DSA émis, apurement, source (manuel / import / mouvement) |
| `dist_notification_dgddi` | Notifications DGDDI (taux de pertes / déchets) |
| `dist_audit_log` | Journal d'audit (table, enregistrement, action, anciennes / nouvelles valeurs, IP) |

### Sites, locaux, stock et inventaires

| Table | Description |
|-------|-------------|
| `dist_site` | Sites d'exploitation (fk_registry, commune, autorisation DGDDI) |
| `dist_local` | Locaux (fk_site, catégorie, régime douanier, fk_entrepot Dolibarr, capacité) |
| `dist_local_alert` | État des alertes de capacité (unique par local et type) |
| `dist_product_map` | Produit Dolibarr → catégorie fiscale / nature / TAV |
| `dist_stock_link` | Mouvements de stock Dolibarr rapprochés ou ignorés → mouvement de registre |
| `dist_inventaire` | En-tête d'inventaire (type, dates, totaux, validation) |
| `dist_inventaire_line` | Lignes par local, catégorie fiscale et nature (théorique, physique, écart, justification, mouvement de régularisation) |
| `dist_inventaire_relance` | Relances envoyées (une par inventaire et niveau) |

### Dictionnaires (créés à l'activation, supprimés à la désactivation)

| Table | Description |
|-------|-------------|
| `dist_dict_fruits`, `dist_dict_etats`, `dist_dict_nettoyage`, `dist_dict_broyage`, `dist_dict_futs`, `dist_dict_alambic` | Fruits, états qualité 1-5, nettoyages (RINCAGE, RINCAGE_EPLUCHAGE, VINAIGRE_BLANC_10, VINAIGRE_BLANC_6), broyages, fûts, alambics |
| `dist_dict_typeregistre` | Types de registres (PRODUCTION, DISTILLATION, STOCK, MOUVEMENTS, PERTES, COMPTE_MATIERES) |

---

## Tâches planifiées

Créées **désactivées** à l'activation du module (Accueil > Configuration > Travaux planifiés) :

| Tâche | Méthode | Rôle |
|-------|---------|------|
| Distillerie : alertes de capacité des locaux | `DistAutomation::cronCapacityAlerts` | E-mail au franchissement du seuil de capacité d'un local |
| Distillerie : relance des inventaires en retard | `DistAutomation::cronInventoryReminders` | E-mail 30 jours avant l'échéance puis chaque semaine en retard |

Les deux disposent d'un bouton « Contrôler maintenant » dans Configuration > Automatisations.

---

## Import Excel (Traçabilité)

Fichier `.xlsx` : **ligne 2** col A = Étape, col B = Champ, col C+ = noms des productions (`TESTxxx` ou `BATCHxxxxxxxx`). Lecture par `includes/SimpleXLSX.php` (sans Composer). Étapes créées avec rang automatique (RECOLTE=10, NETTOYAGE=20, BROYAGE=30…), fruits inconnus ajoutés au dictionnaire, données d'étape en JSON. Références existantes : ignorées (défaut) ou mises à jour (lignes cochées ; production brouillon, distillation encore « Importée »). Détail : `doc/IMPORT_PRODUCTIONS.md`.

---

## Tests et contrôle qualité

```bash
php test/agent_controle.php [chemin_module]   # contrôle statique (CLI)
php test/harness/run.php [filtre]             # harnais d'exécution
```

- **Agent de contrôle** : structure, descripteur ModuleBuilder, bootstrap `main.inc.php`, sécurité (CSRF / GETPOST / échappement), pagination, pièges TCPDF, fichiers `.lang`, classes `CommonObject`, SQL, dictionnaires, migrations de colonnes. À exécuter avant chaque version.
- **Harnais** : exécute le vrai code du module contre des doublures minimales de Dolibarr et une base SQLite créée depuis `sql/llx_*.sql` ; 22 fichiers `tests/t_NN_*.php` (règles dures, langues, versions, schéma, audit, CSRF, XLSX, PDF, import, sites et locaux, mouvements, stock, inventaires, alertes, rapprochement, non-apurement, régime, phase D…). Chaque nouveau test est validé par une mutation volontaire.
- Complémentaires en production : `admin/autodiag.php` (contrôles runtime) et `admin/sql_upgrade.php` (schéma SQL).
- Limites : SQLite n'est pas MySQL ; rien n'est testé contre un vrai Dolibarr, Tuya ou serveur de messagerie. Chaque livraison se termine par une liste « À valider sur ton instance ».

---

## Librairies incluses

| Librairie | Usage | Chemin |
|-----------|-------|--------|
| SimpleXLSX | Lecture .xlsx (ZipArchive natif) | `includes/SimpleXLSX.php` |
| SimpleXLSXWriter | Écriture .xlsx (ZipArchive natif) | `includes/SimpleXLSXWriter.php` |
| Chart.js v3.9.1 | Graphiques des dashboards | CDN |
| TCPDF | PDF (fourni par Dolibarr) | — |

---

## Documentation et versions

- [ChangeLog.md](ChangeLog.md) — historique complet
- [EVOLUTIONS.md](EVOLUTIONS.md) — état des lieux, feuille de route par phases, points à valider avec la douane
- [INSTALL.md](INSTALL.md) — installation et mise à jour
- [CLAUDE.md](CLAUDE.md) — consignes de développement, règles dures, pièges
- `doc/` — conformité ModuleBuilder, règles CGI, plan d'implémentation, import, tests dashboards

**Version actuelle : 0.23.0**
