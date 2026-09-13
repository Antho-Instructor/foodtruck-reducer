---
title: Workshop useReducer - Le Foodtruck Ynov
lang: fr
---

# 🚚 Le Foodtruck Ynov - workshop `useReducer`

Ce TP se fait **seul**. Objectif : appliquer `useReducer` à un cas
**différent** du compteur ou du switch de thème vus en cours - la gestion
du **panier de commande** d'un foodtruck - pour vérifier que tu sais
**transférer** le pattern reducer à un nouveau domaine, pas seulement le
recopier.

À la fin, tu dois savoir répondre à la question :
**« pourquoi un `useReducer` évite-t-il ici les bugs qu'auraient causés
plusieurs `useState` séparés ? »**

Le code (starter + solution) est ici : [{{ site.code_repo }}]({{ site.code_repo }})
{: .alert-info}

## 🎯 Ce que tu vas travailler

- Écrire un **reducer** avec **7 actions métier** qui interagissent entre
  elles (contrairement à un compteur, où les actions sont indépendantes).
- **Typer** ces actions avec une **union discriminée** TypeScript.
- Combiner `useReducer` et **Context** pour exposer un mini store global,
  sans librairie externe.
- Comprendre pourquoi une valeur **dérivée** (ici, le total du panier) ne
  doit **pas** vivre dans le state du reducer.

