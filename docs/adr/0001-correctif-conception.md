# ADR 0001 - Contrôle d'appartenance du panier dans la requête de chargement

- **Statut** : accepté
- **Date** : 2026-09-09
- **Décideur** : Alex
- **Porte sur** : `routes/basket.ts`, fonction `retrieveBasket()`

## Contexte

`GET /rest/basket/:id` charge le panier désigné par l'identifiant d'URL sans vérifier à qui il
appartient : `BasketModel.findOne({ where: { id } })`. La route est pourtant protégée en amont par
`app.use('/rest/basket/:id', security.isAuthorized())` dans `server.ts`. Ce middleware répond
« qui es-tu », il ne répond pas « as-tu droit à **cet objet**-là » — et il ne le peut pas, puisque
ce droit dépend de l'objet demandé, que le middleware ne charge pas.

**Source du finding** : ni Semgrep ni ZAP. Ligne **#1** du tableau STRIDE de
`docs/security/threat-model.md` (catégorie *Tampering*, trust boundary client vers API). Aucun outil
statique ou dynamique ne peut savoir que le panier n° 3 n'est pas le vôtre : il faut connaître le
modèle métier. C'est précisément ce que l'analyse de risque apporte et que l'outillage ne remplace
pas.

**Scénario de risque EBIOS** : un client authentifié incrémente l'identifiant de panier dans l'URL,
lit et modifie les paniers des autres clients. Rattaché à l'événement redouté **ER2** de
`docs/security/contexte.md` — « accès en lecture et en écriture aux paniers et commandes d'autres
clients », gravité 3, bien essentiel **BE2** (commandes et historique d'achat).

**Mesure avant correctif**, compte de Jim (identifiant 2), token valide :

```
basket 1 -> [200]    basket 2 -> [200]    basket 3 -> [200]    basket 4 -> [200]
```

## Décision

Le filtre d'appartenance est placé **dans la requête qui charge l'objet** :

```ts
const user = security.authenticatedUsers.from(req)
if (user?.data?.id == null) { res.status(401)...; return }
const basket = await BasketModel.findOne({ where: { id, UserId: user.data.id }, ... })
if (basket == null) { res.status(403)...; return }
```

Un panier qui n'appartient pas à l'appelant n'est **jamais chargé en mémoire**. La réponse est un
`403` indifférencié, qui ne distingue pas « ce panier n'existe pas » de « ce panier n'est pas le
vôtre » : sans quoi la route deviendrait un oracle d'énumération des paniers existants.

**Mesure après correctif**, même compte, même token :

```
basket 1 -> [403]    basket 2 -> [200]    basket 3 -> [403]    basket 4 -> [403]
```

`npx tsc --noEmit` et `npx eslint server.ts routes/basket.ts` sortent en code 0. Le
`solveIf(challenges.basketAccessChallenge, ...)` devenu inatteignable a été supprimé, ainsi que les
imports `challengeUtils` et `challenges` qu'il était seul à utiliser.

## Options écartées

### 1. Masquer l'accès dans le frontend Angular

Rejetée sans discussion : un contrôle côté client n'est pas un contrôle. La boucle `curl`
ci-dessus n'ouvre jamais le navigateur. C'est la **rustine**, elle est écrite ici pour qu'elle ne
revienne pas au prochain sprint sous couvert d'être « plus rapide à faire ».

### 2. Comparer `req.params.id` au `bid` du jeton, dans un middleware de route

C'est l'option sérieuse, et elle **fonctionne aujourd'hui**. Elle est écartée pour trois raisons,
par ordre d'importance :

1. **Elle sépare le contrôle du chargement.** Le middleware décide, puis la route charge. Toute
   nouvelle route qui chargera un panier — ou tout refactoring qui déplacera ce chargement — devra
   penser à réappliquer le contrôle. Le filtre dans la clause `where` est au contraire **hérité par
   construction** : on ne peut pas charger l'objet en oubliant de vérifier son propriétaire, parce
   que c'est la même opération.
2. **Elle fait confiance à une donnée portée par le client.** Le `bid` est une revendication du
   jeton, pas un fait de la base. Or la ligne **#2** du threat model établit que la clé privée RSA
   est en dur dans `lib/insecurity.ts:21` et que la clé publique est servie par `/encryptionkeys` :
   un jeton est forgeable, donc son `bid` aussi. Le `UserId` de la clause `where` est lu dans la
   base, pas dans l'entrée utilisateur.
3. **Le `bid` est un instantané pris à la connexion**, et le jeton vit 6 h (`expiresIn: '6h'`). Il
   se désynchronise dès qu'un panier change de propriétaire ou qu'un nouveau panier est créé
   pendant la session.

## Conséquences

**Ce que le correctif apporte.** Le scénario d'élévation horizontale sur `GET /rest/basket/:id` est
fermé, et le geste est reproductible : c'est le patron à appliquer partout où un objet est chargé
par un identifiant fourni par le client.

**Risque résiduel n° 1 - le correctif est ponctuel.** Seule `retrieveBasket()` est traitée. Les
autres routes qui chargent un objet par identifiant client — items de panier, commandes, cartes,
retours d'expérience — conservent le patron d'origine. Le correctif réduit ER2, il ne l'élimine
pas. *Assumé par Alex le 2026-09-09, à revoir lors de l'audit de S8, qui doit inventorier ces
routes.*

**Risque résiduel n° 2 - l'authentification reste cassée en amont.** `authenticatedUsers.from(req)`
s'appuie sur le jeton vérifié avec la clé publique exposée. Un attaquant capable de forger un jeton
au nom d'un client obtient le panier de ce client : le correctif empêche l'accès **horizontal**
entre comptes, il ne compense pas la ligne **#2** du threat model. Autrement dit, cet ADR traite
l'autorisation, pas l'authentification. *Assumé le 2026-09-09, dépend du correctif de la gestion
des clés, non planifié à ce jour.*

**Ce qu'il ne faut pas conclure.** Ce n'est pas « le problème est réglé ». C'est « un chemin est
fermé, deux restent ouverts et sont nommés ».

## Décision connexe - la CSP conserve `'unsafe-inline'`

Le second correctif de la séance (durcissement de la configuration Express : CORS restreint à
`server.baseUrl`, `Referrer-Policy: no-referrer`, HSTS 15552000 s, CSP complète) publie une CSP qui
conserve **`'unsafe-inline'` sur `script-src`**.

C'est un arbitrage, pas un oubli. Le retirer casse l'application : le `<script>` inline de
`frontend/src/index.html` et le `<link ... onload="this.media='all'">` généré par le build Angular
en dépendent, et la bannière de consentement disparaît. Une CSP qui casse la boutique ne protège
rien, elle se fait désactiver au premier incident de production.

Ce que cela laisse ouvert : la CSP ne protège **pas** contre l'injection de script inline. Elle
protège contre le chargement de script depuis une origine tierce, contre l'inclusion en iframe
(`frame-ancestors 'none'`), contre l'exfiltration vers une origine tierce (`connect-src 'self'`) et
contre le détournement de formulaire (`form-action 'self'`). Le XSS stocké via l'API REST reste
exploitable.

Condition de levée : passage du frontend à des scripts externes avec `nonce` ou `hash` par requête,
ce qui suppose de modifier le build Angular. *Risque résiduel assumé le 2026-09-09, à réexaminer
lors de la prochaine montée majeure du frontend.*

Effet mesurable attendu sur le scan ZAP, et qui doit être lu correctement : les alertes `10038`
(CSP Header Not Set) et `10098` (Cross-Domain Misconfiguration) disparaissent, et l'alerte `10055`
(CSP unsafe-inline) apparaît. **Ce n'est pas une régression** : ZAP ne pouvait pas juger la qualité
d'une CSP inexistante. Un risque invisible est devenu un risque nommé, mesurable et daté.
