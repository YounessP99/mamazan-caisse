---
name: mamazan-app
description: Modifier l'application de caisse MamaZan (index.html + sw.js). À utiliser dès qu'on touche au code de l'app — ajout de fonctionnalité, correction, refonte d'un écran. Impose les contraintes techniques du projet, le modèle de données localStorage, le vocabulaire de l'interface et la checklist de fin de tâche.
---

# MamaZan — règles de modification de l'app

## Ce qu'est ce projet

Une caisse enregistreuse utilisée en vrai, sur tablette, pendant la vente de nourriture
(rôtis, samoussas, beignets, boissons). Elle tourne hors-ligne. Si elle plante un jour de
vente, on ne vend pas. Ce n'est pas un projet d'apprentissage : la fiabilité passe avant
l'élégance du code.

Dépôts :
- `mamazan-caisse` (public) — le code
- `mamazan-ventes` (privé) — les sauvegardes de ventes envoyées par l'app

## Contraintes non négociables

1. **Un seul fichier** : tout le HTML, le CSS et le JS vivent dans `index.html`.
   Pas de framework, pas de bundler, pas de `npm install`, pas d'import de module.
   On ouvre le fichier, ça marche.
2. **JavaScript ES5 dans l'app** : `var`, `function(){}`, pas de JSX, pas de TypeScript.
   Le code existant est écrit comme ça, on ne mélange pas deux styles.
3. **Aucune dépendance réseau au runtime**. La seule requête sortante est la sauvegarde
   GitHub, et elle doit échouer silencieusement sans casser la vente.
4. **localStorage uniquement**. Toute écriture est enveloppée dans un `try/catch` :
   si le stockage est plein ou bloqué, la session continue en mémoire.
5. **Cible iPad en mode kiosque** : zones tactiles d'au moins 44 px, pas de survol,
   pas de scroll horizontal, pas de dépendance au clavier.

## Le parcours de vente est sacré

Ajouter un produit au panier et valider une commande doivent rester à deux gestes.
Toute nouvelle fonctionnalité de gestion va sous l'onglet **Gestion** (Carte, Stock,
Achats, Argent) ou dans **Fin de journée**, jamais dans l'écran Caisse.

Avant d'ajouter quoi que ce soit à l'écran Caisse, se poser la question : est-ce que
ça sert pendant qu'un client attend ? Si non, ça va ailleurs.

## Modèle de données

Clés localStorage, toutes suffixées par une version :

| Clé | Contenu |
|---|---|
| `mamazan_products_v1` | carte : `{id, name, price, tracked, stock}` |
| `mamazan_caisse_v1` | journée en cours : `{orders, nextOrderNumber, cart, usage}` |
| `mamazan_supplies_v1` | matières et consommables : `{id, name, kind, unit, qty, alertAt, avgCost}` |
| `mamazan_ratios_v1` | `RATIOS[supplyId][productId] = {per, samples}` |
| `mamazan_cash_v1` | argent : `{opening, counted, movements}` |
| `mamazan_archive_v1` | journées clôturées (90 max) |
| `mamazan_github_cfg_v1` | réglages de sauvegarde |

### Règle de compatibilité ascendante

Des tablettes tournent déjà avec des données réelles dedans. Toute lecture de
localStorage passe par une fonction de normalisation qui tolère les champs absents et
retourne une valeur par défaut saine. Ne jamais supposer qu'un champ ajouté aujourd'hui
existe dans les données d'hier.

```js
function normalizeTruc(t){
  return {
    id: t.id,
    qty: (typeof t.qty === 'number' && isFinite(t.qty)) ? t.qty : 0
  };
}
```

### Le mécanisme des ratios

Il n'y a pas de fiche technique au gramme près : la cuisine est faite maison. Le lien
entre un produit vendu et les matières consommées est **appris**. À la clôture, on saisit
ce qui a été réellement utilisé (« 5 paquets de farine »), l'app le rapproche de ce qui
a été vendu (« 50 rôtis ») et en déduit `per = 0,1 paquet par rôti`. Une moyenne glissante
sur les 10 dernières journées affine le chiffre. La saisie manuelle écrase l'apprentissage
et remet `samples` à 0.

Quand plusieurs produits partagent une matière, la répartition est proportionnelle aux
ratios déjà connus. C'est une estimation, et l'interface doit le dire — ne jamais
présenter un ratio estimé comme une certitude.

## Vocabulaire de l'interface

L'interface est lue par des gens qui vendent, pas par des développeurs. En français,
sans jargon, sans emoji.

- « Devrait être en caisse », pas « solde théorique »
- « À racheter », pas « seuil d'alerte franchi »
- « Prévu environ 5 paquets », pas « consommation théorique »
- Montants en euros avec virgule : `formatEuro()` existe déjà, l'utiliser
- Saisie décimale : accepter la virgule ET le point, `parseNum()` existe déjà

Les messages d'erreur donnent un exemple concret : « Prix invalide — exemple : 1,20 ».

## Style visuel

Variables CSS existantes dans `:root` — les réutiliser, ne pas introduire de couleurs
en dur : `--green`, `--green-dark`, `--terracotta`, `--terracotta-dark`, `--cream`, `--line`.

Classes réutilisables déjà en place : `.panel`, `.g-row`, `.g-num`, `.g-btn`, `.g-form`,
`.g-status`, `.g-hint`, `.kpi`, `.money-line`, `.seg-btn`, `.mini-btn`, `.link-chip`.
Avant de créer une classe, vérifier qu'une équivalente n'existe pas.

## Checklist avant de dire que c'est fini

1. **Bumper `CACHE_NAME` dans `sw.js`** (`mamazan-v9` → `mamazan-v10`). Sans ça la
   tablette garde l'ancienne version et le travail est invisible. C'est l'oubli le plus
   fréquent sur ce projet.
2. Vérifier la syntaxe JS et que chaque `getElementById` pointe vers un `id` qui existe.
3. Lancer les tests (voir le skill `mamazan-test`), y compris le test de migration.
4. Vérifier que l'écran Caisse n'a pas bougé.
5. Un commit par étape, avec un message qui dit ce que ça change pour l'utilisateur,
   pas pour le code.

## Ce qu'il ne faut pas proposer

- Réécrire en React, Vue ou Svelte
- Découper en modules ES ou ajouter un build
- Introduire une base de données ou un backend
- Ajouter une librairie de graphiques pour trois chiffres
- Remplacer localStorage par IndexedDB « pour faire propre »

Si une de ces pistes semble vraiment justifiée, l'exposer comme une question à
l'utilisateur avec le coût réel, et ne rien faire avant sa réponse.
