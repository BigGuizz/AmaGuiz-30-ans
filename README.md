# Les 30 ans au Mas

Outil d'organisation du week-end au Mas des 7 Fonts : invités, couchages, budget, to-do et olympiades.
Site statique (GitHub Pages) + base Firebase Firestore. Accès par le lien, base fermée automatiquement le 1er mars 2027.

## Fichiers

- `index.html` : l'outil.
- `config.js` : la configuration Firebase.
- `firestore.rules` : les règles de sécurité à coller dans la console Firebase.

## Mise en place

1. Firebase > Firestore : base créée (eur3, production).
2. Firebase > Authentication > Méthode de connexion : activer **Anonyme**.
3. Firebase > Firestore > Règles : coller `firestore.rules`, puis Publier.
4. Firebase > Authentication > Paramètres > Domaines autorisés : ajouter `<pseudo-github>.github.io`.
5. GitHub > Settings > Pages > Deploy from a branch > `main` / `(root)`.

## Importer les données

Une fois connecté, bouton « Importer des données (.json) » en bas de page.
Ne pas committer l'export : il contient les noms et photos des invités.
