# Week 3 Milestone — Docker / Parse Server / Next.js / ECR / EC2

## Goal

Take a real CRUD application — a Parse Server backend and a Next.js frontend — and
containerize it properly: multi-stage Dockerfiles, a Compose stack with real health
checks, images pushed to a private registry (Amazon ECR), and a working deployment
to a public EC2 instance.

This document explains what was built, the exact steps taken, the problems hit along
the way (and why each fix worked), and how to verify everything afterward.

---

## Milestone spec checklist

| Requirement                                                 | Status                                                                                                                                     |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Multi-stage Dockerfiles for both services                   | ✅ Done                                                                                                                                    |
| Non-root user in both final images                          | ✅ Confirmed — both multi-stage Dockerfiles end in `USER node`                                                                             |
| Minimal final image size                                    | ✅ Measured for both: frontend 1.01GB → 201.32MB, backend 375.64MB → 288.42MB                                                              |
| `docker-compose.yml`, one command, healthchecks             | ✅ Done                                                                                                                                    |
| ECR repo, both images pushed with **semantic version tags** | ⚠️ Everything pushed so far uses `:latest`. The spec asks for semantic versions (e.g. `v1.0.0`) — retag and push before calling this done. |
| Pull onto Week 1 EC2 instance, run via Compose              | ✅ Done                                                                                                                                    |
| README has before/after image-size comparison               | ✅ Done — see table below                                                                                                                  |

---

## What was built

| Component                                                  | Purpose                                                |
| ---------------------------------------------------------- | ------------------------------------------------------ |
| Parse Server (Node/Express, TypeScript compiled via `tsc`) | Backend API for the CRUD app                           |
| Next.js frontend (`output: 'standalone'`)                  | Client app, served as a minimal production server      |
| MongoDB                                                    | Database for Parse Server                              |
| Parse Dashboard                                            | Web UI for inspecting/editing Parse data               |
| Multi-stage Dockerfiles (backend + frontend)               | Small, reproducible production images                  |
| `docker-compose.yml` with real healthchecks                | Local + remote orchestration, correct startup ordering |
| Amazon ECR repositories                                    | Private registry for both images                       |
| EC2 (Ubuntu) deployment                                    | Public-facing host running the full stack via Compose  |

---

## Step-by-step summary

### 1. Building the Dockerfiles — basic vs. multi-stage, for both services

Two Dockerfiles were built and measured for **each** service — a naive single-stage
version and a hardened multi-stage version — to make the size/security difference
concrete rather than just asserted.

**Frontend (Next.js)**

- **Basic** (`node:20-alpine`, single stage): `COPY . .` → `npm install` →
  `npm run dev`. Simple, but ships the entire `node_modules` tree — dev
  dependencies included — and runs as root.
