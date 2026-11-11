# Code walkthrough — `jenkins`

What each file does and why, briefly. Deeper reasons: `Dn` in [design-decisions.md](design-decisions.md).
**Written by:** Claude, at the user's request (2026-11-10); the user reviews.

| # | File | Status |
|---|------|--------|
| 1 | `meta/main.yml` | done |
| 2 | `.yamllint`, `.ansible-lint`, `.gitignore` | done, copied from the registry role (from `mongodb-cis`) |
| 3 | `defaults/main.yml`, `vars/main.yml` | done |
| 4 | `tasks/main.yml`, `install.yml`, `podman.yml`, `config.yml`, `service.yml` | done, lint clean, not run on a VM yet |
| 5 | `handlers/main.yml` | done |
| 6 | `templates/override.conf.j2`, `plugins.txt.j2`, `casc.yaml.j2`, `insecure-registries.conf.j2` | done, rendered and parsed with sample values |
| 7 | `requirements.yml`, `VERSION`, `README.md` | done |

---

## How it works in one picture

```text
install.yml   repo + key ──► dnf: jenkins-2.580.2, Java 21, git, podman, jq, make ──► yq, plugin manager jar
podman.yml    /etc/subuid + /etc/subgid ──► linger ──► registries.conf.d (plain-HTTP registries)
config.yml    JENKINS_HOME ──► override.conf ──► plugins.txt ──► casc.yaml
                                  │                  │              │
                                  │       handler: plugin manager downloads plugins
                                  └──────────────────┴──────────────┴──► handler: restart jenkins
service.yml   flush handlers (plugins on disk before the first start) ──► start + enable ──► firewalld 8080
```

At start, Jenkins reads `casc.yaml` (JCasC): admin login, security, executors, build discarder, credentials. No setup wizard.

---

## 1. `meta/`

| File | Does | Why |
|------|------|-----|
| `meta/main.yml` | Full name `bunnywkwk.jenkins`, EL 9, ansible-core 2.16, no dependencies | Same rules as the registry role; no dependency on it (values pass through variables) |

## 2. `defaults/main.yml` and `vars/main.yml`

| File | Holds |
|------|-------|
| `defaults/main.yml` | Your choices: versions, port, home, JVM options, admin, executors, plugins, credentials, tools, Podman, firewall |
| `vars/main.yml` | Fixed internals: repo URL and key, war path, service and user name, paths of the files the role writes, download URLs |

| Variable | Default | Why this default |
|----------|---------|------------------|
| `jenkins_version` | `2.580.2` | The LTS the workstation runs; the RPM name is `jenkins-<version>` |
| `jenkins_java_package` | `java-21-openjdk-headless` | In the RHEL 9 repos; headless = no desktop libraries |
| `jenkins_home` | `/var/lib/jenkins` | RPM default; point it at the large disk |
| `jenkins_plugins` | 10 plugins | What the reference Jenkinsfile needs ([ci-host-requirements.md](ci-host-requirements.md)) |
| `jenkins_credentials` | `[]` | Placeholders; real ones in `group_vars` with ansible-vault |
| `jenkins_subid_start` | `786432` | Same block as on the workstation; clear of the first login user (`100000`) |
| `jenkins_insecure_registries` | `[]` | No registry assumed; the lab sets `localhost:5000` |
| `jenkins_builds_to_keep` | `30` | Same as `ansible-pipeline`'s `logRotator(numToKeepStr: '30')` |

## 3. Tasks

### `tasks/install.yml`

| Task | Does | Why |
|------|------|-----|
| `INSTALL \| Import the Jenkins package signing key` | `rpm --import` of `pkg.jenkins.io/rpm-stable/repodata/repomd.xml.key` | dnf checks every Jenkins package against it (`gpgcheck`); same key URL as the workstation's repo file |
| `INSTALL \| Add the Jenkins LTS repository` | `/etc/yum.repos.d/jenkins.repo` | `rpm-stable` = LTS releases only |
| `INSTALL \| Install Jenkins, Java and the build tools` | One `dnf` call: `jenkins-2.580.2`, Java, `fontconfig`, `jenkins_build_packages` | One transaction; `fontconfig` is in Jenkins' RHEL install steps (graphs, fonts) |
| `INSTALL \| Install yq` | Binary from GitHub to `/usr/local/bin/yq` | Not in the RHEL 9 repos. `shortcut:` no checksum, no upgrade |
| `INSTALL \| Download the Jenkins plugin manager` | `jenkins-plugin-manager-<ver>.jar` to `/usr/local/lib/` | Installs plugins offline, before the first start (D3). Version in the file name, so a new version downloads again |

