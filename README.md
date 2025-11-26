---
layout: project
title: "PubSub"
slug: pubsub
guilds:
  - lorekeepers
link: "https://github.com/High-Desert-Institute/pubsub"
summary: >
  A C++17 Notcurses reader/publisher for RSS, Atom, JSON, Meshtastic, via http/https/ipfs/ipns/tor/etc with logic bots.
---

# PubSub — C++17 Notcurses RSS Reader & Publisher
A pubsub reader/publisher for rss/atom/json/meshtastic/etc which works on http/https/ipfs/ipns/tor and includes composable, functional logic "bots" including llms.

Read first:
- project-specification.md — Single consolidated source of truth (specification, roadmap, social context, and style guides). Start here.

## Project Documents

- project-specification.md — Consolidated technical specification, roadmap, social context, and style guides

## Overview

PubSub is a C++17 application with a high-performance TUI that:
- Reads and publishes across decentralised protocols
- Organises `subscriptions/` by subscription-type plugin (for example, `rss/`, `meshtastic/`) and stores per-source metadata in `index.md` front matter
- Caches payloads deterministically (`latest.<ext>` and `cache/YYYY-MM-DD-HH-mm-ss.<ext>`)
- Maintains a SQLite database for UI state, latest payload metadata, and parsed posts
- Provides a right-most logic/view column for Filters, Transformations, and Views
- Integrates a LAN-local LLM via Ollama and mesh messaging via Meshtastic

## Interface Philosophy: Speed & Muscle Memory

A central priority of PubSub is to provide a high-speed, high-bandwidth, low-latency interface that rewards mastery. Inspired by the numeric command systems of *Star Trek: Klingon Academy* and the zero-latency navigation of the Sidekick phone, the TUI is designed for rapid execution via muscle memory.

*   **Numeric Chaining:** Every menu item and action is mapped to a static, standardized number. The idea is that these are designed to remain constant forever and not change during updates or new versions so that uer muscle memory will continue to work over time. Users can execute complex, deep commands by typing rapid sequences (e.g., `2-1` to open Sub -> Subscriptions, select Refresh All, and close the menu) without waiting for UI animations. The general principle is that the lower numbers are the more common tasks at each level. Not all of the numbers need be assigned, so that future updates and plugins can add new commands without disrupting existing muscle memory. 
    * Pressing the first number opens that topic as the full-screen view. For example, pressing `2` would open the full-screen Subscriptions view, while `2-1` should queue a refresh all task and open the Refresh All modal. These kinds of modals should have the option to wait and watch them display status as they execute their task, or press escape to dismiss them. (They can still be re-opened from the tasks menu.)
*   **Base menu:**
        1. Home
            1. Show home/dashboard
        2. Pub
            1. New Post To All
            2. Show custom publish screen
        3. Sub
            1. Refresh All
        4. Tasks (number of pending tasks shown in parenthesis)
            1. New Task
        5. Bots
        6. Library
        0. Settings
            1. 
            2. Pub Settings
            3. Sub Settings
                1. HTTP/S Settings
                2. IPFS Settings
                3. Tor Settings
                4. Meshtastic Settings
            4. Task Settings
                1. Task Queue Settings 
            5. Bot Settings
                1. LLM Settings
            6. Library Settings
            9. Exit (opens confirmation modal)
*   **10-Key & D-Pad Parity:** The interface is fully navigable using interchangable approaches on different kinds of devices; for example, a standard desktop 10-key pad, a Steam Deck D-pad + A-Key, or arrow keys + enter. These should all be re-mappable for whatever kind of interface people want to use.
*   **Zero Latency:** Navigation through hierarchies, modals, and context menus must occur instantly with standard enter/escape for yes/no and otherise generally simplified and predictable inputs and results, allowing users to traverse complex trees faster than the screen can refresh.
*   **Muscle Memory:** Frequent actions become second nature through repetition, enabling users to operate the application with minimal cognitive load and maximum efficiency.
*   **Menu Appearance:** The base menu should be listed across the bottom of the screen, and submenus should pop-up from the relevant section of the base menu and then expand side to side through sub-menus. When a terminal command is selected, the menus should collapse back to the base menu.

## Architecture in Brief

- Source/Destination/Logic plugins — Registered via a PluginFactory with clear interfaces
  - RssHttpsPlugin (initial): fetch and parse RSS/Atom/JSON over HTTP/HTTPS
  - MeshtasticSourcePlugin / MeshtasticDestinationPlugin: map channels to subscriptions, watch outbox folders
  - OllamaLogicPlugin: send prompts to `/api/generate` and stream results
- Fetcher (background): Evaluates cadence per subscription, writes timestamped cache files, atomically updates `latest`, and synchronizes the SQLite database with filesystem state
- SQLite tables: subscriptions, fetches, latest_payloads, posts (see project-specification.md for schema)
- TUI (Notcurses): Base Menu navigation (Home, Pub, Sub, Tasks, etc.) with a three-column sub view and a logic/view pipeline in the right-most column
- Prompts: reusable LLM task definitions live under `prompts/` (for example, taxonomy generation)

## Subscriptions Directory Layout

- Root directory: `subscriptions/`
- Immediate children: subscription-type plugin namespaces (for example, `subscriptions/rss`, `subscriptions/meshtastic`). Transport helpers (HTTPS, Tor, IPFS, etc.) never create their own directories—they extend the owning subscription-type plugin.
- RSS/Atom sources: any subdirectory under `subscriptions/rss/` becomes a source when its `index.md` front matter defines all required keys (`source_name`, `source_uri`, `source_slug`, `file_extension`). Directories missing the file or any required field remain navigable categories.
- Meshtastic sources: device roots and channel folders live under `subscriptions/meshtastic/`; the plugin writes its own metadata schema to `index.md` files automatically.
- Cache data: each source may contain a `cache/` folder housing `latest.<ext>` and timestamped payloads (`cache/YYYY-MM-DD-HH-mm-ss.<ext>`); the name `cache` is reserved and skipped when scanning for sources.

## Logic Blocks, Filters, and Views

- Logic blocks: modular units that accept inputs and produce outputs; chain blocks to build workflows
- Data source: always derived from the left-tree selection via a database query that recursively collects child posts
- Filters & Transformations: SQL-based filters, computed fields, joins, and LLM-enrichment (tags, bias, summaries)
- Views: Data Table (default), Charts, Media playlist, Photo gallery, Rendered Markdown — each may include subqueries and prompts

## Meshtastic + Cyberpony Express

- When a Meshtastic device is connected, the plugin creates a device root under `subscriptions/meshtastic/` with per-channel child subscriptions and manages the necessary `index.md` metadata automatically
- Incoming messages become cached payloads and parsed posts
- Outgoing messages are queued via `outbox/` directories and sent by the destination plugin

## Ollama (LAN LLM Gateway)

- Default endpoint: http://ollama.local:11434/api/generate (configurable)
- Blocks can summarise, detect bias, or generate counter-arguments; results persist in the configuration database
- Streaming supported; failures are logged and presented in the UI gracefully

## Logging: Where and How

- Root-level logs/ directory with per-run files named `log.YYYY-MM-DD-HH-mm-ss-ffffff.txt`
- Also write a rolling root-level log.txt for quick inspection
- Capture CLI args, key actions, external process stdout/stderr, and outcomes with timestamps
- Follow the CLI Logging and Interpretability Style Guide section in project-specification.md for levels and message format

## Test Mode (Simulation)

- `--test` runs the app with temporary directories and a temp configuration database
- Simulates Ollama responses and Meshtastic packets without modifying real data
- Exercises the same code paths and produces the same logs for reproducibility

## Build and Run

These instructions are minimal and may need adjustment depending on your platform toolchain. See the CLI-First C++ Development Style Guide (Notcurses) section in project-specification.md for details.

- Linux/macOS (example): Use g++ and pkg-config with Notcurses
- Windows (MSYS2 MinGW-w64): Use the MSYS2 MinGW 64-bit shell and `make`

Example workflow:

1) Ensure Notcurses development files are available
2) Build with your Makefile or standard g++ command
3) Run the executable; consult logs/ for the newest per-run log file
4) Browse `prompts/` if you want to drive LLM-based automation (e.g., generating RSS taxonomy metadata)

## Windows build guide (complete)

Supported Windows flow: MSYS2 MinGW-w64 (GNU toolchain, Makefile-driven)

### MSYS2 MinGW-w64

MSYS2 provides a POSIX-like environment and MinGW-w64 compilers. Your Windows drives are mounted under `/c`, `/d`, etc. For example, `C:\Users\CJ\Documents\GitHub\pubsub` is `/c/Users/CJ/Documents/GitHub/pubsub` inside the MSYS2 shell.

Recommended shell: "MSYS2 MinGW 64-bit" (MSYSTEM=MINGW64)

Install toolchain (first time only):

```bash
pacman -S --needed mingw-w64-x86_64-toolchain mingw-w64-x86_64-notcurses make
```

Build and test:

```bash
cd /c/Users/CJ/Documents/GitHub/pubsub
make clean
make -j2

# Optional helpers added to the Makefile
make run     # runs --help, --version, --test, --init-default-subscriptions
make check   # smoke tests; fails on mismatch
```

Artifacts and bundling:

- Binary: `bin/pubsub.exe`
- The Makefile bundles MinGW runtime DLLs automatically on MSYS2/MINGW: `libstdc++-6.dll`, `libgcc_s_seh-1.dll`, `libwinpthread-1.dll` into `bin/`, so you can double-click `bin/pubsub.exe` from Explorer or run from PowerShell without modifying PATH.


Run CLI tests from PowerShell (after building):

```powershell
powershell -ExecutionPolicy Bypass -File .\run_cli_tests.ps1
```

### Running the executable from Explorer or PowerShell
- After an MSYS2/MinGW build, the Makefile’s `bundle-dlls` step (included in the default `all` target) copies required MinGW runtime DLLs to `bin/`. That allows double-click running outside the MSYS2 shell.
- If you built previously without bundling and see a missing `libstdc++-6.dll` error, run the provided helper to copy the DLLs:

```powershell
powershell -ExecutionPolicy Bypass -File .\ensure_mingw_runtime.ps1
```

### Notcurses headers
- The TUI requires Notcurses.
- Ensure `notcurses.h` is available in your include path (usually handled by `pkg-config`).


## Contributing (for Humans and Agents)

- Read the styleguides and AGENTS.md before contributing
- Keep new user-visible outputs and actions logged (concise, high-signal)
- Update project-specification.md (roadmap section) when features change
- Prefer small, well-scoped patches; ensure changes are testable with `--test`


# Project Specification: C++17 Notcurses RSS Reader & Publisher

Note: This is the single, consolidated source of truth for the project. It includes the technical specification, roadmap, social context, and relevant style guides. Where other standalone files are mentioned historically, their content now lives here.

## Purpose

This project aims to build a modern, cross‑platform **RSS reader and publisher** written in C++17 with a high-performance TUI built on **Notcurses**.  The application will allow users to subscribe to feeds from a wide variety of protocols (HTTP/HTTPS, IPFS/IPNS, Tor, ActivityPub, AT protocol, etc.) and content formats (RSS/XML, Atom, JSON), browse and filter posts, and publish content or comments back to supported protocols.  It replaces typical heavy frameworks with a minimal, CLI‑first design emphasising portability and reproducibility.

