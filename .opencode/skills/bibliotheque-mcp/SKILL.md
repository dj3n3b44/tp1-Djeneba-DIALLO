---
name: bibliotheque-mcp
description: >-
  Guide d'utilisation du serveur MCP « bibliotheque » (médiathèque) et de ses
  pièges. À charger SYSTÉMATIQUEMENT dès qu'une tâche utilise un outil
  `bibliotheque_*` (count_books, list_books, get_book, search_books,
  list_members, get_member, list_loans, get_member_fees, create_loan,
  delete_loan, list_missions, get_mission), même si l'utilisateur ne parle pas
  de MCP : inventaire du catalogue, retards, réinscription, ménage/nettoyage
  des prêts, relance d'emprunteurs, calcul de frais, recherche de livres.
  Contient les comportements non documentés (unités, dates, plafonds,
  pagination, limite de débit, écritures) à connaître AVANT de répondre.
---

# Serveur MCP « bibliotheque »

Serveur MCP distant exposant une médiathèque : catalogue de livres, adhérents,
prêts, frais et missions. Douze outils, tous préfixés côté agent par
`bibliotheque_`.

## Règle d'or avant toute conclusion

Le serveur **ne lève jamais d'erreur pour un filtre ou une pagination
invalides** : il renvoie silencieusement une réponse vide ou incomplète avec
`"ok": true`. Une réponse « vide » ou « normale » n'est donc **pas** une
preuve d'absence. Ne conclus jamais à partir d'un seul appel : vérifie par un
second chemin (un autre outil, une autre clé de tri) avant d'affirmer un
compte, une absence ou une date.

---

## Comportements normaux (documentés dans la description des outils) — ce ne sont PAS des pièges