- **Multi-stage** (`node:24-alpine3.24`, 3 stages — `dependencies` → `builder` →
  `runner`): the `dependencies` stage installs with cache mounts
  (`--mount=type=cache`) and detects npm/yarn/pnpm automatically from whichever
  lockfile is present; the `builder` stage runs the actual `next build`; the
  `runner` stage copies out only `.next/standalone`, `.next/static`, and
  `public/` (Next's `output: 'standalone'` mode), `chown`s everything to a
  non-root `node` user, and switches to `USER node` before `CMD`.

**Backend (Parse Server)**

- **Basic** (`node:jod-alpine3.20`, single stage): `COPY . .` → `npm install` →
  `npm run build` → `npm start`, no user switch, straightforward but carries full
  dev tooling and build output in one layer.
- **Multi-stage** (`node:jod-alpine3.20`, 2 stages — `build` → `release`): the
  `build` stage does the full install and `npm run build`; the `release` stage
  copies only `dist/` and `package.json` out with `--chown=node:node`, reinstalls
  with `npm install --omit=dev`, and also switches to `USER node`.

### Before/after image size

Measured directly via `docker images` / Docker Desktop:

| Image                | Basic (single-stage) | Multi-stage | Reduction    |
| -------------------- | -------------------- | ----------- | ------------ |
| Next.js frontend     | 1.01 GB              | 201.32 MB   | ~80% smaller |
| Parse Server backend | 375.64 MB            | 288.42 MB   | ~23% smaller |

The frontend's reduction is dramatic because the basic version ships the entire
`node_modules` install (including dev dependencies and the Next.js compiler itself)
into the runtime image, while the multi-stage version keeps only the standalone
server output. The backend's reduction is smaller in comparison — Parse Server has
a lighter dev-dependency footprint to begin with (`node:jod-alpine3.20` is already a
fairly minimal base for both variants), so most of what's being trimmed here is
leftover build artifacts and the dev-only portion of `npm install`, not an entire
duplicate dependency tree.

Both multi-stage Dockerfiles end in `USER node` — non-root is satisfied on **both**
final images, not just the frontend.

### 2. Bugs hit while building the Parse Server image

- **`tsc: command not found`** — `tsc-alias` was in `devDependencies`, but
  `typescript` itself wasn't. `tsc-alias` rewrites path aliases _after_
  compilation; it doesn't include the compiler.
  > **Fix:** install `typescript` explicitly before anything else.
- **`npx tsc --init` installed the wrong package** — the first attempt pulled in a
  fake, deprecated `tsc` npm package instead of the real TypeScript compiler.
  > **Fix:** install `typescript` first, _then_ run `npx tsc --init` — npx resolves
  > to the locally installed compiler once it exists.
- **Missing `tsconfig.json` entirely** — `tsc` silently printed its help text
  instead of compiling anything, because there was no config telling it what to do.
  This is a genuinely dangerous failure mode: it looks like the build "did
  something" when it actually did nothing.
- **`tsconfig.json`'s default `include: ["**/\*"]`found zero files** — because the
project only had a root-level`index.js`, no `src/`folder and no`.ts` files at
  all. This surfaced a real project-level finding: the template was scaffolded for
  TypeScript but never actually migrated to it.
  > **Open decision, not yet resolved:** convert the project to real TypeScript, or
  > strip the TS tooling entirely and go plain JS. Worth having an opinion on this
  > before a mentor or interviewer asks.
- **`COPY --from=build /tmp/dist ./` flattened the output** — the trailing `./`
  destination dumped `dist/`'s _contents_ straight into the app root, but
  `package.json`'s `"start": "node dist/index.js"` expected a nested `dist/`
  folder to still exist.
  > **Fix:** change the destination to `./dist` so the folder itself is preserved,
  > not just its contents.
- **`RUN npm install --omit=dev` accidentally left commented out** in an earlier
  release-stage version — the image would have shipped with zero `node_modules`
  and crashed immediately on `npm start`. Caught by inspection before it was ever
  actually deployed.

### 3. `docker-compose.yml` — design decisions

This is the actual final version, updated after the images were pushed to ECR
(secrets replaced with placeholders — see the security note below):

```yaml
services:
  frontend:
    image: <account-id>.dkr.ecr.eu-north-1.amazonaws.com/crud-nextjs-frontend
    build:
      context: ./CRUD-nextjs-frontend
    depends_on:
      backend:
        condition: service_healthy
    ports:
      - "3003:3000"
    healthcheck:
      test:
        [
          "CMD",
          "wget",
          "--no-verbose",
          "--tries=1",
          "--spider",
          "http://127.0.0.1:3000",
        ]

  backend:
    image: <account-id>.dkr.ecr.eu-north-1.amazonaws.com/crud-parse-server-backend
    build:
      context: ./CRUD-parse-server-backend
    depends_on:
      database:
        condition: service_healthy
    ports:
      - "1338:1337"
    healthcheck:
      test:
        [
          "CMD",
          "wget",
          "--no-verbose",
          "--tries=1",
          "--spider",
          "http://localhost:1337/parse/health",
        ]

  backend-dashboard:
    image: parseplatform/parse-dashboard:latest
    depends_on:
      backend:
        condition: service_healthy
    ports:
      - "4041:4040"

  database:
    image: mongo:latest
    ports:
      - "27018:27017"
    healthcheck:
      test: mongosh --eval "db.adminCommand('ping')"
```

- **`depends_on: condition: service_healthy` everywhere**, not plain `depends_on` —
  plain `depends_on` only waits for the container to _start_, not for the service
  inside it to actually be ready to accept connections. Real readiness needs a
  healthcheck.
- **Healthchecks run _inside_ the container**, so they must reference the
  container's internal port, never the host-mapped one — `database` uses `mongosh`
  directly against its own port; `backend` and `frontend` use `wget --spider`
  against `localhost`/`127.0.0.1` on their own internal ports (`1337`, `3000`),
  never the host-side `1338`/`3003`.
- **Both `image:` and `build:` are present together** on `frontend` and `backend` —
  this is deliberate: `docker compose build` builds and tags the image locally
  under the ECR URI given in `image:`, and that same tag is what gets pushed. On a
  machine that already has the image (or pulls it from ECR), `build:` is simply
  ignored — so this one file works both for local iteration and for a pure
  "pull from ECR and run" deployment on EC2.
- **Bind mount for `cloud/` only** on the backend now (the earlier `logs/` bind
  mount was dropped in this version) — Cloud Code edits still don't need a
  rebuild, logs are no longer persisted outside the container.
- **`MASTER_KEY_IPS: 0.0.0.0/0,::0`** allows the Parse master key to be used from
  _any_ IP address. Fine for a personal training project reachable only by you, but
  worth being able to explain unprompted: in a real production setup this should be
  scoped down to specific trusted IPs (office network, VPN range, CI runner), since
  the master key bypasses all ACLs and class-level permissions.
- **Parse Dashboard needs `PARSE_DASHBOARD_ALLOW_INSECURE_HTTP: "1"`** since there's
  no TLS in this local/dev setup.
  > Flagged for later: this needs removing the moment the stack sits behind HTTPS.

> **Security note:** the version of this file actually in use has a real MongoDB
> password hardcoded as the `:-fallback` default for `MONGO_ROOT_PASSWORD` (used in
> both the `database` service and the `backend`'s `DATABASE_URI`). A `:-fallback` in
> Compose syntax is _not_ a placeholder that disappears when unset — it's a real
> value that gets used any time the corresponding `.env` variable is missing. Before
> this file goes into a public repo: rotate that Mongo password, and either put the
> new one only in a git-ignored `.env` file, or drop the fallback so Compose refuses
> to start without an explicit value (`${MONGO_ROOT_PASSWORD:?must be set}`).

> **Lesson learned — the mismatched-port bug, now resolved:** an earlier version of
> this file mapped `"1338:1337"` while also setting the container's internal `PORT`
> to `1338` — which doesn't work, because a host-side port number and the
> container's own listening port are two different things. The fixed version above
> keeps them straight: `PORT: 1337` inside the container, `1338` only as what the
> host sees, and `SERVER_URL`/`NEXT_PUBLIC_PARSE_SERVER_URL` both point at
> `localhost:1338` since that's the port anything _outside_ Docker (a browser, an
> external API client) actually needs to hit.

> **Lesson learned — Compose DNS resolves by service name:** an early version of
> `DATABASE_URI` referenced `mongo` as the Mongo hostname, but the actual Compose
> service was named `database` — Compose's internal DNS resolves by the YAML
> service key, never by the `image` name or `container_name`. The final file above
> consistently uses `database` as both the service key and the hostname in
> `DATABASE_URI`.

### 4. Pushing to Amazon ECR

Only the **multi-stage** images are what actually get pushed and deployed — the
basic versions exist solely for the size comparison above, not as deployment
candidates:

```bash
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com

docker tag crud-parse-server-backend-multistage-final:latest <account-id>.dkr.ecr.<region>.amazonaws.com/crud-parse-server-backend:v1.0.0
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/crud-parse-server-backend:v1.0.0

docker tag crud-nextjs-frontend-multistage:latest <account-id>.dkr.ecr.<region>.amazonaws.com/crud-nextjs-frontend:v1.0.0
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/crud-nextjs-frontend:v1.0.0
```

A dedicated IAM user (`ecr-deploy-user`) with `AmazonEC2ContainerRegistryFullAccess`
handles auth for this. One-time repository creation, before the first push:

```bash
aws ecr create-repository --repository-name crud-parse-server-backend --region <region>
aws ecr create-repository --repository-name crud-nextjs-frontend --region <region>
```

> **Note:** the milestone spec calls for semantic version tags, not `:latest`.
> `v1.0.0` above is a placeholder; bump it to match whatever versioning scheme
> you're actually using (or start one now if there isn't one yet).
>
> **This also means `docker-compose.yml` needs updating to match:** right now its
> `image:` lines for `frontend` and `backend` have no tag at all
> (`.../crud-nextjs-frontend` with nothing after it), which Docker silently reads as
> `:latest`. Pushing `v1.0.0` to ECR does nothing for what actually gets pulled and
> run unless the compose file's `image:` lines are updated to
> `.../crud-nextjs-frontend:v1.0.0` (and the same for the backend) to match.