## Functional Requirements

1. **Subscription management**

  * Support hierarchical subscriptions stored in the on‑disk `subscriptions/` directory.  Each immediate child directory corresponds to a subscription-type plugin namespace (for example, `rss`, `atom`, `meshtastic`).  Transport helpers (e.g., HTTPS, Tor, IPFS) do not own directories; they extend the subscription-type plugins and leave layout unchanged.  Within each namespace, the owning plugin controls the rules for the subdirectories it manages.  For the RSS/Atom plugin, a subdirectory only becomes a source when it contains an `index.md` whose YAML front matter includes every required key: `source_name`, `source_uri`, `source_slug`, and `file_extension`.  Directories that lack the file or omit any required field remain navigable categories within that plugin even if they include other content or auxiliary `index.md` files.  Directories titled "cache" are reserved for cached payloads and are not treated as categories or subscriptions.
    * Store reusable language model prompts under `prompts/`; each `.md` file documents a structured workflow (for example, building a taxonomy from RSS feeds) that the application can provide to an LLM when automating filesystem updates or metadata generation.
   * Automatically create the default directory structure on first run and allow users to add, remove or rename categories and subscriptions via the GUI.
  * Read subscription metadata from the `index.md` file using **YAML front matter**.  The front matter must be at the top of the file and enclosed within triple‑dashed lines.  Each subscription-type plugin defines its own schema: for the RSS/Atom plugin the required fields are `source_name`, `source_uri`, `source_slug`, and `file_extension`, while Meshtastic and other plugins may use different keys or auto-generated metadata.  Transport helpers inherit the schema from their parent subscription-type plugin.  Additional optional keys such as `display_name`, `feed_url`, `website_url`, `logo`, `cadence`, or plugin-specific metadata may be present.
    * Define LLM automation prompts in `prompts/`. Prompts must explain the task, required outputs, directory conventions, and metadata fields so that generated responses can be applied deterministically by the app or agents.
   * Allow editing and updating of metadata through the UI, saving changes back to the `index.md` file.

2. **Feed fetching and parsing**

   * Retrieve feed content over HTTP/HTTPS, IPFS/IPNS, Tor, ActivityPub and AT protocols.  External binaries (e.g., Tor, IPFS daemon) will be bundled under `assets/bin/` and extracted at runtime as needed.
  * Support multiple content formats: RSS/XML, Atom and JSON.  A pluggable **source plugin** architecture will handle protocol‑specific fetching and content parsing.  Subscription-type plugins (such as RSS/Atom or Meshtastic) own the directory namespaces and may compose one or more transport helpers (HTTPS, Tor, IPFS, etc.) to reach remote feeds without altering the filesystem layout.  Plugins must implement methods to declare supported protocols and URIs, fetch raw content and parse posts into a common structure.
   * Store the most recent payload (`latest.<ext>`) and a timestamped snapshot in a `cache/` subdirectory for each subscription.  Respect the configured fetch cadence (default 24 hours).

3. **Publishing and logic**

   * Provide a **publish tab** where users can draft posts, attach media (images, audio, etc.) and publish to supported protocols (ActivityPub, AT, etc.).
   * Offer a **logic layer** with optional plug‑ins (e.g., simple filters, tagging, LLM integration) that can process posts before publication or after retrieval.  Logic plugins must be able to run in a simulation mode during testing.
   * Provide a right‑most column logic pipeline with expandable menu: a "Filters & Transformations" tool for SQL-based filters/joins/derived columns, and a dynamic list of Views (Data Table default, Charts, Media Playlist, Photo Gallery, Rendered Markdown). Views may include subqueries and LLM prompts.
   * Ensure logic blocks are modular units that accept inputs and produce outputs; allow chaining where each block’s output feeds the next.

4. **User interface**

   * **Input Efficiency:** Implement a hierarchical input system where every action is accessible via numeric shortcuts (e.g., `2-1`). Ensure zero-latency transitions between menu states to support rapid, muscle-memory based navigation using 10-key or D-pad inputs.
   * **Navigation:** The primary navigation is driven by a **Base Menu** anchored at the bottom of the screen. Submenus pop up from this base menu.
   * **Views:** The application has primary views corresponding to the base menu items:
   * **Home View (Menu Item 1)**: The default view on launch. Shows a dashboard/summary.
   * **Pub View (Menu Item 2)**:
     * Provide fields for composing a new post (title, content, attachments) and selectors for choosing destination protocols and accounts.
     * Include buttons to preview, publish and schedule posts.  Show a status area for success or error messages.
   * **Sub View (Menu Item 3)**:
     * Present a three-column layout for navigation:
       1. **Subscriptions Column**: A tree view reflecting the directory hierarchy under `subscriptions/`.
       2. **Posts Column**: A list of posts belonging to the selected subscription.
       3. **View Column**: An expanded view of the currently selected post.
   4. Right column also acts as a logic pipeline area with an expandable menu of full‑width buttons to configure Filters & Transformations and choose Views for the current left‑tree selection. Data is sourced by recursively querying posts belonging to the selected folder/subscription via the database.
   * Save UI state (window geometry, splitter positions, selected view) in a local configuration database so that it is restored on startup.

5. **Data storage and configuration**

   * Use **SQLite** as a lightweight embedded database for storing application preferences (UI state, accounts, user settings).  The database will be placed in `data/pubsub_config.db` and accessed via a `DatabaseManager` class.
   * Keep subscriptions and cached payloads in the filesystem to maintain transparency and reproducibility.

6. **Logging and test mode**

   * Implement a `Logger` class that writes plain‑text logs to a root‑level `log.txt` and a per‑run file under `logs/`.  Logs must capture inputs, outputs and contextual metadata in deterministic order.
   * Provide a `--test` flag in the CLI to enable **simulation mode**, which runs the same code paths using temporary directories and does not modify real data.  This helps reproduce behaviour and support automated testing.

7. **Platform support**

   * Target both Linux and Windows builds.  Use a Makefile with `g++` or `clang++` for compilation and static linking where possible.  Support cross‑compilation for Windows using `mingw‑w64`.

## Technical Requirements

1. **Language and standards**: Use C++17.  Adhere to the Vibe coding style guide (one operation per line, small focused functions, descriptive naming).

2. **TUI libraries**: Use **Notcurses** for all rendering and input handling. Notcurses allows for advanced terminal graphics (images, video) and high-performance rendering. Avoid legacy libraries like `ncurses` or `slang`. The input loop must support **numeric chaining** (capturing rapid sequences of digits) to drive the menu system.

3. **Dependencies**: Include libraries for HTTP requests (e.g., libcurl), YAML parsing (for `index.md` front matter), JSON parsing, SQLite, and plugin management.  Bundle any external binaries (Tor, IPFS) under `assets/bin/` and extract at runtime.

4. **Build system**: Provide a `Makefile` that compiles sources in `src/`, links against required libraries, and outputs executables into `bin/`.  Support optional static builds and cross‑compile targets.

5. **Plugin architecture**: Define abstract classes/interfaces for `SourcePlugin`, `DestinationPlugin` and `LogicPlugin`.  Plugins must declare supported protocols or content types and handle the fetch/parse/publish lifecycle.  Within `SourcePlugin`, distinguish subscription-type plugins (which own filesystem namespaces under `subscriptions/`) from transport helpers (which extend those plugins without adding new directories).  Provide a `PluginFactory` for registration and selection.
   * Introduce `OllamaLogicPlugin` for LAN LLM integration. It must expose a configurable API endpoint (default `http://ollama.local:11434/api/generate`), accept a post and prompt template, support streaming, and fail gracefully if unreachable.
   * Add `MeshtasticSourcePlugin` and `MeshtasticDestinationPlugin` to treat Meshtastic as both a source and a destination. Source connects via serial, maps channels to subscriptions, and yields posts; destination watches per‑channel outbox folders and sends messages via the connected node.

6. **File format handling**: Use robust parsers for RSS, Atom and JSON.  Disable external entity resolution in XML parsers for security.  Convert all posts into a standard `Post` structure.

## Logic Blocks, Filters, and Views

Logic blocks are modular, composable units that accept inputs and produce outputs within the right-most (view) column. Each block transforms data, applies filters, or generates metadata (including via LLMs). Users can chain blocks so the output of one becomes the input to the next. A collapsible menu at the top of the view column contains full-width buttons that govern query, transformation, and visualization.

### Data source and context

The view column’s dataset derives from the left-tree selection. Selecting a folder or subscription recursively collects all child posts through a database query, enabling filters and re-queries by subsequent blocks.

### Filters & transformations

The first menu button opens tools to:

* Create SQL‑based filters to subset results
* Apply column transforms, computed fields, joins
* Define custom pipelines that enrich data with LLM‑generated metadata (e.g., tags, bias score, summaries)

Filters operate at the database level against posts and subscriptions tables; transformations can also create derived attributes.

### Views

Below Filters is a dynamic list of configured views for the current path. Default is a Data Table. Additional views: Charts, Media Playlist, Photo Gallery, Rendered Markdown. Each view can include subqueries, transformations, and LLM prompts that produce synthesized outputs for display.

## Ollama LLM Gateway (LAN)

Integrate a networked LLM via an Ollama server reachable on the local network (default port 11434). Logic blocks can send prompts such as "summarise", "analyze bias", or "generate counter-argument" to `/api/generate` and stream back results. Store block outputs in the configuration database for persistence and later review. Provide default prompts and allow customization per block.

## Cyberpony Express via Meshtastic

Treat a connected Meshtastic node as first‑class source/destination:

* When connected, create a node root under `subscriptions/` named after the device ID; mirror discovered channels as child subscriptions with an auto-managed `index.md` containing Meshtastic-specific metadata and a `cache/` for received messages.
* Add `outbox/` per channel; the destination plugin monitors and sends queued messages via `sendText`/`sendData`.
* Expose channels in the **sub** tree and as selectable destinations in **pub**; show queue length and status.

## Fetcher: Cadenced Updates, Caching & DB Sync

Implement a background Fetcher that evaluates cadence per subscription and updates when due:

* Compute `next_due = last_fetched + cadence`; fetch when `now >= next_due`.
* On success, write `cache/YYYY-MM-DD-HH-mm-ss.<ext>` and atomically update `latest.<ext>`.
* Filesystem is the source of truth: each subscription-type plugin interprets the directory structure under its namespace to determine active sources.  The RSS/Atom plugin requires the `index.md` to include `source_name`, `source_uri`, `source_slug`, and `file_extension`; other plugins (e.g., Meshtastic) apply their own validation rules or generate metadata automatically.  Transport helpers never create directories—they attach to the owning subscription-type plugin and share its namespace.
* After fetch or filesystem change, rescan `index.md` YAML, invoke the plugin-specific validation, and upsert into SQLite.
* Use per‑subscription locking, retries, and backoff; log success, skipped, not_due, error. In `--test` mode, operate on temporary directories.

