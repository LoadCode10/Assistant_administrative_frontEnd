# Procédures — application unique, deux tableaux de bord

Frontend statique (HTML + CSS + JavaScript, sans framework ni build). **Une
seule application** : un `index.html`, un routeur, une coque, un `api.js`. Ce
qu'elle affiche dépend du rôle de la personne connectée.

| Rôle | Ce qu'il voit |
|---|---|
| `admin` | la console : Tableau de bord, Documents, Procédures, Pièces requises, Administrations |
| `citizen` | l'espace citoyen : Assistant, Mes procédures |

La barre latérale, les routes atteignables et le sous-titre de l'en-tête sont
tous dérivés du rôle courant. Viser une route qui n'appartient pas à son rôle
renvoie sur l'accueil de ce rôle (`#/dashboard` ou `#/chat`) — jamais sur un
écran à moitié monté.

## Démarrer

Ouvrez `index.html` directement dans un navigateur, ou servez le dossier :

```bash
python -m http.server 5500
```

L'URL du backend est dans **une seule constante** :

```js
// js/config.js
API_BASE_URL: 'http://localhost:8000'
```

Le même fichier contient `USE_MOCK`. Passé à `true`, l'interface tourne sur des
données factices en mémoire (documents, extraction en cours qui se termine au
bout de 12 s, import direct) sans aucun backend — pratique pour vérifier l'UI.

Côté FastAPI, autorisez l'origine du frontend :

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5500", "null"],  # "null" = fichier ouvert en file://
    allow_methods=["*"], allow_headers=["*"],
)
```

### Authentification

L'application s'ouvre sur un écran de connexion (`#/login`) ; on crée un compte
citoyen depuis `#/register`. Les comptes administrateurs se créent hors de
l'application — l'inscription ne propose pas de rôle.

Le jeton est un **cookie HttpOnly posé par le serveur**. Le frontend ne le lit
pas, ne le stocke pas et ne l'attache pas : c'est le navigateur qui le renvoie,
parce que toutes les requêtes partent avec `credentials: 'include'`. Ce choix est
fait à un seul endroit, dans `request()` (`js/api.js`) — aucun appel ne le
repasse. Rien n'est écrit dans `localStorage`.

La seule source de vérité sur la session est **`GET /citizen/me`**, appelé une
fois au démarrage, avant la première résolution de route ; la réponse est gardée
en mémoire par `App.auth`. Un `401` signifie « pas de session ». Le temps de cet
aller-retour, la page n'affiche qu'un état d'attente (classe `is-booting` sur
`<body>`) : monter l'écran de connexion « en attendant » le ferait clignoter
chez les personnes déjà connectées.

Trois règles, toutes tenues par le routeur (`js/router.js`) :

- sans session, tout hash autre que `#/login` et `#/register` renvoie à
  `#/login` ;
- avec une session, `#/login` et `#/register` renvoient à l'accueil du rôle ;
- la restriction par rôle reste celle d'avant : un citoyen n'atteint pas les
  routes admin, et l'inverse.

**Expiration.** Un `401` sur n'importe quelle requête vide la session en mémoire
et renvoie à `#/login` avec un message. C'est traité une seule fois, dans
`request()`, et non écran par écran. Trois appels y échappent volontairement
(`allowUnauthorized`) : la vérification de session au démarrage, la connexion et
la déconnexion — un `401` y est une réponse normale, pas un accident.

**Déconnexion.** `POST /auth/logout` : un cookie HttpOnly ne peut pas être effacé
par le JavaScript de la page, seul le serveur peut le faire. La session en
mémoire n'est vidée qu'une fois qu'il a répondu.

Côté FastAPI, l'authentification par cookie impose `allow_credentials=True`
**et** une liste d'origines explicite :

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5500"],   # plus de "*" ni de "null"
    allow_credentials=True,
    allow_methods=["*"], allow_headers=["*"],
)
```

Les deux vont ensemble. Avec `credentials: 'include'`, le navigateur **rejette**
toute réponse dont l'en-tête `Access-Control-Allow-Origin` vaut `*` : si le
backend n'est pas configuré ainsi, plus aucun appel n'aboutit et l'interface
affiche « Impossible de joindre le serveur » partout, alors que le backend
répond correctement. `"null"` (page ouverte en `file://`) cesse aussi de
fonctionner : servez le dossier.

### Se connecter en mode démonstration

