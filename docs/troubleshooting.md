# Troubleshooting log: `jenkins`

Errors hit while building and testing this role, with the exact message, cause and fix.
Newest last. Format: **Symptom** (exact output) → **Cause** → **Fix** → **Prevent**.

---

## T-J1: `Destination directory /etc/systemd/system/jenkins.service.d does not exist` (2026-11-10)

**Symptom** (first run on `rhel9-ci`, commit `025dfe4`)
```
TASK [jenkins : CONFIG | Install the systemd override (home, port, JVM, JCasC, Podman runtime dir)]
fatal: [rhel9-ci]: FAILED! => {"changed": false, "msg": "Destination directory /etc/systemd/system/jenkins.service.d does not exist"}
PLAY RECAP: ok=21 changed=9 failed=1
```

**Cause:** `ansible.builtin.template` writes a file but does not create missing parent folders. Nothing creates the drop-in folder `jenkins.service.d/`: the RPM ships only `/usr/lib/systemd/system/jenkins.service`, and systemd makes the folder only through `systemctl edit`. The registry role did not hit this because `containers-common` ships `/etc/containers/systemd/`.

**Fix:** new task `CONFIG | Create the systemd drop-in folder` (`root:root 0755`) before the template.

**Prevent:** before a `template` or `copy` into a folder, check the folder exists on a fresh host (`rpm -qf <dir>`); if no package owns it, create it in the role.
