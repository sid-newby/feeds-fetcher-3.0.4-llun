# FeedsFetcher Cartography: Data Flows and Intersections (Comprehensive)

This file contains the full set of Mermaid diagrams mapping the backend behavior when implemented as a GitHub Action, plus per-directory cartography. Diagrams emphasize intersections where files collaborate in workflows.

Note on storageType:
- action.yml defaults to "database"
- createFeedDatabase() checks for "sqlite"
- createFeedFiles() runs when storageType !== 'sqlite'
- buildSite() sets NEXT_PUBLIC_STORAGE=files only when storageType == 'files'

Treat this when configuring workflows.

---

## 1) End-to-end GitHub Action Lifecycle

```mermaid
graph TD
    %% Workflow trigger to Action runtime
    A[".github/workflows/*"] --> B["action.yml<br/>(using: node20, main: action.js)"]
    B --> C["action.js<br/>(resolve Action path)"]
    C --> C1["corepack install"]
    C --> C2["yarn install"]
    C --> C3["node -r @swc-node/register index.ts"]

    %% Orchestrator
    C3 --> D["index.ts"]
    D --> E["setup()<br/>(action/repository.ts)"]
    D --> F{"storageType?"}
    F -->|sqlite| G["createFeedDatabase()<br/>(action/feeds/index.ts)"]
    F -->|else| H["createFeedFiles()<br/>(action/feeds/index.ts)"]
    D --> I["buildSite()<br/>(action/repository.ts)"]
    D --> J["publish()<br/>(action/repository.ts)"]

    %% Artifacts
    G --> G1["public/data.sqlite3<br/>(Action path)"]
    H --> H1["contents/*<br/>(feed snapshots)"]
    H --> H2["public/data/*<br/>(derived JSON)"]
    I --> I1["out/ copied to<br/>$GITHUB_WORKSPACE"]
    J --> J1["push -f HEAD:branch"]
```

---

## 2) Repository Operations (setup, build, publish)

```mermaid
graph TD
    A["index.ts"] --> B["setup()"]
    B --> C1["@actions/core inputs: token, branch"]
    B --> C2["@actions/github context"]
    B --> C3["Octokit listBranches"]
    B --> D{"branch exists?"}
    D -->|yes| E["git clone -b branch --depth 1"]
    D -->|no| F["git clone currentRef → git checkout -B branch"]

    A --> G["buildSite()"]
    G --> H1["rm -rf _next"]
    G --> H2["touch .nojekyll"]
    G --> H3["if storageType == 'files' → set NEXT_PUBLIC_STORAGE=files"]
    G --> H4["yarn build in Action path"]
    G --> H5["cp -rT out → $GITHUB_WORKSPACE"]

    A --> I["publish()"]
    I --> J1["optional CNAME from customDomain"]
    I --> J2["git config user/email"]
    I --> J3["git add -f --all; commit"]
    I --> J4["git push -f HEAD:branch"]
```

---

## 3) SQLite Database Pipeline

```mermaid
graph TD
    A["createFeedDatabase(githubActionPath)"] --> B["read opmlFile from $GITHUB_WORKSPACE"]
    B --> C["readOpml(opmlContent)<br/>(action/feeds/opml.ts)"]
    C --> D["copyExistingDatabase(publicPath)<br/>(database.ts)"]
    D --> E["getDatabase(publicPath)<br/>(knex sqlite)"]
    E --> F["createTables(knex)<br/>(runtime migrations)"]
    F --> G["createOrUpdateDatabase(knex, opml, loadFeed)"]
    G --> H["loadFeed(title, url)<br/>(fetch + parseXML)"]
    H --> I{"rss | atom?"}
    I -->|rss| J["parseRss()<br/>(parsers.ts)"]
    I -->|atom| K["parseAtom()<br/>(parsers.ts)"]
    J --> L["(Site, Entries)"]
    K --> L["(Site, Entries)"]
    L --> M["insertCategory / insertSite / insertEntry"]
    M --> N["removeOldCategories / Sites / Entries"]
    N --> O["cleanup (vacuum, pragmas)"]
    O --> P["public/data.sqlite3<br/>(Action path)"]
```

---

## 4) Files/JSON Pipeline

```mermaid
graph TD
    A["createFeedFiles(githubActionPath)"] --> B["opmlPath = $GITHUB_WORKSPACE/opmlFile"]
    B --> C["loadOPMLAndWriteFiles(contents/, opmlPath)"]
    C --> D["readOpml(opmlContent)"]
    D --> E["For each category → createCategoryDirectory()"]
    E --> F["For each feed → loadFeed(title, xmlUrl)"]
    F --> G{"rss | atom?"}
    G -->|rss| H["parseRss()"]
    G -->|atom| I["parseAtom()"]
    H --> J["write contents/{category}/{siteHash}.json"]
    I --> J["write contents/{category}/{siteHash}.json"]

    %% Derived datasets
    A --> K["prepareDirectories(DEFAULT_PATHS)"]
    A --> L["createRepositoryData(github.json)"]
    A --> M["createCategoryData()"]
    M --> M1["public/data/categories.json<br/>(summary)"]
    M --> M2["public/data/categories/{category}.json<br/>(entries)"]
    M --> M3["public/data/sites/{siteHash}.json<br/>(site + entries)"]
    A --> N["createAllEntriesData()"]
    N --> N1["public/data/all.json<br/>(all entries)"]
```

