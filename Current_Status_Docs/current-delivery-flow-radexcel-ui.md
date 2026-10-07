# radexcel-ux — current delivery flow

Current state only. There is no GitHub Actions workflow, no deployment script, and no IIS or AWS publish path in this repo. Each app is built and viewed on a developer laptop.

`radexcel-app` is a separate repository. It has its own `react/` frontend, and that is the UI the IIS sites (`staging-app`, `production-app`) serve today. This repo does not call those APIs.

## What this repo contains

Three independent Vite apps. Each has its own `package.json`. Nothing at the repo root builds all three.

| Folder | npm name | What it is | Scripts |
|---|---|---|---|
| `app/` | `rad365wms` | Main WMS screens | `npm start` (Vite), `npm run build`, `npm run preview`. `npm test` is left over from Create React App and does not run. |
| `cr/` | `storyboard` | UI storyboard / prototype | its own Vite app |
| `home/` | `rad365-wms-home` | Public marketing page | `npm run dev`, `npm run build` (`tsc && vite build`), `npm run preview`, `npm run lint` |

`app/` listens on `http://localhost:5173` (`app/README.md`). Login in `app/src/services/authService.ts` is a local mock: it waits, then checks `data/users.ts`. The file says no backend is wired up.

## Flow today

```
Developer laptop
  │
  ├─ app/     npm install && npm start     → http://localhost:5173   (mock login)
  ├─ cr/      npm install && npm start     → storyboard, local only
  └─ home/    npm install && npm run dev   → marketing page, local only

No build artifact is published.
No Web Deploy.
No IIS site.
No database.
No AWS.
```

The root `README.md` describes git usage:

1. Branch off `main` (`feature/…`, `fix/…`, `chore/…`).
2. Push the branch.
3. Open a pull request into `main`.
4. Delete the branch after merge.

That README says `main` is protected. Whether GitHub branch protection is actually turned on has to be checked in the GitHub organization. There is no workflow that would fail a pull request if the build breaks, because there is no workflow.

## What a workflow would call

A CI workflow for this repo would install and build each app on its own. There is no shared script equivalent to `radexcel-app`'s `deployment/build_ui.ps1`.

| Future stage | Command | Output |
|---|---|---|
| CI build — WMS | in `app/`: `npm ci` (or `npm install`) then `npm run build` | `app/dist/` |
| CI build — storyboard | same inside `cr/` | `cr/dist/` |
| CI build — home | in `home/`: `npm ci` then `npm run build` | `home/dist/` |
| CI lint | `home/` has `npm run lint`. `app/` does not. | — |
| Security scan | nothing exists | — |
| CD to the IIS EC2 | nothing exists | — |

`app/dist/`, `cr/dist/`, and `home/dist/` are not inputs to `radexcel-app`'s `publish_api_remote.ps1`. That script publishes `radexcel-app/react/` (or a `frontend/` folder that is not in the current `radexcel-app` checkout). Connecting this repo to the IIS sites is a later decision: which of the three apps ships, and whether it replaces `radexcel-app/react`.

## Gaps that matter for CI/CD

- No pull-request build, so a broken `npm run build` is only visible on the laptop that ran it.
- No dependency or secret scan.
- No release artifact and no version recorded anywhere.
- No deploy target. Shipping `app/` to `staging-app` / `production-app` would be a new path, not a script that already runs.
- `app/` cannot be deployed as the product UI until it calls the Auth and Company APIs. Today it does not.
- Branch protection is described in the README and still needs to be confirmed on GitHub.