`USE_MOCK: true` simule l'authentification. **Divergence assumée** : rien ne peut
poser de cookie en mémoire, `js/mock.js` retient donc simplement quel compte est
connecté — un rechargement de page déconnecte, contrairement au flux réel. Deux
comptes sont amorcés, un par rôle, pour relire les deux tableaux de bord :

| Nom d'utilisateur | Mot de passe | Rôle |
|---|---|---|
| `admin` | `demo1234` | console d'administration |
| `citoyen` | `demo1234` | espace citoyen |

Les écrans citoyens s'appuient par ailleurs sur des points d'entrée **qui
n'existent pas encore côté FastAPI** (voir plus bas) : pour les relire
aujourd'hui, passez `USE_MOCK` à `true`.

## Fichiers

```
index.html              structure + sprite d'icônes SVG inline
styles.css              tous les styles (jetons de design en haut du fichier)
js/config.js            URL du backend, intervalle de sondage, tailles limites
js/helpers.js           échappement, dates, toasts, modales, zone de dépôt
js/api.js               appels fetch + normalisation des réponses
js/charts.js            graphiques SVG inline (aire, barres, calendrier, jauges)
js/auth.js              session courante en mémoire (rôles, accueil, expiration)
js/router.js            routeur par hash, session et rôle par route, garde « modifications non enregistrées »
js/shell.js             coque : en-tête (logo, « Bienvenue »), barre latérale par rôle
js/mock.js              backend factice (actif seulement si USE_MOCK)
js/screens/auth-ui.js   en-tête de marque partagé par les deux écrans ci-dessous
js/screens/login.js     connexion (#/login) — carte centrée, sans coque
js/screens/register.js  inscription (#/register) — citoyens uniquement
js/screens/dashboard.js écran 0 — tableau de bord (indicateurs, graphiques, calendrier)
js/screens/documents.js écran 1 — liste + sondage
js/screens/procedures.js écran 2 — procédures, pièces et étapes (3 vues)
js/screens/editor.js    écran 4 — éditeur JSON (accordéon, pagination)
js/screens/chat.js      citoyen — assistant (sources, suivi d'un clic ; l'historique
                        est rendu par la coque, alimenté par cet écran)
js/screens/suivi.js     citoyen — procédures suivies (avancement, pièces, étapes)
js/modals/upload-document.js  écran 2
js/modals/import-json.js      écran 3 — validation client avant envoi
js/modals/approve.js          écran 5 — confirmation + statistiques
js/modals/logout.js           confirmation de déconnexion (barre latérale)
js/app.js               routes et démarrage
```

Les routes sont déclarées dans `js/app.js`, chacune avec le rôle qui peut
l'atteindre — ou avec `{ guest: true }` pour les deux écrans d'authentification,
seuls atteignables sans session et seuls rendus sans la coque.

**Scripts classiques plutôt que modules ES.** Les modules ES sont bloqués par la
politique CORS des navigateurs quand la page est ouverte en `file://` ; comme
l'ouverture directe faisait partie des contraintes, chaque fichier expose son
module sur l'espace de noms `window.App` et `index.html` les charge dans l'ordre.
Passer aux `import`/`export` ne demande que d'ajouter `type="module"` et les
lignes d'export, si vous servez toujours la page par HTTP.

## Contrat d'API attendu

| Appel | Utilisation |
| --- | --- |
| `POST /auth/register` | `{ nom_user, prenom_user, userName, email_user, phone_user, password }` → `201 { message, id_user }`, `409` si l'identifiant ou le courriel est pris |
| `POST /auth/login` | `{ userName, password }` → `200 { user }` + cookie HttpOnly `access_token`, `401 { detail }` sinon |
| `POST /auth/logout` | `204`, efface le cookie |
| `GET /citizen/me` | l'utilisateur connecté (`role` = `admin` ou `citizen`), `401` sans cookie valide |
| `GET /admin/documents` | liste des documents et imports |
| `POST /admin/documents` | multipart : `file`, `titre`, `url_source` |
| `POST /admin/imports` | multipart : `file`, `source` → renvoie l'id d'extraction |
| `GET /admin/extractions/{id}` | le JSON des procédures |
| `PUT /admin/extractions/{id}` | corps = **tableau JSON** des procédures éditées |
| `POST /admin/extractions/{id}/approve` | écriture en base + indexation |

Les réponses sont normalisées dans `js/api.js`, avec une tolérance volontaire sur
les noms de champs, pour éviter de retoucher les écrans si le backend diffère :

- **Liste** : un tableau, ou `{ documents | items | results: [...] }`.
- **Statut** : `published` / `pending_review` / `extracting` / `failed` et leurs
  synonymes courants (`approved`, `needs_review`, `processing`, `error`, …) —
  table `STATUS_MAP` dans `js/api.js`.
