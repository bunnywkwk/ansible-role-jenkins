# Test results — `jenkins`

Checks that prove the role works, with the exact command and the expected result. **Result** says what actually happened; `not run` means no evidence yet.
Test project: `/home/bunny/cicd-automation/ci-host-test` (inventory: `rhel9-ci` = 192.168.122.135, user `frqadmin`).

## Static checks (workstation, no VM)

| # | Check | Command | Expected | Result |
|---|-------|---------|----------|--------|
| S-1 | YAML lint | `yamllint .` | no output | ✅ 2026-11-10, yamllint 1.38.0 |
| S-2 | Ansible lint | `ansible-lint --profile production` | `Passed: 0 failure(s)` | ✅ 2026-11-10, ansible-lint 26.11.0 on ansible-core 2.16.20 (`/opt/ci`) |
| S-3 | Templates render | render all four templates with sample values (password with `"`, `:`, `#`; one credential; two registries); parse `casc.yaml` as YAML and the registries file as TOML | parses; values intact | ✅ 2026-11-10, ansible-core 2.20.7 |

S-3 proves the files are valid YAML/TOML, **not** that JCasC accepts every key; V-4 does that.

## Runtime checks (RHEL 9 VM)

Run from `ci-host-test/` unless the command says `(VM)`. `(VM as jenkins)` means: on the VM, `cd /tmp && sudo -u jenkins -H env XDG_RUNTIME_DIR=/run/user/$(id -u jenkins) <command>` (the jenkins shell is `/bin/false`, so `sudo -iu jenkins` does not work).

| # | Check | Command | Expected | Result |
|---|-------|---------|----------|--------|
| V-1 | First run | `ansible-playbook playbooks/site.yml --tags jenkins` | `failed=0`, `changed` > 0 | ✅ implied, no log kept: the V-2 rerun found every change already applied |
| V-2 | Idempotent | same, rerun | `changed=0` | ✅ 2026-11-11: `ok=20 changed=0 failed=0`; Jenkins already running (Start task `ok`). Log `ci-host-test/runs/jenkins-v1.log` |
| V-3 | Service up | (VM) `systemctl is-active jenkins` and `systemctl show jenkins -p Environment` | `active`; override values present | not run |
| V-4 | JCasC applied, no wizard | `curl -s -o /dev/null -w '%{http_code}\n' http://192.168.122.135:8080/login`, then log in in a browser | `200`; login page, not "Unlock Jenkins"; Manage Jenkins → Configuration as Code shows `casc.yaml` | ✅ 2026-11-11: `/login` → `200`; admin login worked and a job could be created (no wizard). Configuration as Code page not reported |
| V-5 | Login enforced | `curl -s -o /dev/null -w '%{http_code}\n' http://192.168.122.135:8080/api/json` | `403` (anonymous has no read) | ✅ 2026-11-11: `403` (JCasC security active) |
| V-6 | Plugins | Manage Jenkins → Plugins → Installed, or `curl -s -u admin:<pw> 'http://192.168.122.135:8080/pluginManager/api/json?depth=1' \| jq -r '.plugins[].shortName' \| sort` | all of `jenkins_plugins`, no "failed to load" banner | not run |
| V-7 | Build discarder | Manage Jenkins → System → Global Build Discarders | "Specific build discarder", keep 30 | not run |
| V-8 | Tools | (VM) `git --version; podman --version; jq --version; yq --version; make --version \| head -1; java -version` | all print a version | not run |
| V-9 | Rootless Podman as jenkins | (VM as jenkins) `podman info --format '{{.Host.Security.Rootless}} {{.Store.GraphRoot}}'` | `true /var/lib/jenkins/.local/share/containers/storage` | not run |
| V-10 | Reboot | (VM) `sudo reboot`; then V-3, V-4, V-9 | all pass (linger brings back `/run/user/<uid>`) | not run |
| V-11 | Firewall | (VM) `sudo firewall-cmd --list-ports` | contains `8080/tcp` | not run |
| V-12 | Build and push from a job | see [below](#v-12-pipeline-builds-and-pushes) | job green; image in the registry catalog | ✅ 2026-11-11, user-reported: `smoke-push` succeeded (console not saved) |

### V-12: pipeline builds and pushes

Needs the registry role on the same VM (`--tags registry` first, or a full `site.yml` run). In Jenkins: New Item → Pipeline → paste:

```groovy
pipeline {
  agent any
  stages {
    stage('Build and push') {
      steps {
        sh '''
          printf 'FROM docker.io/library/alpine:3.20\\nRUN echo built-by-jenkins > /built\\n' > Containerfile
          podman build -t localhost:5000/test/jenkins-smoke:${BUILD_NUMBER} .
          podman push localhost:5000/test/jenkins-smoke:${BUILD_NUMBER}
          curl -s http://localhost:5000/v2/_catalog
        '''
      }
    }
  }
}
```

Expected: build succeeds; the console ends with a catalog containing `test/jenkins-smoke`. This is the brief's "a test job can build an image and push it to the registry".

## Evidence

Save each run's output under `ci-host-test/runs/` and fill in the Result column with the date and the key numbers.