SQLite schema includes tables:

* `subscriptions` – mirror of YAML front matter and paths; indexed by protocol and path
* `fetches` – per‑attempt history with status, metadata, and cache paths
* `latest_payloads` – one row per subscription’s current latest payload (metadata and optional raw)
* `posts` – parsed items normalized into a query‑friendly structure, de‑duped by content hash

Indexes and upserts are required for performance and consistency. The latest cache timestamp serves as canonical `fetched_at`.

## Test Mode Enhancements

Extend `--test` to simulate Ollama responses and Meshtastic packets. Produce identical logging and DB write patterns to real runs but isolate all filesystem and database paths to temporary locations. Provide CLI commands/flags to dump Fetcher status, DB counts, and plugin registrations for LLM‑friendly inspection.

## Non‑Functional Requirements

1. **Performance**: The application must remain responsive when rendering thousands of posts.  Use background threads for network I/O and parsing, and avoid blocking the UI thread.

2. **Input Latency**: The TUI must process input and update the application state with zero perceptible latency. Animations must never block input; users must be able to "type ahead" of the interface.

3. **Portability**: Build and run on Linux and Windows without third‑party package managers.  All dependencies must be vendored or installable via standard package managers.

4. **Reliability**: Handle network failures gracefully; log errors and retry fetches according to the configured cadence.  Ensure that plugin failures do not crash the application.

5. **Security and privacy**: Respect user privacy; do not send data to third parties unless configured.  Provide sensible defaults for Tor/IPFS connectivity and document any external connections.

## Deliverables

1. Source code organised under `src/` with clear module separation.
2. `Makefile` for building and packaging the application.
3. `README.md` describing installation, usage and contributing guidelines, and linking prominently to this document.
4. `project-specification.md` (this consolidated document including the specification, roadmap, social context, and style guides).
5. A functioning GUI application supporting the described features, along with logs and test mode.

## Social Context and Motivation

### About the High Desert Institute

The **High Desert Institute (HDI)** is a 501(c)(3) non-profit organisation dedicated to “building a foundation for the survival of humanity”. HDI focuses on off-grid land projects, libraries and guilds that provide free, open-source solutions to basic human needs such as housing and infrastructure. A team of community-building experts is fundraising to create outposts in the high deserts of the southwest to research, develop and share these solutions. Key HDI programmes include:

* **The Librarian** – a free, open-source digital librarian that runs on edge devices (like Raspberry Pi) to answer complex questions based on local library content. Communities can load their own collections and the librarian will respond using a small language model running locally.
* **Cyberpony Express** – a free, secure, public mesh network built with Meshtastic nodes. It enables communication between HDI outposts and intentional communities without relying on the fragile infrastructure of the public internet or cell phone towers, and serves as vital disaster response infrastructure.
* **Guilds** – semi-autonomous groups that focus on specific aspects of community development. Guilds are empowered to organise their projects and finances, and the goal is for them to replicate globally.
* **The Library** – a vast, open-source digital library containing everything HDI learns, along with other curated resources on off-grid living. It is designed to be freely accessible and hostable in remote locations.

### Importance of an Offline-Friendly RSS Reader & Publisher

Access to trustworthy, up-to-date information is critical for communities operating off-grid or on unreliable networks. Traditional RSS readers focus only on the mainstream internet. Our C++17 RSS reader and publisher brings together multiple **decentralised protocols**—HTTP/HTTPS, IPFS/IPNS, Tor, ActivityPub and AT protocol—into one application. By supporting both **reading and publishing**, it allows communities to:

* **Consume information across decentralised networks.** IPFS/IPNS feeds can host content without central servers; Tor onion feeds can provide anonymity and censorship resistance; ActivityPub and AT feeds enable interoperability with federated social platforms. This aligns with HDI’s commitment to independence from fragile public infrastructure.
* **Share knowledge back to the world.** Publishing to ActivityPub or AT protocols makes it easy for off-grid communities to broadcast updates—project reports, permaculture tips, disaster alerts—to federated networks and mesh-connected peers.
* **Work offline and off-grid.** Cached subscriptions and the local filesystem are the canonical source of truth. A small footprint C++/Notcurses stack means the app can run on low-power devices similar to those used for the Librarian.
* **Respect privacy and transparency.** By bundling Tor and IPFS binaries and logging everything to plain-text files, the tool preserves user privacy while maintaining verifiable audit trails.

### Fit Within HDI’s Programmes

The RSS reader/publisher complements and extends HDI’s existing projects:

* **Augmenting the Librarian.** The Librarian stores and serves a local library; our RSS reader fetches new content from decentralised feeds and publishes curated summaries. When integrated, the Librarian can answer questions about fresh news items or community posts. Integration with a local-LAN **Ollama** server enables on-device summarisation, tagging, bias analysis and counter-arguments via logic blocks. Results can be appended back into the Library so knowledge remains searchable offline.
* **Leveraging the Cyberpony Express.** The Cyberpony Express mesh network enables secure communication between HDI outposts. By treating **Meshtastic** nodes as first-class sources and destinations, the app can receive messages as channel subscriptions and send updates by watching per-channel outboxes. This turns the reader/publisher into a mesh-connected client, ensuring information flows even when internet access is unavailable.
* **Empowering Guilds and Outposts.** Guilds focus on specific domains (e.g., permaculture, energy, housing). Each guild can maintain its own feed of research, announcements and learning materials within the subscriptions hierarchy, while categories mirror the guild structure. Outposts can publish their progress and read updates from others, fostering collaboration.
* **Strengthening the Library.** As HDI’s digital library grows, the RSS reader/publisher becomes a tool for continually enriching it with decentralised sources. It can automate ingestion of updates from relevant projects, and use logic plugins (e.g., summarisation or tagging via local LLMs) to prepare new entries.

### Ethical and Societal Impact

By distributing news and knowledge across decentralised networks, this project reduces dependence on centralised platforms and fosters resilience. It respects user autonomy—subscriptions are stored locally, publishing is optional, and logs are transparent. Support for Tor and IPFS empowers activists and communities facing censorship to share critical information. LAN-local LLMs via Ollama preserve privacy while enabling advanced analytics offline. Treating Meshtastic as a source/destination extends reach across the Cyberpony Express mesh. Integration with HDI’s education and communication initiatives ensures that research and experience from off-grid outposts can be shared widely, strengthening mutual aid networks.

This RSS reader and publisher is more than a convenience; it is **infrastructure** for a resilient future. By bridging decentralised protocols and feeding into the Librarian, Cyberpony Express, guilds and the Library, it helps realise HDI’s vision of a world where communities control their own information flow and build free, open-source solutions for survival. Its offline-friendly design ensures that no matter the circumstances, knowledge can be shared, stored and acted upon.

## CLI Logging and Interpretability Style Guide

This style guide outlines best practices for command‑line interface (CLI) development tools. Its goal is to make every action that your tool performs visible from the CLI so that both humans and large language models (LLMs) can understand and debug the workflow. The guidance in this document is language‑agnostic and can be applied to any project that exposes a command‑line surface, even if the project also ships a graphical user interface.

### 1. Philosophy

* **CLI first.** The CLI should be the primary interface for your tool. Even if a GUI exists, every operation must be executable from the command line so that automated agents and scripts can drive the tool without a GUI.
* **Transparency.** Everything that happens in your tool should be visible in the CLI. Hiding behaviour behind GUI elements or implicit side effects makes it impossible for developers and LLMs to reason about what the tool is doing.
* **Reproducibility.** A user (or agent) following the documented CLI commands should be able to reproduce any run of your tool. Avoid hidden state or reliance on external environment configuration when possible.
* **Cross-platform reliability.** All code must run correctly on both Windows and Linux without requiring manual tweaks or OS-specific forks.

### 2. Verbose logging

To support LLMs and developers in understanding your tool’s behaviour, every CLI‑based project must create a root‑level log directory (for example, `logs/`). Each run of the application must write to a brand‑new log file inside that directory named with the timestamp of when the run started using a sortable pattern such as `log.YYYY-MM-DD-HH-mm-ss-ffffff.txt`. Every per-run log file must contain:

* **User inputs and actions.** Log the exact command arguments or interactive input received.
* **Outputs and results.** Log what the tool prints to stdout/stderr and any side effects it performs (e.g. files written, network calls made, database queries executed). Include timestamps to aid debugging.
* **Contextual metadata.** Provide information about the component generating the log (module name, function, or class) and the phase of the operation.

Logs should be written in a plain‑text, append‑only format. The goal is to give agents complete visibility into what happened during execution, so do not reuse log files across runs—always start a fresh, timestamped file when the process begins. If multiple processes are spawned (for example, launching Tor or IPFS as sub‑processes), their stdout/stderr should also be captured and appended to that run’s log file.

### 3. Test mode and simulation

LLMs often need to verify that your tool behaves correctly without modifying real data. Provide a dedicated **test mode** (for example, via a `--test` or `-test` flag) that:

* Simulates a broad range of typical user actions.
* Produces the same verbose logs as a normal run.
* Does **not** modify any persistent user data (such as real databases or on‑disk files). Use temporary directories or in‑memory data structures in test mode.

When running in test mode, the tool should exercise enough code paths to make automated testing effective. This allows LLMs to verify functionality without incurring the overhead of running integration tests during every normal startup.

### 4. README.md and style guide references

Every repository that contains CLI‑based tools must include a comprehensive **`README.md`** file. The README must link prominently to this consolidated `project-specification.md` and, where helpful, to specific sections within it (e.g., CLI logging, the C++ development guide, and the roadmap). At a minimum, the README.md should:

* Link to relevant sections of this document (CLI logging guide, C++ Notcurses development guide, roadmap) at the top with brief descriptions
* Describe the high‑level purpose of the project and its architecture.
* Explain where logs live (for example, the root‑level `logs/` directory and the timestamped files it contains) and how to run the project in normal and test modes.

Agents and developers should read `project-specification.md` for detailed guidance. The `README.md` provides quick orientation and build/run steps and links back here.

### 5. Project roadmap and specification requirements

The project specification, social context, style guide, and roadmap are stored in project-specification.md which should always be included in any prompt context. The roadmap must be kept up to date as the project evolves. It should be a living document that reflects the current status of all tasks and features.

#### Roadmap requirements

The roadmap must include:

* **Status legend** at the top explaining checkbox meanings:
  - `[ ]` - Not started
  - `[?]` - In progress / Testing / Development  
  - `[x]` - Completed and tested
  - `[-]` - Blocked / Deferred
* **All major and minor tasks** broken down by phases
* **Current status** clearly marked with appropriate checkboxes
* **Update instructions** requiring roadmap updates whenever tasks change status

#### Specification requirements

The specification must include:

* **Complete technical requirements** for all features
* **Architecture details** and component descriptions
* **Configuration schemas** and data formats
* **Success criteria** and acceptance tests
* **Security and privacy** requirements

#### Social context requirements

The social context document must include:

