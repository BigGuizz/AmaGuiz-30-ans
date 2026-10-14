# Les 30 ans au Mas — contexte projet

Outil web d'organisation d'un week-end d'anniversaire (30 ans) co-organisé par Guilhem et Amandine,
dans un gîte de groupe loué du vendredi soir au dimanche. Utilisé à deux, en ligne, en même temps.

- Repo : `BigGuizz/AmaGuiz-30-ans` (public), branche `main`
- Site : https://bigguizz.github.io/AmaGuiz-30-ans/ (GitHub Pages, deploy from branch `main` / root)
- Base : Firebase, projet `amaguizz-30` (forfait Spark gratuit), Firestore région eur3
- Langue de l'interface et des échanges : français. Guilhem dicte souvent à la voix, donc les noms
  propres peuvent arriver déformés : demander confirmation plutôt que deviner.

## ⚠️ Règle n°1 : le repo est public

Ne jamais committer de données d'invités : noms, photos, exports JSON, captures.
Les données vivent uniquement dans Firestore. `.gitignore` exclut `export/`, `data.json` et les fichiers de travail.
Si un fichier `*.json` ou un export contenant des noms apparaît dans le repo, le supprimer
(et prévenir Guilhem que l'historique Git le contient encore).

## Architecture

Site statique sans build, sans framework, sans dépendance npm.

| Fichier | Rôle |
|---|---|
| `index.html` | Toute l'app : HTML, CSS et JS (module) dans un seul fichier |
| `config.js` | `export const FIREBASE_CONFIG = {…}` : config web Firebase (pas secrète) |
| `firestore.rules` | Règles de sécurité. **Pas déployées automatiquement** : à coller dans la console Firebase (Firestore > Règles > Publier) |
| `README.md` | Mise en place pour un humain |

- Firebase JS SDK **10.12.2** importé en ESM depuis `https://www.gstatic.com/firebasejs/10.12.2/…` (app, firestore, auth).
- Auth : **connexion anonyme automatique** (`onAuthStateChanged` → `signInAnonymously`). Pas de mot de passe,
  choix assumé de Guilhem : toute personne qui a le lien peut lire et modifier.
- Temps réel : un `onSnapshot` par collection, chaque snapshot relance `render()` (rendu complet, innerHTML).
- Un petit objet `adapter` expose une API `db.collection(p).add/onSnapshot` et `db.doc(p).update/set/delete/onSnapshot`.
  Il existe parce que l'app a d'abord tourné comme artifact claude.ai avec cette API. Le code métier passe par lui,
  donc garder cette interface ou tout migrer d'un coup.

### Règles Firestore actuelles

Lecture/écriture si `request.auth != null`, uniquement sur les collections
`guests`, `expenses`, `todos`, `ideas`, `settings`, et **seulement jusqu'au 1er mars 2027** (`request.time < timestamp.date(2027, 3, 1)`).
Toute nouvelle collection doit être ajoutée à cette liste, sinon les écritures échouent en silence côté UI (toast « Pas enregistré »).

## Modèle de données (Firestore)

**`guests/{id}`** : un invité
- `name` string
- `status` : `invite` | `confirme` | `peutetre` | `decline`
- `stay` : `''` (à préciser) | `ven-sam` (2 nuits) | `ven` (1 nuit) | `sam` (1 nuit) | `soiree` (vient sans dormir)
- `arrDay` : `''` | `ven` | `sam` | `dim` ; `arrTime` : `"HH:MM"` ou `''`
- `gare` bool (à aller chercher à la gare), `note` string
- `offert` bool (participation à 0 €, ex. le DJ), `paid` bool
- `room` : id de couchage (`c1-1`, `th-3`…) ou `''`
- `photo` : data URI JPEG 96×96 en base64 (~4 Ko), optionnel
- `order` number (tri de la liste), `createdAt` ms
- `specialBy` : `''` | `guilhem` | `amandine` (organisateur qui accorde un tarif spécial), `specialPrice` number (€ payés par l'invité).
  Champs optionnels : absents sur les anciens invités, ce qui équivaut à « pas de tarif spécial ».

**`expenses/{id}`** : `label`, `amount` (€), `createdAt`.
La ligne dont le libellé contient « mas » sert au calcul « coût du mas par nuit dormie ».

**`todos/{id}`** : `text`, `who`, `done`, `createdAt`

**`ideas/{id}`** : épreuves d'olympiades, `title`, `desc`, `author`, `createdAt`, et optionnels `order` (numéro d'affichage),
`materiel` string, `unsure` bool (« pas sûr »). Modifiables depuis l'UI (bouton Modifier).

**`settings/olympiades`** : `{ cadeau }`, le lot de l'équipe gagnante (texte libre). Le préambule (équipes par gîte, barème) est en dur dans `index.html`.

**`settings/prix`** : `{ deuxNuits, uneNuit, soiree }` en €. Valeurs par défaut dans le code : 80 / 50 / 0.

## Logique métier

- **Tarif d'un invité** = `prices[deuxNuits|uneNuit|soiree]` selon `stay`, 0 si `offert`, non compté si `stay` vide ou `status === 'decline'`.
- **Nuits occupées** : `ven-sam` → ven + sam, `ven` → ven, `sam` → sam, `soiree` → aucune.
  Les jauges vendredi/samedi comparent au total de **36 couchages**.
- **Couchages** : constante `GITES` codée en dur dans `index.html` (composition réelle du gîte, voir plus bas).
  L'occupation d'un espace est vérifiée **par nuit**, donc un « vendredi seul » et un « samedi seul » peuvent partager un lit.
  Passer un invité en `decline` ou sur un séjour sans nuit vide son `room`.
- **Tarifs spéciaux** (bloc dans l'onglet Budget, une colonne par organisateur) : l'invité paie `specialPrice`,
  et l'écart avec le tarif normal (`giftOf`) est **à la charge de l'organisateur qui l'accorde**, pas du budget commun.
  Un tarif spécial passe avant `offert` (l'ajouter décoche `offert`, cocher `offert` retire le tarif spécial).
  Liste des organisateurs : constante `ORGAS`.
- **Budget** : solde = participations attendues + cadeaux des organisateurs − dépenses.
  Mise en page : paramètres à gauche (dépenses, tarifs, tarifs spéciaux, répartition), récap « Équilibre » sticky à droite
  (en haut et non sticky sous 860 px). « De votre poche » = tarifs spéciaux de chacun + moitié du déficit restant. Si le solde est négatif, l'UI propose le % de hausse des tarifs pour tomber juste.
- **Saisie** : un champ texte s'enregistre au `change` (en quittant le champ). Tant qu'un input dans un bloc `[data-live]`
  a le focus, `render()` est différé jusqu'au `focusout`, pour ne pas écraser la frappe pendant une mise à jour distante.
- **Suppression** en deux clics (le bouton s'arme 3 s puis « Confirmer »). Pas de `confirm()`.
- **Onglets** : `#invites`, `#couchages`, `#budget`, `#todo`, `#olympiades`. Le hash ouvre l'onglet, et le dernier onglet est mémorisé en `localStorage`.
- **Import** : bouton en pied de page, fichier JSON `{ guests:{id:{…}}, todos:{…}, expenses:{…}, ideas:{…}, settings:{…} }`,
  écrit par `writeBatch` (lots de 400) en `set` : **écrase** les documents de même id.

## Le lieu : Domaine du Mas des 7 Fonts

Chemin de la Victoire, 34230 Saint-Pons-de-Mauchiens (Hérault). 4 gîtes + 1 suite, 36 couchages, 6 salles d'eau, 7 WC.

| Gîte | Espace | Lits | Places |
|---|---|---|---|
| Corniche 1 (`c1`) | Chambre 1 / Chambre 2 / Salle à manger | double 140 / double + simple / BZ 140 | 2 / 3 / 2 |
| Corniche 2 (`c2`) | Chambre 1 / Alcôve / Salon TV | double 140 / superposés / 3 simples | 2 / 2 / 3 |
| Thau (`th`) | Ch. 1 / Ch. 2 / Ch. 3 / Salle à manger | 2 simples + superposés / 2 simples / double 140 / BZ 140 | 4 / 2 / 2 / 2 |
| Picpoul (`pp`) | Ch. 1 / Ch. 2 / Salle à manger | 2 simples / double + superposés / BZ 140 | 2 / 4 / 2 |
| Suite (`su`) | Chambre / Salon | double / clic-clac 140 | 2 / 2 |

Autres infos utiles : salle de réception climatisée avec piste de danse (pas de limite d'horaire pour le bruit),
office-cuisine pro (vaisselle pour 50, tireuse à bière), barbecue géant, piscine 17 × 7 m, pétanque, parc clos d'1 ha.
**Draps non fournis**, ménage de fin de séjour et taxe de séjour inclus. Source : fiche GrandsGites.com, réf. 5318.

## Où on en est (mi-octobre 2026)

- ✅ App migrée de l'artifact claude.ai vers GitHub Pages + Firebase. L'ancien artifact est **obsolète** :
  la seule source de vérité est Firestore.
- ✅ Firebase : Firestore créé, auth anonyme activée, règles publiées.
- ✅ GitHub Pages actif (le site répond).
- ✅ Le fichier `download` était le `.gitignore` uploadé sous un mauvais nom (aucune donnée d'invité) : renommé en `.gitignore`.
- ⏳ À vérifier : domaine `bigguizz.github.io` ajouté aux domaines autorisés de Firebase Auth, import des données fait.
- Données au moment de la migration : 33 invités (une partie avec séjour et heure d'arrivée, certains à aller chercher à la gare,
  25 avec photo), 8 tâches, 5 dépenses, olympiades vides au départ, puis 9 épreuves ajoutées par Claude (mi-octobre).
- Budget saisi : mas 2 000 €, DJ 350 €, gâteaux 120 € minimum, tartiflette du samedi et brunch du dimanche estimés
  à 150–200 € chacun (175 € saisis). Tarifs annoncés aux invités : 80 € les 2 nuits, 50 € une nuit.
  **Tarif « soirée seulement » pas encore décidé.** Le budget est structurellement déficitaire avec ces tarifs.

## Pistes possibles (non demandées pour l'instant, ne pas faire sans validation)

- Vue « navette gare » : invités `gare` triés par jour et heure, pour organiser les allers-retours.
- Votes sur les idées d'olympiades, constitution d'équipes.
- Ajout d'une photo depuis l'UI (upload → redimensionnement canvas 96 px → data URI).
- Répartition automatique proposée dans les couchages (couples ensemble, nuits compatibles).

## Travailler sur le projet

- **Lancer en local** : `python3 -m http.server 8000` puis http://localhost:8000. Les modules ES ne marchent pas en `file://`,
  et `localhost` est autorisé par défaut dans Firebase Auth. ⚠️ Le local écrit dans **la vraie base** : pas de données de test sans prévenir.
- **Déployer** : push sur `main`, GitHub Pages republie en ~1 min.
- **Règles Firestore** : modifier `firestore.rules` puis demander à Guilhem de les coller dans la console
  (ou mettre en place `firebase.json` + `firebase deploy --only firestore:rules` s'il installe le CLI).
- **Style** : tokens CSS dans `:root` (vert pin `--accent`, jaune `--sun`), thème clair et sombre (via `prefers-color-scheme` et `[data-theme]`),
  polices Bricolage Grotesque / Instrument Sans / JetBrains Mono (Google Fonts). Mobile à 400 px sans scroll horizontal,
  sauf le tableau des invités qui défile dans son conteneur. Garder ce langage visuel.
- Rester sur un seul `index.html` sans build tant que ça reste lisible. Si ça grossit, découper en modules ES natifs, sans bundler.
- Échapper tout texte utilisateur via `esc()` avant de l'injecter en HTML (la base est ouverte à tous ceux qui ont le lien).
