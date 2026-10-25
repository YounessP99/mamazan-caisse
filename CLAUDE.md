# MamaZan — caisse

Application de caisse utilisée en vrai, sur tablette, pendant la vente de nourriture
(rôtis, samoussas, beignets, boissons). Elle tourne hors-ligne. Si elle plante un jour
de vente, on ne vend pas. La fiabilité passe avant l'élégance du code.

## Skills de ce projet

Pour **toute modification du code**, utiliser le skill `mamazan-app`
(`.claude/skills/mamazan-app/`). Il contient les contraintes techniques, le modèle de
données localStorage et la checklist de fin de tâche.

Pour **tester ou valider** une modification, utiliser le skill `mamazan-test`
(`.claude/skills/mamazan-test/`). Aucune modification ne se livre sans que le test de
parcours complet et le test de migration passent.

## Contraintes à ne jamais enfreindre

- Un seul fichier : tout le HTML, CSS et JS vivent dans `index.html`
- Pas de framework, pas de bundler, pas de build, pas de `npm install`
- JavaScript ES5 (`var`, `function(){}`) pour rester cohérent avec l'existant
- Données en localStorage uniquement, chaque écriture dans un `try/catch`
- Cible iPad en mode kiosque : zones tactiles d'au moins 44 px

## Le parcours de vente est sacré

Ajouter un produit au panier et valider une commande restent à deux gestes. Toute
nouvelle fonctionnalité de gestion va sous l'onglet Gestion ou dans Fin de journée,
jamais dans l'écran Caisse.

## À ne jamais oublier

Bumper `CACHE_NAME` dans `sw.js` à chaque modification. Sans ça la tablette garde
l'ancienne version en cache et le travail est invisible.

## Où vivent les choses

- Code : `C:\Dev\mamazan-caisse` (ce dossier), poussé sur GitHub `mamazan-caisse` (public)

## Sécurité

Le jeton GitHub de la sauvegarde est saisi par l'utilisateur et stocké en clair dans le
localStorage de la tablette. Il ne doit jamais être écrit en dur dans le code, ni
committé. Le dépôt `mamazan-caisse` est public.