---

## 5) Frontend Runtime Data Access (Next.js)

```mermaid
graph TD
    A["Next.js app"] --> B["lib/page.tsx"]
    B --> C["getStorage(NEXT_PUBLIC_BASE_PATH)"]
    C --> D{"NEXT_PUBLIC_STORAGE?"}
    D -->|files| E["FileStorage"]
    D -->|sqlite| F["SqliteStorage"]

    E --> E1["fetch {basePath}/data/categories.json"]
    E --> E2["fetch {basePath}/data/categories/{category}.json"]
    E --> E3["fetch {basePath}/data/sites/{siteKey}.json"]
    E --> E4["fetch {basePath}/data/entries/{entryKey}.json"]
    E --> E5["fetch {basePath}/data/all.json"]

    F --> F1["sql.js-httpvfs worker"]
    F1 --> F2["open {basePath}/data.sqlite3"]
    F1 --> F3["load {basePath}/sqlite.worker.js,<br/>{basePath}/sql-wasm.wasm"]

    B --> G["CategoryList, ItemList, ItemContent<br/>(lib/components/*)"]
    B --> H["location + reducer<br/>(lib/reducers/path.ts, lib/utils.ts)"]
```

---

## 6) Data Model Overview (SQLite)

```mermaid
graph TD
    A["Categories<br/>(name PK)"]
    B["Sites<br/>(key PK, title, url, description, createdAt)"]
    C["SiteCategories<br/>(category, siteKey, siteTitle)"]
    D["Entries<br/>(key PK, siteKey, siteTitle, title, url, content, contentTime?, createdAt)"]
    E["EntryCategories<br/>(category, entryKey, entryTitle, siteKey, siteTitle, entryContentTime?, entryCreatedAt)"]

    A --> C
    B --> C
    B --> D
    D --> E
    A --> E

    %% Indices
    D --> D1["idx(siteKey, contentTime, createdAt)"]
    E --> E1["idx(category, siteKey, entryKey, entryContentTime)"]
```

JSON artifacts correspondence:

```mermaid
graph TD
    A["contents/{Category}/{siteHash}.json<br/>(site snapshot)"] --> B["public/data/sites/{siteHash}.json"]
    B --> C["public/data/categories/{category}.json"]
    C --> D["public/data/categories.json"]
    B --> E["public/data/entries/{entryHash}.json"]
    E --> F["public/data/all.json"]
```

---

## 7) Per-directory Cartography (files and intersections)

action/ and root:

```mermaid
graph TD
    A["action.yml"] --> B["action.js"]
    B --> C["index.ts"]
    C --> D["action/repository.ts"]
    C --> E["action/feeds/index.ts"]
    E --> F["action/feeds/opml.ts"]
    E --> G["action/feeds/file.ts"]
    E --> H["action/feeds/database.ts"]
    F --> I["action/feeds/parsers.ts"]

    %% Root configs
    J["package.json"] --> B
    K["next.config.ts"] --> L["app/*"]
    M["postcss.config.mjs"] --> L
    N["tailwind.config.ts"] --> L
    O["tsconfig.json"] --> P["TypeScript runtime via SWC"]
    P --> B
```

app/ and lib/:

```mermaid
graph TD
    A["app/layout.tsx"] --> B["app/page.tsx"]
    B --> C["lib/page.tsx"]
    C --> D["lib/components/CategoryList.tsx"]
    C --> E["lib/components/ItemList.tsx"]
    C --> F["lib/components/ItemContent.tsx"]
    C --> G["lib/components/BackButton.tsx"]
    C --> H["lib/components/ThemeToggle.tsx"]
    C --> I["lib/storage/index.ts"]
    I --> J["lib/storage/file.ts"]
    I --> K["lib/storage/sqlite.ts"]
    C --> L["lib/reducers/path.ts"]
    C --> M["lib/utils.ts"]
```

lib/storage/ runtime assets:

```mermaid
graph TD
    A["lib/storage/sqlite.ts"] --> B["public/sqlite.worker.js"]
    A --> C["public/sql-wasm.wasm"]
    D["lib/storage/file.ts"] --> E["public/data/*"]
```

example/:

```mermaid
graph TD
    A["example/readme.md"] --> B["example/feeds.opml"]
    B --> C["(Used by consumers as canonical input)"]
```

public/:

```mermaid
graph TD
    A["public/favicon.ico"]
    B["public/logo.svg"]
    C["public/site.webmanifest"]
    D["public/sqlite.worker.js"]
    E["public/sqlite.worker.js.map"]
    F["public/sql-wasm.wasm"]
    G["public/vercel.svg"]
```

tests and fixtures:

