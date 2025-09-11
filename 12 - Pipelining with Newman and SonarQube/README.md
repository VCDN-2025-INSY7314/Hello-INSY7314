# PulseVote Backend — CI incorporating Newman and SonarQube 

This guide will take you through the process of:

1. Exporting a Postman collection and testing in the pipeline.  
2. Setting up SonarQube/SonarCloud and running analysis against your code and test coverage.  

We’ll use the `circleci/config.yml` you already have as the base.

## Postman → Newman in CI/CD

### Step 1. Export your Postman collection
1. Open Postman.  
2. Select your collection → click ... → Export → choose Collection v2.1.  
3. Save it in your repo under:  

```
/postman/collection.json
```

> This file will be copied into the Newman container at runtime.

---

### Step 2. Edit collection JSON
Inside the collection JSON, replace hard-coded values to use environment variables. 
For example - passwords:

```json
"body": {
  "mode": "raw",
  "raw": "{ \"email\": \"{{ADMIN_EMAIL}}\", \"password\": \"{{TEST_PASSWORD}}\" }"
}
```

Also, update the protocol and host to environment variables for all endpoints. Remove the port as we don't need it here.
For example:
```json
"url": {
    "raw": "{{PROTOCOL}}://{{HOST}}/api/auth/register-manager",
    "protocol": "{{PROTOCOL}}",
    "host": [
        "{{HOST}}"
    ],
    "path": [
        "api",
        "auth",
        "register-manager"
    ]
}
```
We will inject these variables via `--env-var` flags in Newman.

> Note: look at the end of this guide for a sample `collection.json`
---

### Step 3. Run Newman tests in CircleCI

There is a job  already in your `config.yml` under for building the docker container. Update it to match the below.
Make sure to scroll down and see the summary points explaining the changes.

```yaml
docker_build_and_newman_tests:
    executor: base_docker
    steps:
      - checkout
      - setup_remote_docker

      - run:
          name: Build Docker image
          command: |
            IMAGE_NAME="${IMAGE_NAME:-pulsevote-backend}"
            TAG="${CIRCLE_SHA1:-latest}"
            docker build -t "${IMAGE_NAME}:${TAG}" .

      - run:
          name: Run container
          command: |
            test -n "$MONGO_URI" || { echo "MONGO_URI is NOT set"; exit 1; }
            test -n "$JWT_SECRET" || { echo "JWT_SECRET is NOT set"; exit 1; }

            IMAGE_NAME="${IMAGE_NAME:-pulsevote-backend}"
            TAG="${CIRCLE_SHA1:-latest}"

            docker run -d --name pulsevote \
              -p 5000:5000 \
              -e NODE_ENV=production \
              -e PORT=5000 \
              -e USE_HTTPS=false \
              -e ALLOW_DB_RESET=true \
              -e MONGO_URI=$MONGO_URI \
              -e JWT_SECRET=$JWT_SECRET \
              "${IMAGE_NAME}:${TAG}"

            echo "Waiting for container health..."
            for i in {1..60}; do
              STATUS="$(docker inspect --format='{{.State.Health.Status}}' pulsevote 2>/dev/null || echo starting)"
              echo "Health: $STATUS"
              if [ "$STATUS" = "healthy" ]; then
                echo "Container is healthy"
                break
              fi
              if [ "$STATUS" = "unhealthy" ]; then
                echo "Container is unhealthy"
                docker logs --tail=200 pulsevote || true
                exit 1
              fi
              sleep 2
            done

      - run:
          name: Reset DB before tests
          command: |
            CONTAINER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' pulsevote)
            curl -sSf -X POST http://$CONTAINER_IP:5000/reset
            
      - run:
          name: Run Postman collection
          command: |
            mkdir -p ~/test-results/newman

            docker create --name newman_runner \
              --network container:pulsevote \
              -v ~/test-results/newman:/results \
              postman/newman:alpine run /collection.json \
                --env-var "HOST=localhost:5000" \
                --env-var "TEST_PASSWORD=$TEST_PASSWORD" \
                --reporters cli,junit \
                --reporter-junit-export /results/newman.xml

            docker cp postman/collection.json newman_runner:/collection.json
            docker start -a newman_runner

      - run:
          name: Reset DB after tests
          when: always
          command: |
            CONTAINER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' pulsevote)
            curl -sSf -X POST http://$CONTAINER_IP:5000/reset
      
	  - run:
          name: Cleanup
          when: always
          command: docker rm -f pulsevote || true
```

