# Migration Plan: `rights` -> `roles` (RBAC terminology consistency)

## Purpose

Bring `N2SNUserTools` and the `n2sn_user_tools` Ansible role into terminology
consistency with the NSLS-II RBAC documentation
(`n2sndocs/explanations/security/rbac/intro.md`).

Per the RBAC docs, the AD groups this tool manages (`n2sn-inststaff-*`,
`n2sn-instusers-*`, `n2sn-analysis-*`, etc.) contain **users directly** and
follow the **role** naming convention. In the RBAC model these are therefore
**role groups**, not right groups (right groups contain role groups, never
users directly, and are named `n2sn-right-<system>-<permission>`).

The tool currently calls these `rights`. This is inconsistent with the docs.
This plan renames the config key `rights` -> `roles` and the user-facing CLI
wording `right`/`RIGHT` -> `role`/`ROLE`, using a staged, backward-compatible
migration so no deployed host is ever left with a tool that cannot read the
deployed config.

The `LDAPInsufficientAccessRightsResult` ldap3 exception in `cli.py` is a
*different, correct* use of "rights" (LDAP ACL permissions) and is left
unchanged.

## Confirmed design decisions

- Rename config key `rights` -> `roles`.
- Rename user-facing CLI wording `right`/`RIGHT` -> `role`/`ROLE`.
- **Config parser:** prefer `roles`, fall back to `rights` with a
  `DeprecationWarning` during the transition. At Stage 6 this becomes a **hard
  failure** (`rights`-only config raises `RuntimeError`).
- **CLI wording:** the user-facing positional argument, help text, and output
  strings switch from `right`/`RIGHT` to `role`/`ROLE`. The positional argument
  has always taken a *role name* (`staff`, `user`, ...) as its value, never the
  literal token `right`, and there was never a `--right` flag, so this is a
  pure wording change with no backward-compatibility concern.
