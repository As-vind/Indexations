# Identification Documentaire — V2.8 Pro

> Moteur local d’indexation, de recherche et de rapprochement de catalogues fournisseurs PDF.

**Identification Documentaire V2.8 Pro** est conçu pour centraliser un grand volume de catalogues fournisseurs et rendre leur contenu rapidement exploitable : références, prix HT, désignations, codes, pages et documents sources.

Le principe de la V2.8 est simple :

**indexer le maximum d’informations en amont pour obtenir ensuite des recherches rapides et fluides.**

---

## Fonctionnalités principales

### Indexation documentaire

- Parcours récursif de `data/catalogues/`
- Gestion de milliers de PDF et de sous-dossiers
- Extraction du texte natif avec **PyMuPDF**
- OCR local avec **Tesseract FR + EN** lorsque le texte du PDF est insuffisant
- Base locale **SQLite**
- Recherche plein texte **FTS5**
- Recherche approximative **RapidFuzz** en dernier recours
- Conservation du texte brut pour permettre de nouvelles analyses sans relire tous les PDF
- Indexation progressive avec commits réguliers
- Isolation des PDF/pages problématiques afin d’éviter qu’un document bloque toute l’indexation

### Informations recherchables

La base peut conserver et indexer notamment :

- désignation
- référence article
- famille / préfixe de référence
- prix HT
- dimensions
- finition
- codes secondaires
- coloris / RAL
- éco-participation ou autre montant
- catalogue
- chemin du PDF
- page
- ligne
- texte source

L’interface de résultats reste volontairement simple :

| Référence | Prix HT | Page | Ligne | Catalogue | PDF / Page |
|---|---:|---:|---:|---|---|

Les informations supplémentaires restent dans SQLite pour améliorer la recherche et préparer les fonctions futures.

---

## Recherche optimisée

La priorité est donnée aux index pré-calculés.

Ordre général de recherche :

1. **Référence exacte**
2. **Préfixe / famille de référence**
3. **Informations structurées indexées**
4. **FTS5**
5. **Fuzzy matching** en dernier recours

Exemple :

```text
COL7070
```

peut permettre de retrouver immédiatement les variantes présentes dans les catalogues :

```text
COL7070-TX
COL7070-TX0E
COL7070-T2EX
...
```

Une recherche exacte comme :

```text
COL7070-TXXX
```

doit placer cette référence en priorité.

Le moteur peut également rechercher une désignation, une finition, un code, un coloris ou une autre information préalablement indexée.

**Les PDF ne sont pas reparcourus pendant une recherche normale.**

---

## Exemple d’extraction produit

Ligne fournisseur :

```text
Chaise coque Cendrine 356 x 315 x 445 HNV MAD104 COL2684-T0E 83,35 € 0,21 €
```

Interprétation :

| Champ | Valeur |
|---|---|
| Désignation | Chaise  |
| Dimensions | 356 x 315 x 445 |
| Finition | HNV MAR |
| Référence | COL26114-XXX |
| Prix HT | 83,35 € |
| Éco./Autre | 0,21 € |

Les prix fournisseurs détectés par l’application sont traités comme des **prix HT**.

---

## Analyse d’un e-mail client

La V2.8 peut être utilisée pour rapprocher une demande client du référentiel fournisseur.

Exemple :

```text
Bonjour,

Il nous faudrait 12 chaises Cendrine en finition MAD104
et 4 modèles COL7070.

Merci.
```

L’objectif du moteur est d’identifier les éléments significatifs de la demande puis de rechercher les meilleures correspondances dans les catalogues déjà indexés.

Les résultats peuvent ensuite afficher :

- référence proposée
- prix HT
- catalogue source
- page
- ligne
- accès au PDF/page

Le mail client est analysé ponctuellement : **il ne devient pas un catalogue fournisseur et ne pollue pas le référentiel permanent.**

---

## Rapprochement de devis

Les documents placés dans :

```text
data/devis/
```

peuvent être comparés au même référentiel fournisseur.

Le principe reste identique :

```text
Document client
      ↓
Extraction
      ↓
Références / désignations / codes
      ↓
Index SQLite
      ↓
Meilleures correspondances catalogue
      ↓
PDF + page source
```

---

## Architecture

```text
Identification_Documentaire/
│
├── app.py
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── README.md
│
└── data/
    ├── catalogues/
    │   ├── fournisseur_A/
    │   ├── fournisseur_B/
    │   └── ...
    │
    ├── devis/
    ├── backups/
    └── catalogue_fts.db
```

`data/catalogues/` peut contenir autant de niveaux de sous-répertoires que nécessaire.

---

## Installation

### Prérequis