- **Référence au JSON** : soit un objet imbriqué `extraction: { id, filename,
  procedure_count }`, soit les champs plats `extraction_id`,
  `extraction_filename`, `procedure_count`.
- **Import direct** : reconnu par `kind`/`type`/`source_type` valant
  `import` / `json` / `direct`, par `is_direct_import: true`, ou à défaut par un
  `filename` en `.json` sans document source. Il s'affiche dans la même liste
  avec le badge « Import direct » et sans ligne de document source.
- **Extraction** : un tableau de procédures, ou un objet
  `{ filename, procedures: [...] }` (`data`, `content`, `json`, `items` sont
  aussi acceptés).
- **Erreurs** : le champ `detail` de FastAPI (chaîne ou liste de validation) est
  affiché tel quel à l'admin.

Si votre backend attend autre chose (par exemple un objet enveloppe sur le `PUT`),
la seule chose à changer est `js/api.js`.

## Comportements implémentés

- Sondage de `GET /admin/documents` toutes les 3 s **tant qu'une extraction est
  en cours**, arrêté dès que plus rien n'est en cours et au changement d'écran.
  Un échec pendant le sondage affiche un toast sans effacer la liste déjà à
  l'écran.
- Validation du JSON **avant tout envoi** : tableau à la racine, `proc_title`
  non vide, `proc_administration[0]` non vide, `proc_pieces` / `proc_steps` /
  `proc_law` de type tableau s'ils sont présents. Les problèmes sont listés avec
  le numéro d'entrée et le bouton d'import reste bloqué.
- L'import JSON n'écrit jamais en base : il ouvre l'éditeur pour une dernière
  vérification.
- Éditeur : accordéon à un seul volet ouvert, pagination automatique au-delà de
  20 procédures, champs `proc_description` (textarea) et `proc_law`
  (liste ajout/suppression) gérés même absents ou `null`.
- Une administration vide passe la ligne en ambre avec la note « Administration
  manquante » et désactive l'approbation jusqu'à correction.
- Confirmation avant suppression d'une procédure ; avertissement avant de quitter
  l'écran ou l'onglet avec des modifications non enregistrées.
- Boutons désactivés pendant les requêtes ; l'approbation affiche une barre de
  progression plutôt qu'un bouton figé, et enregistre le brouillon avant
  d'appeler `/approve`.

### Espace citoyen

- L'assistant met plusieurs secondes à répondre : un indicateur de réflexion
  occupe la place de la réponse pendant l'attente. Si l'envoi échoue, la bulle
  de question est retirée et le texte revient dans le champ — un message resté
  sans réponse ferait croire qu'il est parti.
- L'historique des discussions vit dans la barre latérale, sous « Assistant »,
  et n'apparaît que sur `#/chat`. Il est rendu par `js/shell.js` mais appartient
  à l'écran : celui-ci pousse sa liste par `App.shell.setConversations()` et
  reçoit les clics par `App.shell.setConversationHandlers()`. La coque ne fait
  aucun appel réseau.
- Entrée envoie, Maj + Entrée va à la ligne.
- Les réponses peuvent être arabes, françaises ou anglaises : chaque bulle porte
  `dir="auto"` et choisit son sens de lecture seule. L'alignement, lui, dit qui
  parle — une question arabe reste du côté de celui qui l'a posée.
- Cocher une pièce déplace la barre d'avancement **avant** la réponse du
  serveur ; si l'enregistrement échoue, la case revient en arrière et l'échec
  est dit. Une barre qui ne correspond pas à ce qui est enregistré serait pire
  que pas de barre.
- Une procédure dont toutes les pièces sont cochées prend la pastille
  « Terminé ». Le retrait du suivi passe par la modale de confirmation partagée
  (`js/modals/confirm-delete.js`).

## Points d'attention

- **« Nouvelles administrations »** dans la modale d'approbation compte les
  administrations *distinctes du fichier* : le frontend ne sait pas lesquelles
  existent déjà en base. Si vous voulez le vrai nombre de nouveautés, renvoyez-le
  depuis le backend et remplacez `computeSummary` dans `js/modals/approve.js`.
- À l'enregistrement, les chaînes vides redeviennent `null` et les entrées vides
  des listes sont supprimées (`serializeProcedure` dans `js/api.js`). Les champs
  inconnus présents dans le JSON d'origine sont conservés tels quels.


## Navigation

