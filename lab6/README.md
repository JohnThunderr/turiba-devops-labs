# Lab 6 · Test what you ship, ship a release

GitHub user: JohnThunderr · Reference pipeline used (Part 0 fallback): no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores), and <you> in the first line and in section 3. -->

## 1 · Smoke test

- The names of my smoke-test steps in the `image` job: "Build for the smoke test (amd64, kept on the runner)" and "Smoke test the image"
- What my smoke test asks, and what makes it fail: it starts the freshly built image with `docker run -d -p 5000:5000` and asks `GET /` with `curl --fail` and up to 10 retries, one second apart. It fails if there is no 2xx answer, and then prints `docker logs` so the container's own error is visible in the run.
- 🔮 Part 1, before my first run: will my code pass the smoke test? I predicted yes because the Lab 5 image already ran fine locally and `GET /` needs no database. What happened: it passed; `image` was green on PR 9 and the curl printed the JSON name/version.
- 🔮 Part 2, before the checks finished: which checks will go red after `break.sh`? I predicted only `image` (in the smoke step) because lint, unit and integration run the source code on the runner, where `banner.js` exists, while the image only contains what `.dockerignore` lets into the build context.

## 2 · What broke and how I found it

- The check and the step that went red: the `image` check, in the step "Smoke test the image" (PR 10, first commit `c10eb0d`)
- The log line that showed the cause (copied from the log):

```text
Error: Cannot find module './banner'
  code: 'MODULE_NOT_FOUND',
```

- The cause, in my own words: the teammate's commit added `banner.js` to `api/.dockerignore`, so the file never got into the build context. `server.js` now requires `./banner`, so in the container node cannot find it and crashes on start, and `curl` could never connect to port 5000.
- Why `lint`, `unit` and `integration` stayed green: they run the API from source on the runner, where `banner.js` is present. Only the image is built from the filtered build context, and until the smoke test was added nothing ever started the image.
- My fix, and the commit that made it: I deleted only the two `banner.js` lines (the comment and the entry) from `api/.dockerignore`, so `.env` and `node_modules` stay excluded. Commit `8661f59` "fix(api): keep banner.js in the image". All four checks went green and PR 10 was merged.

## 3 · Release

- Release page: https://github.com/JohnThunderr/turiba-devops-labs/releases/tag/v0.2.0
- Image: `ghcr.io/johnthunderr/turiba-devops-labs/api:0.2.0`, platforms (from `docker buildx imagetools inspect`): linux/amd64 and linux/arm64 (the other two entries are attestation manifests)
- One thing in Part 3 that didn't work the first time, and how I fixed it: my first commit of `release.yml` was an empty file because of a shell quoting mistake when I wrote it from a heredoc. I noticed it before the merge, wrote the workflow again properly and pushed a second commit to the same PR.