```mermaid
graph TD
    A["action/feeds/database.test.ts"] --> B["action/feeds/database.ts"]
    C["action/feeds/file.test.ts"] --> D["action/feeds/file.ts"]
    E["action/feeds/opml.test.ts"] --> F["action/feeds/opml.ts"]
    G["action/feeds/parsers#parseAtom.test.ts"] --> H["action/feeds/parsers.ts"]
    I["action/feeds/parsers#parseRss.test.ts"] --> H
    J["lib/utils.test.ts"] --> K["lib/utils.ts"]
    L["action/feeds/stubs/*.xml"] --> H
```

---

## 8) Intersection Flowcharts (process specifics)

index.ts orchestration:

```mermaid
graph TD
    A["index.ts"] --> B["setup()"]
    A --> C{"storageType?"}
    C -->|sqlite| D["createFeedDatabase(getGithubActionPath())"]
    C -->|else| E["createFeedFiles(getGithubActionPath())"]
    A --> F["buildSite()"]
    A --> G["publish()"]
```

feeds/index.ts dispatch:

```mermaid
graph TD
    A["feeds/index.ts"] --> B["@actions/core.getInput('storageType')"]
    B --> C{"=== 'sqlite' ?"}
    C -->|true| D["createFeedDatabase()"]
    C -->|false| E["createFeedFiles()"]
    D --> F["copyExistingDatabase() → getDatabase() → createTables()"]
    D --> G["createOrUpdateDatabase() → loadFeed() → parse*()"]
    E --> H["loadOPMLAndWriteFiles() → write contents/*"]
    E --> I["prepareDirectories() → createRepositoryData()"]
    E --> J["createCategoryData() → createSitesData()"]
    E --> K["createAllEntriesData()"]
```

publish cleanup list (files removed prior to publish):

```mermaid
graph TD
    A["publish()"] --> B["rm -rf (action.yml, index.js, package*.json, .gitignore, .prettierrc.yml, tsconfig.json, .eleventy.js, tailwind.config.js, webpack.config.js, .github, action, readme.md, app, pages, contents, browser, public, lib, .gitlab-ci.yml, yarn.lock, action.js, index.ts, next-env.d.ts, next.config.js, postcss.config.js, css, js)"]
    B --> C["git add -f --all; commit; push -f"]
```

---

## 9) Canonical Workflow (illustrative)

```mermaid
graph TD
    A["User Repo"] --> B[".github/workflows/build.yml"]
    B --> C["uses: llun/feeds@v3<br/>with: opmlFile, storageType, branch, token, customDomain"]
    C --> D["Action Runner"]
    D --> E["action.js → index.ts"]
    E --> F["setup → create* → build → publish"]
    F --> G["Branch with static site artifacts"]
```

---

## 10) File Inventory Covered (nodes appear in graphs above)

```mermaid
graph TD
    subgraph Root
    A1[".gitignore"]
    A2[".gitlab-ci.yml"]
    A3[".prettierrc.yml"]
    A4[".yarnrc.yml"]
    A5["action.js"]
    A6["action.yml"]
    A7["components.json"]
    A8["feeds.opml"]
    A9["index.ts"]
    A10["next-env.d.ts"]
    A11["next.config.ts"]
    A12["package.json"]
    A13["postcss.config.mjs"]
    A14["readme.md"]
    A15["tailwind.config.ts"]
    A16["tsconfig.json"]
    A17["yarn.lock"]
    end

    subgraph action/
    B1["action/repository.ts"]
    B2["action/feeds/index.ts"]
    B3["action/feeds/opml.ts"]
    B4["action/feeds/parsers.ts"]
    B5["action/feeds/file.ts"]
    B6["action/feeds/database.ts"]
    B7["action/feeds/stubs/*.xml"]
    B8["action/feeds/*.test.ts"]
    end

    subgraph app/
    C1["app/globals.css"]
    C2["app/layout.tsx"]
    C3["app/not-found.tsx"]
    C4["app/page.tsx"]
    end

    subgraph lib/
    D1["lib/page.tsx"]
    D2["lib/utils.ts"]
    D3["lib/utils.test.ts"]
    D4["lib/components/BackButton.tsx"]
    D5["lib/components/CategoryList.tsx"]
    D6["lib/components/ItemContent.tsx"]
    D7["lib/components/ItemList.tsx"]
    D8["lib/components/ThemeToggle.tsx"]
    D9["lib/reducers/path.ts"]
    D10["lib/storage/index.ts"]
    D11["lib/storage/file.ts"]
    D12["lib/storage/sqlite.ts"]
    D13["lib/storage/types.ts"]
    end

    subgraph public/
    E1["public/favicon.ico"]
    E2["public/logo.svg"]
    E3["public/site.webmanifest"]
    E4["public/sqlite.worker.js"]
    E5["public/sqlite.worker.js.map"]
    E6["public/sql-wasm.wasm"]
    E7["public/vercel.svg"]
    end

    subgraph example/
    F1["example/.gitignore"]
    F2["example/feeds.opml"]
    F3["example/readme.md"]
    end
```

This concludes the comprehensive data-flow and intersection cartography.
