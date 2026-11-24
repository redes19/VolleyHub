# Spécifications — App répertoire de tournois de volley

## Objectif et périmètre V1

Application mobile (Android d'abord, iOS ensuite) qui répertorie les tournois de volley et renvoie vers les posts Instagram des organisateurs et vers l'app d'inscription.

- Les tournois sont créés par des utilisateurs ayant le rôle **organisateur** (clubs ou diffuseurs), validés au préalable par un admin.
- Les utilisateurs peuvent mettre des tournois en favori et suivre des organisateurs pour être notifiés de leurs nouveaux tournois.
- Les inscriptions, le paiement, le nombre de places et la date limite d'inscription sont gérés par une app externe : notre app stocke seulement le lien d'inscription.
- Un tournoi a un seul format et une seule catégorie, et peut durer plusieurs jours avec des horaires différents chaque jour.

## Stack technique

Tout le projet est en TypeScript, pour partager les types entre l'app et l'API.

| Couche | Techno |
| --- | --- |
| App mobile | React Native + Expo (TypeScript), Expo Router |
| API | NestJS (TypeScript) |
| ORM | Prisma |
| Base de données | PostgreSQL |
| Authentification | JWT (Passport, `@nestjs/jwt`), mots de passe hachés avec bcrypt |
| Notifications push | Expo Push Notifications (`expo-notifications` côté app, `expo-server-sdk` côté API) |
| Validation | `class-validator` + `class-transformer` sur les DTO |
| Dev local | Docker + un seul `docker-compose.yml` |

## Structure du repo et docker-compose

Un monorepo avec un dossier par service. Le `docker-compose.yml` sert au développement local et lance l'API et la base. L'app mobile tourne hors Docker avec `npx expo start`.

```
volley-tournaments/
├── docker-compose.yml
├── .env.example
├── README.md
├── mobile/
│   ├── app.json
│   ├── package.json
│   └── src/
│       ├── app/
│       ├── components/
│       ├── services/
│       └── types/
└── back/
    ├── Dockerfile
    ├── package.json
    ├── prisma/
    │   └── schema.prisma
    └── src/
```

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data

  back:
    build: ./back
    depends_on:
      - db
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      JWT_SECRET: ${JWT_SECRET}
    ports:
      - "3000:3000"

volumes:
  db_data:
```

Le `Dockerfile` du back doit fonctionner seul (sans le compose), pour pouvoir le déployer tel quel sur Render. Toute la configuration passe par des variables d'environnement, listées dans `.env.example`.

## Rôles et permissions

Quatre niveaux d'accès. Le rôle est vérifié par un guard NestJS ; la propriété d'un tournoi est vérifiée dans le service.

| Rôle | Peut faire |
| --- | --- |
| Visiteur (non connecté) | Voir la liste et le détail des tournois publiés ou annulés, voir les profils organisateurs, ouvrir les liens Instagram et d'inscription |
| `USER` | Tout ce que fait le visiteur, plus : favoris, suivre des organisateurs, recevoir des notifications, faire une demande pour devenir organisateur |
| `ORGANIZER` | Tout ce que fait `USER`, plus : créer, modifier, publier, annuler et supprimer **ses propres** tournois |
| `ADMIN` | Tout, plus : recevoir et traiter les demandes organisateur, supprimer n'importe quel tournoi |

Deux contrôles distincts sur chaque action d'un organisateur :

1. Le guard vérifie le rôle (`@Roles('ORGANIZER', 'ADMIN')`).
2. Le service vérifie que `tournament.organizerId` correspond à l'`OrganizerProfile` de l'utilisateur connecté (sauf pour `ADMIN`).

## Modèle de données

Neuf modèles. Un tournoi appartient à un `OrganizerProfile` (créé quand l'admin accepte la demande) et possède un ou plusieurs `TournamentDay`.

- User 1 — 0..1 OrganizerProfile
- User 1 — n OrganizerRequest
- OrganizerProfile 1 — n Tournament
- Tournament 1 — n TournamentDay (au moins un)
- User n — n Tournament via Favorite
- User n — n OrganizerProfile via Follow
- User 1 — n Notification, User 1 — n PushToken

```prisma
enum Role {
  USER
  ORGANIZER
  ADMIN
}

enum OrganizerType {
  CLUB
  BROADCASTER
}

enum RequestStatus {
  PENDING
  APPROVED
  REJECTED
}

enum Surface {
  INDOOR
  BEACH
  GRASS
}

enum Format {
  TWO_PLAYERS
  THREE_PLAYERS
  FOUR_PLAYERS
  SIX_PLAYERS
}

enum Category {
  MALE
  FEMALE
  MIXED
}

enum TournamentStatus {
  DRAFT
  PUBLISHED
  CANCELLED
}

enum NotificationType {
  ORGANIZER_REQUEST_CREATED
  ORGANIZER_REQUEST_APPROVED
  ORGANIZER_REQUEST_REJECTED
  TOURNAMENT_PUBLISHED
}

model User {
  id                String             @id @default(uuid())
  email             String             @unique
  passwordHash      String
  displayName       String
  role              Role               @default(USER)
  createdAt         DateTime           @default(now())
  organizerProfile  OrganizerProfile?
  organizerRequests OrganizerRequest[] @relation("RequestAuthor")
  reviewedRequests  OrganizerRequest[] @relation("RequestReviewer")
  favorites         Favorite[]
  follows           Follow[]
  notifications     Notification[]
  pushTokens        PushToken[]
}

model OrganizerRequest {
  id               String        @id @default(uuid())
  userId           String
  user             User          @relation("RequestAuthor", fields: [userId], references: [id])
  type             OrganizerType
  organizationName String
  instagramHandle  String
  message          String?
  status           RequestStatus @default(PENDING)
  rejectionReason  String?
  reviewedById     String?
  reviewedBy       User?         @relation("RequestReviewer", fields: [reviewedById], references: [id])
  reviewedAt       DateTime?
  createdAt        DateTime      @default(now())
}

model OrganizerProfile {
  id               String        @id @default(uuid())
  userId           String        @unique
  user             User          @relation(fields: [userId], references: [id])
  type             OrganizerType
  organizationName String
  instagramHandle  String
  tournaments      Tournament[]
  followers        Follow[]
  createdAt        DateTime      @default(now())
}

model Tournament {
  id               String           @id @default(uuid())
  name             String
  description      String?
  address          String
  city             String
  postalCode       String
  surface          Surface
  format           Format
  category         Category
  levels           String[]
  instagramPostUrl String?
  registrationUrl  String?
  contact          String?
  status           TournamentStatus @default(DRAFT)
  organizerId      String
  organizer        OrganizerProfile @relation(fields: [organizerId], references: [id])
  days             TournamentDay[]
  favorites        Favorite[]
  createdAt        DateTime         @default(now())
  updatedAt        DateTime         @updatedAt
}

model TournamentDay {
  id           String     @id @default(uuid())
  tournamentId String
  tournament   Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  startsAt     DateTime
  endsAt       DateTime
}

model Favorite {
  userId       String
  tournamentId String
  user         User       @relation(fields: [userId], references: [id])
  tournament   Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  createdAt    DateTime   @default(now())

  @@id([userId, tournamentId])
}

model Follow {
  userId      String
  organizerId String
  user        User             @relation(fields: [userId], references: [id])
  organizer   OrganizerProfile @relation(fields: [organizerId], references: [id])
  createdAt   DateTime         @default(now())

  @@id([userId, organizerId])
}

model Notification {
  id                 String           @id @default(uuid())
  userId             String
  user               User             @relation(fields: [userId], references: [id])
  type               NotificationType
  title              String
  body               String
  tournamentId       String?
  organizerRequestId String?
  isRead             Boolean          @default(false)
  createdAt          DateTime         @default(now())
}

model PushToken {
  id        String   @id @default(uuid())
  token     String   @unique
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
}
```

Règles de validation :

- Un tournoi a au moins un `TournamentDay`, et pour chaque jour `endsAt` est après `startsAt`.
- `levels` contient au moins une valeur.
- Un utilisateur ne peut avoir qu'une seule demande `PENDING` à la fois.
- Les formats s'affichent « 2x2 », « 3x3 », « 4x4 », « 6x6 » dans l'app (une valeur d'enum Prisma ne peut pas commencer par un chiffre).

## Flux métier

Deux flux déclenchent des notifications. Chaque notification est enregistrée en base (`Notification`) puis envoyée en push sur tous les `PushToken` du destinataire via `expo-server-sdk`.

**Demande pour devenir organisateur**

1. Un `USER` envoie une demande : type (club ou diffuseur), nom de l'organisation, compte Instagram, message optionnel. La demande est créée en `PENDING`.
2. Tous les `ADMIN` reçoivent une notification `ORGANIZER_REQUEST_CREATED`.
3. L'admin vérifie le compte Instagram puis accepte ou refuse.
4. Acceptation, dans **une seule transaction Prisma** : la demande passe en `APPROVED` (avec `reviewedBy` et `reviewedAt`), le rôle devient `ORGANIZER`, l'`OrganizerProfile` est créé avec le type, le nom et l'Instagram de la demande.
5. Refus : la demande passe en `REJECTED` avec `rejectionReason`.
6. Le demandeur reçoit `ORGANIZER_REQUEST_APPROVED` ou `ORGANIZER_REQUEST_REJECTED`.

**Cycle de vie d'un tournoi**

1. L'organisateur crée le tournoi en `DRAFT`, invisible du public.
2. Il le publie directement (pas de validation admin en V1) : `DRAFT` → `PUBLISHED`.
3. À la publication, chaque utilisateur qui suit cet organisateur reçoit une notification `TOURNAMENT_PUBLISHED`. Seule la première publication notifie.
4. L'organisateur peut annuler : `PUBLISHED` → `CANCELLED`. Le tournoi reste visible avec la mention « annulé ».
5. La suppression est réservée au propriétaire et à l'admin ; elle supprime aussi ses jours et ses favoris.

## Modules NestJS et endpoints

Un module NestJS par domaine, chacun avec son controller, son service et ses DTO. Prisma est exposé par un `PrismaService` partagé.

```
back/src/
├── auth/
├── users/
├── organizer-requests/
├── organizers/
├── tournaments/
├── favorites/
├── follows/
├── notifications/
├── common/
│   ├── guards/
│   └── decorators/
└── prisma/
```

| Méthode | Route | Accès | Rôle |
| --- | --- | --- | --- |
| POST | `/auth/register` | Public | Créer un compte `USER` |
| POST | `/auth/login` | Public | Renvoie un JWT |
| GET | `/auth/me` | Connecté | Profil de l'utilisateur connecté |
| POST | `/organizer-requests` | `USER` | Envoyer une demande organisateur |
| GET | `/organizer-requests/me` | Connecté | Voir ses propres demandes |
| GET | `/organizer-requests?status=PENDING` | `ADMIN` | Lister les demandes |
| PATCH | `/organizer-requests/:id/approve` | `ADMIN` | Accepter une demande |
| PATCH | `/organizer-requests/:id/reject` | `ADMIN` | Refuser une demande avec une raison |
| GET | `/organizers/:id` | Public | Profil d'un organisateur et ses tournois publiés |
| GET | `/tournaments` | Public | Liste filtrable : ville, surface, format, catégorie, niveau, période |
| GET | `/tournaments/:id` | Public | Détail d'un tournoi avec ses jours |
| GET | `/tournaments/mine` | `ORGANIZER` | Ses tournois, brouillons compris |
| POST | `/tournaments` | `ORGANIZER` | Créer un tournoi en `DRAFT` |
| PATCH | `/tournaments/:id` | Propriétaire | Modifier un tournoi |
| PATCH | `/tournaments/:id/publish` | Propriétaire | Publier et notifier les abonnés |
| PATCH | `/tournaments/:id/cancel` | Propriétaire | Annuler |
| DELETE | `/tournaments/:id` | Propriétaire ou `ADMIN` | Supprimer |
| GET | `/me/favorites` | Connecté | Lister ses favoris |
| POST | `/tournaments/:id/favorite` | Connecté | Ajouter aux favoris |
| DELETE | `/tournaments/:id/favorite` | Connecté | Retirer des favoris |
| GET | `/me/follows` | Connecté | Lister les organisateurs suivis |
| POST | `/organizers/:id/follow` | Connecté | Suivre un organisateur |
| DELETE | `/organizers/:id/follow` | Connecté | Ne plus suivre |
| GET | `/me/notifications` | Connecté | Lister ses notifications |
| PATCH | `/notifications/:id/read` | Connecté | Marquer comme lue |
| POST | `/me/push-tokens` | Connecté | Enregistrer le token Expo du téléphone |
| DELETE | `/me/push-tokens/:token` | Connecté | Supprimer un token (déconnexion) |

Les routes publiques de tournois ne renvoient que les statuts `PUBLISHED` et `CANCELLED`, triés par date du premier jour.

## Écrans de l'app mobile

Navigation par onglets avec Expo Router. Les écrans d'organisateur et d'admin n'apparaissent que pour ces rôles.

| Écran | Pour qui | Contenu |
| --- | --- | --- |
| Liste des tournois | Tous | Cartes (nom, dates, ville, format, catégorie, niveaux), filtres |
| Détail d'un tournoi | Tous | Infos complètes, jours et horaires, boutons « Voir sur Instagram » et « S'inscrire », favori |
| Profil organisateur | Tous | Nom, type, lien Instagram, bouton suivre, liste de ses tournois |
| Connexion / inscription | Visiteur | Email, mot de passe, nom affiché |
| Favoris | `USER` | Tournois mis en favori |
| Notifications | `USER` | Historique, lues / non lues |
| Compte | `USER` | Profil, organisateurs suivis, bouton « Devenir organisateur » |
| Demande organisateur | `USER` | Formulaire : type, nom, Instagram, message ; statut de la demande |
| Mes tournois | `ORGANIZER` | Liste avec statut, créer, modifier, publier, annuler |
| Formulaire tournoi | `ORGANIZER` | Tous les champs, ajout de plusieurs jours avec horaires |
| Demandes à traiter | `ADMIN` | Liste des demandes `PENDING`, accepter ou refuser avec raison |

Les liens Instagram et d'inscription s'ouvrent avec `Linking.openURL` : l'app Instagram si elle est installée, sinon le navigateur. Le token push est enregistré auprès de l'API après la connexion.

## Déploiement

Chaque service est déployé séparément sur une offre gratuite ; seule la publication sur les stores est payante.

| Service | Hébergeur | Coût | Limite à connaître |
| --- | --- | --- | --- |
| API NestJS | Render, via le `Dockerfile` du back | Gratuit | Mise en veille après 15 min sans trafic, environ 1 min pour se réveiller |
| PostgreSQL | Neon | Gratuit | Ne pas utiliser le Postgres gratuit de Render, qui expire après 30 jours |
| Build Android et iOS | EAS Build (Expo) | Gratuit avec quota mensuel | Build iOS possible sans Mac |
| Android hors store | APK partagé directement | Gratuit | Pour la phase de test |
| Google Play | Play Console | 25 $ une fois | Pour la publication publique |
| App Store | Apple Developer Program | 99 $ par an | Nécessaire pour distribuer sur iOS |

Le back lance `prisma migrate deploy` au démarrage en production. L'URL de l'API est fournie à l'app mobile par une variable d'environnement Expo (`EXPO_PUBLIC_API_URL`).

## Hors périmètre V1

- Images (affiche du tournoi, logo de l'organisateur) : à ajouter plus tard avec un service de stockage.
- Validation des tournois par l'admin avant publication.
- Inscription et paiement dans l'app : gérés par l'app externe.
- Tournoi à plusieurs tableaux : on crée un tournoi par format et catégorie.
- Publication iOS sur l'App Store : après la sortie Android.
