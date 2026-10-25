---
name: mamazan-test
description: Écrire et lancer les tests de l'application de caisse MamaZan avant de livrer une modification. À utiliser systématiquement après toute modification de index.html, et dès qu'on parle de tester, vérifier ou valider l'app. Impose un test de parcours complet et un test de migration des données existantes.
---

# MamaZan — tests obligatoires

## Pourquoi

L'app tourne sur une tablette qui contient les vraies données de vente. Une régression
n'est pas un bug de développement : c'est une journée de vente perdue, ou pire, un stock
et une caisse faux pendant des semaines sans que personne ne s'en aperçoive.

Aucune modification n'est livrée sans que les deux tests ci-dessous passent.

## Mise en place

Les tests tournent dans un dossier séparé du dépôt (ne pas polluer `mamazan-caisse`,
qui doit rester à deux fichiers).

```bash
mkdir -p ~/mamazan-tests && cd ~/mamazan-tests
npm install jsdom
```

Le test charge `index.html` dans jsdom avec `runScripts: 'dangerously'`, puis pilote
l'app en cliquant sur de vrais éléments du DOM.

```js
import fs from 'fs';
import { JSDOM } from 'jsdom';

const html = fs.readFileSync('/chemin/vers/mamazan-caisse/index.html', 'utf8');
const dom = new JSDOM(html, {
  runScripts: 'dangerously',
  pretendToBeVisual: true,
  url: 'https://example.org/'
});
const w = dom.window, doc = w.document;
await new Promise(r => setTimeout(r, 100));
```

Pour partir de données existantes, les injecter avant que le script ne s'exécute :

```js
const dom = new JSDOM(html, {
  runScripts: 'dangerously',
  pretendToBeVisual: true,
  url: 'https://example.org/',
  beforeParse(win) {
    win.localStorage.setItem('mamazan_products_v1', JSON.stringify([...]));
  }
});
```

## Piège à connaître

Les grilles sont redessinées après chaque action (`renderCart()` appelle
`renderProducts()`). Une référence à un bouton devient obsolète dès le premier clic.
Toujours re-cibler l'élément à chaque itération :

```js
// faux : les clics 2 à 50 tombent dans le vide
const btn = doc.querySelector('.product-btn[data-product="roti"]');
for (let i = 0; i < 50; i++) btn.click();

// juste
for (let i = 0; i < 50; i++) {
  doc.querySelector('.product-btn[data-product="roti"]').click();
}
```

Les champs de saisie ne déclenchent rien par assignation. Il faut émettre l'événement :

```js
el.value = '50';
el.dispatchEvent(new w.Event('change', { bubbles: true }));
```

## Test 1 — parcours complet

Rejouer une journée de bout en bout et vérifier les chiffres à chaque étape :

1. Démarrage à froid sans aucune donnée, zéro erreur JS
2. Navigation : Caisse, Gestion et ses sous-onglets, Fin de journée
3. Créer une matière première, la rattacher à un produit
4. Enregistrer un achat, vérifier le stock remonté et le coût moyen pondéré
5. Saisir le fond de caisse, vendre, vérifier le total du panier et la commande
6. Vérifier « devrait être en caisse » = fond + encaissé + apports − dépenses caisse − retraits
7. Saisir la consommation réelle, vérifier le stock déduit et le ratio appris
8. Vérifier le coût de revient et la marge
9. Saisir le comptage, vérifier l'écart
10. Clôturer, vérifier l'archive, la remise à zéro des mouvements, et que le stock
    des fournitures et le fond de caisse survivent

Scénario de référence, à garder car il correspond à la réalité du terrain :
20 paquets de farine en stock, achat de 10 paquets pour 15 €, 50 rôtis vendus à 1,40 €,
5 paquets consommés. Attendu : coût moyen 0,50 €/paquet, ratio appris 0,1 paquet/rôti,
coût du rôti 0,05 €, marge 1,35 €, stock restant 25 paquets, 250 rôtis encore possibles.

## Test 2 — migration (le plus important)

Charger l'app avec des données au format **d'avant** la modification et vérifier que
rien n'est perdu ni cassé.

```js
const ancien = {
  'mamazan_products_v1': JSON.stringify([
    { id: 'roti', name: 'Rôti', price: 1.4, tracked: true, stock: 12 }
  ]),
  'mamazan_caisse_v1': JSON.stringify({
    orders: [{ number: 1, time: '10:00',
      items: [{ id: 'roti', name: 'Rôti', price: 1.4, qty: 3 }], total: 4.2 }],
    nextOrderNumber: 2, cart: {}
  }),
  'mamazan_archive_v1': JSON.stringify([
    { dateLabel: '01/10/2026', revenue: 120, ordersCount: 9,
      rows: [{ name: 'Rôti', qty: 80 }] }
  ])
};
```

Vérifier : aucune erreur au chargement, les produits et leur stock sont là, les commandes
du jour sont là, les archives sont lisibles, et tous les nouveaux écrans s'ouvrent sans
planter malgré les champs absents.

Les données injectées doivent volontairement **omettre** les champs ajoutés par la
modification en cours. C'est tout l'intérêt du test.

## Vérifications rapides en plus

Syntaxe du script et cohérence des identifiants DOM, sans lancer jsdom :

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const js=html.match(/<script>([\s\S]*)<\/script>/)[1];
try{ new Function(js); console.log('Syntaxe OK'); }catch(e){ console.log('ERREUR',e.message); }
const ids=[...html.matchAll(/getElementById\('([^']+)'\)/g)].map(x=>x[1]);
const present=new Set([...html.matchAll(/\sid=\"([^\"]+)\"/g)].map(x=>x[1]));
const missing=ids.filter(i=>!present.has(i));
console.log('IDs manquants :', missing.length?missing:'aucun');
"
```

## Règle de lecture des échecs

Quand plusieurs tests échouent d'un coup, chercher la cause unique en amont avant de
corriger quoi que ce soit. Une seule valeur fausse au début du scénario fait tomber tous
les contrôles suivants, et les chiffres affichés par les échecs sont alors cohérents
entre eux — c'est le signe que l'app a raison et que c'est le test qui est mal écrit.

Ne jamais modifier l'application pour faire passer un test sans avoir établi lequel des
deux a tort.
