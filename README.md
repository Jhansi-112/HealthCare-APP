# HealthCare-APP: CI Pipeline for Java Microservices

![Path CI](https://github.com/Jhansi-112/HealthCare-APP/actions/workflows/healthcare-path-ci.yml/badge.svg)

A GitHub Actions CI pipeline for a monorepo with **5 Spring Boot microservices**.
Every push is built, tested, scanned for code quality, packaged as a Docker image
and pushed to Docker Hub. A second pipeline builds **only the service that
changed**. A third workflow runs on **my own AWS EC2 server** (self-hosted runner).

> This is a hands-on DevOps project. I forked the classroom repo shared by my
> trainer, then built, fixed and extended the pipeline step by step.

---

## 1. What this project does

| Need | How the pipeline solves it |
|---|---|
| Manual builds are slow and easy to forget | Every push to `master` starts the pipeline automatically |
| Bugs are found late | Unit tests and a SonarCloud scan run before any image is pushed |
| 5 services build one by one | A **matrix** builds all 5 in parallel |
| One failing service hides the others | `fail-fast: false` so every service reports its own result |
| A one-service change rebuilds everything | **Path-based pipeline** builds only the changed service |
| A hung job wastes build minutes | `timeout-minutes` on every job and every step |
| Images cannot be traced to code | Each image is tagged with the **commit SHA** |
| Credentials in code | Docker and Sonar credentials are **GitHub secrets** |

## 2. Services

All services are in `healthcare-microservices/`. Each has its own `pom.xml`, tests
and multi-stage `Dockerfile`.

| Service | Notes |
|---|---|
| `appointment-service` | Appointment booking, history, cancel |
| `doctor-service` | Spring Boot service |
| `patient-service` | Spring Boot service |
| `report-service` | Medical report upload and download (AWS S3) |
| `user-service` | User registration and login, JWT, roles |

Stack: Java 17, Spring Boot 3.2.8, Maven, Docker.

## 3. Pipeline overview

```
 push to master
      |
      v
 detect-changes  (path workflow only: which service folders changed?)
      |
      v
 unit-test  ---->  sonar  ---->  docker-push
 (mvn test)       (SonarCloud)   (Docker Hub: <commit-sha> and latest)
   x N services in parallel (matrix)
```

### Workflows in `.github/workflows/`

| File | Starts when | What it does |
|---|---|---|
| `healthcare-path-ci.yml` | Push or pull request that changes a service folder | Builds, tests, scans and pushes **only the changed services** |
| `healthcare-ci.yml` | **Manual only** (`workflow_dispatch`) | Full pipeline for all 5 services |
| `healthcare-selfhosted-ci.yml` | **Manual only** | Tests and builds `doctor-service` on my **EC2 self-hosted runner** |
| `java-services-ci.yml`, `java-services-path-ci.yml` | Disabled | The trainer's original files, kept only for reference |

### Pipeline features

- **Matrix strategy:** one job definition runs for 5 services in parallel.
- **Path-based trigger:** a `detect-changes` job uses `dorny/paths-filter` and
  gives the changed service list to the matrix.
- **Timeout handling:** `timeout-minutes` on every job and every step.
- **Error handling:** `fail-fast: false`, and Sonar skips itself if the token is missing.
- **Quality scan:** SonarCloud, one project per service (`jhansi-112_<service>`).
  Sonar runs on JDK 21 because the scanner needs it. Build and tests use JDK 17.
- **Docker:** images are pushed to Docker Hub as
  `<docker-user>/healthcare-<service>:<commit-sha>` and `:latest`.
  Images are **not pushed on pull requests**.
- **Caching:** Maven dependencies are cached with `setup-java`.

## 4. Setup

### Accounts and tools

GitHub, Docker Hub, SonarCloud and Jira. Locally: Ubuntu on WSL2, Git, OpenJDK 17,
Maven, Docker, VS Code.

### Secrets and variable (Settings, Secrets and variables, Actions)

| Name | Type | Value |
|---|---|---|
| `DOCKER_USERNAME` | Secret | Docker Hub user name |
| `DOCKER_PASSWORD` | Secret | Docker Hub **access token** (not the account password) |
| `SONAR_TOKEN` | Secret | SonarCloud token |
| `SONAR_ORGANIZATION` | Variable | SonarCloud organization key (lowercase) |

Paste secret values as **one clean line**, with no line break at the end.

### Build one service by hand

```bash
cd healthcare-microservices/doctor-service
mvn -B clean install
java -jar target/doctor-service-0.0.1-SNAPSHOT.jar
docker build -t doctor-service:test .
```

## 5. Self-hosted runner on AWS EC2

Used to run jobs on my own server instead of a GitHub-hosted machine.

- Server: a small free-tier-size EC2 instance, Ubuntu 24.04 LTS, region ap-south-1,
  20 GiB disk.
- Network: SSH only from my IP. No other inbound port. The runner makes outbound
  calls only.
- Tools: Git, OpenJDK 17, Maven, Docker.
- Runner: installed as a service. Name `ec2-java-build`, label `java-build`.
- Workflow: `runs-on: [self-hosted, linux, java-build]`.
- **Safety on a public repo:** the runner workflow is manual only, fork pull
  requests need approval, and the default token is read-only.
- **Cost:** the server is stopped when not in use.

## 6. Results

- 15 green jobs in the full pipeline (5 unit tests, 5 Sonar scans, 5 Docker jobs).
- 5 SonarCloud projects, one per service.
- 5 Docker Hub repositories, each with a commit SHA tag and `latest`.
- A change in `doctor-service` only builds **doctor-service** in the path pipeline,
  and only its image gets a new tag.
- The job on the EC2 runner passes (tests and Docker build).

Known limits: coverage shows 0% because there is one small test per service and
no coverage report yet. The Sonar Quality Gate does not block the pipeline yet.

## 7. Problems I hit and how I fixed them

| # | Problem | Cause | Fix |
|---|---|---|---|
| 1 | `NO_PUBKEY` when installing Docker | Several commands pasted together, so the key file was not saved | Re-downloaded the key, checked the file, added the source again |
| 2 | Path workflow skipped every job | The trainer's file read `outputs.count`, which the paths-filter action does not have, so the count was empty | Count the services from the `changes` list with `jq length` |
| 3 | Sonar failed: `invalid header value` | `SONAR_TOKEN` secret had a hidden line break from the paste | Saved the secret again as one clean line, re-ran the jobs |
| 4 | Sonar needs a newer Java | The scanner needs JDK 21, the app builds on 17 | Sonar job uses JDK 21 only |
| 5 | Duplicate runs and duplicate image pushes | Two workflows started on every push | Disabled the trainer's two, made the full pipeline manual only |
| 6 | Images pushed on pull requests | `docker-push` had no event check | Added `if: github.event_name != 'pull_request'` |
| 7 | Build folders showed in Git | No `.gitignore` | Added `.gitignore`, removed tracked build output |
| 8 | Sonar Quality Gate showed "Not computed" | First analysis has no earlier run to compare | Expected; checked again after later runs |
| 9 | `java` and `mvn` not found on EC2 | The install line did not run when lines were pasted together | Ran the install as its own command |
| 10 | `docker` permission denied on EC2 | User was not in the `docker` group | `usermod -aG docker`, logged in again |
| 11 | Runner job failed: `No space left on device` | Default 8 GiB disk was full | Freed space, grew the volume to 20 GiB (`growpart`, `resize2fs`) |
| 12 | Typo in the repo name | The original fork name was misspelled | Renamed the repo to `HealthCare-APP` |

## 8. Project stages

| Stage | Work |
|---|---|
| 1 | Accounts: GitHub, Jira, Docker Hub, SonarCloud |
| 2 | Local toolchain on WSL2, manual build of one service, `.gitignore` |
| 3 | GitHub secrets and variable |
| 4 | CI pipeline: build, test, Sonar, Docker push, path-based pipeline |
| 5 | Self-hosted runner on AWS EC2 |

Each stage was tracked as a Jira ticket under the epic **CI Pipeline – HealthCare-APP**.

## 9. What I learned

- Check every command's output before the next one.
- A matrix with `fail-fast: false` gives the full picture in one run.
- Path filters save time in a monorepo, but check which outputs an action really has.
- Never print or commit credentials. Paste secrets without line breaks.
- Tag images with the commit SHA so each image traces to a commit.
- Size the build server disk for Docker images.
- On a public repo, keep self-hosted runner jobs manual and require approval for forks.
- Stop cloud servers when they are idle.

## 10. Next steps

- Make the pipeline fail when the Sonar Quality Gate fails.
- Add a test coverage report.
- Add Snyk for dependency scanning.
- Use ephemeral runners and a cleanup step for old Docker images.
- Add the CD part (deploy to Kubernetes).
