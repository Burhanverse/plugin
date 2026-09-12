# Stremio Official Addons & Cinemeta Deep Probe Report

**Target Repositories & Services Probed:**
- Repository: `https://github.com/Stremio/stremio-official-addons` (Master branch & v2 crate/npm packages)
- Live Cinemeta Service: `https://v3-cinemeta.strem.io`
- Stremio Protocol Validation: `@stremio/stremio-core-validator` standards

---

## 1. Executive Summary

A deep architectural probe of the official Stremio addons repository (`stremio-official-addons`) and live production Cinemeta endpoints (`v3-cinemeta.strem.io`) reveals key architectural patterns that Stremio's core engine (`stremio-core`) relies on for premier UI behavior.

Cinemeta operates as a **pure metadata and catalog provider**, relying strictly on `catalog`, `meta`, and `addon_catalog` resources without handling streams. Stremio treats Cinemeta as the gold standard for catalog navigation, Discover filtering, rich artwork presentation, and deep linking.

Below is an in-depth breakdown of Cinemeta's exact protocol implementation, comparative analysis with `xies-catalog`, and concrete recommendations to elevate `xies-catalog` to official-grade quality.

---

## 2. Key Architecture Findings from `stremio-official-addons`

### A. Manifest Constraints & Guidelines
- **Strict Size Constraint**: The official build pipeline (`scripts/gen.js`) enforces an 8KB hard cap on manifest size:
  ```javascript
  if (JSON.stringify(descriptor.manifest).length > 8192) throw 'manifest bigger than 8kb - aborting!'
  ```
  `xies-catalog`'s manifest is currently ~2.2KB, well within safety margins.
- **Protected & Official Flags**: Official addons are registered with `flags: { official: true, protected: true }` in Stremio client registries to prevent unauthorized overrides.
- **Strict Schema Validation**: The build script passes every descriptor through `@stremio/stremio-core-validator`. Addons failing schema validation are rejected at runtime.

### B. Catalogs Layout & Required Filters
Cinemeta defines 8 dedicated catalogs across `movie` and `series`:
1. **Popular (`id: "top"`)**:
   - `extraSupported: ["search", "genre", "skip"]`
   - Predefined genre options (20 for movies, 22 for series).
2. **New (`id: "year"`)**:
   - `extraSupported: ["genre", "skip"]`
   - `extraRequired: ["genre"]`
   - **Critical Pattern**: In this catalog, `genre` options are **release years** (`["2026", "2025", "2024", ..., "1960"]`).
   - Stremio UI automatically renders an interactive Year picker when `extraRequired: ["genre"]` is set with numeric years.
3. **Featured (`id: "imdbRating"`)**:
   - `extraSupported: ["genre", "skip"]`
4. **Internal Feeds (`last-videos`, `calendar-videos`)**:
   - Bound to `lastVideosIds` and `calendarVideosIds` with `optionsLimit: 100` to feed Stremio's Library notifications and Calendar.

---

## 3. Deep Schema Comparison: Cinemeta vs. Xie's Catalog

### A. Catalog Item Meta Preview

| Field | Cinemeta Production Value | Current `xies-catalog` | Recommended Action |
| :--- | :--- | :--- | :--- |
| `id` | `tt28014327` | `asiaflix:1607_gelboys-2` | Maintain custom prefix (`asiaflix:`, `kissasian:`, `kisskhone:`) |
| `type` | `"movie"` or `"series"` | `"movie"` or `"series"` | Valid |
| `name` | String title | String title | Valid |
| `poster` | Full HTTPS URL | Full HTTPS URL | Valid |
| `posterShape` | `"poster"` or `"regular"` | `"poster"` | Valid |
| `background` | Full backdrop URL | Backdrop URL | Valid |
| `logo` | Transparent PNG logo URL | Not populated | **Add logo support** via TMDB `images.logos` |
| `description` | Full plot synopsis | Synopsis | Valid |
| `releaseInfo` | Year or Year range (`"2008–2013"`) | Year string | Valid |
| `genres` | Array of strings | Array of strings | Valid |
| `imdbRating` | Rating string (`"8.8"`) | Rating string | Valid |
| `trailers` | `[{ source, type: "Trailer" }]` | Populated on detail | Populated on detail |
| `trailerStreams` | `[{ title, ytId }]` | Not present | **Add `trailerStreams`** for Web/TV compatibility |
| `links` | Array of category links | Basic links | **Adopt Discover deep-links** (see Section 4) |
| `behaviorHints` | `{ defaultVideoId, hasScheduledVideos }` | Not in catalog feed | **Add to movies and series previews** |

---

### B. Series Video (Episode) Object Schema

Cinemeta implements dual-property aliasing across video objects for compatibility with all legacy and modern Stremio clients (Desktop v4, Web v5, Android TV, Mobile):