- Docker Desktop
- Docker Compose
- espace disque suffisant pour les catalogues et la base
- ressources recommandées pour le conteneur :
  - **3 Go de RAM**
  - **4 CPU**

### 1. Récupérer le projet

```bash
git clone https://github.com/As-vind/Indexations.git
cd Indexations
```

### 2. Ajouter les catalogues

Copier les PDF fournisseurs dans :

```text
data/catalogues/
```

Exemple :

```text
data/catalogues/Mobilier/LAFA V4 2026.pdf
data/catalogues/Bureaux/Fournisseur-X/catalogue.pdf
```

### 3. Construire l’image

```bash
docker compose build --no-cache
```

### 4. Démarrer l’application

```bash
docker compose up -d
```

### 5. Ouvrir l’interface

```text
http://localhost:8501
```

### Arrêt

```bash
docker compose down
```

---

## Mise à jour du référentiel

Le dossier :

```text
data/catalogues/
```

est la source documentaire principale.

Le fonctionnement recherché est :

| Situation | Action |
|---|---|
| Nouveau PDF | Indexation |
| PDF inchangé | Ignoré |
| PDF modifié/remplacé | Réindexation |
| PDF supprimé | Suppression de ses entrées |
| PDF déplacé/renommé | Mise à jour du référentiel |
| Indexation interrompue | Reprise sans recommencer inutilement |

L’objectif est de ne jamais devoir retraiter l’intégralité des catalogues lorsqu’une petite partie seulement a changé.

---

## OCR

Pour les PDF contenant peu ou pas de texte exploitable :

```text
PDF
 ↓
Extraction texte native
 ↓
Texte suffisant ?
 ├── Oui → indexation
 └── Non → OCR Tesseract
              ↓
           indexation
```

L’OCR est volontairement utilisé comme **fallback**, car il est beaucoup plus coûteux que l’extraction native.

---

## Base SQLite

SQLite constitue le cœur du référentiel local.

La base contient les informations nécessaires à la recherche ainsi que les relations permettant de revenir au document d’origine.

Le texte brut est conservé afin de pouvoir améliorer les règles d’extraction ultérieurement sans nécessairement retraiter tous les PDF.

---

## Sauvegardes

Avant une opération importante sur le référentiel, une sauvegarde de la base est recommandée.

Répertoire prévu :

```text
data/backups/
```

La base SQLite doit être sauvegardée proprement afin d’éviter de copier un fichier pendant une transaction active.

---

## Données à ne pas publier sur GitHub

Le dépôt Git doit contenir le **code de l’application**, pas les données commerciales.

Ne pas publier :

- catalogues fournisseurs confidentiels
- devis clients
- e-mails clients
- bases SQLite de production
- sauvegardes de bases
- documents commerciaux internes

Exemple de `.gitignore` :

```gitignore
data/catalogues/*
data/devis/*
data/backups/*
data/*.db
data/*.db-*
!data/catalogues/.gitkeep
!data/devis/.gitkeep
!data/backups/.gitkeep

__pycache__/
*.pyc
.env
```

---

## Philosophie V2.8

La V2.8 privilégie volontairement une **indexation plus lourde**.

```text
                    INDEXATION
                        │
        ┌───────────────┼────────────────┐
        ↓               ↓                ↓
   Texte/OCR       Structuration     Normalisation
        │               │                │
        └───────────────┼────────────────┘
                        ↓
                SQLite + FTS5
                        ↓
              Index spécialisés
                        ↓
                  RECHERCHE
                        ↓
                    RAPIDE
```

Plus les informations utiles sont préparées pendant l’indexation, moins le moteur a de calcul à effectuer lorsque l’utilisateur lance une recherche.

---

## Évolution vers V3

La V2.8 constitue le socle documentaire avant l’ajout de fonctions IA plus avancées.

Pistes prévues :

- recherche sémantique locale
- compréhension avancée des demandes clients
- rapprochement intelligent désignation ↔ produit
- analyse contextuelle des catalogues
- amélioration automatique de l’extraction
- classement intelligent des références candidates
- assistance au chiffrage
- exploitation du texte brut déjà indexé

L’objectif est de conserver un principe essentiel :

**les catalogues fournisseurs restent la source de vérité.**

---

## Technologies

- Python 3.11
- Streamlit
- SQLite / FTS5
- PyMuPDF
- Tesseract OCR
- RapidFuzz
- Pandas
- Docker / Docker Compose

---

## Auteur

**Asvind**

GitHub : `As-vind/Indexations`

---

## Version

**Identification Documentaire V2.8 Pro**

Architecture orientée indexation locale intensive et recherche documentaire rapide.

---

## Licence

Sauf ajout ultérieur d’un fichier `LICENSE`, ce projet est fourni sans licence open source.

**Tous droits réservés.**
