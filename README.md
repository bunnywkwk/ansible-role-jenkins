# jenkins

Ansible role that sets up **Jenkins LTS** on **RHEL 9** as a CI host: configured as code (no setup wizard), the plugins an app pipeline needs, and **rootless Podman** so the `jenkins` user can run tests in containers, build images and push them.

Built for the lab CI host: Jenkins and the `registry` role on one VM. The two roles do not depend on each other: the registry address reaches this role through `jenkins_insecure_registries`.

## What it does

```text
1. INSTALL  Jenkins repo + key → jenkins-2.580.2, Java 21, git, podman, jq, make → yq, plugin manager
2. PODMAN   subuid/subgid for jenkins → linger → plain-HTTP registries allowed
3. CONFIG   JENKINS_HOME → systemd override → plugins.txt → casc.yaml (JCasC)
4. SERVICE  plugins installed before the first start → Jenkins started and enabled → port opened
```

A second run reports `changed=0`. Jenkins restarts only when the override, the plugin list or `casc.yaml` changes.

What the CI host needs and why, mapped to a real Jenkinsfile: [docs/ci-host-requirements.md](docs/ci-host-requirements.md).

## Requirements

- RHEL 9 with its repos, and access to `pkg.jenkins.io`, `updates.jenkins.io` (plugins) and `github.com` (yq, plugin manager).
- ansible-core 2.16 or later on the control node.
- Collections: `ansible-galaxy collection install -r requirements.yml` (`ansible.posix`, for firewalld).
- firewalld running, unless `jenkins_manage_firewall: false`.

## Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `jenkins_admin_password` | none, **required** | Admin password, from ansible-vault |
| `jenkins_admin_user` | `admin` | Admin login |
| `jenkins_version` | `2.580.2` | Jenkins LTS (RPM) |
| `jenkins_java_package` | `java-21-openjdk-headless` | Java runtime |
| `jenkins_port` | `8080` | HTTP port |
| `jenkins_home` | `/var/lib/jenkins` | Jobs, builds, plugins and jenkins' Podman images; put it on the large disk |
| `jenkins_java_opts` | `""` | Extra JVM options, e.g. `-Xmx1g` |
| `jenkins_url` | `""` | Root URL for links Jenkins sends (GitHub statuses) |
| `jenkins_executors` | `2` | Builds at once |
| `jenkins_builds_to_keep` | `30` | Builds kept per job |
| `jenkins_plugins` | 10 plugins | See `defaults/main.yml`; `name` or `name:version` |
| `jenkins_plugin_manager_version` | `2.13.2` | Plugin installation manager release |
| `jenkins_credentials` | `[]` | `{id, username, password, description}` for `withCredentials` |
| `jenkins_build_packages` | `git`, `podman`, `jq`, `make` | Build tools from the RHEL repos |
| `jenkins_yq_version` | `v4.44.3` | yq release |
| `jenkins_subid_start` | `786432` | First of 65536 subordinate IDs for rootless Podman |
| `jenkins_insecure_registries` | `[]` | Plain-HTTP registries, e.g. `["localhost:5000"]` |
| `jenkins_manage_firewall` | `true` | Open `jenkins_port` in firewalld |

## Quick start

```yaml
# group_vars/ci.yml
jenkins_admin_password: !vault |     # ansible-vault encrypt_string --name jenkins_admin_password
  ...
jenkins_insecure_registries:
  - "localhost:5000"
```

```yaml
# playbooks/site.yml
- name: Set up the CI host
  hosts: ci
  become: true
  roles:
    - role: registry
      tags: registry
    - role: jenkins
      tags: jenkins
```

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass        # rerun: changed=0
```

Then open `http://<host>:8080/` and log in as `admin`: no setup wizard.

## Docs

| File | Read it for |
| --- | --- |
| [docs/ci-host-requirements.md](docs/ci-host-requirements.md) | What the host needs for an app pipeline, and the Jenkinsfile on Podman |
| [docs/code-walkthrough.md](docs/code-walkthrough.md) | What each file and task does, and why |
| [docs/design-decisions.md](docs/design-decisions.md) | Why the drop-in, the plugin manager, JCasC, rootless Podman (`D1`–`D10`) |
| [docs/test-results.md](docs/test-results.md) | Checks that prove the role works: commands, expected and actual results |

## Known limits

- Removing a plugin from `jenkins_plugins` does not delete it from disk.
- `yq` and the plugin manager are downloaded without a checksum, and an existing `yq` is not upgraded.
- Changing `jenkins_home` on a host where Jenkins runs fails at `usermod`; a non-default `jenkins_home` is not tested (SELinux labels for Podman storage).
- Pipeline containers run as `jenkins` and can read `JENKINS_HOME`; separate build agents would isolate them.
