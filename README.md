# GitLab CI/CD → GitOps Manifests Repo (Argo CD) Demo

This repository contains a simple **Node.js HTTP server** and a **GitLab CI/CD pipeline** that:

1. **Builds** a Docker image from this repo and pushes it to the GitLab Container Registry.
2. **Deploys** via GitOps by cloning a **separate manifests repository** (e.g. `root/manifest01`) and updating the image tag inside `manifests/deployment.yaml`.
3. Argo CD (watching the manifests repo) detects the change and syncs it to Kubernetes.

---

## Repository contents

### 1) Application code

**`app.js`**
```js
const http = require('http');

const hostname = '0.0.0.0';
const port = 3000;

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello from GitLab CI/CD and ArgoCD!\n');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

**`package.json`**
```json
{
  "name": "gitlab-argocd-example",
  "version": "1.0.0",
  "description": "A simple Node.js HTTP server for GitLab CI/CD and ArgoCD demo",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "author": "Your Name",
  "license": "MIT",
  "dependencies": {}
}
```

> Note: Your `package.json` says `"start": "node server.js"` but your actual entry file is `app.js`.
> Consider changing it to:
> ```json
> "start": "node app.js"
> ```

### 2) Container image

**`Dockerfile`**
```dockerfile
FROM node:14

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000
CMD [ "node", "app.js" ]
```

---

## GitLab CI/CD pipeline

**`.gitlab-ci.yml`**
```yaml
stages:
  - build
  - deploy

variables:
  IMAGE: "$CI_REGISTRY_IMAGE/api:$CI_COMMIT_SHORT_SHA"

.default_docker:
  image: docker:24.0.7
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY

build-stage:
  stage: build
  tags:
    - k8s
  image: docker:27
  services:
    - name: docker:27-dind
      command: ["--tls=false"]
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_TLS_CERTDIR: ""
  extends: .default_docker
  script:
    - docker build -t "$IMAGE" .
    - docker push "$IMAGE"
  only:
    - main

deploy:
  stage: deploy
  image: alpine:3.20
  before_script:
    - apk add --no-cache git bash curl sed
  script:
    - |
      export IMAGE="$CI_REGISTRY_IMAGE/api:$CI_COMMIT_SHORT_SHA"

      # Set Git identity
      git config --global user.email "ci@mycompany.com"
      git config --global user.name "CI Bot"

      # Clone the GitOps manifests repo
      git clone "https://root:${CI_PASSWORD}@gitlab.sananetco.com/root/manifest01.git"
      cd manifest01
      git checkout main
      git pull origin main

      # Replace image line
      echo "Replacing image line with: image: $IMAGE"
      sed -i "s|image:.*|image: \"$IMAGE\"|" manifests/deployment.yaml

      # Commit & push changes
      git add manifests/deployment.yaml
      git commit -m "Deploy $CI_COMMIT_SHORT_SHA [skip ci]" || echo "No changes to commit"
      git push origin main
  only:
    - main
  tags:
    - k8s
```

### What the pipeline does

- **Build stage**
  - Builds a Docker image using the commit SHA as the tag.
  - Pushes the image to GitLab Container Registry.
- **Deploy stage**
  - Clones the *manifests repo* (`root/manifest01`)
  - Updates `manifests/deployment.yaml` to use the new image tag
  - Commits and pushes that change back to the manifests repo
  - Argo CD sees the manifests change and syncs to Kubernetes

---

## Required CI/CD variables

In the **source code project** (this repo), define:

- `CI_PASSWORD` (**required**) – credential that has **write access** to the manifests repository.
- GitLab Registry variables are already provided by GitLab in CI:
  - `CI_REGISTRY`, `CI_REGISTRY_USER`, `CI_REGISTRY_PASSWORD`, `CI_REGISTRY_IMAGE`

### CI_PASSWORD: what it is and how to obtain it

Your deploy job uses HTTP Basic authentication in the clone URL:

```bash
git clone "https://root:${CI_PASSWORD}@gitlab.sananetco.com/root/manifest01.git"
```

So `CI_PASSWORD` must be a **real credential** that GitLab accepts for that user (here, `root`).
The best options:

#### Option A (Recommended): Personal Access Token (PAT)
1. In GitLab, go to **User Settings → Access Tokens**
2. Create a token with scopes:
   - `write_repository` (and optionally `read_repository`)
3. Make sure this user has **Developer/Maintainer** access to `root/manifest01`
4. In the **src project** go to **Settings → CI/CD → Variables**
   - Key: `CI_PASSWORD`
   - Value: `<YOUR_PAT>`
   - Mark it **Masked** and **Protected** (recommended if `main` is protected)

Then keep the clone URL as-is (or replace `root` with your bot user).

#### Option B: Project Access Token (Manifests Repo)
1. In the **manifests project** (`root/manifest01`), go to **Settings → Access Tokens**
2. Create a token with scope:
   - `write_repository`
3. Store the token value in the src project as `CI_PASSWORD`
4. Update the username part of the URL to the token’s username (GitLab shows it when you create the token)

#### Option C: Use CI_JOB_TOKEN (advanced, no extra secret)
Some GitLab versions allow cross-project Git access using `CI_JOB_TOKEN`, **only if you explicitly allow it** in the manifests project.

Clone format:
```bash
git clone "https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.sananetco.com/root/manifest01.git"
```

You must configure the manifests project to allow inbound job token access from the src project (location depends on your GitLab version/settings).

> Security tip: avoid using `root` tokens in CI. Create a dedicated bot user (e.g., `ci-bot`) with minimal required permissions.

---

## Manifests repository layout (expected)

Your manifests repo should contain something like:

```
manifest01/
  manifests/
    deployment.yaml
```

And `deployment.yaml` should contain an `image:` line that can be replaced, for example:

```yaml
containers:
  - name: api
    image: "registry.example.com/group/project/api:oldtag"
```

---

## Local run (optional)

Run locally without Docker:

```bash
npm install
node app.js
# open http://localhost:3000
```

Build and run with Docker:

```bash
docker build -t demo:local .
docker run --rm -p 3000:3000 demo:local
```

---

## Troubleshooting

- **`fatal: Authentication failed`**
  - `CI_PASSWORD` is missing/incorrect, or the user/token does not have write access to the manifests repo.
- **`No changes to commit`**
  - The image line already matches the new tag (this is fine).
- **`sed` replaced the wrong image line**
  - Make the replacement more specific (target the correct container or file path).
