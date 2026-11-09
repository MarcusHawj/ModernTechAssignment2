# GitHub Actions Workflow Analysis

This document analyzes the `.github/workflows/deploy.yml` workflow used to test and deploy the TechFlow Solutions website.

## 1. What triggers this workflow to run?

The `on:` section defines two triggers:

- **`push` to `main`**: The workflow runs whenever code is pushed or merged into the `main` branch.
- **`pull_request` targeting `main`**: The workflow also runs whenever a pull request is opened or updated against `main`.

Both triggers run the `build-and-test` job, but only a push to `main` continues on to the `deploy` job.

## 2. What are the four main steps this workflow performs?

The `build-and-test` job performs four steps:

1. **Checkout code**: Downloads the repository's code onto the runner.
2. **Validate HTML**: Checks the HTML files for errors using `html5validator-action`.
3. **Check links**: Scans for broken links using `github-action-markdown-link-check`. This step uses `continue-on-error: true`, so a broken link is reported in the logs but does not stop the workflow or block deployment.
4. **Upload artifact**: Packages the website files so they can be deployed to GitHub Pages.

After these steps pass, the separate `deploy` job runs a final step, **Deploy to GitHub Pages**, which publishes the uploaded site.

## 3. What does the "Checkout code" step do and why is it necessary?

The "Checkout code" step uses `actions/checkout@v4` to copy the repository's files onto the GitHub Actions runner, a fresh `ubuntu-latest` virtual machine. The runner starts out empty and has no access to the code until it is checked out. Without this step, the HTML validator and link checker would have nothing to inspect, and there would be no website files to upload and deploy.

## 4. What is the purpose of the environment configuration?

The `environment` section in the `deploy` job sets the deployment target:

- **`name: github-pages`** tells GitHub this job deploys to the `github-pages` environment. GitHub tracks each deployment there, shows its status on the repository page, and can apply protection rules to it.
- **`url: ${{ steps.deployment.outputs.page_url }}`** records the live website's URL, which comes from the output of the deployment step. This makes the link to the deployed site appear in the Actions run and the repository's Deployments section.

The `permissions` section (`pages: write` and `id-token: write`) works alongside it, giving the job only the access it needs to publish to GitHub Pages.

## 5. How does this automated deployment improve reliability compared to manual deployment?

- **Consistent process**: Every deployment follows the same steps in the same order, so nothing is forgotten or done differently from one release to the next.
- **Testing before release**: The `deploy` job uses `needs: build-and-test`, so the site is only deployed if the HTML validation and other build steps succeed. Broken code is stopped before it reaches the live site.
- **Early feedback on pull requests**: Because the tests also run on pull requests, problems are caught during code review, before the changes are merged into `main`.
- **Less human error**: No one has to manually copy files to a server, which removes mistakes like uploading the wrong version or missing a file.
- **Traceability**: Each run is logged in the Actions tab, so it is easy to see which commit was deployed, when, and whether it passed.

## 6. What would happen if you pushed code to a different branch (not main)?

Pushing to a different branch, such as `feature/about-section`, would **not** trigger the workflow, because the `push` trigger only lists `main`. Nothing would be tested or deployed.

If a pull request is then opened from that branch into `main`, the `pull_request` trigger runs the `build-and-test` job to check the changes. The `deploy` job is still **skipped**, because its `if` condition (`github.event_name == 'push' && github.ref == 'refs/heads/main'`) only allows deployment on a push to `main`. You saw this on PR #1: the build-and-test check passed and the deploy check was skipped. The website only updates after the pull request is merged into `main`.
