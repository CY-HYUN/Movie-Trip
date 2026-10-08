# Movie Trip — Technical Details

Companion to the top-level [README](../README.md). Everything here is verified against the code in this repository; file paths are given so claims can be re-checked.

## Contents

- [System diagrams](#system-diagrams)
- [Database schema (15 tables)](#database-schema-15-tables)
- [API reference (14 handlers)](#api-reference-14-handlers)
- [Implementation walkthrough](#implementation-walkthrough)
- [Usage guide](#usage-guide)
- [UI screenshots](#ui-screenshots)
- [Project structure](#project-structure)

## System diagrams

<img src="images/시스템%20구성도.png" alt="System architecture diagram" width="70%" />

<img src="images/무비트립%20시스템의%20상세%20모듈%20및%20API.png" alt="Module and API details" width="70%" />

Three tiers:

1. **Client** — Next.js pages with Recoil state, Kakao Maps JS SDK (loaded client-side), and the browser Geolocation API.
2. **Application** — Next.js App Router serving 12 pages and 14 API handlers; all DB access goes through Prisma.
3. **Data / external** — PostgreSQL (originally Supabase-hosted) with 15 tables; Kakao Mobility REST API for route computation, called directly from the browser (client component `RouteKakaoMap.tsx`), not through the Next.js server.

Auth is a custom JWT implementation inside the API routes (`jsonwebtoken`); Supabase is used only as a Postgres host, not for its Auth product.

## Database schema (15 tables)

Defined in [`prisma/schema.prisma`](../prisma/schema.prisma), applied through 19 migrations.

**Core (3)**

| Model | Purpose | Notes |
|---|---|---|
| `User` | Accounts, points, soft delete | `userId`/`email` unique; `password` stored **plain text** (known limitation); `point` starts at 0; `deletedAt` for soft delete |
| `Post` | Board posts | Defined in schema; **no API route uses it** (planned feature) |
| `Comment` | Post comments | Same — schema-only |

**Content catalog (2)**

| Model | Purpose | Notes |
|---|---|---|
| `Movie` | 10 movies/dramas: title, plot, releaseDate, genre, audience, peekview, rating | Seed source: `public/movieData.json` |
| `MoviePlace` | 82 filming locations: place name/type, description, operating hours, address, `lat`/`lng` | Seed source: `public/data.json` (all 82 rows geo-tagged) |

**Regional places (8)** — one table per region, identical shape (`seqNo` PK, place name/type/description, category, address, `lat`/`lng`):

`SeoulPlace`, `IncheonPlace`, `GyeonggiPlace`, `GangwonPlace`, `ChungcheongPlace`, `GyeongsangPlace`, `BusanPlace`, `JeollaPlace` — 20 curated places each (160 total), seeded from the 8 Korean-named JSON files in `public/`.

**Engagement (2)**

| Model | Purpose | Notes |
|---|---|---|
| `MovieReview` | Reviews keyed by `movieTitle` with 1–5 `rating`, author relation | Newest-first listing |
| `UserSaveRoute` | Saved routes: `content` (title/region), `contentType`, `selectRoute` / `successRoute` (JSON strings), `progress` (0–100) | Progress updated by GPS check-ins |

## API reference (14 handlers)

All routes live under [`src/app/api/`](../src/app/api/). One quirk to know: the dynamic-segment routes (`[title]`, `[category]`) read their value from a **query parameter** of the same name that the client sends alongside the path — see `src/components/movie/MovieDetails.tsx` for the calling pattern.

### `POST /api/user/login`

Body `{ id, pw }`. Matches `userId` + `password` with a plain-text Prisma `findMany` (no hashing), rejects soft-deleted accounts, then signs a JWT (`{ userId, userName }`, 1-hour expiry, `TOKEN_SECRET_KEY`). Returns `{ data: { token, userInfo } }` or a 400-style message on mismatch. File: `src/app/api/user/login/route.ts`.

### `POST /api/user`

Body `{ id, pw, email }`. Creates the user with `point: 0`; returns 201. Uniqueness enforced by the schema (`userId`, `email`). Also exports an `OPTIONS` CORS-preflight handler (not counted among the 14). File: `src/app/api/user/route.ts`.

### `PUT /api/user`

Body `{ id }` (numeric `User.id`). Soft delete — sets `deletedAt` to now. Same file.

### `GET /api/auth?email=...`

Account recovery: looks the user up by email and sends their userId **and plain-text password** via Nodemailer/Gmail. SMTP credentials were hard-coded in this file; the handler now reads `MAIL_USER` / `MAIL_APP_PASSWORD` from env vars, and the old values were removed from the git history on 2026-10-08. They were public before that, so treat them as revoked. File: `src/app/api/auth/route.ts`.

### `GET /api/movie`

Returns all rows of `Movie`. File: `src/app/api/movie/route.ts`.

### `GET /api/movie/[title]?title=...`

Returns `{ findMovie, findMoviePlace }` — the movie row(s) plus all `MoviePlace` rows with that title. File: `src/app/api/movie/[title]/route.ts`.

### `GET /api/content?userId=...`

All `UserSaveRoute` rows for a user. File: `src/app/api/content/route.ts`.

### `DELETE /api/content?userSaveRouteId=...`

Hard-deletes one saved route. Same file.

### `GET /api/content/submit?userId=...`

Completed routes only (`contentType: "지역"`, `progress: 100`) — feeds the "total overview" page. File: `src/app/api/content/submit/route.ts`.

### `GET /api/content/[category]?category=...`

Regional place lookup: a switch on the category string (e.g. `종로-고궁`, `광주`, `중구 - 바다`) selects the matching regional table and returns its places ordered by `seqNo`. File: `src/app/api/content/[category]/route.ts`.

### `POST /api/content/[category]`

Saves a new route (`UserSaveRoute` create). Returns code 403 in the body if a route with the same `content` already exists.

### `PATCH /api/content/[category]`

GPS check-in endpoint. Finds the user's saved route, parses the `successRoute` JSON, guards against double check-ins ("already arrived"), marks the matched place `isSuccess: true`, and recomputes:

```
progress = round(visited_places / total_places × 100)
```

### `GET /api/review?movieTitle=...` / `POST /api/review`

Fetch reviews for a movie (newest first) / create a review `{ content, rating, movieTitle, authorId, authorName }`. File: `src/app/api/review/route.ts`.

## Implementation walkthrough

### Geolocation hooks

- [`src/hooks/useCurrentLocation.ts`](../src/hooks/useCurrentLocation.ts) — one-shot `getCurrentPosition` for initial map centering.
- [`src/hooks/useWatchLocation.ts`](../src/hooks/useWatchLocation.ts) — continuous `watchPosition` subscription. Its call on the saved-route page is commented out; the "기록시작" (start recording) button instead calls `getCurrentPosition()` once per tap (`handleCheckLocation` in `src/app/mypage/myRoute/page.tsx`).

### Haversine distance and the 10 m check-in

[`src/utils/util.ts`](../src/utils/util.ts):

```typescript
export function distance(lat1, lon1, lat2, lon2, dis) {
  const R = 6371;                       // Earth radius, km
  const dLat = deg2rad(lat2 - lat1);
  const dLon = deg2rad(lon2 - lon1);
  const a = Math.sin(dLat / 2) ** 2 +
            Math.cos(deg2rad(lat1)) * Math.cos(deg2rad(lat2)) * Math.sin(dLon / 2) ** 2;
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c * 1000 <= dis;           // meters vs threshold
}
```

The tracking page calls it with a **10-meter** threshold — `distance(lat, lng, latitude, longitude, 10)` in [`src/app/mypage/myRoute/page.tsx`](../src/app/mypage/myRoute/page.tsx) — for every stop on the route, then fires the PATCH described above. (Early prose drafts said 50 m or 100 m; the code says 10.)

### Kakao Mobility route optimization

[`src/components/movie/RouteKakaoMap.tsx`](../src/components/movie/RouteKakaoMap.tsx) posts the selected places to:

```
POST https://apis-navi.kakaomobility.com/v1/waypoints/directions
Authorization: KakaoAK <REST_API_KEY>
{ origin, destination, waypoints, priority: "distance", car_fuel: "GASOLINE", ... }
```

The response's road vertices become the polyline path. Rendering uses a triple-layer effect for readability: a 13px black outline, a 10px red body, and a 2px white dashed center line, plus custom start/waypoint/end marker images (`public/images/startLoad.png` etc.).

### Region polygons

[`src/components/movie/RegionKakaoMap.tsx`](../src/components/movie/RegionKakaoMap.tsx) draws province boundaries from the GeoJSON files in `public/` (e.g. `서울행정구역.json`) as Kakao `Polygon` overlays — used on the completed-routes overview to highlight regions the user has finished.

### State management

Two Recoil atoms under [`src/atom/`](../src/atom):

- `userStore.ts` — the logged-in user's id/email/name for the current tab session.
- `selectPlaceStore.ts` — the currently selected places for route building, shared between the list UI and the map component.

The JWT itself is stored in localStorage; [`src/components/AuthProvider.tsx`](../src/components/AuthProvider.tsx) is a small client-side guard that redirects to the login page when no token is present.

### Auth flow

Login returns a 1-hour JWT stored client-side; `middleware.ts` applies CORS headers to API routes; account deletion is a soft delete. There is no token-refresh or server-side session store — and see the README's Known Limitations for the plain-text password issue.

## Usage guide

1. **Sign up / log in** at `/` (User ID, email, password), landing on `/choice`.
2. **Pick a mode** — movies/dramas (10 titles) or regions (8 provinces).
3. **Movie route**: select a title → its filming locations load with map markers → tick the places you want → the Kakao logo button renders the optimized route → save.
4. **Region route**: pick a province → pick a category (e.g. 종로-고궁) → select places → save.
5. **Track progress**: My Page → saved routes → "start recording" → allow location permission → tapping the button within 10 m of a stop checks it off and updates the progress percentage (one position check per tap, no continuous tracking).
6. **Complete**: at 100% the route moves to the overview page, with the region polygon highlighted.
7. **Review**: movie pages accept 1–5 star reviews with text.

The leaderboard page (`/leaderboard`) renders the intended ranking/achievement UI with sample data — point-awarding server logic was not implemented.

## UI screenshots

| Figure | Screen |
|---|---|
| <img src="images/그림1.%20로그인%20전%20회원가입%20화면.png" width="45%" /> | Registration / login |
| <img src="images/그림2.%20로그인%20후%20메인%20화면.png" width="45%" /> | Main dashboard — 10 movie cards |
| <img src="images/그림3.%20영화%20선택%20후%20촬영지%20선택%20화면.png" width="45%" /> | Filming-location selection |
| <img src="images/그림4.%20각%20촬영지%20이동경로%20표시%20화면.png" width="45%" /> | Optimized route + progress tracking |
| <img src="images/그림5.%20각%20영화에%20대한%20리뷰%20화면.png" width="45%" /> | Review screen |

## Project structure

```
Movie-Trip/
├── prisma/
│   ├── migrations/                 # 19 migrations
│   ├── schema.prisma               # 15 models
│   └── seed.ts                     # active: Busan loader; other loaders commented out
├── public/
│   ├── data.json                   # 82 filming locations (all geo-tagged)
│   ├── movieData.json              # 10 movies/dramas
│   ├── 서울.json … 부산.json        # 8 × 20 regional places
│   ├── 서울행정구역.json 등          # region-boundary GeoJSON
│   └── images/                     # posters, region photos, route markers
├── src/
│   ├── app/
│   │   ├── api/                    # 9 route files, 14 handlers (see API reference)
│   │   ├── page.tsx                # login (home)
│   │   ├── signUp/ choice/ movies/[movie]/ region/[name]/[place]/
│   │   ├── mypage/ myRoute/ totalRoute/
│   │   └── leaderboard/
│   ├── atom/                       # Recoil: userStore, selectPlaceStore
│   ├── components/                 # 33 components (movie, region, leaderboard, common, chart)
│   ├── hooks/                      # useCurrentLocation, useWatchLocation
│   ├── type/                       # movie + map TypeScript types
│   └── utils/                      # Haversine distance, Prisma client, JWT helper
├── middleware.ts                   # CORS for /api/*
└── Final-Report, Presentation/     # original course report + slides
```
