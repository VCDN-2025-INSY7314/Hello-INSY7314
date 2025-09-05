# PulseVote Backend — CI/CD to CircleCI + Render

This guide summarizes the exact steps we followed to wire pulsevote-backend into CircleCI and Render using a CI/CD pipeline.

By the end of this guide, your API will have
* CI on CircleCI that runs ESLint and Jest with no secrets in repo.
* A Docker image build in the pipeline.
* A controlled deploy to Render using a Deploy Hook that triggers after CI passes, followed by a timed /health check.

> Important Note: If CircleCI doesn't connect to your repo in the organisation, fork your repo to your personal account and use that for both CircleCI and Render: See here: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo.
Then, ensure that you also sync the fork as you push to the original repo so changes sync to your forked repo so CircleCI can deploy. See here: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/syncing-a-fork. To skip sync now each time you commit, try one of these: https://chatgpt.com/share/68ba8cf2-ac38-8011-a6a7-459a9b549672

## 1. Prerequisites

1. GitHub Organization & Repo
   * Repo is private in an organization. - done already
   * In your GitHub org: enable Deploy Keys and install the CircleCI GitHub App with access to this repo. - done by EB
   * Authorize the CircleCI app at the org level. - done by EB

2. Node & Docker
   * Node 22 LTS or compatible runtime. - should be done
   * Dockerfile present at repo root (see below). - done already

3. MongoDB Atlas
   * Create a test database user with least privileges. - done already
   * Ensure Network Access allows CI/Render to connect (temporarily `0.0.0.0/0` for staging, or allowlist proper egress IPs). - done already

4. Server Requirements
   * Your Express app must:
     * Bind to `process.env.PORT` and 0.0.0.0. - done already
     * Expose a /health endpoint returning HTTP 200. - done already
   * On Render, you’ll run HTTP only inside the container (set `USE_HTTPS=false`). - done already

Remember: Never commit `.env` with real secrets. Use CircleCI and Render env vars.

## 2. CircleCI Setup

Docs: https://circleci.com/docs/guides/getting-started/getting-started/

### Add the config file

Create `.circleci/config.yml`:

```yaml
version: 2.1

executors:
  node_executor:
    docker:
      - image: cimg/node:22.0
    resource_class: medium

  base_docker:
    docker:
      - image: cimg/base:stable
    resource_class: medium

jobs:
  lint_and_test:
    executor: node_executor
    steps:
      - checkout

      - restore_cache:
          keys:
            - v1-npm-{{ arch }}-{{ checksum "package-lock.json" }}
            - v1-npm-{{ arch }}
            - v1-npm-
      - run:
          name: Install dependencies
          command: npm ci
      - save_cache:
          key: v1-npm-{{ arch }}-{{ checksum "package-lock.json" }}
          paths:
            - ~/.npm

      - run:
          name: Sanity check required env
          command: |
            test -n "$MONGO_URI" || { echo "MONGO_URI is NOT set"; exit 1; }
            test -n "$JWT_SECRET" || { echo "JWT_SECRET is NOT set"; exit 1; }
            echo "Env OK"

      - run:
          name: Run Lint
          command: npm run lint

      - run:
          name: Run tests
          command: npm test

  docker_build_and_healthcheck:
    executor: base_docker
    steps:
      - checkout
      - setup_remote_docker

      - run:
          name: Build Docker image
          command: |
            IMAGE_NAME="${IMAGE_NAME:-pulsevote-backend}"
            docker build -t "$IMAGE_NAME:$CIRCLE_SHA1" .

      - run:
          name: Run container
          command: |
            IMAGE_NAME="${IMAGE_NAME:-pulsevote-backend}"
            test -n "$MONGO_URI" || { echo "MONGO_URI is NOT set"; exit 1; }
            test -n "$JWT_SECRET" || { echo "JWT_SECRET is NOT set"; exit 1; }

            IMAGE_NAME="${IMAGE_NAME:-pulsevote-backend}"
            docker run -d --name pulsevote \
              -e NODE_ENV=production \
              -e PORT=5000 \
              -e USE_HTTPS=false \
              -e MONGO_URI="$MONGO_URI" \
              -e JWT_SECRET="$JWT_SECRET" \
              "$IMAGE_NAME:$CIRCLE_SHA1"

            for i in {1..60}; do
              STATUS="$(docker inspect --format='{{.State.Health.Status}}' pulsevote 2>/dev/null || echo starting)"
              echo "Container health: $STATUS"
              [ "$STATUS" = "healthy" ] && echo "Healthy" && break
              sleep 2
            done

            [ "$(docker inspect --format='{{.State.Health.Status}}' pulsevote 2>/dev/null || echo unhealthy)" = "healthy" ] || {
              echo "Container failed to become healthy"
              echo "---- Logs ----"
              docker logs --tail=200 pulsevote || true
              exit 1
            }
      - run:
          name: Cleanup
          when: always
          command: docker rm -f pulsevote || true

  deploy_to_render:
    docker:
      - image: cimg/node:22.0
    resource_class: medium
    environment:
      HEALTH_DELAY_SECS: "60"
      HEALTH_RETRY_SECS: "5"
      HEALTH_MAX_ATTEMPTS: "24"
      RENDER_HEALTH_PATH: "/health"
    steps:
      - run:
          name: Trigger Render deploy hook
          command: |
            test -n "$RENDER_DEPLOY_HOOK" || { echo "RENDER_DEPLOY_HOOK is NOT set"; exit 1; }
            curl -fsSL -X POST "$RENDER_DEPLOY_HOOK"
            echo "Deploy triggered."

      - run:
          name: Wait, then check health endpoint
          command: |
            test -n "$RENDER_SERVICE_URL" || { echo "RENDER_SERVICE_URL is NOT set"; exit 1; }
            BASE_URL="${RENDER_SERVICE_URL%/}"
            HEALTH_PATH="${RENDER_HEALTH_PATH:-/health}"
            HEALTH_URL="${BASE_URL}${HEALTH_PATH}"

            echo "Sleeping ${HEALTH_DELAY_SECS}s before health check..."
            sleep "${HEALTH_DELAY_SECS}"

            echo "Checking: ${HEALTH_URL}"
            ATTEMPTS=0
            until [ "$ATTEMPTS" -ge "$HEALTH_MAX_ATTEMPTS" ]; do
              HTTP_CODE="$(curl -L -sS -o /tmp/health_body.txt -w "%{http_code}" "$HEALTH_URL" || echo "000")"
              echo "Attempt $((ATTEMPTS+1))/$HEALTH_MAX_ATTEMPTS: $HTTP_CODE"
              if [ "$HTTP_CODE" = "200" ]; then
                echo "----- /health body -----"
                tail -c 1000 /tmp/health_body.txt || true
                echo
                echo "Health check PASSED"
                exit 0
              fi
              ATTEMPTS=$((ATTEMPTS+1))
              sleep "${HEALTH_RETRY_SECS}"
            done

            echo "Health check FAILED after $((HEALTH_MAX_ATTEMPTS*HEALTH_RETRY_SECS))s"
            echo "----- Last response body -----"
            tail -c 2000 /tmp/health_body.txt || true
            exit 1
workflows:
  pulsevote:
    jobs:
      - lint_and_test
      - docker_build_and_healthcheck:
          requires:
            - lint_and_test
          filters:
            branches:
              only: main
      - deploy_to_render:
          requires:
            - lint_and_test
            - docker_build_and_healthcheck
          filters:
            branches:
              only: main
```

### CircleCI → Project → Environment Variables

Add these project env vars:
* `MONGO_URI` → MongoDB connection string
* `JWT_SECRET` → strong value for tests and container health run
* `NODE_ENV` → `test`
* `USE_HTTPS` → `false` (the CI container run is HTTP-only)
* `RENDER_DEPLOY_HOOK` → from Render - Settings → Deploy Hooks
* `RENDER_SERVICE_URL` → e.g. `https://pulsevote-backend.onrender.com` (no trailing slash)

## 3. Render Setup

1. Create Service → New → Web Service → Connect GitHub repo.
2. Environment: `Docker` | Branch: `main` | Auto Deploy: Off | Health Check Path: `/health`.
3. Add Environment Variables (Render dashboard):
   * `MONGO_URI` (Atlas prod/staging)
   * `JWT_SECRET`
   * `NODE_ENV=production`
   * `USE_HTTPS=false`
   * *(Optional)* `CORS_ORIGINS`, `CSP_CONNECT`
4. Settings → Deploy Hooks: copy the Deploy Hook URL and paste into CircleCI `RENDER_DEPLOY_HOOK`.
5. Networking to MongoDB: Ensure MongoDB allows connections from Render (temp `0.0.0.0/0` for staging or use Render Static Egress).

Render performs an initial deploy when the service is first created. Ensure you have Auto Deploy: Off to ensure only CircleCI triggers future deploys.

## 4. Test Pipeline Flow

1. Developer pushes to **main**.
2. CircleCI `lint_and_test job` runs ESLint + Jest.
3. CircleCI `docker_build_and_healthcheck` builds the Docker image and ensures the container becomes healthy using your `/health` route.
4. CircleCI `deploy_to_render` posts the Deploy Hook, waits 60s, then polls `RENDER_SERVICE_URL + /health` until it gets **HTTP 200** or fails with logs.

## 5. Test in Postman
Update postman tests to use your API in render instead of localhost. It should all work fine.