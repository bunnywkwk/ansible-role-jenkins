# Design decisions — `jenkins`

Why the role is built this way. Format: **Decision** → **Why** → **Changed** (if it was).

---

## D1: Copy the workstation's working setup

**Decision:** same Jenkins LTS (`2.580.2`), same repo (`pkg.jenkins.io/rpm-stable`), same rootless-Podman steps (subuid block `786432`, linger), same plugin set minus the role-CI ones.

**Why:** the workstation Jenkins already runs Podman builds as `jenkins` (`ansible-pipeline/docs/ci/setup-guide.md`, steps 4, 5, 10). Copying it uses what is proven instead of new guesses. Left out: `remote-file` (Remote Jenkinsfile Provider, only for the role CI), `gradle`, `ldap`, `email-ext`, `dark-theme`.

## D2: systemd drop-in, not an edited unit

**Decision:** `jenkins.service.d/override.conf` sets home, port, JVM options, JCasC path and the Podman runtime dir.

**Why:** the RPM owns `/usr/lib/systemd/system/jenkins.service` and replaces it on upgrade; a drop-in survives and shows only what we change. The RPM unit reads `JENKINS_HOME`, `JENKINS_PORT` and `JAVA_OPTS` from `Environment=`, so a later line overrides it.

## D3: Plugins by the plugin manager tool, before the first start

**Decision:** `jenkins-plugin-manager` (jenkinsci/plugin-installation-manager-tool) downloads the plugins into `JENKINS_HOME/plugins` while Jenkins is stopped.

**Why:**
- JCasC is a plugin: it must exist before Jenkins first starts, or Jenkins boots with no login (the wizard is off).
- `community.general.jenkins_plugin` needs Jenkins already running and logged in: a window where Jenkins is open, plus an extra collection.
- The tool reads the war, so it picks plugin versions that fit this Jenkins, and pulls dependencies.

**Limit:** it runs only when `plugins.txt` changes. Removing a plugin from the list does not delete it from disk.

## D4: JCasC for login and settings

**Decision:** one `casc.yaml`: local admin user, `loggedInUsersCanDoAnything` (no anonymous read), executors, global build discarder, credentials.

**Why:** the brief asks for no wizard and the admin from Vault. Config in a file is repeatable on a fresh VM. Jenkins re-applies it at every start, so a UI change to these settings is overwritten.

**Unlike the registry, a login is required:** an open Jenkins lets anyone run any command on the host (script console, a pipeline `sh`).

## D5: Rootless Podman as `jenkins`, no Docker

**Decision:** `jenkins` runs `podman` itself (subuid/subgid + linger + runtime dir in the unit). No Docker, no `docker` group.

**Why:** user decision (Podman, 2026-11-10). The `docker` group is root-equivalent (brief gotcha 5): anyone who can start a container with `/` mounted owns the host. Rootless Podman runs as `jenkins` only.

**Limit:** the pipeline's containers can do anything `jenkins` can, including read `JENKINS_HOME` (credentials are encrypted there). Separate build agents fix this; out of scope for one lab VM.

## D6: `JENKINS_HOME` is also the jenkins user's home

**Decision:** `usermod -d <jenkins_home> jenkins` when they differ.

**Why:** rootless Podman keeps images under `~/.local/share/containers`. On the RPM default both are `/var/lib/jenkins`; on the large disk they should move together (brief gotcha 4).

**Not tested:** a non-default `jenkins_home`. SELinux labels for rootless Podman storage on a custom path may need `semanage fcontext`. Default path: same as the workstation.

## D7: Plain-HTTP registry trust only (for now)

**Decision:** `jenkins_insecure_registries` writes `insecure = true` entries; no CA-cert option.

**Why:** the registry role has no TLS yet (registry D3). The brief's "install a CA cert" option comes back with TLS.

## D8: Package signatures checked, repo metadata signature not

**Decision:** `gpgcheck: true`, `repo_gpgcheck` not set (off).

**Why:** every Jenkins RPM is checked against the imported key. Metadata signing (`repo_gpgcheck`) keeps its key in a separate dnf keyring that a non-interactive run may not import, a likely first-run failure. Revisit if the VM test shows it works.

## D9: Downloads without checksums (shortcut)

**Decision:** `yq` and the plugin manager jar come from GitHub releases over HTTPS, pinned by version, without a checksum.

**Why:** lab speed. Before real use, add `checksum:` to both `get_url` tasks (`shortcut:` comments in `tasks/install.yml`).

## D10: Not onboarded to `ansible-pipeline` yet

Same as registry D10: user decision, 2026-11-10.
