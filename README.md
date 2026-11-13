# Jenkins CI/CD Pipeline Optimization (Rails)

A baseline and an optimized declarative Jenkins pipeline for a Rails 8 app, running in Docker on one laptop (WSL2, 4 CPU, 4 GB). Both pipelines were measured and compared. The app itself lives in [rails-ci-demo](https://github.com/sherafganzafar/rails-ci-demo); the Jenkinsfiles here are copies of the ones Jenkins runs from that repo.

## Results

Build time from start to the end of the Test stage (sum of Jenkins stage times). The baseline has no Docker build stage, so that stage is left out of the comparison.

| Run | Pipeline | Network | Install gems | Checks | Total |
|---|---|---|---|---|---|
| rails-baseline #6 | baseline | slow | 44m 09s | Lint 18s, Security 29s, Test 32s (sequential) | 45m 32s |
| rails-optimized #1 | baseline | good | 14m 06s | Lint 17s, Security 1m 28s, Test 35s (sequential) | 16m 33s |
| rails-optimized #3 | optimized, cold cache | good | 15m 41s | Lint 47s, Security 1m 13s, Test 2m 15s (parallel) | 17m 56s |
| rails-optimized #4 | optimized, warm cache | good | 1m 14s | Lint 1m 24s, Security 2m 16s, Test 2m 48s (parallel) | **4m 02s** |

Screenshots: `screenshots/baseline-stage-view.png`, `screenshots/optimized-stage-view.png`.

### What worked
- A Docker volume mounted at `/usr/local/bundle` (gem cache) took the install stage from 14m 06s to 1m 14s on the warm run.

### What did not
- A cold cache run is no faster than the baseline (17m 56s vs 16m 33s). The cache only pays from the second run.
- Parallel Lint, Security and Test did not help. They compete for the same 4 CPUs and 4 GB, so each stage got slower (wall clock 2m 48s vs 2m 20s sequential).

### Caveats
- One run per configuration, not a statistical benchmark.
- The two baseline runs differ mostly because of network conditions (gem install 44m vs 14m on the identical pipeline). That is why the optimized pipeline is compared with the faster baseline.
- Build #6 overlapped for about five minutes with a second build that I aborted.
- rails-optimized #1 and #2 ran the baseline Jenkinsfile because the job's Script Path was not changed at first. Run #1 is used as the second baseline sample.
- The optimized Docker build stage hit the pipeline-wide 20 minute timeout on this network (base image pull). Next step: a separate timeout per stage.

## Pipelines

`Jenkinsfile.baseline`: one Docker agent (`ruby:3.3`), stages run one after another: install gems, RuboCop, Brakeman and bundler-audit, Rails tests.

`Jenkinsfile` (optimized):
- Docker agent with a named volume `gem-cache` for installed gems
- Lint, Security and Test in a `parallel` block
- `timeout` of 20 minutes, `buildDiscarder`, `disableConcurrentBuilds(abortPrevious: true)`
- Docker build stage only on `origin/main`
- `retry(5)` around `bundle install` because of an unstable network

## Security finding

bundler-audit failed the Security stage on real advisories in Rails 8.1.4, nokogiri 1.19.4, loofah 2.25.2 and rails-html-sanitizer 1.7.1. I updated to Rails 8.1.4.1, nokogiri 1.19.7, loofah 2.25.3 and rails-html-sanitizer 1.7.2, and bundler-audit came back clean.

## Problems I hit

- `docker: not found` in the Jenkins container: installed `docker-cli` in the Jenkins image.
- `Killed` (exit 137) during `bundle install`: out of memory in WSL. Fixed with limits in `.wslconfig`.
- Intermittent rubygems timeouts: retries in the Jenkinsfile and in bundler.

## Run it

    cd jenkins-lab
    docker compose up -d --build
    # open http://localhost:8080 and create two Pipeline jobs (Pipeline script from SCM)
    # Script Path: Jenkinsfile.baseline  and  Jenkinsfile

The Jenkins container runs as root with the host Docker socket mounted. That is fine for a local lab, not for production.

## Layout

    Jenkinsfile
    Jenkinsfile.baseline
    jenkins-lab/Dockerfile
    jenkins-lab/docker-compose.yml
    screenshots/