### 5. Deploying to EC2

```bash
sudo apt install -y unzip   # aws-cli install needs this first
curl -fsSL https://awscli.amazonaws.com/v2/install.sh | bash
export PATH=$HOME/.local/bin:$PATH   # aws not found otherwise
sudo usermod -aG docker $USER && newgrp docker
docker compose up -d
```

> **Lesson learned — `aws-cli` install failed on missing `unzip`:** the installer
> assumes it's already present; it usually isn't on a fresh Ubuntu EC2 image.
> `sudo apt install -y unzip` first, then re-run the installer.

> **Lesson learned — `aws` not found after installing:** a PATH issue, not a failed
> install — fixed by adding `$HOME/.local/bin` to `PATH` and persisting it in
> `~/.bashrc`.

> **Lesson learned — `NoCredentials` error:** solved either with `aws configure`,
> or — the better option — an IAM role attached directly to the EC2 instance, which
> in this case was already attached (`assumed-role/ec2-repository-role/...`),
> meaning no access keys ever needed to touch the server at all.

> **Lesson learned — `sudo docker` and plain `docker` don't share a login:**
> `sudo docker compose up` and plain `docker compose up` read Docker config from two
> _different_ locations (`/root/.docker/config.json` vs
> `/home/ubuntu/.docker/config.json`). Logging into ECR without `sudo` left that
> login invisible to any command run _with_ `sudo`. Fixed properly by adding the
> user to the `docker` group (`sudo usermod -aG docker $USER` + `newgrp docker`),
> which removes the need for `sudo` with Docker entirely — rather than just
> remembering to log in twice.

