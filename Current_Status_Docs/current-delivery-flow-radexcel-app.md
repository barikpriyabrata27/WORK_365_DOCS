# radexcel-app — current delivery flow

Current state only. There is no GitHub Actions workflow in this repo. A developer laptop builds and publishes. This file is the map for adding CI (build, security, release) and CD (deploy to the IIS EC2 server) later.

`radexcel-ux` is a separate repository. Nothing in this flow builds or deploys it.

## What this repo ships

Three pieces, each on its own schedule. The two APIs share one SQL Server database.

| Piece | Project | Published name on IIS |
|---|---|---|
| Auth API | `backend/Application/RadExcel.Auth.Microservice` | `AuthManagement.Api` → site `staging-auth` or `production-auth` |
| Company API | `backend/Application/RadExcel.Company.Microservice` | `TenantManagement.Api` → site `staging-company` or `production-company` |
| Frontend | `react/` (Vite) | static site `staging-app` or `production-app` |
| Database | `backend/Databases/RadExcel.Wfm.Database` | one `.dacpac`, deployed by `RadExcel.Wfm.DatabaseMigrationRunner` |

`docker-compose.yml` also runs Redis and a one-shot `db-migrator`. That stack is local development only. Staging and Production do not deploy from Docker. Staging Redis is an Upstash instance loaded from AWS Secrets Manager (`radexcel-dev-redis`). SQL Server stays on its own EC2 instance (SQL Server Express, per the team), not inside the IIS host.

## Environments

| Environment | Where it runs | How config is chosen |
|---|---|---|
| Development | Developer laptop (`dotnet run`, or `docker compose up`) | User secrets via `deployment/Set-DevConnectionString.ps1`. No connection string is committed. |
| Staging | IIS sites on EC2 `172.31.64.196` (`RADXL_HImgAppSvr`), HTTP port 80 | `ASPNETCORE_ENVIRONMENT=Staging` on the API app pools. Secrets from AWS Secrets Manager. |
| Production | Same IIS server, HTTPS port 443 | `ASPNETCORE_ENVIRONMENT=Production` on the API app pools. `deploy_db.ps1` requires an explicit `-ConnectionString`. |

Staging and Production are six top-level IIS sites on one machine, created by `deployment/Setup-IisEnvironments.ps1` (run on the server, as Administrator). Default folders: `C:\wfm\staging` and `C:\wfm\production`, each with `app`, `auth.micro`, and `company.micro`.

| Site | Host | App pool |
|---|---|---|
| Staging UI | `http://staging.radexcel.rad365.com` | `staging-app` (static files) |
| Staging Auth | `http://staging-auth.radexcel.rad365.com` | `staging-auth` |
| Staging Company | `http://staging-company.radexcel.rad365.com` | `staging-company` |
| Production UI | `https://radexcel.rad365.com` | `production-app` |
| Production Auth | `https://auth.radexcel.rad365.com` | `production-auth` |
| Production Company | `https://company.radexcel.rad365.com` | `production-company` |

There is no shared DEV server. Laptops are the development environment.

## Flow today

```
Developer laptop
  │
  ├─ Local run (not a deploy)
  │    docker compose up          auth-api, company-api, redis, react-ui, db-migrator
  │    or dotnet run + pnpm dev   SQL Server on the laptop (user secrets)
  │
  └─ Manual publish (PowerShell in deployment/)
       │
       ├─ Database (one step, build + deploy)
       │    deploy_db.ps1 -Environment Staging|Production
       │      → build RadExcel.Wfm.Database.dacpac
       │      → run DatabaseMigrationRunner against that environment's SQL Server
       │
       ├─ APIs (build, then publish)
       │    build_api.ps1 -Environment <Env>
       │      → dotnet publish both APIs
       │      → artifacts/work/<Env>/api/
       │    then one of:
       │      publish_api.ps1          copy into artifacts/releases/<Env>/current/  (still on the laptop)
       │      publish_api_remote.ps1   Web Deploy to the IIS EC2
       │
       └─ Frontend (build, then publish)
            build_ui.ps1 -Environment <Env>
              → pnpm install --frozen-lockfile
              → pnpm build in react/
              → API URLs baked in (VITE_AUTH_API_URL, VITE_TENANT_API_URL)
              → artifacts/work/<Env>/ui/dist/
            then one of:
              publish_ui.ps1            copy into artifacts/releases/<Env>/current/react/
              publish_api_remote.ps1    Web Deploy of that dist/ to the UI site
```