- **Template dual-write:** use **YAML anchors/aliases**. Each instrument anchors
  its role mapping under `roles:` and aliases it into `rights:`. PyYAML (the
  tool's parser, via `yaml.SafeLoader`) fully supports anchors/aliases.
- `fis` `analysis` typo (`n2sn-analysis-fix` -> `-fis`) already fixed by the
  repo owner.
- **Stages 1 and 3 execute in parallel** (no ordering dependency between them).

## Config-parser helper (`cli.py`)

```python
def get_roles(inst_config):
    if 'roles' in inst_config:
        return inst_config['roles']
    if 'rights' in inst_config:
        warnings.warn(
            "The 'rights' key in n2sn_tools.yml is deprecated; rename it to "
            "'roles'. Support will be removed in a future release.",
            DeprecationWarning, stacklevel=2)
        return inst_config['rights']
    raise RuntimeError("Instrument config missing 'roles' section")
```

At Stage 6 the `rights` fallback branch is removed and becomes a hard
`RuntimeError`.

## Template dual-write pattern (`n2sn_tools.yml.j2`)

```yaml
  amx:
    name: AMX
    roles: &amx_roles
      staff: n2sn-inststaff-amx
      user: n2sn-instusers-amx
      analysis: n2sn-analysis-amx
      guacctrl: n2sn-instusers-guacctrl-amx
      guacview: n2sn-instusers-guacview-amx
      nx: n2sn-instusers-nxopen-amx
    rights: *amx_roles
```

- 33 unique anchor names: `&<acronym>_roles`.
- Instrument-specific extra keys (`nyx`: `gp`/`ccp4`/`phenix`/`hkl`;
  `chx`/`csx`: `offline`; `tst`: commented-out keys) live inside the anchored
  block and are therefore shared automatically by the alias.
- At Stage 5, delete the `rights: *<acronym>_roles` lines and drop the
  `&<acronym>_roles` anchor names, leaving plain `roles:` blocks.

## Affected locations

**`N2SNUserTools/N2SNUserTools/cli.py`:**
- L109 `groups = config['rights']` -> `get_roles(config)`
- L166 `inst_config['rights'].keys()` -> `get_roles(inst_config).keys()`
- L186 `inst_config['rights'][right.lower()]` -> via `get_roles(inst_config)`
- L145 `--purge` help, L148-150 positional arg/metavar/help,
  L171-176 local var + validation text, L241-281 output strings
  -> `role`/`ROLE`
- L7 `LDAPInsufficientAccessRightsResult` -> **unchanged**

**`N2SNUserTools/README.rst`:** add brief config-schema section documenting
`roles:` and the `rights:` deprecation.

**`ansible/roles/n2sn_user_tools/templates/n2sn_tools.yml.j2`:** anchor/alias
dual-write for all 33 instruments.

**`ansible/roles/n2sn_user_tools/tasks/*.yml`:** no change.

## Stages

### Stage 1 || Stage 3 (parallel)

**Stage 1 - `N2SNUserTools` (tool accepts both keys):**
1. Add `import warnings` + `get_roles()` helper.
2. Route the three config reads through `get_roles()`.
3. Rename CLI wording to `role`/`ROLE` (positional argument value has always
   been a role name, so this is a pure wording change).
4. Add README config section.
5. Release new RPM.

**Stage 3 - `ansible` role (template dual-write):**
- Convert template to anchor/alias dual-write; deploy fleet-wide.

*Safe in parallel:* dual-key config reads fine on both old (`rights`) and new
(`roles`) tools; the new tool also reads legacy `rights`-only configs.

### Stage 2 - Roll out new tool everywhere
- Bump the package pin; run playbook fleet-wide.
- Confirm `n2sn_list_users` smoke test (`run_n2sn.yml`) passes.

### Stage 4 - Verify no `rights`-only consumers remain
- Fleet package-version check confirming the Stage-1 tool is universal.
- Grep broader infra/config repos for any other consumer reading `rights` from
  `n2sn_tools.yml`; migrate them.
- Confirm no deprecation warnings in logs.

### Stage 5 - Drop `rights` from template
- Remove `rights: *<acronym>_roles` aliases and the anchor names; leave `roles:`
  only. Deploy fleet-wide; smoke tests confirm.

### Stage 6 - Tighten the tool
- **Config parser:** drop the `rights` fallback -> hard `RuntimeError` on
  `rights`-only config.
- Release final version.

(The CLI wording was already changed to `role`/`ROLE` in Stage 1 and needs no
further work here.)

## Sequencing summary

| Stage | Repo | Change | Tool reads | Config has |
|-------|------|--------|-----------|-----------|
| 1 \|\| 3 | N2SNUserTools | Accept both, prefer `roles`, warn on `rights`; CLI->`role` | both | -- |
| 1 \|\| 3 | ansible role | Template anchors `roles`, aliases into `rights` | -- | both |
| 2 | ops | Deploy new tool everywhere | `roles` | both |
| 4 | ops | Verify no `rights`-only consumers | `roles` | both |
| 5 | ansible role | Drop `rights` aliases from template | `roles` | `roles` |
| 6 | N2SNUserTools | Config: hard-fail on `rights`-only | `roles` | `roles` |

**Ordering invariants:**
- Template drops `rights` (Stage 5) only after the new tool is universal
  (Stage 2).
- Tool config parser hard-fails on `rights` (Stage 6) only after the template
  stops emitting `rights` (Stage 5).
- Stages 1 and 3 are independent and run in parallel.

## Progress notes

- **Stage 1 (N2SNUserTools) — DONE (pending release):**
  - Added `import warnings` and `get_roles()` helper in `cli.py`.
  - Routed all three config reads (`n2sn_list`, and the two in
    `n2sn_change_user`) through `get_roles()`.
  - Renamed CLI wording: positional arg `role` / metavar `ROLE`, help/`--purge`
    text, local var `roles`, validation + output strings (`right`->`role`,
    `RIGHT`->`ROLE`). The positional argument value has always been a role name
    (`staff`, `user`, ...), so the rename is transparent to existing callers.
  - Added a Configuration section to `README.rst` documenting `roles:` and the
    `rights:` deprecation.
  - `LDAPInsufficientAccessRightsResult` import left unchanged (correct,
    different meaning).
  - Verified with `py_compile` and a unit check of `get_roles()`: roles-only
    (no warning), rights-only (DeprecationWarning), both (prefers roles, no
    warning), neither (RuntimeError).
  - Remaining: bump version / release RPM.

- **Stage 3 (ansible role) — DONE (pending deploy):**
  - Rewrote `n2sn_tools.yml.j2` so each of the 33 instruments anchors its role
    mapping under `roles: &<acronym>_roles` and aliases it into
    `rights: *<acronym>_roles`.
  - Verified rendered YAML parses with PyYAML `SafeLoader`: all 33 instruments
    have identical `roles` and `rights`; `fis` typo fix preserved
    (`n2sn-analysis-fis`); `nyx` extras, `cryoem` reduced set, and `tst`
    commented-out keys all correct.
  - Remaining: deploy fleet-wide.

- **Next:** Stage 2 (roll out new tool everywhere) then Stage 4/5/6.