* **Organization details** and mission statement
* **Partnerships and collaborations** with other organizations
* **Use cases and user stories** showing how the project helps people
* **Community impact** and social benefits
* **Future opportunities** and potential for related projects
* **Stakeholder information** and target audiences
* **Cultural and social considerations** relevant to the project
* **Accessibility and inclusion** aspects
* **Ethical considerations** and responsible development practices

All three documents must be kept current whenever project details change, features are added, or development status updates.

### 6. Logging best practices

The following guidelines apply across languages:

* **Consistent format.** Use a structured logging format (e.g. timestamps, log level, module name, message). Consistency makes parsing and analysis easier for tools.
* **Appropriate log levels.** Use `INFO` for normal operations, `WARN` for recoverable issues, and `ERROR` for serious problems. Do not hide exceptions—log stack traces at the `ERROR` level.
* **Context in messages.** Include enough detail in each log entry to understand what was happening. For example, log input parameters, intermediate results, and the outcome of operations.
* **Cross‑language adherence.** When your project consists of components in multiple languages, ensure that each component writes to the same per-run log file in the same format.

### 7. Integration with large language models (LLMs)

LLMs cannot see your GUI or internal state; they rely entirely on textual output. To make your tool LLM‑friendly:

* **Expose state via CLI.** Provide commands or flags that output the current configuration, status, or internal metrics of your tool. Avoid requiring an API or GUI for this.
* **Descriptive errors.** Write error messages that explain what went wrong and how to fix it. Avoid cryptic messages or silent failures.
* **Deterministic output.** When possible, avoid non‑deterministic ordering of logs (e.g. due to concurrency) that could confuse automated analysis. If concurrency is necessary, clearly label log lines with thread or process identifiers.

### 8. Example CLI workflow

Document a typical usage scenario for your tool. For example:

```bash
# run the tool normally and capture logs
./mytool --input data.txt --output results.txt

# inspect the newest log file
ls logs/
cat "$(ls -t logs/log.* | head -n 1)"

# run in test mode
./mytool --test

# run a command to dump current status
./mytool --status
```



## CLI-First C++ Development Style Guide (Notcurses Edition)

This document outlines a lightweight, CLI-first approach to writing, building, and distributing C++ software using **Notcurses** for small, fast, cross-platform TUIs. It focuses on transparent builds, minimal dependencies, and direct control over compilation and linking—without large build systems or IDE lock-in.

---

### 1. Philosophy of CLI-First C++ Development

Most modern C++ projects use heavyweight environments like Visual Studio, Xcode, or large meta-build systems (CMake, Meson, Premake). These tools can:

* Introduce unnecessary abstraction and hidden configuration.
* Enforce non-portable conventions.
* Add dependency sprawl and opaque build steps.
* Slow iteration for small, single-purpose tools.

A CLI-first C++ style instead:

* Uses **standard compiler tools** directly (`g++`, `clang++`, `x86_64-w64-mingw32-g++`).
* Builds from **Makefiles** or **shell/batch scripts**.
* Keeps source trees **transparent, minimal, and reproducible**.
* Emphasizes **small self-contained binaries** that link only what they need.
* Guarantees **dual-platform support**—all code must build and run on both Windows and Linux without divergence.

---

### 2. Goals for TUI: Why Notcurses

This guide standardizes on **Notcurses** as the preferred TUI library for small cross-platform applications.

#### Reasons

* **Multimedia support:** Renders images and video directly in supported terminals (Sixel, Kitty, etc.).
* **High performance:** Uses a dedicated rendering thread and optimized diffing.
* **Z-ordering:** Uses "planes" that can be moved, resized, and stacked, simplifying layout.
* **Thread safety:** Designed for modern multi-threaded applications.
* **Cross-platform:** Works on Linux and Windows (via MSYS2/Mintty or Windows Terminal).
* **Transparent builds:** You can literally see every flag and file in the build command.

#### Goal

> To build small, fast, portable TUI utilities — such as RSS readers, Ollama frontends, and monitoring dashboards — that compile in seconds and distribute as single executables.

---

### 3. Basic Project Structure

A typical CLI-first C++ + Notcurses project:

```
project/
  src/
    main.cpp
    tui.cpp
    tui.h
  assets/
    icons/
    fonts/
  build/        # intermediate object files (ignored by VCS)
  bin/          # final executables
  Makefile
```

* `src/` → C++ source and headers
* `assets/` → static resources embedded or loaded at runtime
* `build/` → compiler outputs (.o, .obj)
* `bin/` → packaged executables

---

### 4. Compiling and Running with Minimal Dependencies

Use `g++` (Linux/macOS) or `x86_64-w64-mingw32-g++` (Windows cross-compile).

```bash
# Compile and link directly
g++ -std=c++17 -O2 -Wall \
  src/*.cpp \
  -o bin/myapp \
  $(pkg-config --cflags --libs notcurses)
```

This produces a single executable — no installers, no runtime dependencies, no DLL sprawl.

---

### 5. Makefile Example

```makefile
APP = myapp
SRC = $(wildcard src/*.cpp)
OBJ = $(SRC:src/%.cpp=build/%.o)
CXX = g++
CXXFLAGS = -std=c++17 -O2 -Wall $(shell pkg-config --cflags notcurses)
LDFLAGS = $(shell pkg-config --libs notcurses)

all: bin/$(APP)

build/%.o: src/%.cpp
	mkdir -p build
	$(CXX) $(CXXFLAGS) -c $< -o $@

bin/$(APP): $(OBJ)
	mkdir -p bin
	$(CXX) $(OBJ) -o $@ $(LDFLAGS)

clean:
	rm -rf build bin
```

Build with `make` or `mingw32-make` (on Windows).

---

### 6. Static Linking for Self-Contained Apps

To avoid runtime library issues and ensure portability:

**Linux static build:**

```bash
g++ -static -static-libgcc -static-libstdc++ -O2 \
  src/*.cpp -o bin/myapp \
  $(pkg-config --cflags --libs notcurses)
```

**Windows cross-compile (from Linux):**

```bash
x86_64-w64-mingw32-g++ -O2 -std=c++17 \
  src/*.cpp -o bin/myapp.exe \
  $(x86_64-w64-mingw32-pkg-config --cflags --libs notcurses)
```

Now you can ship `bin/myapp` or `bin/myapp.exe` as a single binary.

---

### 7. Minimal TUI Example and Notcurses Best Practices

```cpp
#include <notcurses/notcurses.h>
#include <iostream>

int main() {
    struct notcurses_options opts = {0};
    opts.flags = NCOPTION_NO_ALTERNATE_SCREEN; // Optional: keep history
    struct notcurses* nc = notcurses_init(&opts, NULL);
    if (!nc) return -1;

    struct ncplane* stdplane = notcurses_stdplane(nc);
    ncplane_putstr(stdplane, "Hello from PubSub TUI!");

    notcurses_render(nc);

    // Input loop
    uint32_t id;
    struct ncinput ni;
    while((id = notcurses_getc_blocking(nc, &ni)) != (uint32_t)-1) {
        if (id == 'q') break;
        // Handle other input...
    }

    notcurses_stop(nc);
    return 0;
}
```
#### Notcurses Layout Best Practices

*   **Use Planes:** Everything in Notcurses is a plane (`ncplane`). Planes can be moved, resized, and Z-ordered independently.
*   **Standard Plane:** The standard plane (`notcurses_stdplane`) is the background. Create child planes for your widgets and content.
*   **Z-Ordering:** Use `ncplane_move_top()` and `ncplane_move_bottom()` to manage visibility.
*   **Resize Callbacks:** Handle terminal resize events by resizing and repositioning your planes.

### 8. Why Not ncurses or Slang?

* **ncurses:** Legacy API, poor thread safety, limited color support, no native image/video support.
* **Slang:** Better than ncurses but still lacks the modern features and performance of Notcurses.

**Notcurses** provides a modern, high-performance alternative with multimedia capabilities.

---

### 9. Benefits of This Style

* **Minimalism:** one main dependency (Notcurses), pure C.
* **Transparency:** every build step visible in a Makefile or shell script.
* **Reproducibility:** builds identical on Linux and Windows.
* **Speed:** full rebuilds in seconds; no dependency management.
* **Portability:** same source compiles anywhere with g++/clang++.
* **Control:** choose exactly what to link, bundle, or statically embed.

---

### 10. Example Workflow

1. Write code in any editor.
2. Run `make` to:

   * Compile and link sources.
   * Build a single binary in `bin/`.
3. Test locally (`./bin/myapp` or `myapp.exe`).
4. Cross-compile for Windows if needed.
5. Distribute binaries directly — no installers or external libraries.

---

By following this **CLI-first C++ style**, you’ll produce self-contained, portable GUI tools that compile in seconds, run anywhere, and depend on nothing but your own source code. Notcurses provides just enough TUI capability to make tools usable without sacrificing performance or simplicity—perfect for small, independent, and reproducible software projects.

---

### 11. Logging and Interpretability

CLI-first C++ tools should adopt the same verbose logging rules described in the general CLI development guide. At minimum:

* **Root‑level log directory:** Maintain a root-level `logs/` directory (or similarly named folder). Each time the application starts, create a new plain-text log file inside that directory named with the start timestamp (for example, `log.YYYY-MM-DD-HH-mm-ss-ffffff.txt`) and capture all user inputs, outputs, and contextual metadata within that file.
* **Plain text format:** Logs must be plain text so that both humans and automated agents can parse them easily.
* **Include subprocess output:** If your application spawns subprocesses, capture their `stdout` and `stderr` streams and append them to the same per-run log file.
* **Deterministic ordering:** Log entries should appear in the exact order actions occur to make replaying and auditing runs straightforward.
* **See the CLI development guide section above:** For details on log levels (INFO, WARN, ERROR) and cross‑language log formats, refer to the "CLI Logging and Interpretability Style Guide" section in this document.

---

### 12. Test Mode & Simulation

Implement a `--test`, `--dry-run`, or similar command‑line flag that runs the application in simulation mode. In test mode:

* **No real side effects:** Use temporary directories for configuration, storage, and outputs. Do not modify user data or external systems.
* **Same code paths:** Exercise the same code paths and produce the same output and logging as a real run so automated agents can verify behavior.
* **Reproducibility:** Because test mode writes to isolated locations and never modifies real data, results can be replayed for testing and auditing.
* **Documentation:** Explain your test mode in the README and CLI help text so users and agents know how to activate it.

---

### 13. Project Roadmap, Specification & Social Context

This project uses a single consolidated document: `project-specification.md` (this file). It contains:

* The full technical specification (requirements, architecture, data flows, security)
* The project roadmap with milestones and task tracking
* The social context and motivation

Keeping these sections together improves discoverability for humans and agents and ensures there is one authoritative source of truth.

---

### 14. Including External Tools (Tor, IPFS, etc.)

Sometimes a CLI-first C++ tool needs to interface with external binaries—such as **Tor** for anonymous networking or **IPFS** for decentralized storage. To bundle and manage these tools reliably:

1. Place the external binaries in an `assets/bin/` directory within your repository.
2. At runtime, extract the binaries to a temporary directory using standard C++ filesystem utilities.
3. Launch the binaries with `std::system` or a process library (e.g., `Boost.Process`) and communicate via sockets, pipes, or command-line interfaces.
4. Include configuration files in `assets/etc/` and reference them when launching the external tools.

