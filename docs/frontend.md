# Frontend packages (KLB platform)

Client libraries for apps built on the KLB platform, the Karpeles Lab PHP platform REST API, served at `/_rest/<Endpoint>` on each site. `klbfw` is the base JavaScript client; the React, Vue and i18n packages build on top of it. `klbfw-rs` (Rust), `swiftrest` (Swift) and `atonline_api` (Flutter) are the same client for other languages. The API itself is documented in **https://github.com/KarpelesLab/integration-docs**. Read it first for endpoint naming, auth (session, OAuth2, Ed25519 API keys), paging, errors, the upload protocol and the User Flow login. Use `klbfw-describe` to inspect live endpoints.

## Quick pick

| Need | Use |
|------|-----|
| Call the KLB REST API from browser or Node.js | [`@karpeleslab/klbfw`](#klbfw) |
| Explore or type-generate KLB endpoints (CLI or MCP server) | [`@karpeleslab/klbfw-describe`](#klbfw-describe) |
| React app with SSR and cached REST hooks | [`@karpeleslab/react-klbfw-hooks`](#react-klbfw-hooks) |
| React hooks for user/session and API errors | [`@karpeleslab/klb-react-services`](#klb-react-services) |
| New Vue 3 site | [`vuetemplate`](#vuetemplate) + [`klbfw-login`](#klbfw-login) |
| Login / registration UI (User Flow v2, passkeys, OAuth) | [`@karpeleslab/klb-login-vue`](#klbfw-login) |
| i18next translations from the platform | [`@karpeleslab/i18next-klb-backend`](#i18next-klb-backend) |
| Rust / Swift / Flutter client | [`klbfw-rs`](#klbfw-rs), [`swiftrest`](#swiftrest), [`atonline_api`](#flutter-atonline_api) |

---

## klbfw
**Repo:** https://github.com/KarpelesLab/klbfw · **npm:** `@karpeleslab/klbfw` (0.2.x) · **License:** MIT · **Status:** active, the base of every KLB frontend

A CommonJS library that ships TypeScript types (`index.d.ts`) and runs in browser, SSR and Node.js.
- REST calls: `rest(api, method, params, context)`, `restGet(api, params)` and `restSSE`.
- Uploads: `uploadFile` and `uploadManyFiles`, using direct PUT or S3 multipart.
- Context and state accessors: `getLocale`, `getContext`, `getRealm`, `getCurrency`, `getUuid`, `getI18N` and others.
- Cookie helpers and pluggable auth through `setAuth`.

In the browser it relies on the page-injected `FW` global and session cookie, so the site must be served by the platform or by a dev proxy such as `vuetemplate` sets up. Node.js apps use OAuth2 through `require('@karpeleslab/klbfw/auth-node')` (`AuthInfo`, `bearerAuth`). Node uploads also need `node-fetch` and `@xmldom/xmldom`.

```bash
npm install @karpeleslab/klbfw
```

```javascript
import { rest, restGet, uploadFile, getLocale } from '@karpeleslab/klbfw';

// Resolves to RestResponse { result: 'success', data, paging?, request_id, time }
// Rejects with RestError { result: 'error', exception, error, code, token, request }
const res = await rest('User:get', 'GET');
console.log(res.data);

const up = await uploadFile('Misc/Debug:testUpload', file); // File, Blob or Buffer
```

**Gotcha:** `upload.append`/`upload.init` are deprecated; use `uploadFile`.

## klbfw-describe
**Repo:** https://github.com/KarpelesLab/klbfw-describe · **npm:** `@karpeleslab/klbfw-describe` (0.5.x) · **License:** MIT

A CLI that describes KLB API endpoints through OPTIONS requests. It shows methods, parameters and procedures, and can generate TypeScript types with `--ts`. The default host is `ws.atonline.com`; change it with `--host`. It can also run as an MCP server, which is useful for coding agents:

```bash
npx @karpeleslab/klbfw-describe Misc/Debug            # describe an endpoint
npx @karpeleslab/klbfw-describe --ts User              # emit TS types
claude mcp add klbfw-describe -s user -- npx -y @karpeleslab/klbfw-describe --mcp
```

## react-klbfw-hooks
**Repo:** https://github.com/KarpelesLab/react-klbfw-hooks · **npm:** `@karpeleslab/react-klbfw-hooks` (0.4.x) · **License:** no license file

React hooks and SSR runtime for klbfw. Supports React 16.8 to 19 and react-router 5 to 7.
- `run(routes, ...)` replaces `ReactDOM.render`/`hydrate` and handles SSR with the Data Router.
- Hooks: `useRest(path, params, noThrow, cacheLifeTime)` returns `[data, refresh]` and caches results. `useVar` and `useVarSetter` give named shared state. `usePromise` makes SSR wait for data. `useRestRefresh` and `useRestResetter` refresh or clear the cache.

## klb-react-services
**Repo:** https://github.com/KarpelesLab/klb-react-services · **npm:** `@karpeleslab/klb-react-services` (1.0.x) · **License:** no license file

React hooks, context providers and TypeScript definitions for KLB services, organized by API area. Examples: `useUser()` returns `{ user, loading, error }`; `useApiErrorHandler()` is also provided. Depends on klbfw.

## klbfw-login
**Repo:** https://github.com/KarpelesLab/klbfw-login · **npm:** `@karpeleslab/klb-login-core`, `@karpeleslab/klb-login-vue` (1.0.x) · **License:** MIT

The login and registration module for klbfw sites. It implements the User Flow v2 protocol (`User:flow`): password, OTP, OAuth2, passkeys/WebAuthn, email verification and recovery.

The work is split into two packages:
- `klb-login-core` is a framework-agnostic engine and renderer with no dependencies. It is loaded from a CDN pinned to major version 1, so new auth methods reach every site without a redeploy. A bundled copy is the fallback.
- `klb-login-vue` is a thin Vue 3 plugin: `app.use(KlbLogin, { theme })`, then `<KlbLogin @success=...>`.

Prefer it over hand-writing a User Flow UI.

## vuetemplate
**Repo:** https://github.com/KarpelesLab/vuetemplate · **Not on npm**: clone it as a starting point · **License:** no license file

A base template for KLB websites: Vue 3, TypeScript, Vite, klbfw and vue-i18n with locale auto-detection. The dev server injects the `FW` variable and proxies the API. A service worker handles version headers. To start a project:
1. Clone the repo.
2. Set `Realm=usrr-...` in `etc/registry_dev.ini`.
3. Edit `etc/i18n/user_flow.csv`.
4. Run `npm install && npm run dev`.

## i18next-klb-backend
**Repo:** https://github.com/KarpelesLab/i18next-klb-backend · **npm:** `@karpeleslab/i18next-klb-backend` (0.1.x) · **License:** no license file

An i18next backend that loads translations from the platform. It uses the `FW.i18n` global when present, otherwise fetches `/l/<lang>/_special/locale.json` and falls back to `/_special/locale/<lang>.json`. Ships TypeScript types.

```javascript
import i18next from 'i18next';
import { Backend } from '@karpeleslab/i18next-klb-backend';
import { getLocale } from '@karpeleslab/klbfw';
i18next.use(Backend).init({ lng: getLocale(), load: 'currentOnly', fallbackLng: false, ns: ['translation'], defaultNS: 'translation' });
```

## react-autoruby
**Repo:** https://github.com/KarpelesLab/react-autoruby · **npm:** `@karpeleslab/react-autoruby` (0.0.2) · **Status:** old (2020), small

A React component that renders Japanese ruby (furigana) from inline markup: `<AutoRuby text="明日[Ashita]の[]明日[Ashita]"/>`. The delimiter is configurable with `delimiter="()"`.

## klbfw-dynfetch
**Repo:** https://github.com/KarpelesLab/klbfw-dynfetch · **npm:** `@karpeleslab/klbfw-dynfetch` (1.0.x) · **License:** no license file

A Puppeteer-based tool that fetches the fully rendered HTML of a JavaScript-rendered page. From the CLI: `npx @karpeleslab/klbfw-dynfetch <url>`. From code: `const { fetchRenderedHtml } = require('@karpeleslab/klbfw-dynfetch')`.

## fyvue
**Repo:** https://github.com/KarpelesLab/fyvue · **npm:** `@karpeleslab/fyvue` (0.2.6) · **License:** no license file · **Status:** legacy; no changes since 2023

A Vue 3 component library for KLB systems. Its peer dependencies are pinned to the 2023 stack: klbfw ^0.1, pinia 2, i18next 22 and `@fy-/*`. Prefer `vuetemplate` + `klbfw-login` for new work.

## klbfw-rs
**Repo:** https://github.com/KarpelesLab/klbfw-rs · **Crate:** `klbfw` (crates.io 0.1.x) · **License:** no license file

The Rust client, built on a pure-Rust stack (rsurl, purecrypto). Features:
- `Client` (named `RestContext` before 0.1.3).
- OAuth2 tokens with automatic renewal.
- Ed25519 API-key request signing.
- PUT, multipart and S3 multipart uploads with progress.
- Path-based access to response values.

Add it with `klbfw = "0.1"`. Example call: `let user: User = Client::new().apply("Users/Get", "GET", serde_json::json!({"userId": "123"}))?;`

## swiftrest
**Repo:** https://github.com/KarpelesLab/swiftrest · **SwiftPM:** `.package(url: "https://github.com/KarpelesLab/swiftrest.git", from: "0.2.0")` · **License:** BSD-3-Clause

The Swift client, for macOS 12+, iOS 15+, tvOS and watchOS, with Swift 5.9+. It uses async/await and actors. Entry points: `RestClient(config: RestClientConfig(host:))`, then `TokenAuthentication` (OAuth2 with refresh) or `APIKeyAuthentication` (Ed25519), then `client.requestWithRetry("User:get", method: .get)`. It supports chunked uploads with progress.

## Flutter (atonline_api)
**Repo:** https://github.com/KarpelesLab/atonline-flutter · **pub.dev:** `atonline_api` (0.5.x)

The Flutter/Dart client: OAuth2 login widget, REST wrapper, user management, deep links and uploads. Create it with `AtOnline('your_app_id')`. See `klbfw-flutter.md` in integration-docs.
