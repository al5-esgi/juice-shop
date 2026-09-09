# Contexte de sécurité - fork Juice Shop

## 1. Contexte métier

OWASP Juice Shop est une boutique en ligne de jus de fruits et de produits dérivés, exploitée par
un commerçant qui vend directement à des particuliers (B2C) et, via une API dédiée, à des clients
professionnels (B2B). L'application porte l'intégralité du parcours d'achat : création de compte,
navigation dans le catalogue, dépôt d'avis, panier, commande, facturation PDF et suivi de livraison.
Elle manipule donc des données personnelles de clients (identité, adresses de livraison, historique
d'achat), des données d'authentification, et des documents de facturation. Une fuite de la base
clients expose le commerçant à une notification CNIL et à une perte de confiance durable ; une
indisponibilité prolongée lui coûte directement du chiffre d'affaires, puisque la boutique est son
seul canal de vente.

## 2. Biens essentiels

| # | Bien essentiel | Pourquoi il a de la valeur métier | Biens supports qui le portent |
|---|---|---|---|
| BE1 | Les comptes clients et leurs données personnelles | Identité, e-mail, adresses et mots de passe des clients. Leur divulgation est un incident RGPD à notifier, et elle détruit la confiance qui fait revenir l'acheteur. | Table `Users` en base SQLite via Sequelize ; jetons JWT signés (`lib/insecurity.ts`) ; cookies de session (`cookie-parser`, `server.ts`) |
| BE2 | Les commandes et l'historique d'achat | Preuve de la transaction pour le client comme pour le commerçant. Une altération fausse la facturation et le suivi de livraison ; une fuite révèle les habitudes de consommation des clients. | Tables `Baskets`, `BasketItems`, `Orders` (SQLite/Sequelize) ; endpoints `/rest/basket/:id` et `/b2b/v2/orders` ; factures PDF sur le système de fichiers (`ftp/`) |
| BE3 | Les moyens de paiement enregistrés | Cartes enregistrées et cartes-cadeaux. Leur compromission entraîne une fraude financière directe subie par le client, et engage la responsabilité du commerçant. | Tables `Cards` et `Wallets` (SQLite/Sequelize) ; serveur Express qui expose `/api/Cards` |
| BE4 | La disponibilité et l'intégrité de la boutique | Canal de vente unique du commerçant. Indisponible, elle ne vend pas ; défigurée ou peuplée de faux avis, elle vend mal et abîme la réputation de l'enseigne. | Process Express (`server.ts`) ; catalogue `Products` (SQLite) ; avis produits dans MarsDB (NoSQL) ; fichiers servis depuis `ftp/` et `logs/` |

## 3. Sources de risque

Trois profils ont un intérêt à s'en prendre à ces biens. Le **cybercriminel opportuniste** cherche à
revendre une base de comptes clients ou à monétiser des cartes de paiement : c'est la source la plus
probable, car elle scanne massivement sans cibler ce commerçant en particulier. Le **concurrent ou
faux avis rémunéré** vise le catalogue et les avis produits, pour dégrader la réputation de la
boutique ou promouvoir la sienne. Enfin, un **client malveillant authentifié** exploite sa position
légitime pour accéder aux paniers et aux commandes d'autres clients, ou s'octroyer un rôle
administrateur ; il n'a besoin d'aucun accès privilégié pour commencer, seulement d'un compte.

## 4. Événements redoutés

| # | Événement redouté (fait + impact) | Bien essentiel touché | Gravité (1 à 4) | Justification de la gravité |
|---|---|---|---|---|
| ER1 | Divulgation de la base des comptes clients (identités, e-mails et empreintes de mots de passe), préjudice RGPD et perte de confiance des acheteurs | BE1 | 4 | Données personnelles de la totalité des clients, notification CNIL obligatoire sous 72 h et information individuelle des personnes ; les empreintes sont en MD5, donc cassables, ce qui étend l'incident aux autres services où le client a réutilisé son mot de passe |
| ER2 | Accès en lecture et en écriture aux paniers et commandes d'autres clients par un client authentifié, altération de commandes et divulgation d'historiques d'achat | BE2 | 3 | Un simple compte client suffit, sans privilège particulier ; l'impact reste borné à un client à la fois, mais il est répétable à l'échelle et attaque directement la preuve de la transaction |
| ER3 | Indisponibilité de la boutique pendant 24 h en période de soldes, perte de chiffre d'affaires et report des clients vers la concurrence | BE4 | 3 | Canal de vente unique : une journée d'arrêt en pic commercial représente la perte sèche du chiffre d'affaires de la journée, sans destruction de données ni obligation réglementaire, et la situation est réversible |

## 5. Suivi

- Run `ci` de référence : https://github.com/al5-esgi/juice-shop/actions/runs/34329678449 (jobs `build` et `lint` verts, commit `42a515f`)
