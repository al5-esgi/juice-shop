# Variante GitLab CI - document interne, PAS un deck

Le module est produit **GitHub Actions d'abord** : c'est le substrat par défaut de tous les TP,
c'est ce que l'école peut faire tourner gratuitement (dépôts publics = runners hébergés illimités),
et c'est la version testée. Ce dossier fournit l'**équivalent GitLab CI** pour l'étudiant qui
préfère héberger son projet sur `gitlab.com`. Chaque TP concerné porte une annexe
« variante GitLab CI » qui renvoie ici.

## Ce qui change, et ce qui ne change pas

Ne change pas : les outils (Semgrep, gitleaks, `npm audit`, `osv-scanner`, Trivy, ZAP baseline),
leurs versions épinglées, les périmètres de scan, les seuils, les jeux de sortie de référence des
annexes A. Un `.sarif` produit sous GitLab est le même que sous GitHub.

Change :

| GitHub Actions | GitLab CI |
|---|---|
| `.github/workflows/<famille>.yml`, un fichier par famille | un seul `.gitlab-ci.yml` à la racine + `include: local` de `.gitlab/ci/<famille>.yml` |
| `on: [push, pull_request]` | déclencheurs par défaut (branches + MR via `rules` dans les templates) |
| `runs-on: ubuntu-latest` + `uses: actions/...` | `image:` au niveau du job, pas d'actions à composer |
| `actions/setup-node@v7` | `image: node:24` |
| step `continue-on-error: true` + step « seuil de blocage » lisant `steps.X.outcome` | job de scan `allow_failure: true` qui écrit son code retour dans un fichier + job `<famille>-gate` en stage suivant qui le relit. **Même découplage, même intention.** |
| `fetch-depth: 0` sur `actions/checkout` | `variables: { GIT_DEPTH: 0 }` sur le job |
| upload SARIF vers **Security > Code scanning** | `artifacts:reports:*` (voir la limite « tier » ci-dessous) |
| artefact de workflow (`actions/upload-artifact`) | `artifacts:paths:` |
| Dependabot | Renovate (auto-hébergeable) ou le Dependency Scanning natif |

## Limite importante : les tiers GitLab

Sur **GitLab Free** (le cas d'un compte étudiant standard sur `gitlab.com`) :

- les jobs de scan **tournent**, **cassent le pipeline**, et **publient leurs artefacts**
  (`.sarif`, `.json`) téléchargeables depuis la page du pipeline ;
- il n'y a **pas** de *Vulnerability Report*, **pas** de *Security Dashboard*, **pas** de widget
  sécurité dans la merge request, **pas** d'onglet « Security » agrégé. Ces vues sont **Ultimate**.

Conséquence pour le fil rouge : là où le TP GitHub dit « ouvrez l'onglet Security, catégorie
`semgrep` », la version GitLab dit « téléchargez l'artefact `semgrep.sarif` du job et dépliez-le ».
C'est exactement le geste déjà enseigné en **S8** (`jq` sur un SARIF). Le triage, l'ADR, le rapport
d'audit sont identiques : ils se construisent à partir du fichier, pas de l'UI.

Les fichiers `gl-*-report.json` (format natif GitLab) sont produits **en plus** du SARIF : sans
effet visible sur Free, ils rendent le pipeline directement exploitable pour l'étudiant qui aurait
accès à Ultimate (essai, licence éducation, GitLab auto-hébergé de l'école un jour).

## Runners

`gitlab.com` fournit des runners partagés Linux (exécuteur Docker) avec un quota de
**400 minutes CI/mois** sur Free. C'est peu : le `build` de Juice Shop (postinstall = compilation
du frontend Angular, 8 à 12 min) en consomme une bonne part. Conseils :

- limiter les déclencheurs aux branches de travail et aux MR, pas à chaque push (`rules:` dans
  `.gitlab-ci.yml`) ;
- pour le DAST et le scan d'image, préférer l'image publiée `bkimminich/juice-shop:v20.1.1` quand
  le propos n'est pas de tester un correctif (voir `dast.gitlab-ci.yml`) ;
- garder la reproduction locale (annexes B des TP) comme plan A en séance.

## Structure livrée

```
.gitlab-ci.yml                  racine : stages + include local
.gitlab/ci/build.yml            = ci.yml d'amorce (build + lint)          [fourni / S1]
.gitlab/ci/sast.yml             Semgrep + gitleaks                        [S3]
.gitlab/ci/supply-chain.yml     npm audit + osv-scanner + sbom + trivy    [S4]
.gitlab/ci/dast.yml             ZAP baseline                             [S5]
```

Les fichiers de ce dossier portent le suffixe `.gitlab-ci.yml` pour la lisibilité ; chez
l'étudiant ils sont renommés selon l'arborescence ci-dessus.

## Statut de test

Les YAML GitHub du module sont testés sur un vrai runner. Les YAML GitLab de ce dossier sont
**transposés ligne à ligne** depuis eux et vérifiés en syntaxe, mais **doivent recevoir un
passage à blanc sur un runner `gitlab.com`** avant d'être donnés en séance (le point sensible est
le DAST en `docker:dind`). Faire ce dry-run en même temps que le fork de référence, avant le
module.
