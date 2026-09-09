# Politique de mise à jour des dépendances

Périmètre : le fork Juice Shop, 117 dépendances directes (66 de production, 51 de développement)
pour **1457 entrées** dans `package-lock.json`, dont 654 marquées `dev`. Autrement dit, pour un
paquet choisi à la main, une douzaine arrivent sans avoir été décidées. La politique porte sur les
1457, pas sur les 117.

## Cadence

- **Correctifs (patch)** : application automatique en continu. Une pull request par lot
  hebdomadaire, fusionnée dès que la CI est verte. Aucune revue humaine du contenu du diff, la
  compatibilité sémantique est présumée et c'est la CI qui l'infirme.
- **Versions mineures** : lot mensuel. Revue humaine du changelog des seules dépendances de
  production. Fusion sur CI verte.
- **Versions majeures** : au cas par cas, jamais en lot. Chaque montée est un ticket avec un
  effort estimé, parce qu'une majeure implique une lecture du code appelant, pas seulement un
  changement de numéro.
- **CVE critique activement exploitée** : hors cadence, sous **48 h**. Le critère déclencheur
  n'est pas la sévérité seule mais l'**exploitation constatée** (présence au catalogue KEV de la
  CISA, ou preuve de concept publique fonctionnelle). Une critique non exploitée et non
  atteignable depuis notre code suit la cadence normale.

Une précision qui évite la paralysie : la cadence s'applique aux dépendances **de production**.
Une vulnérabilité dans une devDependency n'est pas exécutée par les clients ; elle menace la
chaîne de build, ce qui est un risque réel mais d'une autre nature et d'une autre urgence.

## Qui décide

| Type de montée | Décide | Valide | Trace |
|---|---|---|---|
| Patch de sécurité | automatique (Dependabot / bot de MAJ) | la CI : `deps-scan`, `build`, `lint` verts | PR fusionnée, historique git |
| Mineure | le développeur qui prend le lot | relecture croisée par un pair | PR avec le changelog en description |
| Majeure sur une dépendance de production | le responsable technique du projet | test de non-régression sur le parcours d'achat | ticket + ADR si la montée change une interface publique |
| CVE critique exploitée | le responsable sécurité, sans attendre le comité | test de non-régression réduit au chemin affecté | ticket d'incident, daté, avec l'identifiant GHSA/CVE |
| Risque accepté (pas de correctif, ou coût jugé supérieur) | le responsable technique **et** le responsable sécurité | — | décision écrite, **nommée et datée**, avec une date de revue obligatoire |

La dernière ligne est celle qui compte : accepter un risque est une décision, pas une omission.
Sans nom et sans date de revue, ce n'est pas une acceptation, c'est un oubli déguisé.

## Lien MCO-MCS

Le **maintien en conditions opérationnelles** couvre déjà ce qui empêche l'application de
fonctionner : montées de Node, correctifs de bugs bloquants, compatibilité de la base. Il est
budgété dans la charge de run de l'équipe et déclenché par une panne ou une régression.

Le **maintien en conditions de sécurité** ajoute ce que rien ne déclenche côté fonctionnel : une
dépendance transitive qui devient vulnérable ne casse aucun test et ne fait tomber aucun service.
Sans budget dédié, elle n'est jamais traitée, parce qu'elle ne fait jamais mal. C'est précisément
le cas de `crypto-js` traité ci-dessous : l'application marche parfaitement avec.

Budget : le MCS est provisionné comme une part fixe de la capacité de sprint (de l'ordre de 10 %),
et non comme un projet ponctuel. Le job `deps-scan` de la CI en est le déclencheur : c'est lui qui
transforme une veille passive en tâche datée.

## Cas traité aujourd'hui

Chaîne réelle relevée sur le run `deps-scan` du 2026-09-09 :

```
juice-shop  ->  pdfkit ^0.11.0 (dépendance DIRECTE, génération des factures PDF)
                    -> crypto-js <=4.1.1 (dépendance TRANSITIVE, jamais choisie)
```

| Élément | Valeur |
|---|---|
| Paquet vulnérable | `crypto-js` (transitive) |
| Dépendance directe qui le tire | `pdfkit@^0.11.0` |
| Identifiants | GHSA-xwcq-pm8m-c4vf (PBKDF2 1000 fois plus faible que la spécification) et GHSA-rg76-677x-56q9 (entropie insuffisante) |
| Sévérité | **critical** |
| Correctif disponible | `pdfkit@0.20.2` |
| Est-ce un majeur ? | `"isSemVerMajor": true` |

**Le point à comprendre, et la raison d'être de cette section** : il n'existe aucun correctif à
appliquer sur `crypto-js`. Ce paquet n'a jamais été choisi, il ne figure pas dans le
`package.json`, et aucune décision de l'équipe ne l'a fait entrer. Le seul levier disponible est
une **montée majeure de `pdfkit`**, un paquet qui n'a lui-même aucun problème. Le coût du
correctif ne se mesure donc pas sur la vulnérabilité, mais sur la refonte de la génération de
factures que la montée majeure peut imposer.

**Décision retenue** : montée de `pdfkit` vers `0.20.2` planifiée, traitée comme une **majeure sur
dépendance de production** — donc décidée par le responsable technique, avec un test de
non-régression sur la génération des factures PDF, et **non** sous le délai de 48 h.

Justification du délai : les deux avis concernent les primitives PBKDF2 et la génération
d'aléa de `crypto-js`. Or notre code n'appelle jamais `crypto-js` directement — il est atteint
uniquement par `pdfkit`, pour le chiffrement de documents PDF, fonctionnalité que la boutique
n'utilise pas. La vulnérabilité est donc **présente mais non atteignable** par un chemin
d'exécution réel. C'est exactement le genre de nuance qu'aucun scanner ne peut trancher : `npm
audit` et `osv-scanner` remontent la présence, l'atteignabilité se décide en lisant le code.

Le risque n'est pas accepté pour autant, il est **planifié** : une dépendance non atteignable
aujourd'hui le devient dès qu'une fonctionnalité change, et personne ne relira cet arbitrage à ce
moment-là. Échéance retenue : prochain lot de montées majeures, avec revue de l'atteignabilité à
cette occasion.

## Ce que la politique ne couvre pas

Ce document décide de la cadence et des responsabilités, pas de l'exhaustivité. Les 47 autres
paquets affectés (7 critical, 21 high, 17 moderate, 3 low au 2026-09-09) suivent la même grille
sans être instruits un par un ici. La première application de cette politique consistera à trier
les 7 critical selon le critère d'atteignabilité illustré ci-dessus.
