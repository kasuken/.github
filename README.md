# kasuken/.github

Default community health files for all public repositories owned by [@kasuken](https://github.com/kasuken), including [Brainy](https://github.com/kasuken/Brainy), [LearnStack](https://github.com/kasuken/LearnStack), [Needly](https://github.com/kasuken/Needly) and [MoneyBrain](https://github.com/kasuken/MoneyBrain).

GitHub uses these files automatically for any repository that doesn't have its own version:

| File | Purpose |
|---|---|
| [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) | Contributor Covenant 2.1 |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute |
| [CLA.md](CLA.md) | Contributor License Agreement, required for all contributions |
| [SECURITY.md](SECURITY.md) | How to report vulnerabilities privately |
| [SUPPORT.md](SUPPORT.md) | Where to get help |
| [.github/ISSUE_TEMPLATE](.github/ISSUE_TEMPLATE) | Bug report and feature request forms |
| [.github/pull_request_template.md](.github/pull_request_template.md) | Pull request checklist |

`LICENSE` is not inherited: each repository has its own (AGPL-3.0).

## Production releases

[`.github/workflows/release-azure-webapp.yml`](.github/workflows/release-azure-webapp.yml) is the reusable workflow every app calls to release to Azure App Service. [RELEASING.md](RELEASING.md) explains how to cut a release, roll back, and set up a new app.

## CLA workflow template

[`workflow-templates/cla.yml`](workflow-templates/cla.yml) runs the CLA Assistant bot. Workflows are not inherited, so copy it into each repository as `.github/workflows/cla.yml`. Signatures are stored in each repository's `cla-signatures` branch.