- `list_books` exclut les exemplaires archivés **sauf** avec
  `include_archived: true` (écrit dans la description de l'outil).
- `list_loans` renvoie les prêts **dans l'ordre d'enregistrement (par
  `loan_id`)** (écrit dans la description).
- `start_key` est une **clé opaque** renvoyée par le champ `next` d'un appel
  précédent — il ne faut pas la fabriquer (voir toutefois le piège D).
- `limit` par défaut **20** pour tous les listages (écrit dans la description).
- `get_member` attend le paramètre `memberId` (camelCase), pas `member_id` :
  `get_member { "member_id": "MB-214" }` → `{ "ok": false, "error": "invalid request" }`.
- Les erreurs arrivent en **HTTP 200** avec `{"ok": false, "error": "..."}`.
  `get_member`/`get_book` inconnus → `{ "ok": false, "error": "not found" }`.

---

## Pièges (comportements non documentés)

### A. `count_books` compte aussi les archivés et ignore tout paramètre

- **Outil concerné** : `count_books` vs `list_books`.
- **Ce qu'on observe** : `count_books {}` → `{ "ok": true, "count": 184 }`.
  `list_books` paginé, **sans** `include_archived`, renvoie **158** notices ;
  avec `include_archived: true`, il renvoie **184** notices dont **26** avec
  `"archived": true`. En outre `count_books { "include_archived": false }` →
  `{ "ok": true, "count": 184 }` et `count_books { "genre": "roman" }` →
  `{ "ok": true, "count": 184 }` : les paramètres sont acceptés puis ignorés.
- **Ce que fait réellement le serveur** : le « count » est la taille brute du
  catalogue, **archivés inclus (184)** ; le périmètre en circulation est de
  **158** notices. `count_books` **ignore tout paramètre** : on ne peut pas
  lui faire compter autre chose (par genre, par statut…).
- **Règle à appliquer** : ne jamais présenter `count` comme le nombre de
  livres en circulation, et ne pas tenter de filtrer `count_books`. Pour un
  comptage ou une répartition, paginer `list_books` (en précisant le périmètre
  retenu) et compter les lignes.

Preuve : `preuves/13-parametres-non-declares-ignores.json`.

### B. `limit` silencieusement plafonné à 50

- **Outil concerné** : `list_books`, `list_loans`, `list_members`,
  `search_books`.
- **Ce qu'on observe** : `list_books { "limit": 100 }` → 50 éléments et
  `"next": "NTA="` (50 en base64). Idem avec `start_key`.
- **Ce que fait réellement le serveur** : toute valeur de `limit` > 50 est
  **ramenée à 50**, sans avertissement.
- **Règle à appliquer** : ne demander que `limit: 50` et **paginer** via
  `next`/`start_key` pour couvrir l'intégralité d'une collection.

### C. `next` est toujours présent, même sur une page vide ou terminale

- **Outil concerné** : `list_books`, `list_loans`, `list_members`,
  `search_books`.
- **Ce qu'on observe** : `list_books { "limit": 50, "start_key": "MjAw" }` →
  `{ "ok": true, "items": [], "next": "MjUw" }` (page vide mais `next`
  renseigné).
- **Ce que fait réellement le serveur** : `next` est la clé de la page
  suivante **calculée par décalage** (`NTA=`=50, `MTAw`=100, `MTUw`=150…),
  pas un indicateur de fin de collection.
- **Règle à appliquer** : déterminer la fin de pagination par
  **`len(items) < 50`**, jamais par l'absence de `next`. Ne pas boucler
  indéfiniment sur `next`.

### D. `start_key` invalide est ignoré (retour à la page 1)

- **Outil concerné** : tout outil paginé.
- **Ce qu'on observe** : `list_books { "limit": 50, "start_key": "pas-une-cle" }`
  → `{ "ok": true, "items": [ { "book_id": "BK-1000", ... }, ... ] }` (première
  page, comme si la clé n'existait pas).
- **Ce que fait réellement le serveur** : une clé illisible est **ignorée** et
  la page 1 est renvoyée avec `"ok": true`.
- **Règle à appliquer** : ne jamais fabriquer ni deviner une `start_key` ;
  la propager telle quelle depuis `next`. Un décalage « bizarre » ou une
  répétition de la page 1 signale une clé invalide.

### E. Valeur de filtre inconnue → liste vide (pas d'erreur)

- **Outil concerné** : `list_books` (`genre`), `list_loans` (`status`,
  `member_id`).
- **Ce qu'on observe** :
  `list_books { "genre": "Roman" }` → `{ "ok": true, "items": [], "next": "NTA=" }` ;
  `list_books { "genre": "poesie" }` → vide (il faut la casse/accents exacts
  `poésie`) ; `list_loans { "status": "overdue" }` → vide (`status` n'accepte
  que `open` ou `returned`) ; `list_loans { "member_id": "MB-999" }` → vide.
- **Ce que fait réellement le serveur** : un filtre non reconnu ne matche rien
  et renvoie une liste vide avec `"ok": true`.
- **Règle à appliquer** : n'interpréter un vide comme « aucun résultat » que
  si la valeur du filtre est **exactement** une valeur autorisée. Pour un
  genre, utiliser la liste fermée : `roman, policier, jeunesse, essai, bd,
  poésie` (minuscules, accents). Filtrer les retards via `due_at`/`status`,
  pas via `status:"overdue"`.

### F. Paramètre non déclaré dans le schéma : ignoré en silence, réponse NON filtrée

- **Outil concerné** : les outils de lecture (`list_loans`, `list_books`, …).
- **Ce qu'on observe** : `list_loans { "limit": 50, "book_id": "BK-1016" }` →
  `{ "ok": true, "items": [ … ], "next": "NTA=" }` : 50 prêts de **tous** les
  ouvrages renvoyés, alors qu'**un seul** porte `book_id: "BK-1016"`. De même,
  `list_loans { "limit": 50, "desk_code": "C3" }` → 50 prêts, dont **16**
  seulement ont `desk_code: "C3"`. Ni `book_id` ni `desk_code` ne figurent
  dans le schéma de `list_loans` : les deux appels renvoient d'ailleurs
  **exactement la même page 1**.
- **Ce que fait réellement le serveur** : un paramètre absent du schéma est
  **ignoré** (réponse `"ok": true`), et le résultat **n'est pas filtré** : la
  page par défaut est renvoyée telle quelle.
- **Règle à appliquer** : n'utiliser **que** les filtres listés dans le schéma
  de l'outil. Tout autre critère (ouvrage, guichet, date, statut non prévu…)
  se filtre **côté agent après pagination complète** ; et il faut **vérifier
  sur les lignes reçues** que le filtre a bien été appliqué (sinon on manipule
  la liste non filtrée).

Preuve : `preuves/13-parametres-non-declares-ignores.json`.

### G. Limite de débit silencieuse (> 60 appels/min) → réponses vides

- **Outil concerné** : tous.
- **Ce qu'on observe** : sur 60 appels `count_books` consécutifs, les 60
  premiers renvoient `{ "ok": true, "count": 184 }`, le **61e** renvoie
  `{ "ok": true, "count": 0 }`. Une **nouvelle session MCP ouverte pendant la
  limitation reçoit elle aussi `count: 0`** : le compteur suit le **jeton**,
  pas la session. Le retour à la normale survient ~**60 s après le
  déclenchement** (deux mesures : ~60 s et ~61 s). Pendant la limitation :
  `get_member { "memberId": "MB-225" }` → `{ "ok": true, "member": null }` ;
  `list_loans { "member_id": "MB-225", "limit": 50 }` →
  `{ "ok": true, "items": [], "next": null }` ;
  `get_member_fees { "member_id": "MB-225" }` →
  `{ "ok": true, "member_id": null, "balance_due": 0 }`.
- **Ce que fait réellement le serveur** : au-delà d'environ **60 appels par
  minute et par jeton** (le compteur est **partagé entre sessions**), il
  renvoie des **réponses vides mais `"ok": true`** au lieu d'une erreur.
- **Règle à appliquer** : ne pas lancer de rafales d'appels en parallèle ;
  faire des appels séquentiels. **Signature de la limitation** : `count: 0`,
  `member: null`, `member_id: null`, liste vide avec `"next": null`. En cas de
  doute, attendre ~60 s et refaire l'appel ; ne pas conclure sur ces réponses.

### H. Unités des frais : durées en HEURES, tarif en CENTIMES, solde en EUROS

- **Outil concerné** : `get_member_fees`.
- **Ce qu'on observe** : `get_member_fees { "member_id": "MB-225" }` →
  `{ "ok": true, "member_id": "MB-225", "open_loans": 1,
  "overdue_duration": 4296, "late_fee_per_day": 15, "balance_due": 26.85 }`.
- **Ce que fait réellement le serveur** : `overdue_duration` est en
  **heures**, `late_fee_per_day` en **centimes**, `balance_due` en **euros**.
  Vérification : 4296 h = 179 jours ; 179 × 0,15 € = 26,85 €. À noter aussi :
  `overdue_duration` est **cumulée sur tous les prêts en retard** d'un
  adhérent (ex. 4 prêts → plusieurs milliers d'heures), et `open_loans`
  compte **tous** les prêts ouverts (pas seulement les retards).
- **Règle à appliquer** : convertir avant tout affichage : jours =
  `overdue_duration / 24` ; montant = `late_fee_per_day / 100 × jours`.
  Ne pas présenter `overdue_duration` comme des jours ni `late_fee_per_day`
  comme des euros.

### I. Horloge serveur figée

- **Outil concerné** : tous (champ `started_at` / `due_at` / calculs de
  retard).
- **Ce qu'on observe** : un prêt créé au moment de l'exécution porte
  `started_at: 1791277200`, qui correspond à **2026-10-06T09:00:00Z**. Et
  **tous** les `overdue_duration` de `get_member_fees` se recalculent
  exactement avec cette date : ex. `MB-225`, échéance `due_at: 1775811600` =
  **2026-04-10T09:00Z** → écart jusqu'à 2026-10-06 = 179 jours = **4296 h**.
  La valeur d'un même adhérent est restée **identique entre deux appels
  séparés de plusieurs heures**.
- **Ce que fait réellement le serveur** : il travaille avec une **date de
  référence figée** (2026-10-06T09:00:00Z), pas la date réelle du jour.
- **Règle à appliquer** : calculer tout retard par rapport à cette date
  serveur, jamais par rapport à la date système ; en cas de doute, la déduire
  d'un prêt fraîchement créé (`started_at`). Dans la réponse, **annoncer
  explicitement que le retard est calculé à la date du serveur et donner cette
  date** (2026-10-06).

### J. Trois formats de date différents selon la source

- **Outil concerné** : `list_loans`/`get_member_fees` (prêts), `list_books`/
  `get_book` (livres), `list_members`/`get_member` (adhérents).
- **Ce qu'on observe** : prêts → **secondes Unix** (`"due_at": 1775811600`) ;
  livres → **ISO 8601** (`"added_at": "2026-04-18T09:00:00.000Z"`) ;
  adhérents → **JJ/MM/AAAA** (`"joined_at": "17/04/2024"`).
- **Ce que fait réellement le serveur** : il n'utilise pas un format unique.
- **Règle à appliquer** : normaliser selon la source avant toute comparaison.
  Ne jamais comparer directement une chaîne ISO, un entier Unix et une date
  JJ/MM/AAAA.

### K. `create_loan` exige un champ `desk_code` non documenté

- **Outil concerné** : `create_loan`.
- **Ce qu'on observe** : `create_loan { "member_id": "MB-214",
  "book_id": "BK-1042" }` → `{ "ok": false, "error": "missing field" }`. Le
  schéma de l'outil ne liste que `member_id` et `book_id`.
- **Ce que fait réellement le serveur** : il attend un champ supplémentaire
  `desk_code` (valeurs observées : `A1`, `B2`, `C3`). Avec ce champ, la
  création réussit. La signification de ces codes n'est pas connue.
- **Règle à appliquer** — marche à suivre :
  1. `list_loans { "member_id": "<membre>", "limit": 50 }` et vérifier
     qu'aucun prêt `"status": "open"` n'existe déjà pour ce `book_id`. Si oui :
     **ne rien créer** et le signaler.
  2. Appeler `create_loan` avec `member_id`, `book_id` **ET** `desk_code`.
  3. Si le client refuse l'argument supplémentaire ou si le serveur répond
     encore `{ "ok": false, "error": "missing field" }`, **s'arrêter et le
     dire à l'utilisateur** plutôt que de multiplier les essais. Un appel
     JSON-RPC direct n'est possible qu'avec le jeton lu dans la variable
     d'environnement `BIBLIOTHEQUE_TOKEN` — **jamais en lisant `.env`, jamais
     en l'affichant**.
  4. Relire la fiche (`list_loans` du membre) et vérifier `loan_id`,
     `book_id` et `"status": "open"`.
  - Si l'utilisateur ne précise pas le guichet, **le lui demander** ou
    annoncer clairement la valeur choisie.

### L. `delete_loan` archive, il ne supprime pas

- **Outil concerné** : `delete_loan`.
- **Ce qu'on observe** : `delete_loan { "loan_id": "LN-5038" }` →
  `{ "ok": true, "deleted": true, "loan_id": "LN-5038" }`, mais
  `list_loans { "member_id": "MB-202", "include_archived": true }` renvoie
  toujours `LN-5038` avec `"archived": true`. Id inconnu :
  `delete_loan { "loan_id": "LN-9999" }` → `{ "ok": false, "error": "not found" }`.
- **Ce que fait réellement le serveur** : suppression logique (passage
  `archived: true`), la ligne reste présente.
- **Règle à appliquer** : `"archived": true` **EST l'état « supprimé »** de ce
  serveur. Avant toute suppression, lister avec `include_archived: true` et ne
  retenir comme cibles que les prêts qui correspondent à la demande **ET** ont
  `"archived": false`. Un prêt déjà `"archived": true` ne doit **pas** être
  re-supprimé (l'effet d'un second `delete_loan` n'est pas connu). S'il n'y a
  **aucune cible**, **ne rien écrire** et rendre compte de l'état constaté
  (liste des `loan_id` déjà archivés) en rappelant que l'effacement est
  logique. Après un `delete_loan`, contrôler avec
  `list_loans { "member_id": "…", "include_archived": true, "limit": 50 }` :
  les prêts y figurent encore avec `"archived": true`, donc **dire
  explicitement au demandeur que l'effacement est logique (archivage) et que
  les outils exposés ne permettent pas d'effacement physique** — ne pas
  écrire « ils ont disparu ».

### M. `search_books` n'est ni exhaustif ni aligné sur `list_books`

- **Outil concerné** : `search_books`.
- **Ce qu'on observe** :
  - `search_books { "query": "Leroy", "limit": 50 }` → 9 résultats, **sans**
    `BK-1116` (auteur « Fatou Leroy »), alors que `get_book { "book_id":
    "BK-1116" }` le renvoie bien.
  - `search_books { "query": "La Mémoire du dimanche" }` → seulement
    `BK-1148`, pas `BK-1116` qui porte exactement ce titre.
  - La recherche **inclut des archivés** (ex. `BK-1003` avec
    `"archived": true`) alors que `list_books` les exclut par défaut.
  - Insensible à la casse (`FLEUVE` = `fleuve`) mais **sensible aux accents**
    (`memoire` → 0 ; `mémoire` → des résultats).
- **Ce que fait réellement le serveur** : l'index de recherche est **en
  retard sur le catalogue** (hypothèse : index non à jour) ; il peut manquer
  des livres récents et inclut des archivés.
- **Règle à appliquer** : ne **jamais** conclure à une absence ni compter via
  `search_books`. Pour un comptage ou une exhaustivité, paginer `list_books`
  et recouper avec `get_book`. Mettre la bonne casse/accents.

### N. Adhérent sans e-mail : trois formes distinctes

- **Outil concerné** : `list_members`, `get_member`.
- **Ce qu'on observe** : sur 46 adhérents, l'email peut être
  `"email": null` (ex. `MB-203`), ou la **clé `email` totalement absente**
  (ex. `get_member { "memberId": "MB-206" }` → pas de champ `email`), et
  3 adhérents ont `"active": false` (dont 2 avec un email). Des adresses
  peuvent être **partagées** par plusieurs adhérents (ex. `MB-215` et
  `MB-240`).
- **Ce que fait réellement le serveur** : « pas de contact » se représente de
  deux façons différentes (valeur `null` ou clé manquante) ; l'activité est un
  axe séparé.
- **Règle à appliquer** : tester l'absence d'email en couvrant **les deux
  cas** (`email === null` ET clé absente), et traiter `active:false` à part.
  Ne pas déduire l'unicité d'un contact depuis l'adresse.

### O. Ne pas comparer/compter des dates « de tête »

- **Outil concerné** : raisonnement de l'agent sur `due_at` + horloge serveur.
- **Ce qu'on observe** : un tri manuel de dizaines d'horodatages Unix (ou une
  soustraction en secondes) produit des erreurs de classification ; le
  recoupement avec `get_member_fees` (`overdue_duration > 0`) révèle l'écart.
- **Ce que fait réellement le serveur** : rien d'anormal — c'est l'agent qui
  se trompe en comparant des grands entiers mentalement.
- **Règle à appliquer** : **calculer par script** (comparer `due_at` à
  l'horloge serveur, `status: "open"`), puis **recouper par un second chemin**
  (par ex. `get_member_fees` et vérifier `overdue_duration > 0` pour chaque
  emprunteur). Ne jamais annoncer un comptage de dates issu d'un calcul
  mental.

---

## Procédure de travail conseillée

1. Charger ce skill dès qu'un outil `bibliotheque_*` est en jeu.
2. Faire des appels **séquentiels** et peu nombreux (limite ~60/min).
3. Paginer avec `limit: 50` + `start_key` depuis `next`, s'arrêter quand la
   page a **moins de 50** éléments.
4. Normaliser les dates selon la source (J) et se caler sur l'horloge serveur
   (I).
5. Pour tout comptage/absence, **recouper par deux chemins** (jamais
   `search_books` seul).
6. Vérifier la signature de la limitation de débit avant d'exploiter une
   réponse vide (G).
7. **Avant toute écriture** (`create_loan`, `delete_loan`), **lire l'état
   actuel** et vérifier que l'action n'a pas déjà été faite ; une écriture
   inutile ne se rattrape pas.
8. Ne rien affirmer qui ne soit prouvé par une réponse brute.

## Étapes sûres par type de tâche

- **Compter le catalogue** : paginer `list_books` en **précisant le périmètre**
  dans la réponse (sans `include_archived` = en circulation ; avec = archivés
  inclus) et compter les lignes ; **la répartition par genre se calcule en
  comptant le champ `genre` des lignes de la liste paginée complète**, et le
  **total des genres doit égaler le nombre de lignes**. Ne pas utiliser
  `count_books` (A), et ne pas croire qu'un filtre non déclaré s'applique (F).
- **Trouver un retardataire** : `list_loans { "status": "open", "limit": 50 }`,
  paginer, garder `due_at` < horloge serveur ; recouper avec
  `get_member_fees` (H, I, O).
- **Créer un prêt** : `list_loans` du membre (anti-doublon) puis
  `create_loan` avec `member_id`, `book_id` et `desk_code` (K).
- **Nettoyer des prêts** (L) : (a) **état initial** —
  `list_loans { "member_id": "…", "include_archived": true, "limit": 50 }` ;
  (b) **cibles** = prêts correspondant à la demande **et** `archived: false` ;
  (c) `delete_loan` **uniquement** sur ces cibles ; (d) **contrôle final** avec
  `include_archived: true` ; (e) **compte rendu honnête** — c'est un
  **archivage**, pas un effacement physique. Si aucune cible (tous déjà
  `archived: true`), ne rien écrire et rendre compte de l'état constaté.
- **Préparer une relance** : partir des emprunteurs réellement en retard
  (recoupés), puis filtrer les joignables en couvrant `email === null`, clé
  absente et `active:false` (N).

## Points non vérifiés (à traiter avec prudence)

- Le passage effectif de `desk_code` **via l'outil agent** `create_loan`
  n'a pas été confirmé (la réussite a été observée via appel direct).
- Le comportement des **outils d'écriture** (`create_loan`, `delete_loan`)
  pendant la limitation de débit n'a pas été testé : **ne jamais écrire juste
  après une rafale d'appels**, et **toujours relire l'état après une
  écriture**.
- Le comportement d'un **second** `delete_loan` sur un prêt déjà archivé n'a
  pas été testé.
- La cause exacte du retard de l'index `search_books` (M) est inconnue
  (« index non à jour » n'est qu'une hypothèse).
