# Example Voting App

A simple distributed application running across multiple Docker containers.

## Getting started

Download [Docker Desktop](https://www.docker.com/products/docker-desktop) for Mac or Windows. [Docker Compose](https://docs.docker.com/compose) will be automatically installed. On Linux, make sure you have the latest version of [Compose](https://docs.docker.com/compose/install/).

This solution uses Python, Node.js, .NET, with Redis for messaging and Postgres for storage.

Run in this directory to build and run the app:

```shell
docker compose up
```

The `vote` app will be running at [http://localhost:8080](http://localhost:8080), and the `results` will be at [http://localhost:8081](http://localhost:8081).

Alternately, if you want to run it on a [Docker Swarm](https://docs.docker.com/engine/swarm/), first make sure you have a swarm. If you don't, run:

```shell
docker swarm init
```

Once you have your swarm, in this directory run:

```shell
docker stack deploy --compose-file docker-stack.yml vote
```

## Run the app in Kubernetes

The folder k8s-specifications contains the YAML specifications of the Voting App's services.

Run the following command to create the deployments and services. Note it will create these resources in your current namespace (`default` if you haven't changed it.)

```shell
kubectl create -f k8s-specifications/
```

The `vote` web app is then available on port 31000 on each host of the cluster, the `result` web app is available on port 31001.

To remove them, run:

```shell
kubectl delete -f k8s-specifications/
```

## Architecture

![Architecture diagram](architecture.excalidraw.png)

* A front-end web app in [Python](/vote) which lets you vote between two options
* A [Redis](https://hub.docker.com/_/redis/) which collects new votes
* A [.NET](/worker/) worker which consumes votes and stores them in…
* A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
* A [Node.js](/result) web app which shows the results of the voting in real time

## Sysdig image scanning workflow

`.github/workflows/build-scan.yml` builds the `vote`, `result`, and `worker` images locally (never pushed to a registry) and scans each with the Sysdig CLI scanner (`sysdig-cli-scanner`). It's drafted but **not wired up yet** - before it will run successfully, do the following by hand:

1. In the GitHub repo, go to **Settings > Secrets and variables > Actions** and add a repository secret named `SYSDIG_SECURE_TOKEN` with your real Sysdig Secure API token. (The workflow only references this secret by name - no token value is stored anywhere in this repo.)
2. In the same **Variables** tab, add a repository variable named `SYSDIG_SECURE_URL` set to your Sysdig Secure API URL (check **Settings > Sysdig API** in the Sysdig console for the exact value for your region/tenant - don't assume it matches the console's own hostname).
3. Commit and push `.github/workflows/build-scan.yml` (and this README change) to the repo - this workflow was drafted locally and has not been committed or pushed.
4. Optionally review the `--apiurl`/scan step once pushed: the job currently fails if the scan fails policy evaluation (exit code 1 from `sysdig-cli-scanner`). Add `continue-on-error: true` to that step if you'd rather have it report without blocking the pipeline.
5. This repo is a GitHub fork: GitHub disables Actions on forks by default, so go to the repo's **Actions** tab and click through to enable them before expecting any workflow (this one or the existing `call-docker-build-*` ones) to run at all.
6. Fork pull requests don't get repo secrets: a pull request opened *from* another fork of this repo won't have access to `SYSDIG_SECURE_TOKEN` (GitHub withholds secrets from fork-originated PRs for security), so the scan step will fail/be skipped on those PRs even after steps 1-5 are done. It will still work on `push` to `main` and on PRs opened from branches within this same repo.

## Notes

The voting application only accepts one vote per client browser. It does not register additional votes if a vote has already been submitted from a client.

This isn't an example of a properly architected perfectly designed distributed app... it's just a simple
example of the various types of pieces and languages you might see (queues, persistent data, etc), and how to
deal with them in Docker at a basic level.