Prérequis : le support de cours `useReducer` (pattern reducer, anatomie
d'un reducer, union discriminée, `useReducer` + `Context`), et les bases
React (`useState`, Context API, `useContext`).
{: .alert-warning}

---

# Partie 1 - Pourquoi un panier, et pas un compteur ?

**_10 minutes_**

Un compteur n'a qu'**une seule valeur** et des actions qui ne se marchent
jamais sur les pieds (`+1`, `-1`, `reset`). Ça montre la **syntaxe** de
`useReducer`, mais pas *pourquoi* on en a besoin en vrai projet.

Le panier du foodtruck a un état **composé de plusieurs valeurs qui
s'influencent** :

- les lignes du panier déterminent le sous-total ;
- le **happy hour** change le prix des boissons **dans** ce sous-total ;
- un **code promo** s'applique **après** le happy hour ;
- vider le panier doit remettre **les trois** à zéro d'un coup, de façon
  cohérente.

Avec plusieurs `useState`, rien ne t'empêche d'oublier de remettre
`happyHour` à `false` en vidant le panier - deux bugs indépendants, faciles
à manquer en review. Avec `useReducer`, toute la logique vit au même
endroit : `RESET_CART` a son propre `case` explicite, impossible de
n'en remettre à zéro qu'une partie.
{: .alert-info}

## Le domaine métier

```ts
interface Product {
  id: string;
  name: string;
  price: number;
  category: "burger" | "side" | "drink";
  emoji: string;
}

interface CartLine {
  product: Product;
  quantity: number;
}

interface CartState {
  lines: CartLine[];
  discountCode: string | null;
  discountPercent: number;
  happyHour: boolean;
}
```

Remarque de conception, à bien comprendre avant de coder : **le total
n'est pas dans `CartState`**. Il se calcule à partir du reste
(`reducer/cartSelectors.ts`, déjà fourni). S'il était stocké, il faudrait
le recalculer à la main dans chaque `case` qui touche au panier - un
oubli, et le total affiché mentirait par rapport au contenu réel du
panier. Un reducer ne gère que la **source de vérité**, jamais ce qu'on
peut recalculer à partir d'elle.
{: .alert-warning}

## ✅ Point de contrôle Partie 1

Avant de coder, tu dois pouvoir expliquer à voix haute :

- une situation où plusieurs `useState` deviendraient incohérents entre
  eux sur ce panier ;
- pourquoi le total n'a pas sa place dans `CartState`.

---

# Partie 2 - Mise en route

**_5 minutes_**

```bash
git clone {{ site.code_repo }}
cd tp-foodtruck-reducer/starter
npm install
npm run dev
```

Ouvre `http://localhost:5173`. Tu dois voir la liste des produits à gauche
et un panier vide à droite. **C'est normal que rien ne fonctionne encore.**

Tant que le TODO 3 (Context) n'est pas fait, l'appli affiche une erreur
React au chargement (`useCart() doit être appelé à l'intérieur d'un
<CartProvider>`). C'est un message d'erreur **volontaire** : il te dit
exactement quoi corriger, ce n'est pas un bug du starter.
{: .alert-warning}

Tout est déjà fourni et câblé **sauf 3 fichiers** : `src/types.ts`,
`src/reducer/cartReducer.ts` et `src/context/CartContext.tsx`. C'est là
que tu vas travailler.

---

# Partie 3 - Les 3 TODOs

**_1 heure_**

> Sous chaque TODO, des blocs **`▸ Indice`** à dérouler **un par un**,
> seulement quand tu bloques. Ils ne donnent jamais la ligne de code
> directement : ils pointent le bon outil ou la bonne question. Essaie
> **au moins 10 minutes** avant d'en ouvrir un.
{: .alert-warning}

## 🔹 TODO 1 · `src/types.ts` - typer les actions

**_15 minutes_**

Le reducer doit gérer **7 actions**. Écris l'union discriminée
`CartAction` qui les modélise (remplace le `export type CartAction =
never;` actuel) :

| `type`                  | Payload à transporter | Déclenchée par                       |
| ------------------------ | ----------------------- | -------------------------------------- |
| `"ADD_ITEM"`             | `product: Product`      | bouton "Ajouter" d'une carte produit  |
| `"INCREMENT_ITEM"`       | `productId: string`     | bouton `+` dans le panier             |
| `"DECREMENT_ITEM"`       | `productId: string`     | bouton `−` dans le panier             |
| `"REMOVE_ITEM"`          | `productId: string`     | icône poubelle dans le panier         |
| `"APPLY_DISCOUNT_CODE"`  | `code: string`          | formulaire de code promo              |
| `"TOGGLE_HAPPY_HOUR"`    | *(aucun)*                | bouton "Happy Hour"                   |
| `"RESET_CART"`           | *(aucun)*                | bouton "Vider le panier"              |

<details markdown="1">
<summary>▸ Indice · la syntaxe d'une union discriminée</summary>

Rappelle-toi l'exemple du compteur vu en cours :

```ts
type Action =
  | { type: "increment" }
  | { type: "decrement" }
  | { type: "reset" }
  | { type: "set"; payload: number };
```

Chaque ligne du tableau ci-dessus devient une variante de l'union, sur le
même modèle. Une action sans payload n'a que le champ `type`.
</details>

**Vérif.** `npm run build` doit faire disparaître les erreurs `is not
assignable to parameter of type 'never'` dans les fichiers
`components/*.tsx`. Il restera des erreurs dans `cartReducer.ts` et
`CartContext.tsx` : normal, ce sont les TODOs suivants.

## 🔹 TODO 2 · `src/reducer/cartReducer.ts` - écrire le reducer

**_30 minutes_**, le cœur de l'exercice.

Trois règles à respecter dans chaque `case` :

1. **Fonction pure** - pas de `fetch`, pas de `Math.random()`, pas de
   mutation d'une variable extérieure au reducer.
2. **Immutabilité** - jamais `state.lines.push(...)` ni
   `line.quantity++`. Toujours `{ ...state, ... }`, `.map()`, `.filter()`,
   `[...state.lines, nouvelleLigne]`.
3. **Action invalide → état inchangé** - un code promo inconnu ne doit
   RIEN changer, pas planter.

<details markdown="1">
<summary>▸ Indice · ADD_ITEM et INCREMENT_ITEM</summary>

Les deux se ressemblent : trouve la ligne concernée (`.find()` ou
`.map()`), et soit incrémente sa `quantity`, soit ajoute une nouvelle
ligne si le produit n'était pas encore dans le panier.
</details>

<details markdown="1">
<summary>▸ Indice · DECREMENT_ITEM</summary>

Fais-le en deux étapes séparées (plus lisible qu'un seul `.reduce()`) :
`.map()` pour décrémenter la bonne ligne, puis `.filter()` pour retirer
les lignes tombées à `0`.
</details>

<details markdown="1">
<summary>▸ Indice · APPLY_DISCOUNT_CODE</summary>

`DISCOUNT_CODES` (déjà importé) est un `Record<string, number>`. Cherche
`action.code` dedans (normalise la casse avec `.toUpperCase()`). Si le
résultat vaut `undefined`, retourne `state` sans y toucher.
</details>

<details markdown="1">
<summary>▸ Indice · TOGGLE_HAPPY_HOUR et RESET_CART</summary>

Les deux plus courts : `!state.happyHour` pour le premier,
`initialCartState` (déjà déclaré en haut du fichier) pour le second.
</details>

**Vérif.** Ajoute un burger deux fois de suite → une seule ligne,
quantité `2`. Clique `−` jusqu'à `0` → la ligne disparaît. Le bouton
Happy Hour divise par deux le prix affiché des boissons. `ETUDIANT10`
affiche `-10%` sur le total ; un code inventé n'affiche rien et ne
plante pas la page.

## 🔹 TODO 3 · `src/context/CartContext.tsx` - useReducer + Context

**_15 minutes_**

Structure identique au `ThemeContext` vu en cours, appliquée au panier.
Dans `CartProvider` :

1. Appelle `useReducer(cartReducer, initialCartState)`.
2. Retourne `<CartContext.Provider value={{ state, dispatch }}>{children}</CartContext.Provider>`
   au lieu du fragment actuel.

Le hook `useCart()` en bas du fichier est **déjà fourni**, tu n'as rien à
y changer.

**Vérif.** L'erreur `useCart() doit être appelé...` disparaît, toute
l'appli fonctionne de bout en bout.

---

# Partie 4 - Validation

**_10 minutes_**

Déroule ce scénario en entier :

1. Ajoute 2 Cheeseburgers et 1 Frites → 2 lignes, quantités correctes,
   sous-total juste.
2. Ajoute un Soda, active le **Happy Hour** → son prix affiché est divisé
   par deux, le sous-total se met à jour.
3. Applique `CAMPUS20` → le total baisse de 20% par rapport au sous-total
   (happy hour inclus).
4. Essaie un code bidon → rien ne change, pas d'erreur console.
5. Clique `−` sur les Frites jusqu'à `0` → la ligne disparaît seule.
6. **Vide le panier** → retour à zéro complet (lignes, code promo, happy
   hour).

---

# 🎁 Bonus (si tu as fini en avance)

Aucun n'est noté :

- **Initialisation paresseuse (lazy init)** : `useReducer` accepte un
  **3ᵉ argument**, une fonction `init(initialArg)` appelée une seule fois
  au premier rendu - utile pour un calcul coûteux (ici : lire un panier
  sauvegardé dans `localStorage`). Essaie `useReducer(cartReducer,
  undefined, init)` + un `useEffect` qui réécrit le panier dans
  `localStorage` à chaque changement.
- **Quantité maximale** : refuse `INCREMENT_ITEM` au-delà de 10 unités
  d'un même produit, sans planter.
- **`document.title` dynamique** : un `useEffect` dans `App` qui affiche
  le total du panier dans l'onglet du navigateur.
- **Historique des actions** : garde les 5 dernières actions dispatchées
  et affiche-les - le principe de base des devtools Redux.

---

# 📋 Auto-évaluation (/20)

Pas de rendu pour ce TP : sers-toi de cette grille pour vérifier que tu as
bien tout couvert avant de comparer avec `solution/`.

| Ce qu'on regarde                                                       | Points  |
| ------------------------------------------------------------------------| :-----: |
| `npm run dev` démarre, `node_modules` non commité                      |    2    |
| **TODO 1** - union discriminée complète et correctement typée          |    4    |
| **TODO 2** - les 7 actions du reducer fonctionnent, state jamais muté  |    8    |
| **TODO 3** - Context + hook `useCart` fonctionnels                     |    4    |
| **Scénario de validation** (Partie 4) passe en entier                  |    2    |
| **Total**                                                               | **/20** |
{: .alert-warning}

---

# ☝️ Récap

Reformule pour toi-même, sans regarder le code :

- une situation où plusieurs `useState` seraient devenus incohérents sur
  ce panier ;
- ce qu'une union discriminée apporte par rapport à `type Action = {
  type: string; payload?: any }` ;
- pourquoi le total n'est pas stocké dans `CartState` ;
- ce que Context apporte à `useReducer`, et inversement.

Si un de ces points est encore flou, relis la Partie 1 et compare avec
`solution/` qui te sera fournie après la remise.
