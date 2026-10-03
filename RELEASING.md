# Releasing to production

Brainy, LearnStack, MoneyBrain and Needly all ship the same way, through the shared workflow
[`release-azure-webapp.yml`](.github/workflows/release-azure-webapp.yml). Each app has a small
`.github/workflows/release.yml` that calls it with the app's names and URLs.

Merging to `main` never deploys. Production changes only when you cut a release.

## Cut a release

1. Merge the changes to `main` and let CI pass.
2. Optional: in the last PR, move the `## [Unreleased]` entries in `CHANGELOG.md` under
   `## [X.Y.Z] - YYYY-MM-DD`. That section becomes the release notes; without it GitHub
   generates them from the merged PRs.
3. Run the **Release** workflow on `main` and pick the bump:

   ```bash
   gh workflow run release.yml -R kasuken/<App> -f bump=minor   # patch | minor | major
   gh run watch -R kasuken/<App>
   ```

The workflow then:

1. works out the next version from the highest `vX.Y.Z` tag,
2. waits for `ci.yml` to pass on that commit (it never deploys a red commit),
3. builds once with that version stamped into the assemblies,
4. deploys through the `production` environment, logging in to Azure with OIDC,
5. runs any database migrations first (Needly only; the others migrate on startup),
6. smoke tests `/health/ready`, any extra pages, and the HTTP to HTTPS redirect,
7. only then creates the `vX.Y.Z` tag and GitHub release.

Publishing a release by hand in the GitHub UI (with a new `vX.Y.Z` tag) also works: it deploys
that tag.

## Roll back

Redeploy an earlier tag. Nothing new is tagged.

```bash
gh workflow run release.yml -R kasuken/<App> -f redeploy=v1.4.2
```

Startup migrations are not reversed, so roll back only to versions that work with the
current database schema.

## Where configuration lives

| What | Where |
| --- | --- |
| Defaults that suit self-hosting (billing, licensing and integrations off) | `appsettings.json` in the repo |
| Hosted-only switches (`Billing__Provider=Stripe`, `Licensing__Enabled=true`, ...) and every secret | App Service application settings |
| Azure login for the workflow | `production` environment secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` |

The workflow never changes app settings, and no `appsettings.Production.json` turns hosted
features on. A self-hosted instance in Production mode keeps the open-source defaults.

## One-time Azure setup for a new app

Each app has a user-assigned managed identity `id-<app>-github-deploy`. It has a federated
credential for the repo's `production` environment and the **Website Contributor** role on that
web app only (Needly also has **SQL Server Contributor**, for the migration firewall rule).

```bash
app=myapp; rg=MyApp.Prod; site=myapp-prod-001; repo=kasuken/MyApp
az identity create -g $rg -n id-$app-github-deploy
subject="$(gh api repos/$repo/actions/oidc/customization/sub --jq .sub_claim_prefix):environment:production"
az identity federated-credential create -g $rg --identity-name id-$app-github-deploy \
  -n github-release-production --issuer https://token.actions.githubusercontent.com \
  --subject "$subject" --audiences api://AzureADTokenExchange
az role assignment create --assignee-object-id "$(az identity show -g $rg -n id-$app-github-deploy --query principalId -o tsv)" \
  --assignee-principal-type ServicePrincipal --role "Website Contributor" \
  --scope "$(az webapp show -g $rg -n $site --query id -o tsv)"
```

Then on GitHub: create the `production` environment, allow deployments from `main` and `v*`
tags only, and add the three `AZURE_*` secrets to it. Publish profiles and basic-auth
publishing credentials stay off.