Important points:
1. We need to update how we run the container to ensure that it can accessed by our newman tests - newman is a tool to run our postman collection.
2. Notice that we introduced a new env called `ALLOW_DB_RESET` - this will be used in our API to offer an endpoint that will only be accessible when set to true (inside the docker container where we are testing). This endpoint will allow mongodb to be cleared. This is acceptable in a test environment.
3. In the `Reset DB before tests` step, we call the `/reset` endpoint which resets the db. Code for this is included below. We call this before and after our newman tests.
4. In the `Run Postman collection`, we use newman to use the `collection.json` file and run our tests. We run newman in its own container and give it the `HOST` and `TEST_PASSWORD` environment variables that we passed in. Note that you need to set the `TEST_PASSWORD` in CircleCI as we are reading it from there.
5. We clean up when done.

Before you can run the pipeline, you need to add the workflow steps. Add the following into your steps in `config.yml`
```yaml
      - docker_build_and_newman_tests:
          requires:
            - lint_and_test
          filters:
            branches:
              only: main
```

### Step 4. Required code changes

Add a new controller called `controllers/resetController.js`
```js
const mongoose = require("mongoose");

async function resetDatabase(req, res) {
  try {
    if (process.env.ALLOW_DB_RESET !== "true") {
      return res.status(403).json({ error: "Database reset not allowed" });
    }

    await mongoose.connection.dropDatabase();

    return res.status(200).json({ message: "Test database cleared" });
  } catch (err) {
    console.error("Error resetting DB:", err);
    return res.status(500).json({ error: "Failed to reset database" });
  }
}

module.exports = { resetDatabase };
```

Add the endpoint in `app.js`
```js
const { resetDatabase } = require("./controllers/resetController");

if (process.env.ALLOW_DB_RESET === "true") {
  app.post("/reset", resetDatabase);
}
```

### Step 5. Push and test on pipeline
Push a commit to git and let the pipeline run.
You need to debug and errors.
Notes:
1. The pipeline will stop running if any step fails.
2. During execution, Newman results are displayed in the `Run Postman collection` step

## SonarQube / SonarCloud Setup

We are using SonarCloud for static code analysis and test coverage integration.

---

