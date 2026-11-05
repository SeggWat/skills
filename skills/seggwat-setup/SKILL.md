---
name: seggwat-setup
description: Add SeggWat user feedback collection to a web app with the SeggWat MCP server. Trigger this skill whenever the user wants to add a feedback button, feedback widget, bug report button, "send feedback" form, user feedback collection, or in-app feedback to a website or web app, or mentions setting up or installing SeggWat in a codebase. Also trigger on "let users report bugs", "how do I collect feedback from my users", or "add a feedback widget to my Next.js/Nuxt/React/Vue/Astro/Svelte site". Covers connecting the MCP server, creating the project, installing the widget snippet in the right file for the framework, tagging feedback with the app version and signed-in user, and verifying that the first feedback arrived.
---

# SeggWat Setup

Add the SeggWat feedback widget to a web app: a button on every page that lets users send bugs, ideas and questions, with optional screenshots. Feedback lands in the SeggWat dashboard.

Three MCP calls do the work: `setup_project` creates the project and returns the snippet, you install it, `verify_setup` confirms the first feedback arrived.

## 1. Connect the MCP server

Check whether the `seggwat` MCP server is already connected (its tools include `setup_project` and `verify_setup`). If not:

```bash
# Claude Code: OAuth, no token. Run /mcp afterwards and approve in the browser.
claude mcp add --transport http seggwat https://seggwat.com/mcp
```

Other clients (Cursor, Windsurf, VS Code) take the same URL in their MCP config:

```json
{
  "mcpServers": {
    "seggwat": { "type": "http", "url": "https://seggwat.com/mcp" }
  }
}
```

Clients without OAuth need an Organization Access Token as `Authorization: Bearer oat_…` (Dashboard → Settings → API Tokens).

**No SeggWat account yet?** The sign-in page creates one: the user enters their email, confirms the one-time code, and gets an organization with a 14-day Pro trial. No credit card. Connecting needs **admin or owner** access to the organization.

## 2. Create the project

Call `setup_project`:

| Field | Value |
|---|---|
| `name` | The app's name, e.g. from `package.json` `name` or the repo name |
| `allowed_origins` | Every origin the widget runs on: production **and** the local dev server, e.g. `["https://acme.com", "http://localhost:3000"]`. Omit to allow every origin |

`setup_project` is idempotent: an existing project with the same name is returned instead of a duplicate. `allowed_origins` only applies on creation, so get it right the first time. An origin missing from the list rejects the widget's submissions, so include the dev server's origin (with its port) or the test in step 5 fails. Origins can be edited later in the dashboard.

It fails with a billing link when the organization has no active trial or has reached its plan's project limit. Show the user that link; don't retry.

The result:

- `resource.id`: the project id, needed for `verify_setup`
- `resource.dashboard_url`: where the user configures the widget's look
- `env`: `SEGGWAT_PROJECT_KEY`. It is **public**: it ships in every page, so it is not a secret
- `install`: one snippet per target (`html`, `nextjs`, `nuxt`), each with a `file_hint`

## 3. Install exactly one snippet

Add it **once**, in the layout rendered on every page:

| Framework | File | Snippet |
|---|---|---|
| Next.js (App Router) | `app/layout.tsx`, inside `<body>` | `nextjs` |
| Next.js (Pages Router) | `pages/_app.tsx` or `pages/_document.tsx` | `nextjs` |
| Nuxt | `app.vue` | `nuxt` |
| Vite (React, Vue, Svelte, Solid) | `index.html`, before `</body>` | `html` |
| SvelteKit | `src/app.html`, before `</body>` | `html` |
| Astro | The base layout used by every page, before `</body>` | `html` |
| Remix / React Router | `app/root.tsx`, inside `<body>` | `html` (as JSX) |
| Rails, Django, Laravel, Phoenix, plain HTML | The base template, before `</body>` | `html` |

Use the snippet from the `setup_project` result: it carries the real project key and script URL. Never invent a project key.

Don't wrap the script in a component that re-renders: the widget installs itself once per page load and survives client-side navigation.

## 4. Optional: version and user

Only when it applies to the app:

- **Release version.** If the app has a version (e.g. `version` in `package.json`), add `data-version="<version>"` to the script tag (in Next.js, as a `data-version` prop on `<Script>`). Feedback is then grouped by release in the dashboard.
- **Signed-in users.** If the app has accounts, identify the user after sign-in and reset on sign-out, so feedback is attributed and the email field is pre-filled:

  ```js
  window.SeggwatFeedback?.setUser(user.id, user.email);
  window.SeggwatFeedback?.logout(); // on sign-out
  ```

  Use `logout()`, not `setUser(null)`: only `logout()` clears the stored email and chat token.

- **Feedback in the app's own navigation.** Only if the user asks for no floating button, or for feedback in their nav, sidebar or footer. Add `data-button-position="none"` to the script tag, and `data-seggwat-open` with an empty value to the existing nav or footer item:

  ```html
  <button data-seggwat-open>Feedback</button>
  <button data-seggwat-open="screenshot">Report a bug</button>
  ```

  The empty form opens the widget and gets a red dot when a reply is waiting. `"screenshot"` starts a capture, which also needs `data-enable-screenshots="true"`. Module names such as `bug_report` open that form. The dot needs a signed token: mint a short-lived HS256 JWT on the backend with the project's end-user signing secret, then call `setUser(user.id, user.email, token)`. Never send the secret to the browser. Docs: https://seggwat.com/docs/features/portal-authentication

Change nothing else. Color, position, label, screenshots and language are set in the dashboard (`resource.dashboard_url`), not in code. Adding `data-*` appearance attributes overrides the dashboard and confuses the user later. The navigation setup above is the one exception, because its markup depends on that attribute.

## 5. Verify

1. Start the dev server (or deploy) and open a page.
2. Ask the user to click the feedback button and send a test message. Or, with a browser tool, do it yourself.
3. Call `verify_setup` with `{ "id": "<resource.id>" }`.

`waiting` means nothing has arrived yet, and comes with a hint. `receiving` means it works. If it stays `waiting`:

- The browser console shows a 403 on `/api/v1/feedback/submit` → the page's origin isn't in `allowed_origins`. Add it in the dashboard under the project's settings.
- No button is visible → expected with `data-button-position="none"`: use the `data-seggwat-open` element instead. Otherwise the script isn't in a layout this page renders, or a Content Security Policy blocks `https://seggwat.com`. Add `https://seggwat.com` to `script-src`, `style-src` and `connect-src`.

## 6. Report back

Tell the user, briefly:

- The file(s) you changed
- The dashboard link (`resource.dashboard_url`) for appearance, screenshots and notifications
- Whether `verify_setup` reported `receiving`

## Beyond the feedback button

Each needs its own snippet. Offer them, don't add them unasked:

- **Helpful rating** (thumbs up/down at the end of docs or articles): `seggwat-helpful.js`
- **Surveys** (NPS, CSAT, CES), configured in the dashboard: `seggwat-survey.js`
- **Public feedback board and changelog**: no code, enabled in the dashboard

Docs: https://seggwat.com/docs/widget/installation
