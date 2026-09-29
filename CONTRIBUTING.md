# Contributing

Thanks for your interest in contributing! This guide applies to every project under [@kasuken](https://github.com/kasuken), including Brainy, LearnStack, Needly and MoneyBrain. A project may add its own `CONTRIBUTING.md` with specific details; when it does, that file takes precedence.

## Before you start

- Read and follow the [Code of Conduct](CODE_OF_CONDUCT.md).
- **Security issues must not be reported in public issues.** See the [Security Policy](SECURITY.md).
- For questions and help, see [SUPPORT.md](SUPPORT.md).

## Licensing and the Contributor License Agreement (CLA)

These projects are licensed under the **GNU Affero General Public License v3.0 (AGPL-3.0)**. They are also operated as hosted SaaS products.

All contributors must sign the [Contributor License Agreement](CLA.md) before a pull request can be merged. You keep the copyright on your work. The CLA grants the maintainer the rights needed to distribute your contribution under the AGPL-3.0 and under other licenses, for example to offer commercial licenses.

Signing takes one comment. On your first pull request, a bot will ask you to post this:

```
I have read the CLA Document and I hereby sign the CLA
```

You only need to sign once per repository.

## Ways to contribute

- **Report a bug.** Open an issue using the bug report template. Include steps to reproduce, expected vs. actual behavior, and your environment.
- **Suggest a feature.** Open an issue using the feature request template. Describe the problem before the solution.
- **Improve the docs.** Fixes to typos, unclear setup steps and missing explanations are always welcome.
- **Write code.** Look for issues labelled [`good first issue`](https://github.com/search?q=user%3Akasuken+label%3A%22good+first+issue%22+state%3Aopen&type=issues) or `help wanted`.

For anything larger than a small fix, **open or comment on an issue first** so we can agree on the approach before you invest time.

## Development workflow

1. Fork the repository and create a branch from `main`:
   `git checkout -b fix/short-description`
2. Follow the project's README to set up your local environment.
3. Make your change, with tests where it makes sense.
4. Make sure the build and tests pass locally:
   ```bash
   dotnet build
   dotnet test
   ```
5. If you change the data model, add an EF Core migration. Do not edit existing migrations.
6. Push your branch and open a pull request against `main`, filling in the pull request template.

### Pull request guidelines

- Keep pull requests focused: one change per PR.
- Write a clear title and description, and link the related issue (`Closes #123`).
- Update documentation and the `CHANGELOG.md` (if the project has one) when behavior changes.
- Make sure CI passes. A maintainer will review your PR as soon as possible.
- Never commit secrets. Use `dotnet user-secrets` or environment variables for local configuration.

### Commit messages

Write short, imperative commit messages, for example `Fix due date parsing for UTC offsets`. [Conventional Commits](https://www.conventionalcommits.org/) prefixes (`feat:`, `fix:`, `docs:`…) are welcome but not required.

### Code style

- Follow the existing style of the code you are changing, and any `.editorconfig` in the repository.
- Run `dotnet format` before committing.
- Prefer small, readable methods and meaningful names over comments.

### AI-assisted contributions

AI coding assistants are welcome. Some repositories include agent instructions (`AGENTS.md`, `.github/copilot-instructions.md`, `.github/skills/`). You are responsible for every line you submit: review, understand and test AI-generated code before opening a pull request.

## Hosted service vs. self-hosting

Paid features of the hosted services (such as Stripe billing) live in the open source code behind configuration flags and are disabled by default. A self-hosted instance runs without them. Contributions must keep the application working with these features disabled.

The project names and logos are trademarks of the maintainer and are not covered by the AGPL-3.0. If you run a modified public instance, please use a different name.
