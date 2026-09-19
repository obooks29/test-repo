# Hands-On DevSecOps Lab: Docker Build Caching & GitHub Container Registry (GHCR)

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Topic** | Docker Layer Caching + GitHub Container Registry (GHCR) |
| **Estimated Time** | 20–25 minutes |
| **Builds On** | *Step Outputs vs Job Outputs* lab |

---

## 1. Objective

By the end of this lab, you should be able to:

1. Understand what a **Docker layer** is, and why rebuilding an image from scratch every time is wasteful.
2. Use `docker/build-push-action` with the **GitHub Actions cache backend** (`type=gha`) to cache Docker build layers between workflow runs.
3. Tell the difference between **`cache-from`** and **`cache-to`**, and why you need both.
4. Authenticate to GitHub Container Registry (GHCR) using the built-in `GITHUB_TOKEN`.
5. Build and push a Docker image, tagged using an output from a previous job.
6. Understand the difference between:
   - **Docker layer caching** — speeding up the *build itself* by reusing unchanged layers
   - **A registry** — a permanent, versioned storage location for the *finished image*
7. Chain job outputs across three jobs to keep one consistent image tag from build to publish.

---

## 2. Scenario

You are still working as a DevSecOps Engineer, extending the pipeline from the previous lab.

Right now your pipeline looks like this:

```
BUILD
  ↓
SECURITY
```

Your team has two new problems:

1. **The Docker build is slow.** Every run rebuilds every single layer of the image from scratch — even the layers that didn't change (like installing OS packages or dependencies). Your team lead wants Docker's own **layer cache** reused between runs.
2. **The built image goes nowhere.** Right now the pipeline only *simulates* a build and a scan — no image is actually built or stored. Your team wants the image **built and pushed to GitHub Container Registry (GHCR)** once it passes the security scan.

Your pipeline must become:

```
BUILD (Docker image, with layer caching)
  ↓
SECURITY
  ↓
PUBLISH (push to GHCR)
```

---

## 3. What You Need to Build

```
┌───────────────────────────────┐
│           BUILD JOB            │
│                                 │
│ Step 1: Build image              │
│  (restore cached layers,         │
│   save new/changed layers)       │
│          ↓                      │
│ Step 2: Generate Image Tag      │
└────────────┬────────────────────┘
             │ image_tag (job output)
             ▼
┌───────────────────────────────┐
│         SECURITY JOB           │
│                                 │
│ Receive image_tag               │
│          ↓                      │
│ Scan image                      │
│          ↓                      │
│ Output: scan_result = PASS      │
└────────────┬────────────────────┘
             │ image_tag + scan_result
             ▼
┌───────────────────────────────┐
│         PUBLISH JOB            │
│                                 │
│ Login to GHCR                   │
│          ↓                      │
│ Build & tag image (cache hit)   │
│          ↓                      │
│ Push image to GHCR               │
└───────────────────────────────┘
```

---

## 4. Requirements

Create (or extend) this file:

```
.github/workflows/docker-cache-ghcr-lab.yml
```

Your workflow must contain **three** jobs:

- **Job 1:** `build` — builds the Docker image using layer caching, produces `image_tag`
- **Job 2:** `security` — depends on `build`, simulates a scan, produces `scan_result`
- **Job 3:** `publish` — depends on `build` and `security`, rebuilds (cache-hit, so it's fast) and pushes the image to GHCR, but **only if `scan_result == PASS`**

For the task-by-task walkthrough, see [`STEP-BY-STEP-GUIDE.md`](./STEP-BY-STEP-GUIDE.md).

---

## 5. Two New Concepts, In Plain Terms

### Docker Layer Caching

Every instruction in a `Dockerfile` (`FROM`, `RUN`, `COPY`, etc.) creates a **layer**. Docker normally caches layers *locally on one machine* — but GitHub Actions gives you a fresh, empty machine on every run, so that local cache never survives between runs unless you explicitly store and restore it.

`docker/build-push-action` can do this for you with two options:

- **`cache-from`** — where to look for existing layers before building (the source to restore from)
- **`cache-to`** — where to save this run's layers afterward (the destination to write to)

Using `type=gha` tells both options to use **GitHub Actions' own cache storage** — no extra registry or secret needed.

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

`mode=max` caches **every** layer, including intermediate build stages — not just the final image layers. This matters most in multi-stage Dockerfiles.

### GitHub Container Registry (GHCR)

A **registry** is the opposite of a cache: it is a **permanent, versioned, pullable** storage location for the finished container image. Once you push `ghcr.io/<owner>/<repo>:app-v1.0`, that exact image can be pulled by anyone with access, today or a year from now.

| | Docker Layer Cache | Registry (GHCR) |
|---|---|---|
| Purpose | Speed up the **build step** | Store and distribute the **finished image** |
| Lifetime | Temporary, can be evicted | Permanent until deleted |
| What's stored | Intermediate build layers | The final, tagged image |
| Who uses it | The workflow itself, next time it builds | Humans, deployments, other pipelines |
| Example | `cache-from: type=gha` | `ghcr.io/org/app:v1.0` |

---

## 6. Expected Result

When the workflow runs successfully, you should see something like:

```
Restoring cached layers from GitHub Actions cache...
[+] Building 4.2s (12/12) FINISHED
 => CACHED [2/5] RUN apt-get update && apt-get install -y curl
 => CACHED [3/5] COPY package*.json ./
 => [4/5] RUN npm ci
The image is app:v1.0
Scanning image: app:v1.0
Scan result: PASS
Logging in to ghcr.io...
Building app:v1.0 (cache hit — fast rebuild)...
Pushing ghcr.io/<owner>/<repo>:app-v1.0...
Image pushed successfully
```

The lines starting with `CACHED` are the whole point of this lab — they show layers that were **restored, not rebuilt**. The image tag created in `build` should be the exact same tag used all the way through to the GHCR push, reinforcing the job-output chaining from the previous lab.

---

## 7. DevSecOps Extension

Once the basic lab works, modify it further:

1. Change one line in your `Dockerfile` (e.g. add a new `RUN` command near the end) and re-run the workflow. Compare which layers say `CACHED` vs which get rebuilt — this teaches you **layer ordering**: put things that change often (your app code) *after* things that rarely change (OS packages, dependencies).
2. Add a **second tag** when pushing to GHCR: also push `:latest` alongside `:app-v1.0`, but only on the `main` branch.
3. Make the `publish` job conditional using `if:` so it only runs when `needs.security.outputs.scan_result == 'PASS'` — this is your first real **security gate**.

This is the natural next step after the previous lab: you now have a full pipeline where **layer caching makes builds fast**, and **GHCR makes the result durable**.
# Hands-On DevSecOps Lab: GitHub Actions — Step Outputs vs Job Outputs

| | |
|---|---|
| **Difficulty** | Beginner to Intermediate |
| **Topic** | GitHub Actions Outputs |
| **Estimated Time** | 30–45 minutes |

---

## 1. Objective

By the end of this lab, you should be able to:

1. Create a step output using `$GITHUB_OUTPUT`.
2. Access a step output from another step in the same job.
3. Convert a step output into a job output.
4. Pass a job output from one job to another job.
5. Understand the difference between:
   - `steps.<step_id>.outputs.<output_name>`
   - `needs.<job_id>.outputs.<output_name>`
6. Use job outputs to control what happens in a DevSecOps pipeline.

---

## 2. Scenario

You are working as a DevSecOps Engineer.

Your team has a CI/CD pipeline with two jobs:

```
BUILD
  ↓
SECURITY
```

The `BUILD` job creates a Docker image tag.

The `SECURITY` job needs to know which image was created so that it can scan that image.

For this lab, we will simulate the Docker build and security scan using `echo` commands. The goal is to understand how data moves between steps and jobs.

---

## 3. What You Need to Build

Your workflow should look like this:

```
┌──────────────────────────┐
│       BUILD JOB          │
│                           │
│ Step 1                    │
│ Generate image tag        │
│          ↓                │
│ Step 2                    │
│ Display image tag         │
└────────────┬──────────────┘
             │
             │ Job Output
             │ image_tag
             ▼
┌──────────────────────────┐
│      SECURITY JOB         │
│                           │
│ Receive image_tag         │
│          ↓                │
│ Scan image                │
└──────────────────────────┘
```

---

## 4. Requirements

Create this file:

```
.github/workflows/outputs-lab.yml
```

Your workflow must contain:

- **Job 1:** `build`
- **Job 2:** `security`

The `security` job must depend on the `build` job.

---

## 5. Task 1: Create the Build Job

Create a job called:

```
build:
```

It should run on:

```
ubuntu-latest
```

---

## 6. Task 2: Generate a Step Output

Inside the `build` job, create a step called:

```
Generate Image Tag
```

Give the step this ID:

```
generate
```

The step must create an output called:

```
image_tag
```

The value should be:

```
app:v1.0
```

Use `$GITHUB_OUTPUT`.

**Hint:** You need something similar to:

```bash
echo "name=value" >> "$GITHUB_OUTPUT"
```

Your output should therefore contain:

```
image_tag=app:v1.0
```

---

## 7. Task 3: Use the Step Output

Create a second step called:

```
Display Image Tag
```

This step must display:

```
The image is app:v1.0
```

You must retrieve the value using the step output syntax:

```
steps.<step_id>.outputs.<output_name>
```

Remember:
- step ID = `generate`
- output name = `image_tag`

So you need to construct the correct expression yourself.

---

## 8. Task 4: Create a Job Output

Now comes the important part.

The `security` job is a different job. Therefore, it cannot directly use:

```
steps.generate.outputs.image_tag
```

from the `build` job.

You must expose the step output as a job output.

Create a job output called:

```
image_tag
```

The job output should receive its value from:

```
steps.generate.outputs.image_tag
```

Your structure should look conceptually like:

```yaml
build:
  outputs:
    image_tag: ...
```

Complete the expression yourself.

---

## 9. Task 5: Create the Security Job

Create a second job called:

```
security
```

It must depend on the `build` job. Use:

```
needs:
```

The relationship should be:

```
build
  ↓
security
```

---

## 10. Task 6: Retrieve the Job Output

Inside the `security` job, create a step called:

```
Security Scan
```

Print:

```
Scanning image: app:v1.0
```

This time you cannot use:

```
steps.generate.outputs.image_tag
```

because `generate` belongs to another job.

Instead, use:

```
needs.<job_id>.outputs.<output_name>
```

You know:
- job ID = `build`
- output name = `image_tag`

Construct the correct expression.

---

## 11. Expected Result

When you run the workflow, you should see output similar to:

```
The image is app:v1.0
Scanning image: app:v1.0
```

The important thing is that the value `app:v1.0` was created in one step and eventually consumed by a step in another job.

---

## 12. Your Challenge

Before looking at the solution, try to complete the workflow yourself. You should be able to answer these questions:

**Question 1**
What is the step output?

______________________________________

**Question 2**
What syntax is used to access a step output?

______________________________________

**Question 3**
What is the job output?

______________________________________

**Question 4**
What syntax is used to access a job output from another job?

______________________________________

**Question 5**
Why can't the security job directly use `steps.generate.outputs.image_tag`?

______________________________________

---

## 13. Solution

After attempting the lab, compare your answer with this solution.

```yaml
name: Step and Job Outputs Lab
on:
  workflow_dispatch:

jobs:
  # =================================
  # JOB 1: BUILD
  # =================================
  build:
    runs-on: ubuntu-latest
    # Expose the step output as a job output
    outputs:
      image_tag: ${{ steps.generate.outputs.image_tag }}
    steps:
      # -----------------------------
      # STEP 1
      # -----------------------------
      - name: Generate Image Tag
        id: generate
        run: |
          echo "image_tag=app:v1.0" >> "$GITHUB_OUTPUT"

      # -----------------------------
      # STEP 2
      # -----------------------------
      - name: Display Image Tag
        run: |
          echo "The image is ${{ steps.generate.outputs.image_tag }}"

  # =================================
  # JOB 2: SECURITY
  # =================================
  security:
    needs: build
    runs-on: ubuntu-latest
    steps:
      # -----------------------------
      # STEP 1
      # -----------------------------
      - name: Security Scan
        run: |
          echo "Scanning image: ${{ needs.build.outputs.image_tag }}"
```

---

## 14. Understand the Data Flow

This is the most important part of the lab.

Step 1 creates the value with `id: generate` and:

```bash
echo "image_tag=app:v1.0" >> "$GITHUB_OUTPUT"
```

This creates `steps.generate.outputs.image_tag`.

So:

```
Generate Image Tag
       │
       ▼
steps.generate.outputs.image_tag
```

---

## 15. The Build Job Exposes the Output

This section:

```yaml
outputs:
  image_tag: ${{ steps.generate.outputs.image_tag }}
```

takes the step output and exposes it as a job output.

Think of it as:

```
STEP OUTPUT
     │
     ▼
JOB OUTPUT
```

The job output is now:

```
build.outputs.image_tag
```

---

## 16. The Security Job Receives It

The `security` job has:

```yaml
needs: build
```

Therefore, it can access outputs from the `build` job. It uses:

```
needs.build.outputs.image_tag
```

So the complete flow is:

```
BUILD JOB
Generate Image Tag
       │
       ▼
steps.generate.outputs.image_tag
       │
       ▼
jobs.build.outputs.image_tag
       │
       ▼
SECURITY JOB
needs.build.outputs.image_tag
       │
       ▼
Security Scan
```

---

## 17. The Main Difference

### Step Output

A step output is mainly used for communication **within the same job**.

```
Step A
  ↓
Step B
  ↓
Step C
```

**Reference:**
```
steps.<step_id>.outputs.<output_name>
```

**Example:**
```
steps.generate.outputs.image_tag
```

### Job Output

A job output allows information to move **from one job to another**.

```
Job A
  ↓
Job B
```

**Reference:**
```
needs.<job_id>.outputs.<output_name>
```

**Example:**
```
needs.build.outputs.image_tag
```

---

## 18. The Rule to Remember

| Scope | Syntax | Example |
|---|---|---|
| Same job | `steps` | `steps.generate.outputs.image_tag` |
| Different job | `needs` | `needs.build.outputs.image_tag` |

A simple memory trick:

```
STEP → STEP  = steps
JOB  → JOB   = needs
```

---

## 19. DevSecOps Extension

Once the basic lab works, modify it.

Add a security result to the `security` job:

```
scan_result=PASS
```

Make it a step output. Then expose it as a job output.

Finally, create a third job:

```
deploy
```

Your pipeline should become:

```
BUILD
  │
  │ image_tag
  ▼
SECURITY
  │
  │ scan_result
  ▼
DEPLOY
```

The deployment should only happen when:

```
scan_result == PASS
```

This will be your next level because you will practice: **step outputs → job outputs → security gates → deployment conditions.**

# Step-by-Step Guide: Step Outputs vs Job Outputs

This guide walks through each task in order. See [`README.md`](./README.md) for the overview, objective, and full scenario.

---

## Task 1: Create the Build Job

Create a job called:

```
build:
```

It should run on:

```
ubuntu-latest
```

---

## Task 2: Generate a Step Output

Inside the `build` job, create a step called:

```
Generate Image Tag
```

Give the step this ID:

```
generate
```

The step must create an output called:

```
image_tag
```

The value should be:

```
app:v1.0
```

Use `$GITHUB_OUTPUT`.

**Hint:** You need something similar to:

```bash
echo "name=value" >> "$GITHUB_OUTPUT"
```

Your output should therefore contain:

```
image_tag=app:v1.0
```

---

## Task 3: Use the Step Output

Create a second step called:

```
Display Image Tag
```

This step must display:

```
The image is app:v1.0
```

You must retrieve the value using the step output syntax:

```
steps.<step_id>.outputs.<output_name>
```

Remember:
- step ID = `generate`
- output name = `image_tag`

So you need to construct the correct expression yourself.

---

## Task 4: Create a Job Output

Now comes the important part.

The `security` job is a different job. Therefore, it cannot directly use:

```
steps.generate.outputs.image_tag
```

from the `build` job.

You must expose the step output as a job output.

Create a job output called:

```
image_tag
```

The job output should receive its value from:

```
steps.generate.outputs.image_tag
```

Your structure should look conceptually like:

```yaml
build:
  outputs:
    image_tag: ...
```

Complete the expression yourself.

---

## Task 5: Create the Security Job

Create a second job called:

```
security
```

It must depend on the `build` job. Use:

```
needs:
```

The relationship should be:

```
build
  ↓
security
```

---

## Task 6: Retrieve the Job Output

Inside the `security` job, create a step called:

```
Security Scan
```

Print:

```
Scanning image: app:v1.0
```

This time you cannot use:

```
steps.generate.outputs.image_tag
```

because `generate` belongs to another job.

Instead, use:

```
needs.<job_id>.outputs.<output_name>
```

You know:
- job ID = `build`
- output name = `image_tag`

Construct the correct expression.

---

## Your Challenge

Before looking at the solution, try to complete the workflow yourself. Answer these questions — hints are included, but try answering first before reading them.

**Question 1: What is the step output?**


It’s `image_tag`. That’s the specific name created in Task 2 when `id: generate` saved `image_tag=app:v1.0` into `$GITHUB_OUTPUT`.

______________________________________

*Hint: Look back at Task 2. You created something with `id: generate` that wrote a value using `$GITHUB_OUTPUT`. The step output is that named value itself — what did the `generate` step produce, and what did you name it?*

**Question 2: What syntax is used to access a step output?**


You just call the step ID (`generate`), followed by `outputs`, and then the variable name (`image_tag`).


______________________________________

*Hint: You already wrote this in Task 3, when `Display Image Tag` needed to print the value. It follows the pattern `steps.<something>.outputs.<something>` — fill in the step ID and output name from Task 2.*

**Question 3: What is the job output?**

It’s also called `image_tag`. It was set up under the `build` job’s `outputs:` section, pointing directly to `${{ steps.generate.outputs.image_tag }}` so the whole job can share it.

______________________________________

*Hint: This is what you defined under the `build:` job's `outputs:` block in Task 4. What did you name it, and which step output does it point to?*

**Question 4: What syntax is used to access a job output from another job?**

Since `security` listed `needs: build`, you swap out `steps` for `needs`, mention the job name (`build`), and grab `outputs.image_tag`.

______________________________________

*Hint: Compare this to Question 2, but for cross-job access. It starts with `needs` instead of `steps`, because `security` declared `needs: build`. Fill in the job ID and output name.*

**Question 5: Why can't the security job directly use `steps.generate.outputs.image_tag`?**


Because `steps` only works inside the exact job where it was created. Each job runs in its own separate virtual environment, so the `security` job can't see what happened inside the `build` job's steps. Exposing it as a job output is the only way to pass that information over to the next job.

______________________________________

*Hint: Think about scope. `steps.*` only exists within the job where those steps ran. The `security` job is a completely separate runner — it never saw the `build` job's steps at all. That's exactly why Task 4 (exposing a job output) is necessary in the first place — it's the bridge between jobs.*

> **Self-check:** if you can explain *why* `steps` breaks across jobs but `needs` doesn't, you've understood the core lesson, not just memorized the syntax.

---

## Solution

After attempting the lab, compare your answer with this solution.

```yaml
name: Step and Job Outputs Lab
on:
  workflow_dispatch:

jobs:
  # =================================
  # JOB 1: BUILD
  # =================================
  build:
    runs-on: ubuntu-latest
    # Expose the step output as a job output
    outputs:
      image_tag: ${{ steps.generate.outputs.image_tag }}
    steps:
      # -----------------------------
      # STEP 1
      # -----------------------------
      - name: Generate Image Tag
        id: generate
        run: |
          echo "image_tag=app:v1.0" >> "$GITHUB_OUTPUT"

      # -----------------------------
      # STEP 2
      # -----------------------------
      - name: Display Image Tag
        run: |
          echo "The image is ${{ steps.generate.outputs.image_tag }}"

  # =================================
  # JOB 2: SECURITY
  # =================================
  security:
    needs: build
    runs-on: ubuntu-latest
    steps:
      # -----------------------------
      # STEP 1
      # -----------------------------
      - name: Security Scan
        run: |
          echo "Scanning image: ${{ needs.build.outputs.image_tag }}"
```

---

## Understanding the Data Flow

This is the most important part of the lab.

Step 1 creates the value with `id: generate` and:

```bash
echo "image_tag=app:v1.0" >> "$GITHUB_OUTPUT"
```

This creates `steps.generate.outputs.image_tag`.

So:

```
Generate Image Tag
       │
       ▼
steps.generate.outputs.image_tag
```

### The Build Job Exposes the Output

This section:

```yaml
outputs:
  image_tag: ${{ steps.generate.outputs.image_tag }}
```

takes the step output and exposes it as a job output.

Think of it as:

```
STEP OUTPUT
     │
     ▼
JOB OUTPUT
```

The job output is now `build.outputs.image_tag`.

### The Security Job Receives It

The `security` job has `needs: build`. Therefore, it can access outputs from the `build` job using `needs.build.outputs.image_tag`.

So the complete flow is:

```
BUILD JOB
Generate Image Tag
       │
       ▼
steps.generate.outputs.image_tag
       │
       ▼
jobs.build.outputs.image_tag
       │
       ▼
SECURITY JOB
needs.build.outputs.image_tag
       │
       ▼
Security Scan
```

---

## The Main Difference

### Step Output

A step output is mainly used for communication **within the same job**.

```
Step A
  ↓
Step B
  ↓
Step C
```

**Reference:** `steps.<step_id>.outputs.<output_name>`
**Example:** `steps.generate.outputs.image_tag`

### Job Output

A job output allows information to move **from one job to another**.

```
Job A
  ↓
Job B
```

**Reference:** `needs.<job_id>.outputs.<output_name>`
**Example:** `needs.build.outputs.image_tag`

---

Next: try the DevSecOps Extension in the [README](./README.md#7-devsecops-extension) to practice step outputs → job outputs → security gates → deployment conditions.