> **Lesson learned — site unreachable after enabling UFW:** UFW defaults to
> deny-all-incoming; only `http` (port 80) was allowed by default, but the app used
> custom ports (3000, 1337, 4040). Fixed with `sudo ufw allow <port>/tcp` for each
> port actually in use.
> **Second layer, easy to forget:** the AWS Security Group is a completely separate,
> independent firewall — it must also explicitly allow the same ports, or UFW being
> correct won't matter.

> **Still open / not yet confirmed:** Parse Dashboard showed `(unhealthy)` in
> `docker ps` despite logs showing it running fine — suspected to be the same
> host-port-vs-container-port healthcheck mismatch pattern seen elsewhere, but not
> yet verified.

### 6. Verification

```bash
docker ps                                  # all services show (healthy)
curl http://localhost:1337/parse/health    # from inside the EC2 instance
```

Visit `http://<ec2-public-ip>:3000` in a browser to confirm the frontend is served
and talking to the backend.

---

## Deliverables produced

Per the milestone spec — repo with Dockerfiles + `docker-compose.yml`, a before/after
image-size comparison in this README, and the full stack running on EC2 pulled
straight from ECR:

- [x] Four Dockerfiles built and measured (basic + multi-stage, for both frontend
      and backend), demonstrating the size/security trade-offs of each
- [x] Non-root user confirmed in **both** final images (`USER node` in both
      multi-stage Dockerfiles)
- [x] `docker-compose.yml` with real healthchecks and correct startup ordering
- [ ] Both images pushed to ECR with **semantic version tags** (currently `:latest`
      — see the note in the ECR section above)
- [x] Before/after image-size comparison table filled in for both images
- [x] Full working deployment on a public EC2 instance, pulled from ECR

---

## Key concepts to be ready to explain

- Why multi-stage builds shrink image size so dramatically, and specifically what
  `output: 'standalone'` does for a Next.js production image
- The difference between a build-time failure and a _silent_ one (the missing
  `tsconfig.json` case) — and why silent failures are the more dangerous category
- Why `depends_on: condition: service_healthy` matters over plain `depends_on`
- How Docker Compose's internal DNS resolves hostnames (by service key, not image
  name or container name)
- Why a healthcheck must reference a container's _internal_ port, never its
  host-mapped one
- The two independent firewall layers on EC2 (UFW vs. AWS Security Group) and why
  both must agree
- Why an IAM role attached to an EC2 instance is preferable to `aws configure` with
  long-lived access keys
- Why `sudo docker` and non-`sudo docker` can behave as if logged into two different
  Docker daemons, and the actual fix (docker group membership) versus the workaround
  (logging in twice)
