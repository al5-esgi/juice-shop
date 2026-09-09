# Triage des findings SAST et secrets - S3

- **Périmètre scanné** : `lib routes models` (Semgrep), arbre et historique git (gitleaks)
- **Outils** : Semgrep OSS 1.145.0 (`p/ci` + `.semgrep/`), gitleaks 8.x
- **Volumétrie** : 9 findings Semgrep (5 avec `p/ci` seul, +4 apportés par les règles ciblées)
- **Date** : 2026-09-09 - **Branche** : `tp3-sast`

Règle du jeu : aucun verdict n'est rendu sur le titre du finding. Chaque ligne ci-dessous a été
tranchée après ouverture du fichier.

## Findings triés

| Fichier:ligne | Règle | CWE | Sévérité | Vrai / faux positif | Action décidée |
| --- | --- | --- | --- | --- | --- |
| `routes/login.ts:34` | `express-sequelize-injection` (p/ci) | CWE-89 | error | **Vrai positif à corriger** | Correctif planifié : requête paramétrée. Rattaché à l'exigence **H #5** du threat model (pas de concaténation dans une clause de requête). Le second défaut de la même ligne fait l'objet d'une exigence distincte, voir ci-dessous |
| `lib/insecurity.ts:41` | `juiceshop-weak-hash-md5` (règle ciblée) | CWE-327 | error | **Vrai positif à corriger** | Correctif planifié : argon2id. Rattaché à l'exigence **H #4** du threat model. `p/ci` ne remonte pas ce finding : son catalogue `use-of-md5` couvre Clojure, Java et Python, pas JavaScript — d'où la règle ciblée |
| `lib/insecurity.ts:150` | `juiceshop-hardcoded-private-key` (règle ciblée) | CWE-798 | error | **Vrai positif à corriger** | Correctif planifié : secret HMAC distinct, hors du dépôt. Rattaché à l'exigence **H #2** du threat model |
| `routes/login.ts:64` | `generic-api-key` (gitleaks) | CWE-798 | high | **Vrai positif, risque accepté** | Exclusion tracée et datée dans `.gitleaksignore`. Voir la justification écrite ci-dessous |

## Instruction des trois cas

### `routes/login.ts:34` - deux problèmes sur une seule ligne

La ligne est :

```ts
models.sequelize.query(`SELECT * FROM Users WHERE email = '${req.body.email || ''}' AND password = '${security.hash(req.body.password || '')}' AND deletedAt IS NULL`, ...)
```

Elle porte **deux défauts de nature différente** :

1. **Le finding Semgrep, c'est l'injection SQL** : `req.body.email` est interpolé dans le littéral
   de requête sans paramétrage. C'est ce que `express-sequelize-injection` détecte, et c'est
   CWE-89.
2. **Le second défaut n'est pas ce finding** : `security.hash()` est le MD5 de
   `lib/insecurity.ts:41`. Un mot de passe est comparé sur son empreinte MD5 non salée. Ce défaut
   relève d'une **exigence de sécurité distincte** (exigence H #4 du threat model), remontée par
   une autre règle, sur une autre ligne, et corrigée par un autre correctif.

Conclusion de triage : corriger l'injection ne corrige pas le hachage, et inversement. Deux
tickets, pas un.

### `lib/insecurity.ts:150` - aucun littéral sur la ligne, et pourtant vrai positif

La ligne est :

```ts
const hmac = crypto.createHmac('sha256', privateKey)
```

Il n'y a effectivement aucun secret littéral ici : le premier réflexe serait de classer faux
positif. C'est faux. `privateKey` est la constante définie **ligne 21** du même fichier, qui
contient une clé RSA complète en dur. Semgrep propage les constantes : le finding désigne bien un
secret en dur, simplement à travers une indirection d'une variable.

Le verdict change l'action : ce n'est pas une exclusion, c'est un correctif. Et l'instruction met
au jour un défaut supplémentaire que ni la règle ni le titre du finding ne disent — **la clé privée
RSA de signature des jetons est réutilisée comme secret HMAC** pour le jeton « deluxe ». Un même
secret sert deux usages cryptographiques distincts, ce qui interdit toute rotation indépendante.

### `routes/login.ts:64` - faux positif ou risque accepté ?

La ligne est :

```ts
challengeUtils.solveIf(challenges.oauthUserPasswordChallenge, () => { return req.body.email === 'bjoern.kimminich@gmail.com' && req.body.password === 'bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=' })
```

Décodage de la valeur signalée :

```
bW9jLmxpYW1nQGhjaW5pbW1pay5ucmVvamI=  --base64-->  moc.liamg@hcinimmik.nreojb
                                       --inverse-->  bjoern.kimminich@gmail.com
```

