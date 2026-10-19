# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The parent `../CLAUDE.md` describes what every formula in this collection shares
(template layout, `map.jinja` merge chain, TOFS, Kitchen platforms, CI, lint
rules). This file only covers what is specific to `rkhunter-formula`.

## Commands

```bash
bundle install
bin/kitchen list                                  # 3 instances, all in suite `debian`
bin/kitchen test debian-debian-13-3008-2          # one instance: <suite>-<platform>
bin/kitchen converge debian-debian-11-3006-27
bin/kitchen verify debian-debian-11-3006-27
pre-commit run --all-files
```

The only Kitchen suite is named `debian` (not `default`), hence the doubled
`debian-debian-…` instance names. Its InSpec profile still lives under
`test/integration/default/`, and it feeds `test/salt/pillar/debian.sls`.

## What the formula does

Two states, no service: rkhunter runs from the cron jobs its Debian package
ships, which this formula drives through `/etc/default/rkhunter`.

- `rkhunter.package` — `pkg.installed` (plus an EPEL repo on Amazon Linux 2 only).
- `rkhunter.config` — two `file.managed`, both through `files_switch()`:
  - `rkhunter.default_file` (`/etc/default/rkhunter` on Debian,
    `/etc/sysconfig/rkhunter` elsewhere) rendered from the `rkhunter:default` dict;
  - `rkhunter.config_file` (`/etc/rkhunter.conf`) rendered from the
    `rkhunter:config` dict.

Almost all Debian behaviour comes from the `Debian:` block of
`rkhunter/osfamilymap.yaml`; `defaults.yaml` only holds the cross-family base.
`osmap.yaml` and `osfingermap.yaml` are empty placeholders, and `osarchmap.yaml`
only sets `arch`, which no state reads.

## The rendering macro — `rkhunter/macro.jinja`

Both templates are a header plus `print_config(dict)`. Each key becomes an
upper-cased rkhunter directive, and the value's type decides the output:

| Value | Output |
|---|---|
| string | `KEY=value` (unquoted) |
| boolean | `KEY="true"` / `KEY="false"` |
| number | `KEY=1` |
| list, key `disable_tests` | one line, `DISABLE_TESTS='a b c'` |
| any other list | one `KEY=item` line per item |

Consequences:

- A pillar value meant as the word `yes`/`no` must be quoted (`'yes'`), otherwise
  YAML turns it into a boolean and it renders as `"true"`. The test pillars do
  exactly that for `allow_ssh_root_user`.
- `/etc/rkhunter.conf` is generated from scratch: any directive the Debian
  package's own `rkhunter.conf` sets and that is absent from the merged
  `config` dict disappears from the host.
- A directive rkhunter does not know is still written. Check it exists in
  rkhunter 1.4.6 (the version on Debian 11 and 13) before adding it to a map —
  `DISABLE_UNHIDE` was removed from the Debian map for that reason, and
  `config_spec.rb` asserts it stays absent.

## Tests

- `test/integration/default/controls/config_spec.rb` asserts on lines of the two
  rendered files, branching on `os[:family]`. Only `check_debian` runs; the
  RedHat and SUSE branches, and `test/salt/pillar/redhat.sls` / `suse.sls`, are
  not wired to any suite.
- When a Debian map value or `debian.sls` changes, update the matching
  `should include` line — assertions spell out the exact rendered directive.
- `rkhunter._mapdata` is in `state_top`, but this fork has no `_mapdata`
  control and no reference dumps under `test/integration/default/files/`.
- The pre-commit config has no rubocop hook; yamllint runs on the whole tree,
  salt-lint on the matching files.
