# Rapport d'audit de sécurité applicatif - <nom du projet>

> Gabarit fourni (S8). Versionné dans `docs/security/rapport-audit.md`.

## Périmètre et méthode

- **Système audité** : <nom, URL du dépôt, commit>
- **Périmètre** : <ce qui est dans le périmètre / hors périmètre>
- **Méthode** : revue de code guidée par la checklist + résultats des outils de CI (SAST, SCA,
  secrets, conteneur, DAST) + tests manuels ciblés sur les trust boundaries du threat model.
- **Date** : <date> - **Auditeur** : <nom>

## Synthèse

| Sévérité | Nombre |
|---|---|
| Critical | |
| High | |
| Medium | |
| Low | |

## Findings

### F-01 - <titre court>

- **Sévérité** : High - **CVSS 4.0 (base)** : <score>
  `CVSS:4.0/AV:_/AC:_/AT:_/PR:_/UI:_/VC:_/VI:_/VA:_/SC:_/SI:_/SA:_`
  (score **et** vecteur, calculateur FIRST : <https://www.first.org/cvss/calculator/4.0>)
- **CWE** : CWE-<n>
- **Emplacement** : `<fichier>:<ligne>` ou `<endpoint>` - la ligne de la **cause**, pas celle où
  l'outil s'est arrêté
- **Description** : <en quoi consiste la faiblesse, au niveau conception>
- **Preuve reproductible** :
  ```
  <requête curl, capture, sortie d'outil - étapes exactes pour reproduire>
  ```
- **Impact** : <ce qu'un attaquant obtient> - **Exploitabilité** : <faible / moyenne / élevée>
- **Recommandation** : <correctif de fond, pas rustine>

### F-02 - ...

(3 à 5 findings au total)

## Classement priorisé

Ordonner les findings par risque décroissant. Risque approché par impact multiplié par
exploitabilité multiplié par exposition. Chaque axe est noté de 1 à 3, le produit va de 1 à 27.

| Rang | Finding | Impact | Exploitabilité | Exposition | Risque | CVSS | Justification du rang |
|---|---|---|---|---|---|---|---|
| 1 | F-0x | | | | | | |

> Ce classement doit diverger de l'ordre CVSS décroissant sur au moins un rang, et la raison de
> l'écart doit être écrite dans la colonne de justification.

> Critères de priorisation retenus : voir `docs/adr/0002-criteres-priorisation.md`.

## Amorce de plan de remédiation (si le temps le permet en S8, sinon S9)

| Finding | Action | Effort | Priorité | Responsable |
|---|---|---|---|---|
| F-01 | | S / M / L | | |
