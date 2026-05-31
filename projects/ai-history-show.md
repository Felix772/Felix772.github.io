---
layout: page
permalink: /projects/ai-history-show/
title: AI History Show
---

## AI History Show

- Built an interactive front-end exhibition app for major milestones in AI history
- Designed for large-screen gallery displays with adaptive single-screen, mobile, and dual-screen layouts
- Uses **Three.js** for a 3D globe tied to event geography
- Supports Chinese/English runtime switching with bilingual milestone content
- Includes content-generation tooling, a local admin UI, deployment validation, Docker packaging, and CI-oriented quality checks
- [GitHub](https://github.com/bjzgcai/AI-History-Show)

## Table of contents
- [Experience Goal](#experience-goal)
- [Display Experience](#display-experience)
- [Content Model](#content-model)
- [Internationalization](#internationalization)
- [Admin & Generation Workflow](#admin-generation-workflow)
- [Layout & Interaction Engineering](#layout-interaction-engineering)
- [Quality & Deployment](#quality-deployment)
- [Technical Stack](#technical-stack)

---

## Experience Goal

AI History Show is a museum-style timeline app rather than a conventional website. The goal is to present AI milestones as a spatial, visual exhibition:

- historical narrative organized by era
- event details with locations, people, quotes, images, and videos
- a globe-based geographic context for each event
- large-screen readability for live exhibition settings
- bilingual content for Chinese and English audiences

The project currently organizes **21 events** across **4 categories**:

| Category | Events | Timespan |
|---|---:|---|
| Genesis of AI | 3 | 1950s-1970s |
| Neural Networks and Connectionism | 4 | 1980s-2000s |
| Deep Learning and Unified Paradigms | 7 | 2010s-2020s |
| Large Models and Scientific Intelligence | 7 | 2018-2025 |

---

## Display Experience

The presentation layer includes:

- adaptive `index.html` entry for single-screen, mobile, and ultrawide/dual-screen setups
- fixed `dual-screen.html` entry for exhibition hardware
- Three.js globe rendering with event location markers
- keyboard navigation with left/right arrows
- touch swipe navigation for large interactive screens
- image viewer with fullscreen browsing
- embedded YouTube metadata and optional local commentary videos
- auto-play mode for dual-screen exhibition loops

The dual-screen path is designed for on-site display where the browser may be stretched across two physical screens or a GPU-composed ultrawide display.

---

## Content Model

The data model separates curated source content from generated runtime data.

```text
manage/catalog.js
manage/events.js
manage/figure-avatars.js
resources/videos/*.json
        |
        v
node manage/generate.js
        |
        v
milestones-data.js
```

`catalog.js` controls category order and which events appear. `events.js` contains the event-level content:

- year
- title
- location name, country, and coordinates
- description
- key figures and roles
- quotes and sources
- local image paths
- video ids

`figure-avatars.js` acts as a canonical registry for portraits, and `scripts/report-figure-avatars.js` can audit missing portrait metadata.

---

## Internationalization

The app ships with built-in Chinese/English support:

- supported locales: `zh` and `en`
- default locale: Chinese
- active locale stored in `localStorage` as `ai-history-locale`
- runtime language toggle in both single- and dual-screen layouts
- content fields can be plain strings or bilingual objects like `{ zh, en }`
- missing locale values fall back gracefully

This is important because the project is not simply translated at the UI-label level; milestone titles, descriptions, figure roles, quotes, and location fields can all carry bilingual content.

---

## Admin & Generation Workflow

The local admin service is a lightweight content management tool:

```bash
npm run start:admin
```

It serves an admin page at `http://localhost:3001/admin` for editing categories and event content. After saving, the admin workflow regenerates `milestones-data.js` so the static exhibition page can refresh with updated content.

Security boundary:

- the static exhibition page can be hosted publicly
- the admin service has no authentication and is intended for local/protected use only
- deployment notes recommend SSH tunneling or Nginx Basic Auth if remote access is needed

---

## Layout & Interaction Engineering

### Adaptive Layout Routing

`shared/layout-router.js` decides whether the browser should use single-screen or dual-screen mode. It considers:

- explicit `?layout=single` or `?layout=dual` overrides
- viewport width
- aspect ratio
- coarse pointer detection for handheld devices
- stable dual-screen entry behavior

The dual-screen threshold is intentionally high so normal desktop windows stay in single-screen mode, while ultrawide exhibition layouts route to the dual-screen entry.

### Swipe Navigation

`shared/swipe-navigation.js` implements pointer/touch navigation with guardrails:

- minimum horizontal swipe distance
- vertical-cancel threshold
- ignore selectors for controls, iframes, timeline elements, photo viewers, and videos
- support for touch, pen, and touch-emulated mouse contexts
- cleanup through a returned `destroy()` method

That separation makes the gesture logic testable outside the full page.

---

## Quality & Deployment

The project includes repeatable local and CI checks:

```bash
npm run quality
npm run validate:deployment
```

`quality` runs ESLint, Prettier format checks, and JavaScript tests. `validate:deployment` regenerates data, runs layout/swipe tests, starts the presentation and admin servers, and verifies HTTP startup behavior.

Container deployment is also supported:

```bash
docker build -t ai-history-show .
docker run --rm -p 8000:8000 ai-history-show
```

The deployment guide covers:

- static Nginx hosting
- Docker and Docker Compose
- PM2 for the admin service
- Gitee Pages as a quick demo option
- Windows dual-screen/kiosk presentation setup
- limitations of browser UI, `--app`, `F11`, GPU display composition, and true ultrawide fullscreen

---

## Technical Stack

- **Frontend runtime**: HTML5, CSS3, JavaScript ES6+
- **3D rendering**: Three.js from CDN
- **Content tooling**: Node.js scripts
- **Quality tooling**: ESLint, Prettier, Node-based validation scripts
- **Deployment**: static server, Nginx, Docker, Docker Compose, optional PM2 admin process

The architecture keeps the exhibition itself static and portable, while using Node.js only for content generation, admin editing, and validation.