Ce n'est donc **pas une clé d'API**, contrairement à ce que dit le nom de la règle. C'est l'adresse
e-mail du compte, inversée puis encodée en base64, utilisée **comme mot de passe** : le point de
la vulnérabilité pédagogique `oauthUserPasswordChallenge`, qui enseigne qu'un mot de passe dérivé
mécaniquement de l'identifiant n'est pas un secret.

Le verdict n'est donc **pas « faux positif »**, et c'est toute la distinction demandée :

- gitleaks a **raison** sur le fond : il y a bien un identifiant en clair dans le code source, et
  ce mot de passe est réellement valide pour un compte de la base de démonstration.
- gitleaks se trompe seulement sur la **classification** (`generic-api-key` au lieu d'un
  identifiant de compte). Une erreur de nom de règle n'est pas un faux positif.

Verdict retenu : **vrai positif, risque accepté**. Justification : cette valeur est du code de
challenge intrinsèque à la finalité de Juice Shop, qui est d'être délibérément vulnérable. La
supprimer reviendrait à retirer la vulnérabilité que l'application existe pour enseigner. Le risque
est nul hors de ce contexte, puisque le compte n'existe que dans le jeu de données de
démonstration. La décision est **datée et tracée dans `.gitleaksignore`**, et devrait être revue si
le fork était un jour déployé ailleurs qu'en local.

## Rattachement au threat model

| Finding | Ligne STRIDE (S2) | Exigence |
| --- | --- | --- |
| `routes/login.ts:34` | #5 - Tampering, injection | H - pas de concaténation dans une clause de requête |
| `lib/insecurity.ts:41` | #4 - Information disclosure, MD5 | H - dérivation de clé lente et salée |
| `lib/insecurity.ts:150` | #2 - Spoofing, clé privée en dur | H - secret hors du dépôt, rotationnable |
| `routes/login.ts:64` | **aucune exigence associée** | Trou du threat model de S2 : les identifiants en dur du jeu de données de démonstration n'y figuraient pas. À ajouter |

Le dernier rattachement est en soi un résultat : le tableau STRIDE de S2 ne couvrait pas les
identifiants en clair présents dans le code applicatif. C'est un angle mort révélé par l'outil, pas
par l'analyse.

## Écarts constatés entre les outils et le périmètre

- L'injection NoSQL identifiée en S2 (ligne #5 du threat model, `routes/chat.ts:149`,
  `$where: 'this.product == ' + productId`) **n'est pas remontée** par `p/ci`, alors que le fichier
  est dans le périmètre scanné. Aucun outil ne remplace la lecture : le finding vient du threat
  model, pas du SAST. À reprendre comme finding de revue manuelle en S8.
- gitleaks scanné en local sur l'arbre complet remonte 126 entrées, dont l'écrasante majorité vient
  de `build/`, du cache Angular et des fichiers de test, non versionnés ou hors périmètre. En CI,
  le scan porte sur l'historique git : seules les entrées réellement commitées comptent.

## Incident d'outillage : gitleaks et le compte d'organisation

Le job `secret-detection` a d'abord été écrit avec `gitleaks/gitleaks-action@v3`, comme le prévoit
le gabarit. Le premier run l'a fait échouer **en 0,0 seconde, sans produire d'artefact** : l'action
n'a jamais scanné quoi que ce soit. Cause : elle est gratuite sur un compte personnel mais exige
une licence sur un compte d'**organisation**, et ce fork appartient à `al5-esgi`.

Le job était donc rouge pour une raison qui n'a rien à voir avec la sécurité du code — le pire cas
pour une porte de CI, puisqu'il produit exactement le même signal qu'une vraie détection.

Décision : passage à l'image officielle `ghcr.io/gitleaks/gitleaks:v8.30.1` en `docker run`, qui
fait le même travail sans licence. La sous-commande retenue est `git` et non `dir`, pour scanner
l'historique et non l'arbre courant. Le même découplage que dans le job `sast` est conservé :
`continue-on-error` sur le scan, artefact publié dans tous les cas, puis step de seuil explicite.

## Une exclusion produite par le triage lui-même

Le scan de l'historique remonte à présent `docs/security/triage-s3.md` : ce document cite la chaîne
base64 de `routes/login.ts:64` pour justifier son verdict. Le finding est un **faux positif** au
sens strict — il ne s'agit pas d'un identifiant utilisable, mais d'une citation dans une analyse.

C'est un cas instructif : documenter un secret suffit à le faire redétecter. L'exclusion est tracée
au même titre que les autres, et elle illustre pourquoi une exclusion se pose sur une empreinte
précise et jamais sur la règle entière — désactiver `generic-api-key` pour faire taire cette ligne
aveuglerait l'outil sur les 60 autres.
