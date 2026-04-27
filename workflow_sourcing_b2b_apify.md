# Workflow de sourcing B2B automatisé avec Apify + Claude Code

> Scrapez des contacts de prospection (cold email / cold call) sans passer par Apollo, Kaspr ou Hunter.
> Coût estimé : **moins de $0.15 pour 30 résultats**.

---

## Prompt de départ (à coller dans Claude Code)

```
Tu vas scraper des contacts de prospection via Apify.

Contexte : je teste un workflow de sourcing B2B automatisé depuis Claude Code,
avec l'objectif de l'enseigner à des entrepreneurs qui veulent prospecter en cold
email ou cold call sans passer par des outils payants comme Apollo ou Kaspr.

Variable d'environnement à définir avant tout :
export APIFY_TOKEN="apify_api_XXXXXXXXXXXXXXXXXXXX"

Étape 1 — Installe le CLI Apify si absent :
npm install -g apify-cli

Étape 2 — Lance un scraping Google Maps :
- Cible : "restaurant" à [VILLE], France
- Volume : 30 résultats
- Champs à récupérer : nom, adresse, site web, téléphone, note Google, nombre d'avis
- Exclure les grands groupes

Étape 3 — Une fois le run terminé, récupère les résultats et exporte-les en CSV
propre avec les colonnes : Entreprise, Adresse, Site, Telephone, Note, Nb_Avis, Email

Étape 4 — Affiche un résumé : nombre de résultats obtenus, top 5 par note,
entreprises sans site web identifiées.
```

> Remplace `APIFY_TOKEN` par ton token et `[VILLE]` par ta cible géographique.

---

## Prérequis

- [Node.js](https://nodejs.org) installé (v18+)
- Un compte [Apify](https://apify.com) gratuit → récupérez votre token dans *Settings > Integrations*
- Claude Code CLI

---

## Étape 1 — Installer le CLI Apify

```bash
npm install -g apify-cli
```

Vérifiez l'installation :

```bash
apify --version
```

> Si la commande n'est pas trouvée, ajoutez le dossier bin npm global à votre PATH :
> ```bash
> export PATH="$(npm root -g)/../bin:$PATH"
> ```

---

## Étape 2 — Définir votre token Apify

```bash
export APIFY_TOKEN="votre_token_ici"
```

---

## Étape 3 — Lancer le scraping Google Maps

Remplacez les paramètres selon votre cible :

```bash
curl -s -X POST \
  "https://api.apify.com/v2/acts/compass~crawler-google-places/runs?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "searchStringsArray": ["restaurant Paris 11ème France"],
    "maxCrawledPlacesPerSearch": 30,
    "language": "fr",
    "countryCode": "fr",
    "exportPlaceUrls": false,
    "includeHistogram": false,
    "includeOpeningHours": false,
    "includePeopleAlsoSearch": false,
    "additionalInfo": false
  }'
```

**Paramètres à adapter :**
| Paramètre | Exemple | Description |
|---|---|---|
| `searchStringsArray` | `"restaurant Paris 11ème"` | Secteur + ville + pays |
| `maxCrawledPlacesPerSearch` | `30` | Volume (max ~250 pour rester < $1) |
| `language` | `"fr"` | Langue des résultats |
| `countryCode` | `"fr"` | Code pays ISO |

La commande retourne un JSON contenant :
- `data.id` → l'ID du run
- `data.defaultDatasetId` → l'ID du dataset (résultats)

---

## Étape 4 — Surveiller le run

```bash
RUN_ID="votre_run_id"

curl -s "https://api.apify.com/v2/actor-runs/${RUN_ID}?token=$APIFY_TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin)['data']; print(d['status'])"
```

Statuts possibles : `READY` → `RUNNING` → `SUCCEEDED` / `FAILED`

---

## Étape 5 — Récupérer les résultats et exporter en CSV

```bash
DATASET_ID="votre_dataset_id"

curl -s "https://api.apify.com/v2/datasets/${DATASET_ID}/items?token=$APIFY_TOKEN&format=json&limit=50" \
| python3 -c "
import sys, json, csv

data = json.load(sys.stdin)

# Filtrer les grands groupes (optionnel)
GRANDS_GROUPES = ['mcdonald', 'burger king', 'kfc', 'subway', 'quick', 'pizza hut', 'domino', 'flunch']

def is_grand_groupe(name):
    return any(g in name.lower() for g in GRANDS_GROUPES)

rows = []
for item in data:
    if is_grand_groupe(item.get('title', '')):
        continue
    rows.append({
        'Entreprise': item.get('title', ''),
        'Adresse':    item.get('address', ''),
        'Site':       item.get('website', '') or '',
        'Telephone':  item.get('phone', '') or '',
        'Note':       item.get('totalScore', '') or '',
        'Nb_Avis':    item.get('reviewsCount', '') or '',
        'Email':      ''
    })

with open('prospects.csv', 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=['Entreprise','Adresse','Site','Telephone','Note','Nb_Avis','Email'])
    writer.writeheader()
    writer.writerows(rows)

print(f'{len(rows)} résultats exportés dans prospects.csv')
"
```

---

## Étape 6 — Analyser les résultats

```bash
python3 -c "
import csv

with open('prospects.csv', 'r', encoding='utf-8') as f:
    rows = list(csv.DictReader(f))

sans_site = [r for r in rows if not r['Site']]
scored = [(r['Entreprise'], float(r['Note']), int(r['Nb_Avis'] or 0)) for r in rows if r['Note']]
top5 = sorted(scored, key=lambda x: (-x[1], -x[2]))[:5]

print(f'Total : {len(rows)} | Avec site : {len(rows)-len(sans_site)} | Sans site : {len(sans_site)}')
print()
print('Top 5 par note :')
for name, note, avis in top5:
    print(f'  {note}/5 ({avis} avis) — {name}')
print()
print('Sans site web (opportunités) :')
for r in sans_site:
    print(f'  {r[\"Entreprise\"]} | {r[\"Telephone\"]}')
"
```

---

## Coûts Apify (Google Maps Scraper)

| Action | Prix |
|---|---|
| Par lieu scrappé | $0.004 |
| Démarrage du run | ~$0.0002 |
| **30 résultats** | **~$0.12** |
| **250 résultats** | **~$1.00** |

> Le compte gratuit Apify inclut **$5 de crédit mensuel**, suffisant pour ~1 200 résultats.

---

## Secteurs à fort potentiel pour ce workflow

- Restaurants, hôtels, hébergements
- Artisans (plombiers, électriciens, menuisiers)
- Professions libérales (avocats, comptables, kinés)
- Commerces de proximité
- Agences immobilières indépendantes

---

## Prochaines étapes possibles

- [ ] Enrichir les emails avec [Hunter.io](https://hunter.io) API (gratuit jusqu'à 25/mois)
- [ ] Automatiser l'envoi cold email avec [Instantly](https://instantly.ai) ou [Lemlist](https://lemlist.com)
- [ ] Planifier des runs hebdomadaires avec `cron` ou Claude Code `/schedule`
- [ ] Croiser avec LinkedIn Sales Navigator pour cibler les décideurs