By vendoring external binaries you avoid relying on system installations and maintain control over versions and configurations.

---

### 15. AI Integration

Two kinds of integrations will be used.

1. **Ollama:** If a local Ollama server is available on the LAN, your application can send prompts to it for tasks like summarization, tagging, or bias analysis. Use HTTP requests to communicate with the Ollama API and handle responses appropriately. Ensure that all interactions are logged verbosely. This is preferable but more complex fo rmost users to set up.
2. **Local LLMs:** For simpler setups, integrate with llama.cpp to run models locally. The llm.cpp library is present in external/llama.cpp. See the following integration details;

**Use `llama.cpp` (libllama)**, to run prompts against a small local model:

#### Minimal way to embed `llama.cpp` in C++

Add as a submodule and link the library:

```bash
git submodule add https://github.com/ggerganov/llama.cpp external/llama.cpp
git -C external/llama.cpp submodule update --init --recursive
```

**CMakeLists.txt**

```cmake
add_subdirectory(external/llama.cpp)
add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE llama)
```

**src/main.cpp** (ultra-short example)

```cpp
#include "llama.h"
#include <iostream>

int main() {
    llama_init_params ip = llama_init_params_default();
    llama_model_params mp = llama_model_default_params();
    llama_context_params cp = llama_context_default_params();
    // e.g., 4096 ctx tokens
    cp.n_ctx = 4096;

    // Load a small GGUF (quantized) model
    llama_model *model = llama_load_model_from_file("models/Qwen3-4B-Instruct-2507-Q8_0.gguff", mp);
    llama_context *ctx   = llama_new_context_with_model(model, cp);

    // Load prompt from promts/ or hardcode for testing
    const char *prompt = "You are a helpful assistant. Q: 2+2? A:";
    llama_batch batch = llama_batch_init(512, 0, 1);

    // tokenize
    std::vector<llama_token> toks(512);
    int n = llama_tokenize(model, prompt, std::strlen(prompt), toks.data(), toks.size(), true, false);
    toks.resize(n);

    // evaluate prompt
    llama_batch_clear(batch);
    for (int i = 0; i < (int)toks.size(); ++i) {
        llama_batch_add(batch, toks[i], i, {0}, false);
    }
    llama_decode(ctx, batch);

    // sample a few tokens
    std::string out;
    llama_token cur = 0;
    for (int t = 0; t < 64; ++t) {
        cur = llama_sample_token_greedy(ctx, llama_get_logits_ith(ctx, toks.size() - 1 + t));
        if (cur == llama_token_eos(model)) break;
        out += llama_token_to_piece(model, cur);
        llama_batch_clear(batch);
        llama_batch_add(batch, cur, toks.size() + t, {0}, true);
        llama_decode(ctx, batch);
    }
    std::cout << out << "\n";

    llama_batch_free(batch);
    llama_free(ctx);
    llama_free_model(model);
    llama_backend_free();
}
```

#### Quick picks for “small, local” models

* **CPU-only laptops/mini-PCs:** 1–3B or quantized 7B (Q4/Q5) runs fine.
   * Qwen3-4B-Instruct-2507-GGUF
* **Modest GPU (8–12 GB VRAM):** 7B–14B Q4/Q5 with CUDA/Metal/Vulkan.
* Use **GGUF quantized** checkpoints (most popular models have ready-made GGUF builds).



### 16. Cross‑Referencing this Guide

This C++ guide is part of a larger style guide collection. For general logging, interpretability, and agent‑integration rules that apply to all CLI‑based projects, see the "CLI Logging and Interpretability Style Guide" section in this document.


## Core Data Schemas

All persistent features must converge on shared SQLite schemas so the GUI, CLI, and automation prompts operate on consistent structures. **The `subscriptions/` filesystem remains the source of truth.** Every time the application fetches, creates, renames, or deletes content under `subscriptions/`, the database must be re-synchronised to mirror that state (including removals). Table names, required columns, and intended indexes are defined below; implementations must use these definitions verbatim.

### Task Queue (`tasks`)

| Column | Type | Notes |
| ------ | ---- | ----- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Unique task identifier |
| `task_type` | TEXT NOT NULL | `fetcher` or `llm` |
| `status` | TEXT NOT NULL | `queued`, `running`, `paused`, `succeeded`, `failed` |
| `payload` | TEXT NOT NULL | JSON blob describing work to perform |
| `result` | TEXT | JSON blob summarising outcome or error |
| `last_updated_at` | DATETIME NOT NULL | Updated whenever task state changes |
| `created_at` | DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP | Queue time |
| `started_at` | DATETIME | Optional start timestamp |
| `completed_at` | DATETIME | Optional finish timestamp |

Indexes: `(task_type, status)` to drive dashboards; `(last_updated_at)` for pruning.

### LLM Prompt History (`llm_prompts`)

| Column | Type | Notes |
| ------ | ---- | ----- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Unique prompt run |
| `prompt_text` | TEXT NOT NULL | Final rendered prompt |
| `model_name` | TEXT NOT NULL | Selected model identifier |
| `destination_type` | TEXT NOT NULL | `local` or `api` |
| `destination_uri` | TEXT | API endpoint or local socket path |
| `queued_at` | DATETIME NOT NULL | When request entered queue |
| `started_at` | DATETIME | Execution start |
| `completed_at` | DATETIME | Execution end |
| `tokens_evaluated_per_sec` | REAL | Collected from inference metrics |
| `tokens_generated_per_sec` | REAL | LLM throughput |
| `metadata` | TEXT | JSON containing additional stats (temperatures, lengths, etc.) |
| `task_id` | INTEGER REFERENCES tasks(id) ON DELETE SET NULL | Link back to task queue |

### Subscription Sources (`subscription_sources`)

| Column | Type | Notes |
| ------ | ---- | ----- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Surrogate key |
| `subscription_path` | TEXT NOT NULL UNIQUE | Filesystem path (e.g., `subscriptions/rss/tech/hackernews`) |
| `category_path` | TEXT NOT NULL | Category path including parent directories |
| `source_name` | TEXT NOT NULL | From YAML front matter |
| `source_uri` | TEXT NOT NULL | Canonical feed URI |
| `source_slug` | TEXT NOT NULL | Stable slug |
| `file_extension` | TEXT NOT NULL | Expected cache extension |
| `plugin_type` | TEXT NOT NULL | Owning subscription plugin |
| `cadence_minutes` | INTEGER | Optional cadence override |
| `metadata` | TEXT | JSON for plugin specific fields |
| `yaml_last_hash` | TEXT | Hash of last parsed front matter |
| `updated_at` | DATETIME NOT NULL | Last schema load |
| `created_at` | DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP | Initial discovery |

### Posts (`posts`)

| Column | Type | Notes |
| ------ | ---- | ----- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Surrogate key |
| `subscription_id` | INTEGER NOT NULL REFERENCES subscription_sources(id) ON DELETE CASCADE | Foreign key |
| `title` | TEXT NOT NULL | Normalised title |
| `source_slug` | TEXT NOT NULL | Cached for fast joins |
| `link` | TEXT | Canonical link |
| `guid` | TEXT | Feed GUID/hash |
| `summary` | TEXT | Short description |
| `content` | TEXT | Full content |
| `post_image` | TEXT | Resolved hero image path/URL |
| `source_logo_image` | TEXT | Optional logo |
| `audio_file` | TEXT | Media URL/path |
| `video_file` | TEXT | Media URL/path |
| `pubdate` | DATETIME | Publication timestamp |
| `last_updated_at` | DATETIME NOT NULL | When record last changed |
| `fetched_at` | DATETIME NOT NULL | Source fetch timestamp |
| `content_hash` | TEXT NOT NULL UNIQUE | Deduplication key |

Indexes: `(subscription_id, pubdate DESC)`, `(content_hash)`.

### Meshtastic Messages (`meshtastic_messages`)

| Column | Type | Notes |
| ------ | ---- | ----- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Surrogate key |
| `local_device_id` | TEXT NOT NULL | Our node identifier |
| `channel_id` | TEXT NOT NULL | Channel or DM identifier |
| `source_device_id` | TEXT NOT NULL | Message origin |
| `message_id` | TEXT NOT NULL | Stable message UID |
| `content` | TEXT | Text payload |
| `reply_to` | TEXT | Optional parent message id |
| `raw_payload` | BLOB | Binary body if present |
| `data_type` | TEXT | Text, data, telemetry, etc. |
| `received_at` | DATETIME NOT NULL | When message arrived |
| `updated_at` | DATETIME NOT NULL | Last update (edits/acks) |
| `metadata` | TEXT | JSON for extra fields |

Unique index `(local_device_id, channel_id, message_id)`.

### Meshtastic Nodes (`meshtastic_nodes`)

| Column | Type | Notes |
| ------ | ---- | ----- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Surrogate key |
| `local_device_id` | TEXT NOT NULL | Our device observing |
| `remote_device_id` | TEXT NOT NULL | Device identifier seen |
| `short_name` | TEXT | Broadcast short name |
| `long_name` | TEXT | Broadcast long name |
| `first_seen_at` | DATETIME NOT NULL | First observation |
| `last_seen_at` | DATETIME NOT NULL | Most recent observation |
| `metadata` | TEXT | JSON for capabilities/firmware |

Unique index `(local_device_id, remote_device_id)`.

### Meshtastic Sightings (`meshtastic_sightings`)

| Column | Type | Notes |
| ------ | ---- | ----- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Surrogate key |
| `local_device_id` | TEXT NOT NULL | Reporting node |
| `remote_device_id` | TEXT NOT NULL | Observed node |
| `recorded_at` | DATETIME NOT NULL | Timestamp of sighting |
| `rssi` | REAL | Signal strength |
| `reported_latitude` | REAL | Optional latitude |
| `reported_longitude` | REAL | Optional longitude |
| `reported_altitude` | REAL | Optional altitude (meters) |
| `telemetry_payload` | TEXT | JSON of telemetry/health data |

Index `(local_device_id, recorded_at DESC)` to power history views.

Implementations must update this section whenever schema changes occur and add migration steps to the roadmap.


## Disk‑First Tasks System (Fetcher & LLM)

This section specifies a filesystem‑first task system, modeled after the `subscriptions/` pattern. Markdown files with YAML front matter are the single source of truth for both Fetcher and LLM tasks. The database mirrors these files for querying, dashboards, and automation. The goal is to make tasks human‑writable and agent‑friendly while preserving transactional integrity and auditability.

### Purpose & scope

- Track two task types: `fetcher` and `llm`.
- Persist authoritative fields in YAML: state, timestamps (created/updated/completed), payload, and result.
- Mirror to SQLite tables: `tasks` (all tasks) and `llm_prompts` (for LLM tasks).
- Provide end‑to‑end operability via GUI and CLI, with test mode that isolates all side effects.

### Filesystem layout

Root directory: `tasks/`

Namespace (type‑owned), suggested date bucketing for hygiene:

- `tasks/fetcher/YYYY/MM/<uid>/`
- `tasks/llm/YYYY/MM/<uid>/`