### Step 1. Setup SonarCloud account
1. Go to [SonarCloud.io](https://sonarcloud.io).  
2. Log in with GitHub and import your repository.  
3. Create a project and link your repo → copy the values for:  
   - **Project Key**  - get this from project information
   - **Organization**  - get this from project information
   - **Token** - you need to create this under your profile  

### Step 2. Add CircleCI environment variables
In **CircleCI Project Settings → Environment Variables**, set:

- `SONAR_PROJECT` → your project key  
- `SONAR_ORGANIZATION` → your SonarCloud organization key
- `SONAR_TOKEN` → secure token generated in SonarCloud  

The project key and organization are set by you as your configure your project in SonarCloud.
They are also available under your project information tab if you need to access them after creating.

### Step 3. Review Sonar Analysis Job
Add a new step in `config.yml` under `sonar-analysis`:

```yaml
sonar-analysis:
    docker:
      - image: sonarsource/sonar-scanner-cli:latest
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run:
          name: Run SonarCloud analysis
          command: |
            sonar-scanner \
              -Dsonar.projectKey=$SONAR_PROJECT \
              -Dsonar.organization=$SONAR_ORGANIZATION \
              -Dsonar.sources=. \
              -Dsonar.exclusions=test/**,coverage/**,node_modules/** \
              -Dsonar.tests=test \
              -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
              -Dsonar.host.url=https://sonarcloud.io \
              -Dsonar.token=$SONAR_TOKEN \
              -Dsonar.qualitygate.wait=true

      - run:
          name: Fail if any Sonar issues exist (historic or new)
          command: |
            TOTAL=$(curl -s -u $SONAR_TOKEN: \
              "https://sonarcloud.io/api/issues/search?projectKeys=$SONAR_PROJECT&statuses=OPEN,CONFIRMED,REOPENED" \
              | sed -n 's/.*"total":[ ]*\([0-9]*\).*/\1/p')
            echo "Total open Sonar issues: $TOTAL"
            [ "$TOTAL" -eq 0 ] || (echo "Sonar issues found" && exit 1) # comment this out to ignore historical issues.

```
Important points:
1. In sonar-scanner, we use the environment variables.
2. Using an AI tool of your choice, explore what the parameters do.
3. There is also a step to see if any previous issues exist and this will block your pipeline. You can comment out the last line for now while you attend to the issues.

Before you can run the pipeline, you need to add the workflow steps. Add the following into your steps in `config.yml`
```yaml
- sonar-analysis:
    requires:
    - docker_build_and_newman_tests
    filters:
    branches:
        only: main
```

Also, update the next step in the workflow.
```yaml
- deploy_to_render:
	requires:
	- sonar-analysis
	filters:
	branches:
		only: main 
```

### Step 4. Push and test on pipeline
Push a commit to git and let the pipeline run.
You need to debug and errors.
Notes:
1. The pipeline will stop running if any step fails.
2. During execution, Sonar outcome is  displayed in the `Run SonarCloud analysis` step, but actual report needs to be viewed in SonarCloud.

### Step 5. Coverage integration
Update `jest.config.js` to include the various directories.

```js
module.exports = {
  testEnvironment: "node",
  testMatch: ["**/test/**/*.test.js"],
  verbose: true,

  collectCoverage: true,
  coverageDirectory: "coverage",
  coverageReporters: ["lcov", "text", "text-summary"],

  collectCoverageFrom: [
    "controllers/**/*.js",
    "routes/**/*.js",
    "middleware/**/*.js",
    "app.js"
  ]
};

```
> Add in unit tests if your pipeline fails due to test coverage.
> You need to fix any security or maintainability issues so your pipeline can succeed.

### Step 6: Setup up a cron to run every day at midnight (optional)

Add this to the end of the `config.yml`
```yaml
  daily_keepalive:
    triggers:
      - schedule:
          cron: "0 0 * * *"
          filters:
            branches:
              only: main
    jobs:
      - lint_and_test
```
## config.yaml and collection.json - incase you need it.

`config.yml`
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
          name: Run tests with coverage
          command: |
            mkdir -p ~/test-results/jest
            npx jest --ci --coverage --reporters=default --reporters=jest-junit --outputFile=~/test-results/jest/junit.xml

      - store_test_results:
          path: ~/test-results/jest

      - store_artifacts:
          path: ~/test-results/jest
          destination: jest

      - store_artifacts:
          path: coverage/lcov-report
          destination: coverage-html

      - persist_to_workspace:
          root: .
          paths:
            - coverage

  docker_build_and_newman_tests:
    executor: base_docker
    steps:
      - checkout
      - setup_remote_docker

      - run:
          name: Build Docker image
          command: |
            IMAGE_NAME="${IMAGE_NAME:-pulsevote-backend}"
            TAG="${CIRCLE_SHA1:-latest}"
            docker build -t "${IMAGE_NAME}:${TAG}" .

      - run:
          name: Run container
          command: |
            test -n "$MONGO_URI" || { echo "MONGO_URI is NOT set"; exit 1; }
            test -n "$JWT_SECRET" || { echo "JWT_SECRET is NOT set"; exit 1; }

            IMAGE_NAME="${IMAGE_NAME:-pulsevote-backend}"
            TAG="${CIRCLE_SHA1:-latest}"

            docker run -d --name pulsevote \
              -p 5000:5000 \
              -e NODE_ENV=production \
              -e PORT=5000 \
              -e USE_HTTPS=false \
              -e ALLOW_DB_RESET=true \
              -e MONGO_URI=$MONGO_URI \
              -e JWT_SECRET=$JWT_SECRET \
              "${IMAGE_NAME}:${TAG}"

            echo "Waiting for container health..."
            for i in {1..60}; do
              STATUS="$(docker inspect --format='{{.State.Health.Status}}' pulsevote 2>/dev/null || echo starting)"
              echo "Health: $STATUS"
              if [ "$STATUS" = "healthy" ]; then
                echo "Container is healthy"
                break
              fi
              if [ "$STATUS" = "unhealthy" ]; then
                echo "Container is unhealthy"
                docker logs --tail=200 pulsevote || true
                exit 1
              fi
              sleep 2
            done

      - run:
          name: Reset DB before tests
          command: |
            CONTAINER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' pulsevote)
            curl -sSf -X POST http://$CONTAINER_IP:5000/reset
            
      - run:
          name: Run Postman collection
          command: |
            mkdir -p ~/test-results/newman

            docker create --name newman_runner \
              --network container:pulsevote \
              -v ~/test-results/newman:/results \
              postman/newman:alpine run /collection.json \
                --env-var "HOST=localhost:5000" \
                --env-var "TEST_PASSWORD=$TEST_PASSWORD" \
                --reporters cli,junit \
                --reporter-junit-export /results/newman.xml

            docker cp postman/collection.json newman_runner:/collection.json
            docker start -a newman_runner

      - run:
          name: Reset DB after tests
          when: always
          command: |
            CONTAINER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' pulsevote)
            curl -sSf -X POST http://$CONTAINER_IP:5000/reset
          
      - run:
          name: Cleanup
          when: always
          command: docker rm -f pulsevote || true

  sonar-analysis:
    docker:
      - image: sonarsource/sonar-scanner-cli:latest
    steps:
      - checkout
      - attach_workspace:
          at: .
      - run:
          name: Run SonarCloud analysis
          command: |
            sonar-scanner \
              -Dsonar.projectKey=$SONAR_PROJECT \
              -Dsonar.organization=$SONAR_ORGANIZATION \
              -Dsonar.sources=. \
              -Dsonar.exclusions=test/**,coverage/**,node_modules/** \
              -Dsonar.tests=test \
              -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
              -Dsonar.host.url=https://sonarcloud.io \
              -Dsonar.token=$SONAR_TOKEN \
              -Dsonar.qualitygate.wait=true

      - run:
          name: Fail if any Sonar issues exist (historic or new)
          command: |
            TOTAL=$(curl -s -u $SONAR_TOKEN: \
              "https://sonarcloud.io/api/issues/search?projectKeys=$SONAR_PROJECT&statuses=OPEN,CONFIRMED,REOPENED" \
              | sed -n 's/.*"total":[ ]*\([0-9]*\).*/\1/p')
            echo "Total open Sonar issues: $TOTAL"
            [ "$TOTAL" -eq 0 ] || (echo "Sonar issues found" && exit 1) # comment this out to ignore historical issues.

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
      - lint_and_test:
          filters:
            branches:
              only: main
      
      - docker_build_and_newman_tests:
          requires:
            - lint_and_test
          filters:
            branches:
              only: main
      - sonar-analysis:
          requires:
            - docker_build_and_newman_tests
          filters:
            branches:
              only: main
      - deploy_to_render:
          requires:
            - sonar-analysis
          filters:
            branches:
              only: main

```

`collection.json`

```json
{
	"info": {
		"_postman_id": "a1089c93-9521-4b67-a505-d13ef9a7db99",
		"name": "PulseVote RBAC Test",
		"description": "Add the values for the usernames and so on. If your API is configured correctly, and runs at the relevant ports, all tests should run in postman",
		"schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json",
		"_exporter_id": "10616096"
	},
	"item": [
		{
			"name": "0. Register First Admin",
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{ADMIN_EMAIL}}\",\n  \"password\": \"{{ADMIN_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/register-admin",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"register-admin"
					]
				}
			},
			"response": []
		},
		{
			"name": "1. Login Admin",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('ADMIN_TOKEN', res.token);",
							"// Optional: pm.collectionVariables.set('ADMIN_ID', res.userId || res.id);"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{ADMIN_EMAIL}}\",\n  \"password\": \"{{ADMIN_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/login",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"login"
					]
				}
			},
			"response": []
		},
		{
			"name": "2. Register Manager",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{ADMIN_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{MANAGER_EMAIL}}\",\n  \"password\": \"{{MANAGER_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/register-manager",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"register-manager"
					]
				}
			},
			"response": []
		},
		{
			"name": "3. Login Manager",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('MANAGER_TOKEN', res.token);",
							"// Optional: pm.collectionVariables.set('MANAGER_ID', res.userId || res.id);"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{MANAGER_EMAIL}}\",\n  \"password\": \"{{MANAGER_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/login",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"login"
					]
				}
			},
			"response": []
		},
		{
			"name": "4. Create Organisation",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"if (res.organisation && res.organisation._id) {",
							"    pm.collectionVariables.set('ORG_ID', res.organisation._id);",
							"}",
							"if (res.organisation && res.organisation.joinCode) {",
							"    pm.collectionVariables.set('JOIN_CODE', res.organisation.joinCode);",
							"}",
							"console.log('Organisation ID:', pm.collectionVariables.get('ORG_ID'));",
							"console.log('Join Code:', pm.collectionVariables.get('JOIN_CODE'));"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"name\": \"{{ORG_NAME}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/organisations/create-organisation",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"organisations",
						"create-organisation"
					]
				}
			},
			"response": []
		},
		{
			"name": "5. Register User",
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{USER_EMAIL}}\",\n  \"password\": \"{{USER_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/register-user",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"register-user"
					]
				}
			},
			"response": []
		},
		{
			"name": "6. Login User",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('USER_TOKEN', res.token);",
							"// Optional: pm.collectionVariables.set('USER_ID', res.userId || res.id);"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"email\": \"{{USER_EMAIL}}\",\n  \"password\": \"{{USER_PASSWORD}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/auth/login",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"auth",
						"login"
					]
				}
			},
			"response": []
		},
		{
			"name": "7. Join Organisation",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{USER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"joinCode\": \"{{JOIN_CODE}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/organisations/join-organisation",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"organisations",
						"join-organisation"
					]
				}
			},
			"response": []
		},
		{
			"name": "8. Generate Join Code",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('JOIN_CODE', res.joinCode || res.code || (res.data && res.data.joinCode));"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": ""
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/organisations/generate-join-code/{{ORG_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"organisations",
						"generate-join-code",
						"{{ORG_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "9. Create Poll",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"pm.collectionVariables.set('POLL_ID', res._id || res.id);"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"organisationId\": \"{{ORG_ID}}\",\n  \"question\": \"{{POLL_QUESTION}}\",\n  \"options\": {{POLL_OPTIONS}}\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/create-poll",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"create-poll"
					]
				}
			},
			"response": []
		},
		{
			"name": "10. Get Organisation Polls",
			"event": [
				{
					"listen": "test",
					"script": {
						"exec": [
							"const res = pm.response.json();",
							"if (Array.isArray(res) && res.length) { pm.collectionVariables.set('POLL_ID', res[0]._id || res[0].id); }",
							"if (res && res.polls && res.polls.length) { pm.collectionVariables.set('POLL_ID', res.polls[0]._id || res.polls[0].id); }"
						],
						"type": "text/javascript",
						"packages": {}
					}
				}
			],
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "GET",
				"header": [],
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/get-polls/{{ORG_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"get-polls",
						"{{ORG_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "11. Vote in Poll",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{USER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"optionIndex\": {{OPTION_INDEX}}\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/vote/{{POLL_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"vote",
						"{{POLL_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "12. Get Poll Results",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "GET",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/get-poll-results/{{POLL_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"get-poll-results",
						"{{POLL_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "13. Close Poll",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"organisationId\": \"{{ORG_ID}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/close/{{POLL_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"close",
						"{{POLL_ID}}"
					]
				}
			},
			"response": []
		},
		{
			"name": "14. Re-open Poll",
			"request": {
				"auth": {
					"type": "bearer",
					"bearer": [
						{
							"key": "token",
							"value": "{{MANAGER_TOKEN}}",
							"type": "string"
						}
					]
				},
				"method": "POST",
				"header": [
					{
						"key": "Content-Type",
						"value": "application/json"
					}
				],
				"body": {
					"mode": "raw",
					"raw": "{\n  \"organisationId\": \"{{ORG_ID}}\"\n}"
				},
				"url": {
					"raw": "{{PROTOCOL}}://{{HOST}}/api/polls/open/{{POLL_ID}}",
					"protocol": "{{PROTOCOL}}",
					"host": [
						"{{HOST}}"
					],
					"path": [
						"api",
						"polls",
						"open",
						"{{POLL_ID}}"
					]
				}
			},
			"response": []
		}
	],
	"event": [
		{
			"listen": "prerequest",
			"script": {
				"type": "text/javascript",
				"packages": {},
				"exec": [
					""
				]
			}
		},
		{
			"listen": "test",
			"script": {
				"type": "text/javascript",
				"packages": {},
				"exec": [
					""
				]
			}
		}
	],
	"variable": [
		{
			"key": "PROTOCOL",
			"value": "http"
		},
		{
			"key": "HOST",
			"value": "localhost"
		},
		{
			"key": "PORT",
			"value": "5000",
			"disabled": true
		},
		{
			"key": "ADMIN_EMAIL",
			"value": "admin@pulsevote.com"
		},
		{
			"key": "ADMIN_PASSWORD",
			"value": "{{TEST_PASSWORD}}"
		},
		{
			"key": "MANAGER_EMAIL",
			"value": "manager@pulsevote.com"
		},
		{
			"key": "MANAGER_PASSWORD",
			"value": "{{TEST_PASSWORD}}"
		},
		{
			"key": "USER_EMAIL",
			"value": "user@pulsevote.com"
		},
		{
			"key": "USER_PASSWORD",
			"value": "{{TEST_PASSWORD}}"
		},
		{
			"key": "ORG_NAME",
			"value": "INSY7314"
		},
		{
			"key": "OPTION_INDEX",
			"value": "1"
		},
		{
			"key": "POLL_QUESTION",
			"value": "What's your favourite programming language?"
		},
		{
			"key": "POLL_OPTIONS",
			"value": "[\"JavaScript\", \"Python\", \"C#\"]"
		},
		{
			"key": "ADMIN_TOKEN",
			"value": ""
		},
		{
			"key": "MANAGER_TOKEN",
			"value": ""
		},
		{
			"key": "USER_TOKEN",
			"value": ""
		},
		{
			"key": "ORG_ID",
			"value": ""
		},
		{
			"key": "JOIN_CODE",
			"value": ""
		},
		{
			"key": "POLL_ID",
			"value": ""
		}
	]
}
```