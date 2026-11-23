# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project purpose

Monorepo POC of a banking web app that exists mainly as a **target for Cypress E2E/API test automation**. The app code (back + front) is deliberately simple; the test suites under `Cypress-bank-front/cypress/` are the real deliverable. The README (in French) holds the full business rules and a roadmap (Phase 2 = more Cypress suites, custom `cy.login` with `cy.session()`, `cy.intercept` mocks, Page Object Model in `cypress/support/pages/`, fixtures). Project docs, comments and test names are written in French — keep that convention.

## Commands

Backend (Java 17, Spring Boot 3.2, Maven) — port 8080:
```bash
cd Cypress-bank-back && mvn spring-boot:run
```

Frontend (React 19 + Vite) — port 5173:
```bash
cd Cypress-bank-front
npm install
npm run dev      # dev server
npm run build
npm run lint     # eslint .
```

Cypress (both servers must be running first):
```bash
cd Cypress-bank-front
npx cypress open                                        # interactive
npx cypress run                                         # headless, all specs
npx cypress run --spec cypress/e2e/bank_login.cy.js     # single spec
```

There are no backend unit tests yet.

## Architecture

**Backend — single file** `Cypress-bank-back/src/main/java/com/cypressbank/CypressBankApplication.java` contains everything: the app class, Java `record` DTOs (Account, Transaction, Beneficiary, OfferRange, ApiSuccess…) and one `BankController` (`/api`, `@CrossOrigin("*")`). State is **in-memory lists** on the controller, seeded by `resetData()` (also called from the constructor). Endpoints: `POST /auth/login`, `GET /dashboard`, `POST /beneficiaries`, `POST /transfers`, `POST /subscriptions`, `POST /reset`. Business-rule violations map to specific HTTP codes (400 invalid input / terms not accepted, 401 bad credentials, 409 duplicate, 422 rule violation such as 3 000 € transfer cap, insufficient balance, invalid IBAN, income too low) — tests assert on these, so keep them stable. Test credentials: client `12345678` / `Banque2026!` (user "Jean Dupont", "Gamme Essentiel", accounts `ACC_COURANT` and Livret A).

**Frontend — single component** `Cypress-bank-front/src/App.jsx`: all state in `useState`, no router. Views are switched by state (`user` null → login card; otherwise tabs `activeTab` ACCUEIL / virements / gammes, with `rangeCategoryTab` CARTES / épargne). API base URL is hard-coded as `API_URL = "http://localhost:8080/api"`; after each mutation the UI re-fetches `/dashboard`. Login is not a real session — there's no token sent on later calls.

**Test contract — `data-cy` attributes**: every interactive/asserted element has a `data-cy` attribute (e.g. `login-card`, `input-client-id`, `btn-login`, `top-navbar`, `alert-error`, and id-suffixed ones like `account-balance-ACC_COURANT`). Tests select only via `[data-cy=...]`. When changing UI, preserve or add `data-cy` attributes and update specs if renaming.

**Test isolation**: each spec's `beforeEach` calls `cy.request("POST", "http://localhost:8080/api/reset")` then `cy.visit("/")`, so tests never depend on each other's state.

**CI** (`.github/workflows/cypress.yml`, on push to `main`): starts the backend with `mvn spring-boot:run &`, waits on `:8080/api/dashboard`, then `cypress-io/github-action@v6` starts `npm run dev` and runs Cypress headless.

## Lint notes

- Specs under `cypress/` are linted with `eslint-plugin-cypress` (flat config in `eslint.config.js`), so Cypress globals need no `/* global */` comment.