### `tasks/podman.yml`

| Task | Does | Why |
|------|------|-----|
| `PODMAN \| Give jenkins a subordinate UID and GID range` | `jenkins:786432:65536` in `/etc/subuid` and `/etc/subgid` | Rootless Podman maps container users (root, 1, 2…) to these host IDs. The RPM makes `jenkins` a system user, which gets none |
| `PODMAN \| Enable linger for jenkins` | `loginctl enable-linger jenkins` (once: `creates:`) | `jenkins` never logs in; linger starts its user manager and `/run/user/<uid>` at boot, which rootless Podman needs |
| `PODMAN \| Allow the plain-HTTP registries` | `/etc/containers/registries.conf.d/50-jenkins-insecure.conf` | Podman refuses HTTP registries unless listed `insecure = true` (registry D3) |

### `tasks/config.yml`

| Task | Does | Why |
|------|------|-----|
| `CONFIG \| Read the jenkins user's UID` | `getent passwd jenkins` | The UID goes into `/run/user/<uid>` in the override; it differs per host |
| `CONFIG \| Create JENKINS_HOME` | `jenkins:jenkins 0750` | Needed when `jenkins_home` is not the RPM default |
| `CONFIG \| Make JENKINS_HOME the jenkins user's home` | `usermod -d` when they differ | Rootless Podman stores images in `~/.local/share/containers`: keep them on the large disk too. Fails if Jenkins is running (change it on a fresh host) |
| `CONFIG \| Create the systemd drop-in folder` | `/etc/systemd/system/jenkins.service.d/`, `root:root 0755` | No package creates it, and `template` does not make parent folders (T-J1) |
| `CONFIG \| Install the systemd override` | `jenkins.service.d/override.conf` | Changes only home, port, JVM options, JCasC path and the Podman runtime dir; the RPM's unit stays as shipped (D2) |
| `CONFIG \| Write the plugin list` | `plugins.txt` in `JENKINS_HOME` | A change runs the plugin manager, then restarts |
| `CONFIG \| Write the JCasC file` | `casc.yaml`, `0600`, `diff: false` | Holds the admin password; `diff: false` keeps it out of `--diff` output |

### `tasks/service.yml`

| Task | Does | Why |
|------|------|-----|
| `SERVICE \| Install plugins and apply changes before the start` | `meta: flush_handlers` | The JCasC plugin must be on disk before Jenkins first starts, or it boots without a login |
| `SERVICE \| Start and enable Jenkins` | started, enabled | A normal RPM unit (not generated), so `enabled: true` works here |
| `SERVICE \| Open the Jenkins port` | firewalld `jenkins_port/tcp` | Browser and webhooks from other hosts |

## 4. `handlers/main.yml`

| Handler | Does | Why |
|---------|------|-----|
| `Install jenkins plugins` | `runuser -u jenkins -- java -jar <plugin manager> --war jenkins.war --plugin-download-directory <home>/plugins --plugin-file plugins.txt` | Runs as `jenkins` so the files belong to it; `--war` makes it pick plugin versions that fit this Jenkins; it pulls dependencies |
| `Restart jenkins` | `daemon-reload`, restart | The unit is `Type=notify`, so the restart waits until Jenkins says it is up; a broken `casc.yaml` fails here |

Handlers run in the order of this file: plugins first, then the restart.

## 5. Templates

| Template | Becomes | Key lines |
|----------|---------|-----------|
| `override.conf.j2` | `/etc/systemd/system/jenkins.service.d/override.conf` | `JENKINS_HOME` + `WorkingDirectory`; `JENKINS_PORT`; `JAVA_OPTS` with `-Djenkins.install.runSetupWizard=false`; `CASC_JENKINS_CONFIG`; `XDG_RUNTIME_DIR` and `DBUS_SESSION_BUS_ADDRESS` so `podman` works in `sh` steps without exporting them in every Jenkinsfile |
| `plugins.txt.j2` | `<home>/plugins.txt` | One plugin per line, `name` or `name:version` |
| `casc.yaml.j2` | `<home>/casc.yaml` | Local user database, no sign-up; logged-in users can do anything, anonymous nothing; executors; global build discarder; credentials. Values pass through `to_json`, so quotes and colons in passwords are safe; only `${` is not (JCasC variable syntax) |
| `insecure-registries.conf.j2` | `/etc/containers/registries.conf.d/50-jenkins-insecure.conf` | One `[[registry]] insecure = true` block per entry |