Per‑task contents:

- `index.md` — YAML front matter holds authoritative metadata; Markdown body is the human description.
- `artifacts/` — optional output files (e.g., summaries, cached payload snippets, JSON reports). Store large results here and link to them from YAML `result` fields.

This mirrors the `subscriptions/` arrangement where the owning type/namespace governs rules and the front matter is canonical.

### YAML front matter schema (canonical)

Common required keys:

- `task_type`: `fetcher` | `llm`
- `uid`: stable string identifier (matches folder name recommended)
- `title`: short title
- `status`: `queued` | `running` | `paused` | `succeeded` | `failed`
- `created_at`: ISO8601 timestamp
- `updated_at`: ISO8601 timestamp
- `completed_at`: ISO8601 timestamp or null
- `payload`: object — type‑specific inputs/parameters
- `result`: object or null — type‑specific outputs/summary
- `metadata`: object — optional free‑form tags/labels

Fetcher recommended fields:

- `payload.subscription_path`: string (e.g., `subscriptions/rss/tech/hackernews`)
- `payload.cadence_minutes`: integer (override)
- `payload.reason`: enum (`manual`, `schedule`, `retry`, `fs-change`, ...)
- `payload.transport`: enum (`https`, `tor`, `ipfs`, ...)
- `result.cache_path`: string (last written cache file)
- `result.status_detail`: string (HTTP status or parser detail)
- `result.items_parsed`: integer
- `result.error`: string (on failure)

LLM required/recommended fields:

- `prompt_text`: the final rendered prompt
- `model_name`: selected model identifier
- `destination_type`: `local` | `api`
- `destination_uri`: API endpoint or local socket path
- `payload.params`: object (e.g., temperature, max_tokens)
- `payload.context_ref`: object linking context (e.g., post_id, source_slug)
- `result.output_file`: path under `artifacts/` for large text output
- `result.tokens_evaluated_per_sec`: number
- `result.tokens_generated_per_sec`: number
- `result.metrics`: object (extra stats)
- `result.error`: string (on failure)

The Markdown body of `index.md` contains a human description and can be copied into the DB payload as `description` for convenience.

### Database mirroring

Tables used (see Core Data Schemas): `tasks` and `llm_prompts`.

`tasks` mapping (applies to both task types):

- `task_type` <= YAML.task_type
- `status` <= YAML.status
- `payload` (JSON) includes at minimum:
  - `file_path`: absolute path to `index.md`
  - `uid`: YAML.uid
  - `yaml_hash`: SHA‑256 hash of normalized front matter
  - `description`: Markdown body (optional/truncated)
  - `user_payload`: entire YAML.payload object
  - `artifacts`: optional list of discovered files under `artifacts/`
- `result` (JSON) <= YAML.result (+ optional small computed extras)
- `created_at` <= YAML.created_at
- `started_at` <= derived when status first transitions to `running` (if absent in YAML)
- `completed_at` <= YAML.completed_at
- `last_updated_at` <= YAML.updated_at

Identity & upsert:

- Logical key: `(task_type, uid)`; do not rely on DB `id` across syncs.
- Upsert by selecting where `task_type = ? AND json_extract(payload, '$.uid') = ?`.

`llm_prompts` mapping (only for `llm` tasks):

- `prompt_text`, `model_name`, `destination_type`, `destination_uri` from YAML
- `queued_at` <= YAML.created_at
- `started_at` <= first running timestamp
- `completed_at` <= YAML.completed_at
- `tokens_evaluated_per_sec`, `tokens_generated_per_sec` <= result
- `metadata` (JSON): merge `payload.params`, `payload.context_ref`, `artifacts`, `file_path`, `uid`
- `task_id`: FK to the owning `tasks` row

Deletion mirroring:

- Filesystem is authoritative. If a DB row corresponds to a task whose directory (or `index.md`) is removed, delete the `tasks` row and clear or delete the linked `llm_prompts` row accordingly.

### Sync algorithm & change detection

Contract:

- Inputs: `tasks/` tree
- Outputs: `tasks` and `llm_prompts` reflect the filesystem exactly (including removals)

Algorithm:

1. Discover `tasks/**/index.md` on startup and on demand.
2. Parse front matter and body; compute `yaml_hash`; optionally list `artifacts/`.
3. Validate required keys and enums; log and skip malformed entries.
4. Upsert into `tasks` by `(task_type, uid)`; update when `yaml_hash` differs or file mtime increases.
5. For `llm` tasks, upsert into `llm_prompts` linked by `task_id`.
6. Detect deletions: any DB row whose `payload.file_path` no longer exists is removed.
7. Wrap each file’s mirror work in a small transaction; batch for performance when scanning many.

Conflict resolution:

- Filesystem wins. The app must write YAML first, then DB. Avoid mutating DB rows that would diverge from YAML.

### App writes & atomicity

When the app changes a task (e.g., status → running, adding result):

1. Generate updated front matter (set `updated_at`, possibly `completed_at`).
2. Write `index.tmp` next to `index.md`; flush; atomic rename to `index.md`.
3. Trigger a focused sync for that file (with short debounce to coalesce bursts).
4. Mirror to DB in a transaction; log the action type: insert/update/skip.

### UI integration

- Tasks Dashboard: filter by `task_type` (Fetcher/LLM), columns: Title, Status, Created, Updated, Completed, Result.
- Actions: open `index.md` in editor; reveal task folder; create new task (scaffold folder + YAML); retry; pause.
- LLM extras: show model and destination; quick‑open `result.output_file` when present.

### CLI integration

- `--tasks-sync` — rescan filesystem and mirror into DB (verbose summary of insert/update/skip/delete)
- `--tasks-list [fetcher|llm|all]` — list tasks with key fields
- `--tasks-show <uid>` — dump YAML and the mirrored DB row(s)
- `--tasks-new <type> --title <t> --uid <u> [--...params]` — scaffold directory and starter YAML
- `--tasks-update <uid> --status <s> [--result-file <path>]` — atomic YAML update then DB mirror

### Test mode & edge cases

Test mode (`--test`):

- Redirect `tasks/` to a temporary root (e.g., `data/test/tasks/`).
- Use deterministic UIDs for reproducibility.
- Simulate fetcher/LLM results; write outputs under `artifacts/`.
- Use a temp SQLite DB; produce identical logging and DB writes as real runs.

Edge cases:

- Large outputs: store in `artifacts/` and reference via `result.output_file`.
- Partial writes/crash: atomic rename avoids torn `index.md`; clean up stray `.tmp` files on startup.
- Duplicate UID: detect and log as error; ignore subsequent duplicate until resolved.
- Time skew: `yaml_hash` is authoritative for change detection; mtime is a fast path.
- Orphaned `llm_prompts`: ensure cleanup on task deletion.

### Acceptance criteria

- Creating/editing/removing task folders updates the DB and UI after sync.
- Status/timestamps displayed correctly; result is shown or linked via artifact file.
- LLM tasks populate both `tasks` and `llm_prompts` consistently.
- CLI operations are deterministic and fully logged.

### Logging & observability

- Log every sync decision with file path, uid, action, and counts.
- Include timing and error details for malformed YAML or IO failures.
- All actions adhere to the project’s logging style guide and per‑run log file policy.


## LLM Structured Transformations (Prompt‑as‑Function, No Autonomy)

This application primarily uses the local LLM as a deterministic formatter/renderer, not as an autonomous agent. A “prompt program” supplies exact instructions and an expected output contract. The model receives only the required input text (e.g., the head of a podcast RSS feed or scraped metadata) appended to a reusable prompt from `prompts/`, and must return only the requested structured output (typically a YAML front matter block). The application validates the output against a schema, writes it to the filesystem if valid, or feeds back a concise correction prompt for another attempt.

Key properties:

- No autonomy: the model never decides to take actions. It only returns a specific, schema‑conformant document.
- Filesystem is canonical: the app writes YAML to `subscriptions/` or `tasks/` and runs the sync to mirror into SQLite.
- Deterministic runs: low temperature, constrained output format, and strict extraction of the first valid YAML block.
- Retry with critique: on validation errors, provide a minimal diff/error explanation and request a corrected output; limited retries.

### Contract: prompt → output

Inputs

- A base prompt (from `prompts/`, e.g., `prompts/taxonomy_from_feeds.md`) that describes:
  - Required keys, field semantics, and examples
  - Output rules (YAML only, no prose, no extra text)
  - Any path/category conventions the app will use when saving
- The raw source text to transform (e.g., `feed.xml` head, HTTP response headers, scraped page head)

Outputs

- Exactly one YAML document that matches the required schema. The app will:
  - Extract the first fenced YAML block (or triple‑dash delimited front matter)
  - Validate against the schema
  - On success, write `index.md` with this front matter and an optional body stub
  - On failure, re‑prompt with a short list of validation errors

Example expected output envelope (one of):

```yaml
---
source_name: Example Podcast
source_uri: https://example.org/podcast.xml
source_slug: example-podcast
file_extension: xml
display_name: Example Podcast
cadence: daily
---
```

The app will ignore any text before/after the first valid YAML block and fail the render if no valid block is found.

### Inference settings (libllama default model)

- Model: `Qwen3-4B-Instruct-2507-Q8_0.gguf`
- Temperature: 0.0–0.2 (default 0.0)
- Top‑p/Top‑k: conservative defaults (e.g., top_p 0.9, top_k 40) or grammar‑constrained where possible
- Stop sequences: encourage exact output (e.g., stop on a second fenced code block)
- Optional grammar: when practical, use a YAML‑like grammar with llama.cpp to reduce format drift

### Validation & correction loop

1. Extract candidate YAML.
2. Validate against the plugin’s schema (e.g., RSS/Atom requires `source_name`, `source_uri`, `source_slug`, `file_extension`).
3. If invalid, re‑prompt with a concise list of errors and the previous output for correction. Limit to N retries (e.g., 2–3).
4. On success, write to filesystem using app logic to choose the correct path (the LLM does not pick paths).

### Filesystem integration

- Subscriptions: write `index.md` front matter and create `cache/` as needed under `subscriptions/<plugin>/<categories>/<slug>/`.
- Tasks: for LLM/fetcher tasks, render the YAML for `tasks/<type>/.../index.md` when appropriate (e.g., generated from a higher‑level operation), but typically the app scaffolds tasks directly and uses the LLM for metadata generation only.

### Safety & scope limits

- No tool use, no network autonomy: the app fetches content; the model only transforms text to YAML.
- Strict output requirement: any prose in output is considered an error.
- All runs logged: prompt, input snippet hash, output, validation result, and retries are logged to the per‑run file.

### Integration with tasks & UI/CLI

- Each render is tracked as an `llm` task (queued → running → succeeded/failed); `llm_prompts` stores the final prompt, timings, and minimal metrics.
- UI: “Render from prompt” action shows the base prompt (read‑only), the input snippet, and the YAML result; user can accept to save.
- CLI: `--llm-render --prompt prompts/taxonomy_from_feeds.md --in feed_head.xml --out subscriptions/.../index.md --dry-run`

### Acceptance criteria

