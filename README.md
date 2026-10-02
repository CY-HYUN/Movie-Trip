# Movie Trip

A full-stack Next.js 14 web app that turns filming locations from Korean movies and dramas into GPS-tracked travel routes: pick a title, select its real shooting locations, get an optimized driving route from the Kakao Mobility API, and check in on-site as live GPS tracking marks each stop visited.

![Next.js](https://img.shields.io/badge/Next.js-14.2-black?logo=next.js)
![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue?logo=typescript)
![Prisma](https://img.shields.io/badge/Prisma-5.x-2D3748?logo=prisma)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Supabase-316192?logo=postgresql)

## Verified numbers

Every figure below is counted directly from this repository's code and data (path given so you can re-count):

| What | Count | Where |
|---|---|---|
| Geo-tagged filming locations | **82** (all with lat/lng) | `public/data.json` |
| Movies and dramas covered | **10** | `public/movieData.json` |
| Curated regional attractions | **160** (8 regions × 20) | `public/서울.json` … `public/부산.json` |
| GPS check-in threshold | **10 m** Haversine proximity | `src/utils/util.ts`, `src/app/mypage/myRoute/page.tsx` |
| Route optimization | Kakao Mobility multi-waypoint directions | `src/components/movie/RouteKakaoMap.tsx` |
| HTTP API handlers | **14** across 9 route files | `src/app/api/**/route.ts` |
| PostgreSQL tables (Prisma models) | **15** | `prisma/schema.prisma` |
| Prisma migrations | **19** | `prisma/migrations/` |
| Pages / React components | **12** pages, **33** components | `src/app/`, `src/components/` |

<img src="docs/images/그림4.%20각%20촬영지%20이동경로%20표시%20화면.png" alt="Optimized route rendered on Kakao Map with visited-location checkmarks" width="70%" />

*Optimized multi-stop route over Kakao Maps, with per-location completion tracking.*

## Quick Start

Clone the repo:

```bash
git clone https://github.com/CY-HYUN/Movie-Trip.git
cd Movie-Trip
npm install                     # postinstall runs `prisma generate`
```

Create a `.env` file in this folder:

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=public"
TOKEN_SECRET_KEY="any-random-string"            # JWT signing key
NEXT_PUBLIC_KAKAO_API_KEY="your-kakao-javascript-key"
NEXT_PUBLIC_KAKAO_REST_API_KEY="your-kakao-rest-api-key"
NEXT_PUBLIC_LOCAL_URL="http://localhost:3000"
```

Then:

```bash
npx prisma migrate dev          # creates the 15 tables
npx prisma db seed              # see data notes below
npm run dev                     # http://localhost:3000
```

Verified locally: `npm run dev` compiles and serves the login page (HTTP 200). DB-backed pages and API routes need a reachable PostgreSQL (notes below).

### Data notes (read before running)

- **Bring your own PostgreSQL.** The original Supabase instance from the course is no longer live — DB-backed API routes fail against the old connection string. Any Postgres works; a free Supabase project is what the app was built against.
- **All source data ships in the repo**: `public/data.json` (filming locations), `public/movieData.json` (titles), and 8 regional JSON files. No private or external dataset is needed.
- **The committed seed script loads only one region as-is.** `prisma/seed.ts` actively seeds the Busan table; commented-out loaders exist only for movies, filming locations, and Seoul (run once during development). The other 6 regional JSON files are imported but have no loader code — to seed the full dataset, uncomment those three blocks and copy the Busan loop for the remaining regions.
- **Kakao keys are free**: create an app at [developers.kakao.com](https://developers.kakao.com), enable the Web platform with `http://localhost:3000`, and copy the JavaScript and REST API keys. Map and routing features do not render without them.

## Architecture

```
Browser ── Next.js pages + Recoil state + Kakao Maps SDK + Geolocation API
   │
   ▼  HTTP
Next.js App Router ── 14 API handlers (/api/*) ── Prisma ORM
   │                                                 │
   ▼                                                 ▼
Kakao Mobility REST API                    PostgreSQL (15 tables)
(multi-waypoint route optimization)        users · movies · 82 filming places
                                           8 regional place tables · reviews
                                           saved routes + progress
```

One Next.js codebase serves both the UI (12 pages) and the backend (14 API handlers); all persistence goes through Prisma. Kakao services split cleanly: the Maps JS SDK renders client-side, the Mobility REST API computes driving routes server-of-record style, and the browser Geolocation API supplies live position for check-ins.

## How the core features work

**Route optimization** — selected places are sent to Kakao Mobility `POST /v1/waypoints/directions` (origin, destination, waypoints, `priority: "distance"`), and the returned road geometry is drawn as a triple-layer polyline (black outline, red body, white dashes) on the Kakao map.

**GPS check-in** — `useWatchLocation` wraps `navigator.geolocation.watchPosition()`; each position update runs a Haversine distance calculation (`distance()` in `src/utils/util.ts`) against the next unvisited stop. Within **10 m**, the client PATCHes `/api/content/[category]`, which flips that stop's `isSuccess` flag and recomputes `progress = visited / total × 100` on the saved route. Routes reaching 100% appear in the completed-routes overview.

**Auth and sessions** — custom JWT flow: login matches userId + password and signs a 1-hour token (`jsonwebtoken`); the token is kept in localStorage, user info in a Recoil atom, and a lightweight `AuthProvider` guard redirects unauthenticated visitors back to the login page; account deletion is a soft delete (`deletedAt` timestamp). Read the limitations section before judging this as production auth.

**Reviews** — per-movie reviews with 1–5 star ratings stored in `MovieReview`, listed newest-first on each movie page.

## API surface

| Method(s) | Path | Purpose |
|---|---|---|
| GET | `/api/movie` | List all movies/dramas |
| GET | `/api/movie/[title]` | One movie + its filming locations |
| GET / POST / PATCH | `/api/content/[category]` | Regional places / save a route / update progress |
| GET / DELETE | `/api/content` | List / delete a user's saved routes |
| GET | `/api/content/submit` | Completed (100%) regional routes |
| GET / POST | `/api/review` | Fetch / create movie reviews |
| POST | `/api/user/login` | Login, returns 1-hour JWT |
| POST / PUT | `/api/user` | Sign up / soft-delete account |
| GET | `/api/auth` | Account-recovery email |

Full request/response reference, schema detail, and implementation walkthrough: [docs/DETAILS.md](docs/DETAILS.md).

## Known limitations

This was a university team project (2024). The security shortcuts below are real and are listed instead of papered over:

- **Passwords are stored and compared in plain text.** bcrypt is not integrated anywhere (it is not in `package.json`), and the account-recovery email sends the user's password back in plain text.
- **SMTP credentials were hard-coded** in `src/app/api/auth/route.ts`; the handler now reads env vars (`MAIL_USER`, `MAIL_APP_PASSWORD`). The old values remain retrievable in git history until the history is scrubbed — treat them as compromised.
- **CORS is wide open** (`Access-Control-Allow-Origin: *`) on API responses.
- **Leaderboard and points are a UI prototype** — the ranking page renders sample data; no server code awards points yet.
- **No automated tests.**

## Tech stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14.2 (App Router), React 18, TypeScript 5 |
| State | Recoil 0.7 (session info + route-building selection) |
| Styling / UI | Tailwind CSS 3.3, Nivo charts |
| Data | Prisma 5 ORM, PostgreSQL |
| Auth / email | jsonwebtoken (JWT), Nodemailer |
| Maps / geo | Kakao Maps JS SDK, Kakao Mobility REST API, browser Geolocation API |

## Screenshots

| | |
|---|---|
| <img src="docs/images/그림3.%20영화%20선택%20후%20촬영지%20선택%20화면.png" alt="Filming location selection" width="100%" /> | <img src="docs/images/그림5.%20각%20영화에%20대한%20리뷰%20화면.png" alt="Movie review screen" width="100%" /> |
| Filming-location selection for a chosen movie | Per-movie review and rating screen |

All five UI screenshots plus the system diagrams are in [docs/DETAILS.md](docs/DETAILS.md).

## Repo layout

```
Movie-Trip/
├── prisma/            # schema (15 models), 19 migrations, seed.ts
├── public/            # 82 filming locations + 160 regional places (JSON), posters, GeoJSON
├── src/
│   ├── app/           # 12 pages + 9 API route files (App Router)
│   ├── components/    # 33 components (map, movie, region, leaderboard, common)
│   ├── atom/          # Recoil atoms (session, selected places)
│   ├── hooks/         # useCurrentLocation, useWatchLocation
│   └── utils/         # Haversine distance, Prisma client, JWT helpers
├── docs/              # DETAILS.md + screenshots
└── Final-Report, Presentation/   # original course report and slides
```
