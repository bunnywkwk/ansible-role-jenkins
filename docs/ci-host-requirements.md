# CI host requirements — `jenkins`

What the CI host needs so an app pipeline like the reference `Jenkinsfile` (test → build image → push → update GitOps repo) runs, and where each need comes from. Sources: the reference Jenkinsfile (user, 2026-11-10) and the workstation Jenkins, which already runs this kind of CI (`jenkins-2.580.2`, rootless Podman as `jenkins`, checked 2026-11-10).

## From the Jenkinsfile to the host

| Jenkinsfile uses | Needs on the host | Done by |
|---|---|---|
| `pipeline { ... stages ... }`, `sh`, `script`, `when`, `environment` | Pipeline plugins | `workflow-aggregator` |
| `env.BRANCH_NAME`, `env.TAG_NAME` | A multibranch job from GitHub that discovers branches **and tags** | `github-branch-source`, `git` |
| Tag builds (`v1.0.0` → production) | Tags are discovered but **never built** without a build strategy | `basic-branch-build-strategies`; set "Build strategies → Tags" in the job |
| `withCredentials([usernamePassword(...)])` | Credentials plugin + the credentials themselves | `credentials-binding`; credentials from `jenkins_credentials` (JCasC) |
| `sh 'git rev-parse ...'`, `git clone/commit/push` | `git` CLI | `jenkins_build_packages` |
| `docker run ... pytest` | A container runtime the `jenkins` user can use | rootless Podman: `podman`, subuid/subgid, linger, runtime dir |
| `docker build`, `docker push` | Build and push images | `podman build`, `podman push` (no Docker, no `docker` group) |
| Push to the lab registry (plain HTTP) | Podman must allow HTTP for that registry | `jenkins_insecure_registries` |
| `sed`, `rm`, `echo` | Base OS | already on RHEL 9 |
| `jq`, `yq`, `make`, `curl` (brief, common in scripts) | CLI tools | `jq`, `make` from RHEL; `yq` from GitHub; `curl` is in the OS |
| Test results (`junit` step, optional) | JUnit plugin | `junit` |
| Login, no setup wizard | Configured on first boot | `configuration-as-code` + `casc.yaml` |

**No Python, pytest or pip on the host:** the tests run inside a throwaway `python:3.11-slim` container, so each project brings its own versions. The host only needs Podman.

## The reference Jenkinsfile on Podman

Only the `docker` lines change. Three things differ from Docker:

```groovy
// Tests: :Z relabels the workspace for SELinux; full image name (no TTY to answer a short-name prompt)
sh '''
  podman run --rm -v "$PWD":/app:Z -w /app docker.io/library/python:3.11-slim \
    sh -c "pip install --no-cache-dir -r requirements.txt && PYTHONPATH=. pytest -v --junitxml=reports/junit.xml"
'''
// optional, shows the results in Jenkins:  post { always { junit 'reports/*.xml' } }

// Build and push to the lab registry on this VM: no login (registry D4), HTTP allowed by registries.conf.d
sh """
  podman build -t localhost:5000/${env.APP_NAME}:${env.IMAGE_TAG} .
  podman push localhost:5000/${env.APP_NAME}:${env.IMAGE_TAG}
"""
```

| Difference | Why |
|---|---|
| `-v "$PWD":/app:Z` | SELinux blocks a container from the workspace (`var_lib_t`) unless it is relabelled for the container |
| `docker.io/library/python:3.11-slim` | RHEL 9 Podman resolves short names in `enforcing` mode; a build has no TTY to answer the prompt, so a short name can fail |
| No `docker login` for the lab registry | It has no login for now; pushing to Docker Hub instead still needs `podman login docker.io` with a credential |

The GitOps stage needs only `git` and the `github-credentials` credential (`jenkins_credentials`). It puts the token in the clone URL: it can then show in `.git/config` of the workspace. The `ansible-pipeline` Release stage passes it through a git credential helper instead; better for real use.

## Not on the host (job setup)

Done once in the Jenkins UI (or later as code), not by this role:

- The multibranch job for the app repo, with tag discovery and the tag build strategy.
- The GitHub webhook (the workstation relays it through smee.io, because GitHub cannot reach a lab IP).
