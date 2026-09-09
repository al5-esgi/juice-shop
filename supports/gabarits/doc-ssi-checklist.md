# Checklist - documentation SSI livrée avec le projet

> Gabarit fourni (S9). À cocher pour la soutenance. Versionné dans
> `docs/security/doc-ssi-checklist.md`.

Le dossier de sécurité d'un projet, tel qu'on le laisse à une équipe qui reprend le code :

- [ ] **Contexte et biens essentiels** (`docs/security/contexte.md` ou en tête du threat model) :
      ce que le système protège, événements redoutés, gravité.
- [ ] **Threat model** (`docs/security/threat-model.md`) : DFD, trust boundaries, tableau STRIDE
      priorisé, correspondance EBIOS.
- [ ] **Pipeline de sécurité** : workflows `.github/workflows/` documentés (quel outil, quel seuil,
      quoi faire quand un job casse). Un `README` sécurité ou une section dédiée.
- [ ] **ADR sécurité** (`docs/adr/`) : au moins les décisions structurantes (correctif de
      conception, critères de priorisation).
- [ ] **Rapport d'audit** (`docs/security/rapport-audit.md`) : findings, preuves, priorisation.
- [ ] **Plan de remédiation** (`docs/security/plan-remediation.md`) : *conditionnel* - actions
      ordonnées, effort, responsable. Si absent, le justifier.
- [ ] **Registre des traitements** (`docs/security/registre-traitements.md`) : traitements de
      données personnelles, bases légales, durées, sous-traitants.
- [ ] **Points de conformité ouverts** : ce qui reste à traiter côté RGPD / NIS / sous-traitance.

Règle : un document manquant est acceptable s'il est **explicitement listé comme non fait et
pourquoi**. Un document absent et non mentionné est une non-conformité.