- Given a valid feed head and the taxonomy prompt, the model produces a front matter YAML block that validates and is saved without manual edits.
- On common validation errors (missing keys, wrong types), the correction loop yields a valid YAML within the retry budget.
- Outputs are deterministic in practice (no extra chatter) using the specified inference settings.


## LLM Tool Use Architecture (Agent‑Accessible Ops)

A stretch goal is to eventually enable the local llm (default: Qwen3‑4B‑Instruct Q8 via libllama/llama.cpp) to create and manage resources (subscriptions, tasks, taxonomy) using the same primitives available to users. Design all operations so they are callable by an LLM acting as an optional “pair‑programmer” that can explain changes, propose actions, and execute tools when authorized.

### Goals

- Provide a least‑common‑denominator tool interface that works across diverse models (function‑calling, JSON tools, ReAct‑style).
- Expose safe, idempotent, CLI‑first operations that the LLM can invoke to perform work identical to a user’s actions.
- Keep filesystem (subscriptions/, tasks/) as source of truth; tools write YAML first, then mirror into SQLite.
- Maintain full logging and human‑in‑the‑loop approval for impactful changes.
   - Ie a model could be tasked to create a yaml file based on a subscription url, but not just random arbitrary access to the internet or filesystem.

### Tooling model

Define tools in a central Tool Registry with:

- name: stable identifier (e.g., `create_subscription_from_url`)
- description: concise purpose and constraints
- input_schema: JSON Schema for inputs
- output_schema: JSON Schema for outputs
- idempotency_key: how to dedupe (e.g., `(plugin, source_uri, target_path)`)
- side_effects: description + safety class (none/read‑only/write‑fs/write‑db/network)
- preconditions/postconditions: validation rules
- supports_dry_run: bool (simulate, no writes)

Each tool is implemented once in C++ and exposed through:

- GUI: buttons/flows (with the same underlying tool calls)
- CLI: `--tools-invoke <name> --input-json <...>`
- LLM adapter: prompt+parser glue for the active model profile

All invocations produce structured logs and, in `--test`, run against a sandboxed root and temp DB.

### Adapters for model differences

To support a broad set of models, we provide multiple prompting/IO adapters:

1. JSON Tool Call (default)
   - System prompt lists available tools and JSON Schemas.
   - Model returns a single JSON object:
     {"tool":"create_subscription_from_url","input":{...}}
   - We parse, validate against schema, then execute.

2. ReAct‑style (Thought/Action/Observation)
   - The model writes steps; when it emits `Action: <tool>\nInput: <json>`, we execute and return `Observation: <result>` back in a loop until `Final Answer`.

3. Function‑calling (OpenAI‑style mimic)
   - We render function signatures; model returns a function call object; we execute and provide tool output as a message.

Model profile selects adapter and minor formatting details. Default Qwen3‑4B‑Instruct profile uses JSON Tool Call with fenced code blocks preferred for reliability.

### Core tool set (initial)

- create_subscription_from_url
  - Inputs: `url`, `category_path` (optional), `plugin` (default `rss`), `file_extension` (default `xml`), `slug` (optional)
  - Behavior: fetch URL (network via app), discover or resolve feed link and metadata, choose path under `subscriptions/<plugin>/<category>/<slug>/`, write `index.md` with required front matter, create `cache/`, return `subscription_path` and `created` flag.

- update_subscription_metadata
  - Inputs: `subscription_path`, partial metadata
  - Behavior: modify front matter; atomic write; resync DB.

- create_task
  - Inputs: `task_type` (fetcher|llm), `title`, `payload`, plus LLM fields for llm tasks
  - Behavior: scaffold `tasks/<type>/YYYY/MM/<uid>/index.md`; optionally create `artifacts/`.

- update_task_status
  - Inputs: `uid`, `status`, optional `result` or `result_file`
  - Behavior: YAML update; resync DB; timestamps maintained.

- fs_list_subscriptions / fs_list_categories
  - Read‑only listing for context planning; useful in ReAct.

- db_query_readonly
  - Constrained SELECT with row/time limits; results returned as JSON for analysis.

All write tools support `dry_run` for simulation and report the planned diffs and resolved paths without making changes.

### Safety & governance

- Scoped capabilities: tools advertise side‑effects; policy can restrict which tools are enabled for unattended runs.
- Confirmation for destructive ops (delete/rename) unless explicitly allowed.
- Sandboxing in `--test`: temp roots for `subscriptions/` and `tasks/`, temp DB, simulated network if needed.
- Rate limits and backoff for network tools; consistent error reporting.
- Comprehensive logging: inputs, outputs, file diffs, DB mutations.

### Pair‑programming UX

- The in‑app assistant surfaces suggestions with proposed tool calls.
- Users can approve, edit, or deny each call; optional "auto" mode within specified policies.
- Every call shows a preview of file changes (front matter diff) and the resulting DB actions.

### Integration with existing flows

- Tools must write YAML first, then trigger the sync layer to mirror into SQLite (identical to user actions).
- The same tools power GUI buttons and CLI commands to ensure parity.
- Prompts under `prompts/` should include concise tool descriptions and examples to bootstrap models.

### Default model & configuration

- Ship with `Qwen3-4B-Instruct-2507-Q8_0.gguf` via libllama as default.
- Provide a model profile setting: chooses adapter, formatting, and guardrails.
- Allow additional models to be configured; unknown models fall back to JSON Tool Call adapter.

### Acceptance & testability

- Creating a subscription from a URL via the assistant produces the correct folder and `index.md`, and the GUI reflects the change.
- Creating and updating tasks via the assistant mirrors correctly to DB and task dashboards.
- `--test` mode yields identical logs and DB writes (to temp paths) without touching real data.


## Project Roadmap

### Overview

Before doing anything else, read this consolidated document: [project-specification.md](project-specification.md). Then review the [README.md](README.md) for installation and quick start instructions.

This roadmap outlines the sequential tasks and milestones for developing the C++17 RSS reader/publisher. It follows the style guide’s recommendation to keep commits atomic and tasks well-scoped. Use the status legend below to track progress.

### Legend

| Symbol | Meaning |
| ------ | -------- |
| `[ ]` | Not started |
| `[?]` | In progress / Testing / Development |
| `[x]` | Completed |
| `[-]` | Blocked / deferred |

### Milestone 1: Project Setup & Infrastructure

| Status | Task |
| ------ | ----- |
| [ ] | Create Git repository and initialise with `.gitignore` |
| [ ] | Set up directory structure (`src/`, `assets/`, `data/`, `subscriptions/`, `build/`, `bin/`) |
| [ ] | Write initial `Makefile` for building with Notcurses, SQLite and other dependencies |
| [ ] | Implement basic `Logger` class writing to root-level `log.txt` and per-run logs |
| [ ] | Implement CLI argument parsing in `main.cpp` (support `--help`, `--version`, `--init-default-subscriptions`, `--test`) |
| [ ] | Create `project-specification.md` (this consolidated document) with initial content |

### Milestone 2: TUI Skeleton (No-Op Controls)

| Status | Task |
| ------ | ----- |
| [ ] | Initialise Notcurses; create the main application planes including the **Base Menu** anchored at the bottom. |
| [ ] | Implement the **Numeric Input System** to handle chained commands (e.g., `2-1`) and map them to menu actions with zero latency. |
| [ ] | Implement the **Home View** (Menu Item 1) as the default view. |
| [ ] | Implement the **Pub View** (Menu Item 2) with input fields for title and content; buttons are present but disabled/no-op |
| [ ] | Implement the **Sub View** (Menu Item 3) with its three-column layout (Subscriptions, Posts, View). |
| [ ] | Render a placeholder directory tree in the **Subscriptions** column. |
| [ ] | Render a placeholder list of posts in the **Posts** column. |
| [ ] | Render a placeholder post detail view in the **View** column. |
| [ ] | Add a status bar just above the Base Menu to display context and feedback. |

### Milestone 3: Database

| Status | Task |
| ------ | ----- |
| [ ] | Add a **Settings View** section dedicated to database configuration, surfacing current SQLite path, size, and connection status |
| [ ] | Embed a **DB Explorer** view with resizable panes for table list, SQL editor, and asynchronous results viewer |
| [ ] | Introduce a **Tasks View** (Menu Item 4) with fetcher/LLM sub-views to stage workflow automation features |
| [ ] | Extend **Settings** with general guidance plus LLM local/API configuration (endpoint, auth, prompt editor, model picker) |
| [ ] | Tasks system (disk-first): define `tasks/` filesystem with `fetcher/` and `llm/` namespaces; each task folder contains `index.md` with YAML front matter as source of truth and optional `artifacts/` for large outputs |
| [ ] | Tasks YAML schema: common fields (`task_type`, `uid`, `title`, `status`, `created_at`, `updated_at`, `completed_at`, `payload`, `result`, `metadata`); LLM-specific fields (`prompt_text`, `model_name`, `destination_type`, `destination_uri`) |
| [ ] | Implement `TaskManager` with FS↔DB sync that upserts into `tasks` and, for LLM tasks, mirrors into `llm_prompts`; include `file_path`, `uid`, and `yaml_hash` embedded in JSON payload |
| [ ] | Change detection: full scan on startup and on-demand; detect updates via `mtime` + YAML hash; mirror deletions (remove DB rows when task folders are removed); per-file transactional upserts |
| [ ] | App writes & atomicity: update tasks by writing YAML first (`index.tmp` + atomic rename to `index.md`), then mirror into DB inside a transaction; log every operation (insert/update/skip/delete) |
| [ ] | UI wiring: extend Tasks dashboard to filter by `task_type` (Fetcher/LLM), display Title/Status/Created/Updated/Completed/Result, open the underlying `index.md`, reveal folder, and create/edit tasks |
| [ ] | CLI: add `--tasks-sync`, `--tasks-list [type]`, `--tasks-show <uid>`, `--tasks-new <type> ...`, `--tasks-update <uid> ...` to drive the tasks system without GUI |
| [ ] | Test mode & edge cases: in `--test`, redirect `tasks/` into a temp root, use deterministic UIDs, store large results in `artifacts/`, handle duplicate UIDs, and recover from partial writes |
| [ ] | Implement the core schemas described above (`tasks`, `llm_prompts`, `subscription_sources`, `posts`, `meshtastic_messages`, `meshtastic_nodes`, `meshtastic_sightings`) with migrations and unit tests |
| [ ] | Wire the tasks dashboard to a persisted Fetcher queue, update statuses in real time, and connect LLM tasks to logic pipelines |
| [ ] | Expand the database explorer to support transactional edits, guardrails (row limits, timeout), and structured logging for every statement |
| [ ] | Persist database settings from the UI with path validation, error reporting, and live reconnection |
| [ ] | Persist LLM profiles (local/API) with hydrated prompt/model selectors and integrate selections with upcoming Ollama/llama.cpp plugins |
| [ ] | Capture UI interactions (task actions, SQL execution, settings changes) in the Logger and expose equivalent CLI commands for query runner and task status output |
| [ ] | Document task queue, database workflow, and LLM configuration flows as prompts/docs so agents can automate them |
| [ ] | Ensure all database management actions invoke `DatabaseManager` APIs with transactional safety and verbose logging |
| [ ] | Document database maintenance workflows and tie them to prompts (e.g., add or update `prompts/` entries for automated assistance) |

