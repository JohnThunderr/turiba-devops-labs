# Lab 2 · Containerize it, then debug it

- **Image:** `ghcr.io/<you>/course-api:lab2` (public, linux/amd64 + linux/arm64)
- **Base image digest:** `node:24-alpine@sha256:…`

## Part 1 · Images and layers

| Image | Size | Distro | Default user |
|-------|-----:|--------|--------------|
| node:24        | 1.66Gb | Debian GNU/Linux 12 (bookworm) | root -> User=|
| node:24-slim   | 242Mb  | Debian GNU/Linux 12 (bookworm) | root -> User= |
| node:24-alpine | 332MB  | Alpine Linux v3.24             | root -> uid=0(root) |

cow:bad = 65.5 MB · cow:good = 12.9 MB

| Build | RUN step CACHED? | Build time |
|-------|------------------|-----------:|
| b · app.txt changed         | Yes | 3.8s for the whole build and about 0.2s for the RUN |
| c · deps.txt changed        | No | 19s for the whole build and about 15.2s for the RUN |
| d · app.txt changed, wrong order | No | 19s for the whole build and about 15.2s for the RUN |

## Part 2 · The course API image

sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1

| Step | Image | Size |
|------|-------|-----:|
| Naive | course-api:naive | 1.75 Gb |
| + .dockerignore, npm ci --omit=dev, exec form | course-api:step1 | 1.65 Gb |
| Multi-stage, node:24-alpine, non-root | course-api:lab2 | |
| Reduction against naive | | … % |

docker stop before the SIGTERM handler: … s, exit code … · after: … s, exit code …

Output of `docker run --rm course-api:lab2 id`:

Output of `docker ps` showing (healthy):

## Part 3 · Linux drills

3.1 … 3.7 (outputs and one-line explanations)

## Part 4 · Broken containers

### lab2-broken:N
- Symptom:
- Cause:
- Fix:

## Answers

1. `cow:bad` contains no `/big.file`, yet it is 50 MB bigger than `cow:good`. Why?
    
    Because each RUN command creates a new layer that does not have an access to the previous one. In cow:good example a single Run command was able to use the big.file and delete it right away. The cow:bad did use a big.file in the first run command, then created a second RUN command to delete that file, but was not able to reach back in to do so. It could only mark it as hidden, not actually remove it.

2. Why does the order of `COPY` and `RUN` lines decide how long a rebuild takes?

    Well. Run installs the dependecies, which is the longest step in the exercise (15s sleep time really makes the difference here). Copy only does a small copy of a file whihch does not take as long. Then there is CACHED status which skips the Layer/command completley, it gets ignored. Layer gets cahced only if all the previous layers (on top of it) stays unchanged. Therefore, the ordering of the layers makes sense when they are sorted in a way where at the bottom sits layers that get changed the most. That way, each build can use as much CACHED layers as possible, reducing the execution time .

3. Why did `docker stop` take 10 seconds before you added the SIGTERM handler?


4. Name three things the naive image contained that `course-api:lab2` does not.