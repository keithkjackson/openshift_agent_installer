# Spec: Localhost FIPS detection

## Purpose & user problem

Operators need to know whether **FIPS mode is enabled on the machine running this Ansible role** (the control node / `localhost`), separate from the OpenShift install-config flag `fips_enabled`. Today the role does not inspect OS-level FIPS; users may assume cluster FIPS settings match the provisioning host or may need to verify compliance on the jump box.

## Success criteria

1. During prerequisites (or an equivalent early, always-run path), the role **detects** whether FIPS is active on the current host when the play targets localhost (consistent with existing `tasks/prerequisites.yml` localhost assertion).
2. The result is exposed as a **clear Ansible fact** (or registered variable) whose name does **not** collide with install-config `fips_enabled` (e.g. `localhost_fips_active` or `provisioning_host_fips_enabled`).
3. Default behavior is **informational**: log or debug the detected state without failing the play, unless the operator opts into stricter behavior via a new default variable.
4. Documentation (`README.md`) briefly describes the variable, detection method, and any opt-in failure/warn behavior.

## Scope & constraints

**In scope**

- Detection on **Linux** hosts where this role already runs (RHEL-family primary; align with OpenShift support matrix).
- Placement in `tasks/prerequisites.yml` after localhost validation and fact gathering (reuse existing `ansible.builtin.setup` when present).
- New defaults in `defaults/main.yml` for feature toggle and strictness (names TBD in implementation; see open questions).

**Constraints**

- Must not redefine or repurpose `fips_enabled` (used in `templates/install-config.yaml.j2` and `deploy.yml` for **cluster** FIPS).
- Follow existing patterns: `ansible.builtin.stat` / `slurp` / `command` with `changed_when: false`, minimal new dependencies.
- Tag behavior: include under existing `prerequisites` / `always` flow so the check runs when prerequisites run.

## Technical considerations

1. **Primary signal (RHEL / kernel FIPS user space):** read `/proc/sys/crypto/fips_enabled` when the file exists; value `1` means FIPS mode enabled at the level this interface reports.
2. **Fallback / corroboration (optional):** if `fips-mode-setup` exists, `fips-mode-setup --is-enabled` (or documented equivalent) can confirm RHEL “FIPS mode” vs. the proc knob; spec allows implementing proc-only first if fallback adds complexity without benefit.
3. **Non-Linux / missing file:** set fact to a well-defined value (e.g. `unknown` or `false` with a debug note) rather than failing, unless strict mode requests otherwise.
4. **Display:** optional one-line `debug` after detection, similar to installer version / Python checks.

## Out of scope

- FIPS state on **cluster nodes** or **iDRAC** targets.
- Changing OpenShift `fips` in `install-config` based on localhost detection (automatic coupling is a policy decision and not assumed here).
- Auditing cryptographic modules (FIPS 140 validation).

## Open questions (please confirm)

1. **Strictness:** Should the default be **warn-only** (debug message if mismatch between `fips_enabled` and localhost FIPS), **fail** if cluster `fips_enabled: true` but localhost is not FIPS, **fail** if localhost is FIPS but cluster `fips_enabled: false`, or **never fail** unless a new flag like `localhost_fips_check_enforce: true`? **fail**
2. **Variable names:** Preference for the fact name: `localhost_fips_active` (bool), `provisioning_host_fips_enabled`, or another? `localhost_fips_active`
3. **RHEL-only fallback:** Is `fips-mode-setup` required in v1, or is `/proc/sys/crypto/fips_enabled` alone sufficient for your environments? no

---

**Does this capture your intent? Any changes needed?** When you are satisfied, type **GO!** to proceed with implementation.
