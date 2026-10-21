# Lab 5 · Your first CI pipeline

GitHub user: johnthunderr · Reference files used (Lab 2 fallback): no

![ci](https://github.com/johnthunderr/turiba-devops-labs/actions/workflows/ci.yml/badge.svg?branch=main)

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores), and your user name placeholder in the first line and in the badge URL: write your GitHub user name exactly as GitHub shows it. -->

## 1 · Graph predictions

Part 1, step 1.2: fill in this section **before** you run `lab05 graph`. A wrong prediction that you explain earns full marks.

My drawing of the `lab05 graph` jobs, before the first run (boxes and arrows in text are fine):

```
grph-build and graph-lint start together, since they do not need anything to start. -->> then graph-test starts up, since it depends on (needs) graph-build -->> Afterwards graph-package starts up because it requires [graph-lint, graph-test] -->> And lastly  graph-report because it needs all the previous services to boot up [graph-build, graph-lint, graph-test, graph-package]
____
```

| # | Question | I predicted, and why | What the run showed |
|---|----------|----------------------|---------------------|
| P1 | Which jobs start together? | grph-build and graph-lint start together, since they do not need anything to start, so they can start in parallel | Yes. they started simultainiously |
| P2 | Will `graph-package` find `build.txt`? | No. txt files do not travel around the jobs. they are transformed as artifacts and used in that way. Also, package never downloads an artifact, it can not read the artifact without downloading it. | build.txt: not here: jobs don't share files <<-- Just as predicted |
| P3 | With `break = build`: which jobs run, which fail, which are skipped? | lint runs so does report (due to if: allways()). build fails (due to break). Package, test, graph gets skipped, due to build failure. | Yes. exactly like predicted. |

## 2 · Pipeline

- B5, before I filled it in: `localhost` or `db`? I predicted localhost because the job’s steps run directly on the runner VM, not in a container. db can only be resolved by name WITHIN a container. 
- My first red run in Part 2 (the job, the first red step, the error line copied from the log): All was green and no red runs 
- What I changed to make it green: no changes
- My lint findings, and how I fixed them (fixed, not silenced): npm run lint: no findings. npm test: 5 of 5 unit tests passed.
- Part 2 pull request (URL): https://github.com/JohnThunderr/turiba-devops-labs/pull/5
- Part 3: will the PR run push an image? I predicted All would run and image would get pushed successfully unless some kind of a trap built in the code that I have missed (just like in the step 1 where build = break occured). What happened: no. The step “Log in to GHCR (main only)” was skipped, and the build step ran with push: false, because github.event_name was pull_request. PR runs only build the image; the push happens only on main.
- My image on GHCR, from the `main` run: `ghcr.io/johnthunderr/turiba-devops-labs/api:cc6f6ac5386ece095a0279c7aa0ef0ceada6d24b@sha256:e0aead1d06ef3b4e3990a7490a8872686bbf6be0cf268f4df7667e66135531d0`

## 3 · Red PR

- Red PR (URL):https://github.com/JohnThunderr/turiba-devops-labs/pull/6
- The test that failed, with Expected and Received (copied): Integration test    
    Expected: 404
    Received: 500
- The cause line from "On failure: the API log" (copied): DELETE /api/todos/abc failed: invalid input syntax for type integer: "abc"
- The merge box while it was red (the text, copied): 
    Some checks were not successful
    1 failing, 1 skipped, 2 successful checks
    ✘ ci / integration (pull_request) — Failing after 25s — Required
    ⊘ ci / image (pull_request) — Skipped — Required
    ✔ ci / lint (pull_request) — Successful in 13s — Required
    ✔ ci / unit (pull_request) — Successful in 8s — Required
- Which commit did the red check test: your branch alone, or your branch merged with `main`? How do you know? 
    It tested my branch combined with main, not my branch on its own. For a pull request, GitHub creates a temporary preview of “what main would look like after this merge” and runs the checks on that. The checkout log shows it: HEAD is now at 5925137 Merge 99451f0… into cc6f6ac…, which is my branch’s last commit merged into the current main.

## 4 · Answers

**A1.** 
graph-lint     ✔ success
graph-build    ✘ failure
graph-test     ⊘ skipped
graph-package  ⊘ skipped
graph-report   ✔ success

report ran due to if: allways() which ruyns regardless of status of previous jobs

**A2.** 
because test used 'download' step to read the build.txt artifact where package just tried to read it directly, and directly the file does not exists, its only stored as an artifact. An actions/download-artifact step in graph-package would make it find the file.

**A3.** 
In CI, the job’s steps run directly on the runner VM, not in a container. The PostgreSQL service container publishes port 5432 to that VM, so the API reaches it as localhost:5432. In Lab 4, api was itself a container on the back network, where Docker’s DNS resolves the service name db.

**A4.** 
The required check integration failed (Expected: 404, Received: 500), and image was skipped because it needs all three checks. The run tested my branch combined with main: the temporary merge commit 5925137 (Merge 99451f0… into cc6f6ac…). The merge was stopped by my protect-main ruleset, which requires lint, unit, integration and image to pass, so the merge box said “Some checks were not successful” and the merge button was blocked. On my Part 2 PR, image ran because all three checks it needs were green, but it didn’t push: on a pull_request run the step “Log in to GHCR (main only)” is skipped and the build runs with push: false. Only the push run on main publishes the image.