### Milestone 4: Local LLM Structured Transformations (Prompt-as-Function)

| Status | Task |
| ------ | ----- |
| [ ] | Implement deterministic LLM renderer that builds prompt from `prompts/` + input snippet, runs libllama (Qwen3-4B-Q8) with low temperature, and captures output |
| [ ] | YAML extraction & validation: extract the first YAML/front-matter block; validate against per-plugin schema with clear, actionable errors |
| [ ] | Correction loop: on validation failure, re-prompt with concise error list and previous output; retry up to N times (e.g., 2–3) |
| [ ] | Filesystem write & sync: on success, write `index.md` front matter to the correct path (app chooses path), then trigger FS→DB sync |
| [ ] | UI: add “Render from prompt” flow that previews base prompt, input snippet, and YAML result; user can accept to save or request correction |
| [ ] | CLI: add `--llm-render --prompt <path> --in <file> [--out <index.md>] [--dry-run]` with exit codes for success/validation-failure/error |
| [ ] | Logging: record prompt hash, input hash, output hash, validation errors, retries, timings, and final write target in per-run log |
| [ ] | Tests/fixtures: add sample podcast feed head and golden YAML; verify pipeline yields valid YAML within retry budget under `--test` |
| [ ] | Prompt updates: ensure `prompts/taxonomy_from_feeds.md` explicitly requires YAML-only output, lists required keys, and includes a minimal valid example |
| [ ] | LLM tasks integration: record each render as an `llm` task with status transitions (queued→running→succeeded/failed); upsert into `tasks` and `llm_prompts` with `file_path`, `uid`, `yaml_hash`, and link any artifact/output files |
| [ ] | Tasks dashboard wiring: display render tasks with model, destination, input snippet hash, validation status; provide actions to open result, reveal `artifacts/`, and retry correction |
| [ ] | CLI task helpers: `--tasks-new llm --title <t> --prompt <p> --in <file> [--uid <u>]` scaffolds a render task; `--tasks-update <uid> --status <s> [--result-file <path>]` transitions state; `--tasks-sync` ingests outputs into DB |
| [ ] | Error handling policy: after N validation failures mark task `failed`, attach concise error summary in `result`, and persist the last invalid YAML into `artifacts/invalid.yaml` |
| [ ] | Test mode parity: ensure llm render tasks in `--test` run against temp roots/DB, write artifacts there, and produce identical logs and state transitions |

### Milestone 5: Subscription System & Data Storage (wired to TUI skeleton)

| Status | Task |
| ------ | ----- |
| [ ] | Implement `Subscriptions` manager: create `subscriptions/` if missing, ensure each subscription-type plugin subfolder (e.g., `rss/`, `meshtastic/`) exists, walk directory trees, and load YAML front matter  |
| [ ] | Automatically create a taxonomy.txt file within the subscriptions directory which recursively lists all category paths starting with "subscriptions/" (only list non-source, non-cache directories). This process should happen on every startup, and any time something happens tha would affect the contents of the list. |
| [ ] | Design and implement `DatabaseManager` using SQLite to store preferences and UI state; migrate temporary UI state persistence from file to SQLite |
| [ ] | Create a theme table in the database to store user-selected colours and styles |
| [ ] | Implement a `ThemeManager` to save, load, and apply user-selected themes from the database. Include resizing windows, window position, etc. |
| [ ] | Build a filesystem synchroniser that diff-scans `subscriptions/` on startup and after every change, reconciling additions, renames, and deletions into `subscription_sources` and cascading removals into dependent tables |
| [ ] | When fetchers write cache files or parsed posts, update the `posts` table to mirror the filesystem (including removing pruned cache entries) |
| [ ] | Write helper functions to parse and validate `index.md` front matter, applying plugin-specific schemas (e.g., enforce `source_name`, `source_uri`, `source_slug`, `file_extension` for RSS/Atom) using a YAML library |
| [ ] | Wire the left tree in the TUI to real subscription data; selection updates the centre and right planes (still showing placeholders until Fetcher lands) |
| [ ] | Implement functions to create, rename and delete categories and subscriptions through the TUI (menus and dialogs) |

### Milestone 6: Basic Fetcher (HTTP/S + RSS/Atom) integrated with TUI

| Status | Task |
| ------ | ----- |
| [ ] | Integrate libcurl (or equivalent) for HTTP/HTTPS requests |
| [ ] | Implement asynchronous fetching on background threads; avoid blocking the TUI |
| [ ] | Parse RSS and Atom payloads into `Post` structures |
| [ ] | Populate the post list in the centre column with parsed items and update the detail view on selection |
| [ ] | Implement background Fetcher tick (1–5 min) to evaluate cadence per subscription |
| [ ] | On success, write `cache/YYYY-MM-DD-HH-mm-ss.<ext>` and atomically update `latest.<ext>` |
| [ ] | Maintain SQLite tables: `subscriptions`, `fetches`, `latest_payloads`, `posts` with indexes and upsert semantics |
| [ ] | Rescan filesystem changes to keep DB in sync with `subscriptions/` and YAML front matter |
| [ ] | Implement per-subscription locking, retries, and backoff with robust logging |
| [ ] | Add a simple in-TUI activity indicator and last-fetch status per subscription (non-blocking notifications area or status bar) |

### Milestone 7: Core Plugin Architecture with TUI wiring

| Status | Task |
| ------ | ----- |
| [ ] | Define abstract interfaces for `SourcePlugin`, `DestinationPlugin` and `LogicPlugin` |
| [ ] | Implement `PluginFactory` for registering and retrieving plugins by protocol and content type |
| [ ] | Implement initial `RssHttpsPlugin` supporting HTTP/HTTPS, parsing RSS and Atom feeds; expose enable/disable in the TUI |
| [ ] | Implement a simple logic plugin (e.g., keyword filter) and surface it in a TUI debug panel for per-subscription tests |
| [ ] | Implement `OllamaLogicPlugin` for LAN LLM calls with streaming support and prompt templates (TUI toggle only for now) |
| [ ] | Implement a stub destination plugin that logs published content for testing; add a TUI “Test publish” button |

### Milestone 8: Logic Blocks, Filters & Views TUI

| Status | Task |
| ------ | ----- |
| [ ] | Implement right-most column as a logic pipeline area with expandable menu |
| [ ] | Add "Filters & Transformations" tool to define SQL-based filters and transformations against SQLite tables |
| [ ] | Implement dynamic Views list: Data Table (default), Charts, Media Playlist, Photo Gallery, Rendered Markdown |
| [ ] | Enable chaining logic blocks so one block’s output becomes the next block’s input |
| [ ] | Persist view configurations and logic block results in the configuration database |

### Milestone 9: Fetcher Protocol Extensions (beyond HTTP/S RSS/Atom)

| Status | Task |
| ------ | ----- |
| [ ] | Integrate IPFS/IPNS via external binary; extract and launch the daemon on demand |
| [ ] | Integrate Tor via bundled binary and configure SOCKS proxy for HTTP requests |
| [ ] | Handle JSON Feed parsing (in addition to RSS/Atom) |

### Milestone 10: Publishing Features

| Status | Task |
| ------ | ----- |
| [ ] | Implement TUI components for the **Pub View** (Menu Item 2): fields for title, content, media attachments, and protocol selectors |
| [ ] | Implement `DestinationPlugin` for ActivityPub publishing (e.g., send posts to a configured server) |
| [ ] | Implement destination plugin for AT protocol (e.g., BlueSky) |
| [ ] | Add ability to schedule posts and display publishing status to the user |
| [ ] | Persist drafts and published posts for future editing |

### Milestone 11: Logic Layer & Advanced Features

| Status | Task |
| ------ | ----- |
| [ ] | Implement logic plugins: keyword filter, tagger, summariser using local or remote LLMs |
| [ ] | Provide TUI to enable/disable logic plugins per subscription and configure parameters |
| [ ] | Implement test mode path: run logic plugins and publishing in simulation without side effects |
| [ ] | Integrate Ollama-backed summarisation, bias analysis, and counter-argument logic blocks |

### Milestone 12: Meshtastic (Cyberpony Express) Integration

| Status | Task |
| ------ | ----- |
| [ ] | Detect Meshtastic node via USB (auto or user-provided path) and create node root under `subscriptions/` |
| [ ] | Mirror channels as child subscriptions with plugin-managed `index.md` metadata and a `cache/` for received messages |
| [ ] | Implement `MeshtasticSourcePlugin` to subscribe to text/data and convert to `Post` objects |
| [ ] | Implement `MeshtasticDestinationPlugin` to watch per-channel `outbox/` and send via `sendText` / `sendData` |
| [ ] | Expose channels in **sub** tree; enable channel selection in **pub** destinations with queue status |

### Milestone 13: User Settings & Theming

| Status | Task |
| ------ | ----- |
| [ ] | Implement a comprehensive **Settings View** (Menu Item 0) to configure fetch cadence defaults, TUI themes, and account credentials, etc |
| [ ] | Allow users to change global colours; save preferences in the database and apply across the TUI |
| [ ] | Provide a log viewer accessible from the TUI that reads `log.txt` and run logs |

### Milestone 14: Packaging & Distribution

| Status | Task |
| ------ | ----- |
| [ ] | Finalise static builds for Linux and Windows; ensure external binaries (Tor, IPFS) are packaged and extracted at runtime |
| [ ] | Write scripts or Make targets for cross-compiling with `mingw-w64` |
| [ ] | Write installation instructions and sample configuration in the README |
| [ ] | Tag a beta release and solicit user feedback |

### Milestone 15: Documentation & Final Polishing

> **Evergreen requirement:** treat these tasks as part of every change, not a one-time endgame pass. Whenever code, specs, or assets change, rerun the checklist below so agents always keep documentation and polish current.

| Status | Task |
| ------ | ----- |
| [ ] | Update `README.md` with usage examples, screenshots and troubleshooting tips |
| [ ] | Maintain this `project-specification.md` throughout development |
| [ ] | Add inline code comments and docstrings; ensure all functions are documented |
| [ ] | Review logs and refine error handling and user messages |
| [ ] | Final release: package binaries, update documentation and create release notes |

### Milestone 16: Test Mode & Simulation Enhancements

| Status | Task |
| ------ | ----- |
| [ ] | Extend `--test` mode to simulate Ollama API responses and Meshtastic packets without external dependencies |
| [ ] | Ensure simulation uses temporary directories and produces the same logs and DB writes (to temp DB) as real runs |
| [ ] | Add CLI flags to dump current Fetcher status, DB counts, and plugin registrations for LLM-friendly inspection |

### Deliverables

1. Source code organised under `src/` with clear module separation.
2. `Makefile` for building and packaging the application.
3. `README.md` describing installation, usage and contributing guidelines, and linking prominently to this document.
4. `project-specification.md` (this consolidated document including the specification, roadmap, social context, and style guides).
5. A functioning TUI application supporting the described features, along with logs and test mode.
