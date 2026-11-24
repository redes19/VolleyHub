# VolleyHub

Monorepo d’une application mobile de répertoire de tournois de volley.

## Structure

- `mobile/` : application React Native avec Expo, TypeScript et Expo Router
- `back/` : API NestJS avec Prisma et PostgreSQL

## Lancer le back avec Docker

Depuis la racine :

```bash
cp .env.example .env
docker compose up --build
```

L’API est disponible sur `http://localhost:3000` et sa route de santé sur `http://localhost:3000/health`.

Le Dockerfile peut aussi être utilisé seul depuis `back/` :

```bash
cd back
docker build -t volleyhub-back .
docker run --env-file ../.env -p 3000:3000 volleyhub-back
```

## Lancer l’application mobile

```bash
cd mobile
npm install
npx expo start
```
