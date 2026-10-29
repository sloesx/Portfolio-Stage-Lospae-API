Sécurisation d'API - Application Lospae (Extraits de code)

Contexte
Ce dépôt présente des extraits de code réalisés lors de mon stage de première année de BTS SIO (option SLAM) au sein de l'entreprise éditant l'application Lospae.

Objectif principal : Sécuriser une API et implémenter un filtrage strict des données. L'enjeu technique majeur était de s'assurer qu'un étudiant connecté via l'application mobile ne puisse récupérer que ses propres notes, historiques et critères d'évaluation, et en aucun cas les données des autres élèves.

(Note : Pour des raisons de confidentialité liées à l'entreprise, ce dépôt ne contient que des extraits de la logique d'authentification et de filtrage, et non l'intégralité du code source).

Technologies utilisées
Framework : Symfony 6/7
API : API Platform
Base de données : MySQL, Doctrine ORM (QueryBuilder)
Sécurité : JSON Web Tokens (LexikJWTAuthenticationBundle)

1. Configuration de l'authentification (JWT)
Mise en place des firewalls et du contrôle d'accès dans security.yaml pour bloquer les routes de l'API en exigeant un token valide, tout en laissant la route de login publique.

YAML
```yaml
# config/packages/security.yaml
security:
    firewalls:
        login:
            pattern: ^/api/login
            stateless: true
            json_login:
                check_path: /api/login_check
                success_handler: lexik_jwt_authentication.handler.authentication_success
                failure_handler: lexik_jwt_authentication.handler.authentication_failure
        
        api:
            pattern: ^/api
            stateless: true
            jwt: ~

    access_control:
        - { path: ^/api/login, roles: PUBLIC_ACCESS }
        - { path: ^/api,       roles: IS_AUTHENTICATED_FULLY }
```

2. Filtrage global des données avec Doctrine (CurrentUser)
Afin d'intercepter les requêtes avant qu'elles n'atteignent la base de données, j'ai créé une extension Doctrine (CurrentUserExtension.php) implémentant QueryCollectionExtensionInterface et QueryItemExtensionInterface.

Voici la logique algorithmique utilisant le QueryBuilder pour réaliser les jointures SQL nécessaires selon l'entité demandée :

PHP
```php
<?php
// src/Doctrine/CurrentUserExtension.php

// 1. Filtrage simple pour la table Note
if (Note::class === $resourceClass) {
    // La colonne 'student' de cette note doit correspondre à l'étudiant actuellement connecté
    $queryBuilder->andWhere(sprintf('%s.student = :utilisateur', $alias));
    $queryBuilder->setParameter('utilisateur', $etudiantConnecte);
}

// 2. Filtrage de l'historique (NoteChange) via une jointure simple
if (NoteChange::class === $resourceClass) {
    // Jointure SQL vers la table "Note" associée
    $queryBuilder->join(sprintf('%s.note', $alias), 'la_note');
    // Vérification que cette note jointe appartient à l'étudiant connecté
    $queryBuilder->andWhere('la_note.student = :utilisateur');
    $queryBuilder->setParameter('utilisateur', $etudiantConnecte);
}

// 3. Filtrage complexe des critères (Criteria) via une double jointure
if (Criteria::class === $resourceClass) {
    // Jointure SQL vers la table "Teacher" (Professeur)
    $queryBuilder->join(sprintf('%s.teacher', $alias), 'prof');
    // Deuxième jointure SQL vers la table "Note" appartenant au professeur
    $queryBuilder->join('prof.notes', 'les_notes');
    // Vérification de l'appartenance des notes à l'étudiant connecté
    $queryBuilder->andWhere('les_notes.student = :utilisateur');
    $queryBuilder->setParameter('utilisateur', $etudiantConnecte);
}
```
