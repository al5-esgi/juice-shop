# Registre des traitements de données personnelles - <nom du projet>

> Gabarit fourni (S9), inspiré du modèle CNIL simplifié. Versionné dans
> `docs/security/registre-traitements.md`.

## Responsable de traitement

- Organisme : <projet / étudiant>
- Contact : <email>
- DPO désigné : <oui / non / non applicable à ce stade>

## Traitements

### T-01 - <nom du traitement, ex. « Gestion des comptes utilisateurs »>

| Rubrique | Contenu |
|---|---|
| Finalité(s) | ex. authentifier les utilisateurs, personnaliser l'expérience |
| Base légale | ex. exécution du contrat, consentement, intérêt légitime |
| Catégories de personnes concernées | ex. utilisateurs inscrits |
| Catégories de données | ex. email, mot de passe haché, pseudo, historique de connexion |
| Données sensibles | ex. aucune / à préciser |
| Destinataires | ex. équipe technique uniquement |
| Sous-traitants | ex. hébergeur <nom>, service d'envoi d'email <nom> - contrat art. 28 ? |
| Transferts hors UE | ex. non / oui vers <pays>, garanties : <SCC, adequacy> |
| Durée de conservation | ex. compte actif + 2 ans, puis suppression |
| Mesures de sécurité | ex. mots de passe hachés (argon2), TLS, contrôle d'accès par rôle, journalisation |

### T-02 - <ex. « Messages temps réel » ou « Positions des livreurs »>

(même tableau)

## Points d'attention identifiés

- Minimisation : <données collectées mais non nécessaires ?>
- Durées : <une durée de conservation est-elle absente ou excessive ?>
- Sous-traitance : <un sous-traitant sans clause de sécurité (art. 28) ?>
- Information des personnes : <existe-t-il une politique de confidentialité ?>

Ces points alimentent le plan de remédiation (S9) au même titre que les findings techniques.
