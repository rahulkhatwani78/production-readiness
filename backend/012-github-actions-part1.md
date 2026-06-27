# GitHub Actions & CI/CD Pipelines: Basics

In professional software development, you don't manually run tests, build docker images, or copy files to servers every time you write code. Doing this manually is slow, boring, and highly prone to human error.

**CI/CD (Continuous Integration / Continuous Deployment)** solves this by automating the entire lifecycle of your application. **GitHub Actions** is a automation tool built directly into GitHub that lets you build, test, and deploy your code automatically.

---

> **Reference Material:** This guide is based on the YouTube tutorial [Deploy Node.js Application with CI CD and GitHub Actions](https://youtu.be/y7S2oSjJ8PA?si=by3c0iDWb6JSfj18) by Piyush Garg.

---

## 1. What is CI/CD? (Layman's Terms)

### The Car Factory Assembly Line Analogy

Imagine you run a factory that manufactures cars.

*   **Traditional Approach (Manual):**
    A mechanic builds a car from scratch. Once done, they manually inspect it, take it for a test drive, spray-paint it, and drive it to the dealership showroom. 
    *   *Drawback:* It takes days. If the mechanic is tired, they might forget to tighten a wheel bolt (a bug), causing a crash later.
*   **CI/CD Approach (Automated Assembly Line):**
    You put the car parts on an automated conveyor belt.
    *   **Continuous Integration (CI):** As the car moves along the belt, specialized robots automatically weld the chassis, install the engine, and run safety diagnostics at each step. If any robot detects a faulty component, the belt stops immediately (failed build) so you can fix it.
    *   **Continuous Deployment (CD):** Once the car passes all the checks at the end of the line, a robot automatically loads it onto a delivery truck and ships it to the dealership showroom (production environment) without human intervention.

```
       [ CONVEYOR BELT (CI/CD PIPELINE) ]
       
Code Push ---> [ Build & Install ] ---> [ Test & Lint ] ---> [ Deploy to Production ]
                   (Robot 1)               (Robot 2)             (Robot 3)
```

In software:
*   **CI (Continuous Integration):** Automates the process of installing dependencies, checking code styles (linting), and running unit tests every time a developer pushes code.
*   **CD (Continuous Deployment / Delivery):** Automates the process of packaging your app (like building a Docker image) and deploying the new code to your live production servers once all tests pass.

---

## 2. What is GitHub Actions?

**GitHub Actions** is a platform that allows you to automate workflows directly inside your GitHub repository. It listens for events (like when you push code or open a Pull Request) and automatically triggers your conveyor belt (the pipeline).

---

## 3. Core Terminology of GitHub Actions

To write automation rules, you must understand these six key concepts:

```
+-------------------------------------------------------------+
|                      WORKFLOW (.yml file)                   |
|                                                             |
|   Triggered by: EVENT (e.g., Code Push)                     |
|                                                             |
|   +-----------------------------------------------------+   |
|   |                     JOB (Run on Runner)             |   |
|   |                                                     |   |
|   |  Step 1: Check out code (Action)                    |   |
|   |  Step 2: Run npm install (Command)                  |   |
|   |  Step 3: Run npm test (Command)                     |   |
|   +-----------------------------------------------------+   |
+-------------------------------------------------------------+
```

1.  **Workflow:** The overall automated process. It is defined by a YAML file placed in the `.github/workflows/` directory of your repository.
2.  **Event:** The specific trigger that starts a workflow (e.g., `push` to the main branch, opening a `pull_request`, or a scheduled cron time).
3.  **Runner:** The physical or virtual server hosted by GitHub (usually running Ubuntu Linux, Windows, or macOS) where your workflow tasks execute.
4.  **Job:** A series of steps that execute on the same Runner. By default, multiple jobs run in parallel, but you can configure them to depend on each other (e.g., run the Deploy Job only *after* the Test Job completes).
5.  **Step:** An individual task within a job. A step can run a shell command (e.g., `npm run test`) or execute a pre-made tool called an Action.
6.  **Action:** A reusable, packaged plugin that performs complex tasks so you don't have to write them yourself (e.g., checking out code `actions/checkout` or setting up Node.js `actions/setup-node`).

---

## 4. GitHub Secrets Management

Your automation pipeline might need to log into a Docker registry, access your database, or SSH into a VPS. **Never hardcode passwords or private API keys inside your YAML files!**

GitHub solves this with **Repository Secrets**:
1.  Go to your GitHub repository -> **Settings** -> **Secrets and variables** -> **Actions**.
2.  Add your keys (e.g., `DOCKER_PASSWORD`, `DATABASE_URL`) as encrypted secrets.
3.  Reference them securely inside your YAML file using this syntax:
    ```yaml
    ${{ secrets.DOCKER_PASSWORD }}
    ```
GitHub automatically hides these values in the build logs (masking them as `***`) so no one can see them.

---

## 5. Hands-On YAML Configuration Example

To create a workflow, create a folder structure `.github/workflows/` in your project and create a file named `ci-pipeline.yml`:

```yaml
# 1. The name of the workflow displayed in GitHub Actions UI
name: Node.js CI Pipeline

# 2. Define the Event (Trigger)
on:
  push:
    branches:
      - main # Run this workflow whenever code is pushed to main branch
  pull_request:
    branches:
      - main # Run when a PR is opened against main branch

# 3. Define the Jobs to run
jobs:
  run-tests:
    # Specify the OS Runner to use
    runs-on: ubuntu-latest

    # The sequence of Steps in this job
    steps:
      # Step A: Download the code from the repo into the runner
      - name: Checkout repository code
        uses: actions/checkout@v3

      # Step B: Set up Node.js runtime environment on the runner
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm' # Speeds up builds by caching node_modules

      # Step C: Install dependencies
      - name: Install dependencies
        run: npm ci # 'npm ci' is faster and cleaner for automation pipelines

      # Step D: Run Code Linter
      - name: Run code linter
        run: npm run lint

      # Step E: Execute tests
      - name: Run unit tests
        run: npm run test
```

---

## 6. GitHub Actions Production Checklist

- [ ] **Pin Action Versions:** Always use specific tag versions (e.g., `actions/checkout@v3`) rather than `@main` to prevent unexpected breaking changes if the action creator updates their code.
- [ ] **Configure Cache Optimization:** Always use caching for dependencies (like `cache: 'npm'`) to speed up workflow execution and save action minutes.
- [ ] **Enforce Branch Protections:** Configure GitHub to block pull requests from merging if the GitHub Actions CI pipeline fails.
- [ ] **Restrict Permissions:** Configure your workflows with minimal token permissions (readonly by default) to protect your repository from supply-chain attacks.
- [ ] **Set Job Timeout limits:** Set explicit timeout limits on jobs (e.g., `timeout-minutes: 10`) to prevent a hanging test from running forever and consuming all your monthly free billing minutes.
- [ ] **Add Path-based Filters:** Avoid running your test pipeline when updating non-code files (like `README.md` or `.gitignore`).
    *   *Example:*
        ```yaml
        on:
          push:
            paths-ignore:
              - '**.md'
        ```
