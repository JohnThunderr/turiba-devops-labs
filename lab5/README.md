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

- Red PR (URL): ____
- The test that failed, with Expected and Received (copied): ____
- The cause line from "On failure: the API log" (copied): ____
- The merge box while it was red (the text, copied): ____
- Which commit did the red check test: your branch alone, or your branch merged with `main`? How do you know? ____

## 4 · Answers

**A1.** ____

**A2.** ____

**A3.** ____

**A4.** ____
