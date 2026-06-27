---
name: deployer-cicd
description: Run Deployer from a CI/CD pipeline (GitHub Actions, GitLab CI, Bitbucket Pipelines) with deploy concurrency guards, SSH private-key and known_hosts secrets, and a dotenv-secrets upload task. Use when automating Deployer runs in a pipeline — editing .github/workflows/*.yml, .gitlab-ci.yml, or bitbucket-pipelines.yml, or wiring deploy SSH keys and secrets into CI.
---

# Running Deployer in CI/CD

Automate `dep deploy` from a pipeline. Three rules apply everywhere:

1. **Guard against concurrent deploys** — never let two deploys run at once.
2. **Inject the SSH private key (and known_hosts) as CI secrets** — never commit them.
3. **Keep app secrets out of the repo** — upload them with a Deployer task.

## GitHub Actions

Use the official [`deployphp/action`](https://github.com/deployphp/action). The
`concurrency:` key is what prevents overlapping deploys.

```yaml
name: deploy

on:
  push:
    branches: [master]

concurrency: production_environment

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: "8.1"

      - name: Install dependencies
        run: composer install

      - name: Deploy
        uses: deployphp/action@v1
        with:
          private-key: ${{ secrets.PRIVATE_KEY }}
          dep: deploy
```

Rules of thumb:
- `concurrency: production_environment` is **important** — it prevents concurrent deploys. Don't omit it.
- Pass the SSH key through `private-key: ${{ secrets.PRIVATE_KEY }}`; store it as a repository/environment secret, never inline.
- `dep:` is the Deployer command the action runs (here `deploy`).

## GitLab CI/CD

Set two CI/CD variables in the GitLab project first:

- `SSH_KNOWN_HOSTS` — contents of `~/.ssh/known_hosts`. Get the host keys with `ssh-keyscan`, e.g. `ssh-keyscan deployer.org`.
- `SSH_PRIVATE_KEY` — the private key used to connect to the hosts. Generate one with `ssh-keygen -t ed25519 -C 'gitlab@deployer.org'`.

```yaml
stages:
  - deploy

deploy:
  stage: deploy
  image:
    name: deployphp/deployer:v8
    entrypoint: [""]
  before_script:
    - mkdir -p ~/.ssh
    - eval $(ssh-agent -s)
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - chmod 644 ~/.ssh/known_hosts
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - > /dev/null
  script:
    - dep deploy -vvv
  resource_group: production
  only:
    - master
```

Concurrency on GitLab:
- `resource_group: production` ensures only one deployment job runs at a time.
- Enable **skip outdated deployment jobs** (on by default) so older deployment jobs are cancelled automatically when a newer deploy starts.

## Bitbucket Pipelines

1. Generate a new SSH key and add it to your workspace for the server, then add the public key to the server.
2. Define the environment variables you reference in your deploy commands.

```yaml
pipelines:
  branches:
    develop:
      - stage:
          # target deployment name — it inherits that environment
          deployment: staging
          name: Deploy Staging
          steps:
            - step:
                name: Composer Install
                image: composer/composer:2.2
                caches:
                  - composer
                script:
                  - composer install --quiet
                artifacts:
                  # persist so the deploy step can pick them up
                  - vendor/**
            - step:
                name: NPM Install
                image: node:22-bullseye-slim
                caches:
                  - node
                script:
                  - npm install --silent
                artifacts:
                  - public/build/**
            - step:
                name: Deployer Deploy
                timeout: 6m # error out if it takes longer
                image: deployphp/deployer:v8
                script:
                  # pass env vars from the "staging" deployment environment
                  - php /bin/dep deploy --branch=$DEVELOP stage=$STAGING
```

Notes:
- Build steps (composer, npm) write to `artifacts:` so the later deploy step can pick the files up.
- The deploy step pulls values like `$DEVELOP` / `$STAGING` from the `deployment:` environment.

## Deploy secrets (dotenv) — applies to all pipelines

Don't commit secrets. Store the dotenv contents in a CI **file variable** (e.g.
a GitLab file variable named `DOTENV`) and upload it as part of the deploy. Many
frameworks read secrets from a dotenv file, so write it into `shared/.env`:

```php
task('deploy:secrets', function () {
    upload(getenv('DOTENV'), '{{deploy_path}}/shared/.env');
});
```

Run this task **immediately after updating the code** so the new release has its
secrets in place.