```json
{
  "id": "tt0903747:1:1",
  "name": "Pilot",
  "title": "Pilot",
  "season": 1,
  "episode": 1,
  "number": 1,
  "firstAired": "2008-01-21T05:00:00.000Z",
  "released": "2008-01-21T05:00:00.000Z",
  "rating": "7.7",
  "overview": "Episode synopsis...",
  "description": "Episode synopsis...",
  "thumbnail": "https://episodes.metahub.space/tt0903747/1/1/w780.jpg"
}
```

**Key Takeaways for `xies-catalog`:**
- Provide both `name` and `title`.
- Provide both `episode` and `number`.
- Provide both `overview` and `description`.
- Provide both `released` and `firstAired`.
- Provide `thumbnail` (still image from TMDB).

---

### C. Movie Metadata & BehaviorHints

Cinemeta handles movies distinctively:
1. `videos: []` (empty array, never fake episode arrays).
2. `behaviorHints`:
   ```json
   "behaviorHints": {
     "defaultVideoId": "tt1375666",
     "hasScheduledVideos": false
   }
   ```
   Setting `defaultVideoId: meta.id` instructs Stremio to immediately query stream providers for the movie upon clicking "Play".

---

## 4. Navigation & Discover Deep-Linking (`links`)

Cinemeta uses Stremio internal URI protocols to create an interconnected user experience:

### 1. In-Addon Discover Links for Genres
Instead of falling back to a global search, Cinemeta deep-links genres directly back into its own Discover feed:
```
stremio:///discover/{encoded_manifest_url}/{type}/{catalog_id}?genre={genre}
```
**Example:**
`stremio:///discover/https%3A%2F%2Fv3-cinemeta.strem.io%2Fmanifest.json/movie/top?genre=Action`

**Benefit for Xie's Catalog:**
Clicking "Romance" on any Asian Drama or Movie detail page will immediately take the user to Xie's Catalog Discover tab filtered by Romance, keeping the user immersed in curated Asian content.

### 2. Rating Badge Link
Cinemeta formats the IMDB rating as an active link:
```json
{
  "name": "8.8",
  "category": "imdb",
  "url": "https://imdb.com/title/tt1375666"
}
```
Stremio renders items with `category: "imdb"` as an official golden star rating badge.

### 3. Share Link
```json
{
  "name": "Inception",
  "category": "share",
  "url": "https://www.strem.io/s/movie/inception-1375666"
}
```

### 4. Cast & Crew Links
```json
{
  "name": "Leonardo DiCaprio",
  "category": "Cast",
  "url": "stremio:///search?search=Leonardo%20DiCaprio"
}
```

---

## 5. Trailer Delivery: `trailers` vs. `trailerStreams`

Modern Stremio interfaces (especially Stremio Web and Stremio Android TV) support direct in-player trailer playback via `trailerStreams`:

```json
"trailers": [
  { "source": "cdx31ak4KbQ", "type": "Trailer" }
],
"trailerStreams": [
  { "title": "Official Trailer", "ytId": "cdx31ak4KbQ" }
]
```

- Older desktop clients consume `trailers` (`source: YouTubeID`).
- Modern Stremio clients and streaming cards consume `trailerStreams` (`ytId: YouTubeID`).
- Sourcing both simultaneously from TMDB guarantees 100% device compatibility.

---

## 6. Actionable Roadmap for Xie's Catalog

| Priority | Feature / Improvement | Implementation Scope |
| :---: | :--- | :--- |
| **High** | **Dual Trailer Support** | Add `trailerStreams: [{ title, ytId }]` alongside `trailers: [{ source, type: "Trailer" }]` in TMDB enrichment. |
| **High** | **Discover Deep-Linking** | Replace `stremio:///search?search={genre}` with `stremio:///discover/{manifestUrl}/{type}/{catalogId}?genre={genre}`. |
| **High** | **Transparent Title Logos** | Fetch `images.logos[0].file_path` from TMDB and pass `logo` in `MetaDetail` and `MetaPreview`. |
| **Medium** | **Video Object Schema Aliasing** | Populate `name` + `title`, `episode` + `number`, `overview` + `description`, `released` + `firstAired` on series episodes. |
| **Medium** | **Movie BehaviorHints** | Set `behaviorHints: { defaultVideoId: meta.id, hasScheduledVideos: false }` for movie meta responses. |
| **Medium** | **Release Year Catalog** | Add a `year` ("Release Year") catalog with `extraRequired: ["genre"]` populated with years (`2026`, `2025`, `2024`, etc.). |
| **Low** | **IMDB Category Badge Link** | Format rating link with `category: "imdb"` and `name: rating` if IMDB ID or rating exists. |