| Route | Écran | Rôle |
|---|---|---|
| `#/dashboard` | Tableau de bord (accueil `admin`) | admin |
| `#/documents` | Documents et extractions | admin |
| `#/procedures` | Toutes les procédures | admin |
| `#/procedures/pieces` | Inventaire des pièces requises | admin |
| `#/procedures/etapes` | Inventaire des étapes | admin |
| `#/extractions/{id}` | Éditeur JSON | admin |
| `#/chat` | Assistant (accueil `citizen`) | citizen |
| `#/suivi` | Mes procédures | citizen |

## Points d'entrée backend attendus

Les trois derniers sont nouveaux ; sans eux l'application fonctionne toujours,
en mode dégradé (voir `deriveStats` dans `js/api.js`).

| Méthode | Chemin | Utilisé par |
|---|---|---|
| GET | `/admin/documents` | liste des documents |
| POST | `/admin/documents` | import d'un document |
| POST | `/admin/imports` | import d'un JSON |
| GET | `/admin/extractions/{id}` | éditeur |
| PUT | `/admin/extractions/{id}` | enregistrement |
| POST | `/admin/extractions/{id}/approve` | approbation |
| GET | `/admin/me` | nom affiché dans l'en-tête (repli : « Administrateur ») |
| GET | `/admin/stats` | tableau de bord (repli : recalcul depuis `/admin/documents`) |
| GET | `/admin/procedures` | écran Procédures (champs `statut_proc`, `date_obsolete`) |
| PATCH | `/admin/procedures/{id}/obsolete` | mise hors vigueur ; renvoie `{ affected_users }` |
| DELETE | `/admin/procedures/{id}` | suppression définitive ; `409` si des citoyens la suivent |
| GET | `/admin/logs` | écran Journal ; filtres `action`, `user_id`, `limit` (100 par défaut) |

### Espace citoyen — à écrire

Aucun de ces points d'entrée n'existe côté backend. Le contrat détaillé (formes
de réponse, noms de champs acceptés) est en commentaire dans `js/api.js`,
au-dessus des normaliseurs ; les mocks correspondants sont dans `js/mock.js`.

| Méthode | Chemin | Utilisé par |
|---|---|---|
| GET | `/citizen/conversations` | historique, dans la barre latérale |
| GET | `/citizen/conversations/{id}` | fil d'une discussion |
| POST | `/citizen/conversations` | première question — crée la discussion |
| POST | `/citizen/conversations/{id}/messages` | question suivante |
| GET | `/citizen/tracked` | « Mes procédures » |
| POST | `/citizen/tracked` | « Suivre cette procédure » |
| DELETE | `/citizen/tracked/{id}` | retrait du suivi |
| PATCH | `/citizen/tracked/{id}/pieces/{pieceId}` | cocher une pièce, écrire une note |

Une réponse de l'assistant porte ses **sources** : pour chacune, le titre de la
procédure, son administration et son identifiant — c'est cet identifiant que
`POST /citizen/tracked` reçoit. Le `PATCH` renvoie le suivi **complet** : c'est
le serveur qui fait foi sur l'avancement, l'écran s'y réaligne après coup.

## Contenu bidirectionnel

Le contenu peut être arabe, français ou anglais, y compris mélangé dans une même
procédure. Le parti pris : l'interface reste en français donc de gauche à droite,
et seuls les champs porteurs de contenu sont marqués `dir="auto"` — chacun choisit
son sens d'après son premier caractère fort. Voir le bloc « Contenu
bidirectionnel » de `styles.css`. Deux pièges déjà traités :

- la vue « JSON brut » force `dir="ltr"`, sinon les accolades et les virgules
  partent à la mauvaise extrémité ;
- le CSS cible `:dir(rtl)` et non `[dir="rtl"]` : l'attribut vaut `auto`, c'est
  le sens *calculé* qu'il faut interroger.

## Graphiques

`js/charts.js` ne dépend d'aucune bibliothèque : tout est du SVG écrit à la
largeur réelle du conteneur, redessiné par un `ResizeObserver`. La palette a été
validée (bande de clarté, plancher de chroma, séparation daltonienne, contraste
sur fond blanc) — la rampe bleue est monochrome et ordonnée du clair au foncé.
Trois règles à ne pas casser en la modifiant :

- une série = une seule couleur ; ne jamais colorer une barre selon sa valeur ;
- la valeur reste lisible sans survol (étiquette directe, axe, ou bouton
  « Tableau » qui révèle le jumeau tabulaire) ;
- les couleurs de statut vont toujours avec un libellé écrit, jamais seules.