Staging commands, from `deployment/DEPLOYMENT.md`:

```powershell
.\deployment\deploy_db.ps1 -Environment Staging
.\deployment\build_api.ps1 -Environment Staging
.\deployment\build_ui.ps1  -Environment Staging
.\deployment\publish_api_remote.ps1 -Environment Staging
```

`publish_api_remote.ps1` syncs both APIs and the frontend in one run. `-SkipFrontend` publishes APIs only. Default target is `172.31.64.196` port `7281` (Web Deploy / WMSvc). The server hostname does not resolve from a typical dev machine, so the script uses the IP. The deploy password is a Windows credential on that server (`RADXL_HImgAppSvr\kc` in the script remarks), prompted or loaded from a DPAPI-encrypted local file. It is not stored in git.

`publish_api.ps1` and `publish_ui.ps1` do not touch AWS. They only copy the build into `artifacts/releases/<Environment>/current/` on the laptop. Someone still has to get that folder onto the server.

The database script is not split into build and publish. It builds the dacpac and applies it in the same run. For Staging it reads `ConnectionStrings:Default` from the `radexcel-dev-db` secret (AWS credentials on the laptop, for example `AWS_PROFILE=radexcel-dev`). For Production the connection string must be passed on the command line.

## What a workflow would call

These scripts are the CI and CD steps. They already exist. GitHub Actions would run them on a runner instead of a laptop.

| Future stage | Script | Output |
|---|---|---|
| CI build — APIs | `deployment/build_api.ps1` | `artifacts/work/<Env>/api/` |
| CI build — UI | `deployment/build_ui.ps1` | `artifacts/work/<Env>/ui/dist/` plus `build-info.json` |
| CI build — database | the build half inside `deployment/deploy_db.ps1` | `RadExcel.Wfm.Database.dacpac` |
| CD — IIS | `deployment/publish_api_remote.ps1` | Web Deploy to the six sites |
| CD — database | `deployment/deploy_db.ps1` | dacpac applied to that environment's SQL Server |

Security and release do not exist as steps. There is no `.github/` folder, no dependency scan, no secret scan, no code scan, and no versioned release artifact. `artifacts/work/<Env>/` is overwritten on every build. Nothing records which commit was published.

`publish_api_remote.ps1` can also take `-FrontendSource frontend` and look for `build_frontend_ui.ps1` plus a `frontend/` folder. That script and folder are not in this checkout. The UI this repo actually builds is `react/`.

## Server prerequisites (already manual)

Run on the IIS EC2 once, before the first deploy. Documented in `deployment/DEPLOYMENT.md` section 2.2.

1. `deployment/Setup-IisEnvironments.ps1` or `Setup-StagingEnvironment.ps1` — sites and app pools.
2. `deployment/Set-AwsAppPoolCredentials.ps1` — AWS access key on the API pools so they can read Secrets Manager, Cognito, and S3.
3. IIS URL Rewrite module — the React `web.config` needs it, or a refresh on a client-side route returns 404.
4. Web Deploy (WMSvc) listening on port 7281.

Staging API pools need `secretsmanager:GetSecretValue` on `radexcel-dev-db`, `radexcel-dev-cognito`, `radexcel-dev-superadmin`, `radexcel-dev-smtp`, `radexcel-dev-redis`, and `radexcel-dev-service-api-key`, plus Cognito admin actions and S3 on `dev-radexcel-s3-bucket` (Company API). Secret names still say `dev` because they were created before this environment was called Staging.

## Gaps that matter for CI/CD

- No pull-request build. A push is not compiled until someone runs the scripts.
- No scan before publish.
- No approval between Staging and Production. Both are sites on the same EC2, and the same laptop can publish either.
- Staging is HTTP. The refresh token is a `Secure` cookie, so sign-in on Staging does not work until those sites move to HTTPS (`deployment/DEPLOYMENT.md` section 2.1).
- Deploy credentials and AWS keys live on developer machines and in IIS app-pool environment variables, not in a pipeline secret store.
- `radexcel-ux` is outside this pipeline. The screens in that repo are not what `staging-app` / `production-app` serve.
