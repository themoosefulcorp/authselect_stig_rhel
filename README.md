# Ansible Role: authselect STIG (RHEL)

An Ansible role that builds, activates, and maintains a custom `authselect`
profile on Red Hat Enterprise Linux 8 and 9, configured to satisfy the DISA
STIG requirements that are applied through the PAM authentication stacks and
through `authselect` features.

This role manages only the pieces of STIG compliance that legitimately belong
to `authselect`: the contents of the `system-auth`, `password-auth`, and (on
RHEL 8) `smartcard-auth` PAM templates, plus the set of `authselect` features
that map to STIG findings. It deliberately does **not** manage settings that
live elsewhere, such as `/etc/security/faillock.conf`, `/etc/security/pwquality.conf`,
`/etc/security/pwhistory.conf`, `/etc/sssd/sssd.conf`, SELinux contexts, audit
rules, `dconf`, or `/etc/pam.d/su` and `/etc/pam.d/sudo`. Those requirements
are real, but they are the responsibility of separate roles. See
[Scope and STIG coverage](#scope-and-stig-coverage) below for the full boundary.

> **Disclaimer:** This role is provided as-is, without warranty of any kind.
> It modifies PAM authentication stacks; a misconfiguration can lock users out
> of a system. Test in a non-production environment and verify you can
> authenticate before applying to systems that matter. You are responsible for
> validating STIG compliance in your own environment.

## Requirements

- RHEL 8 (8.4 or newer) or RHEL 9. The role asserts the OS family and major
  version and fails fast on anything else, and fails on RHEL 8 releases older
  than 8.4 because the STIGs are applied differently on those versions.
- A system that uses (or will use) `sssd` for authentication. The custom
  profile is based on the built-in `sssd` profile. Both `authselect` and
  `sssd` are installed by the role.
- Privilege escalation (`become`), since the role writes under
  `/etc/authselect/` and runs `authselect` subcommands.

## Role Variables

Available variables are listed below, along with their default values (see
`defaults/main.yml`).

    authselect_profile_name: stig

The short name of the custom profile this role creates and manages. The profile
is created as `custom/{{ authselect_profile_name }}` and based on the built-in
`sssd` profile. Changing this changes which profile the role owns; it does not
rename an existing profile.

    authselect_desired_features:
      - with-mkhomedir
      - with-subid
      - with-sudo
      - with-smartcard
      - with-smartcard-lock-on-removal
      - with-faillock
      - without-nullok
      - with-pwhistory

The list of `authselect` features the role will ensure are **enabled** on the
managed profile. Several of these map directly to STIG findings (for example,
`with-faillock` and `without-nullok`); others are operational
(`with-mkhomedir`, `with-sudo`). This must always be defined as a list, even if
empty (`[]`). The role does not enforce that it is defined; supplying a string
instead of a list will silently corrupt the feature math, because the role uses
set operations (`difference`, `intersect`) that iterate a string character by
character.

    authselect_unwanted_features: []

The list of `authselect` features the role will ensure are **disabled** on the
managed profile. Anything that appears here is removed if present, and is also
excluded from the enable set if it somehow appears in
`authselect_desired_features` (unwanted wins over desired). Like the desired
list, this must always be a list, even when empty.

## Dependencies

None. This role installs the packages it needs (`authselect`, `sssd`) and does
not depend on other roles. It is intended to be consumed as one stage of a
larger STIG-application playbook, alongside separate roles that handle the
out-of-scope settings listed above.

## Example Playbook

    - hosts: rhel_servers
      become: true
      roles:
        - role: authselect_stig_rhel
          vars:
            authselect_unwanted_features:
              - with-smartcard-lock-on-removal

The example disables `with-smartcard-lock-on-removal` (for instance, on a
headless server where smartcard removal locking is irrelevant) while leaving
the rest of the defaults in place.

## How the role works

`authselect` has a non-obvious lifecycle, and this role handles four distinct
starting states. Understanding the branch logic is worth the few minutes it
takes, because the behavior around an existing profile is intentional and not
obvious from a quick read.

After gathering facts, asserting the OS, and installing packages, the role
determines the full profile name (`custom/<name>`), lists the available
profiles, and reads the currently active profile via `authselect current`. It
then routes to exactly one of the following paths:

**1. No custom profile exists.** The role creates the profile from the `sssd`
base (`create.yml`), which also computes the full set of features to enable
(desired minus unwanted). It then activates the new profile (`activate.yml`). A
brand-new profile is inherently valid, so it skips straight to activation with
no validation step.

**2. The profile exists and is already the active profile.** The role runs
`update.yml`, which validates the live profile with `authselect check`. If the
check fails (the generated files were edited by hand, for example), the profile
is deleted and recreated from base. Otherwise the role reads the profile's
**current** feature set and reconciles it against your declared intent: it
enables any desired feature that is missing, and disables any unwanted feature
that is present.

**3. The profile exists but is not active** (a different profile is selected,
or none is). The role selects the profile first, which makes it the active
profile, and **then** runs the same reconciliation as path 2. This ordering
matters: `authselect current` only ever reports the *active* profile's
features, so an inactive profile's feature set cannot be read until after it is
activated. Activating first is what makes the reconcile possible.

**4. A freshly imaged or unconfigured system.** If `authselect current` returns
nonzero or its output does not match the expected format, the parsed current
profile name resolves to an empty string rather than failing, and the role
treats the profile as not-active (path 1 or 3 as appropriate).

The reconcile in paths 2 and 3 is a **full** reconcile, not an additive one. If
you have enabled or disabled features out of band (for example, while testing),
the role reverts them to match `authselect_desired_features` and
`authselect_unwanted_features` on the next run. The role is treated as the
source of truth: if a change matters, it belongs in the role's variables, not
applied manually to a host.

State is classified once, before any activation, with a single
`__profile_already_active` fact. This keeps the active-profile and
inactive-profile reconcile paths mutually exclusive, so `update.yml` runs at
most once per play regardless of which path is taken.

### PAM template files

This role manages only the files it actually changes. Every file it does ship
is the stock vendor `sssd` profile template with nothing more than the minimal
STIG-required edits applied on top; any file the role does not ship is left
entirely to the `authselect` `sssd` profile's own defaults. In other words, the
role is a thin set of deltas over the vendor baseline, not a wholesale
replacement of the PAM stacks.

The PAM stack files that are managed are shipped as static, version-specific
copies under `files/`, selected by major version at copy time
(`system-auth.el8`, `system-auth.el9`, and so on). They are deployed with the
`copy` module, not `template`, because they contain no Ansible/Jinja
interpolation. A PAM file is deployed only if a copy exists for the running
major version, so a file is managed on a given release purely by virtue of
shipping an `.elN` variant for it.

`postlogin`, `nsswitch.conf`, `user-nsswitch.conf`, and `fingerprint-auth` are
intentionally not managed, because no in-scope STIG requires changes to them;
they remain whatever the `sssd` profile generates by default.

## Scope and STIG coverage

This role is based on the following DISA STIG benchmarks:

| OS      | Benchmark           | Release | Benchmark date |
| ------- | ------------------- | ------- | -------------- |
| RHEL 8  | U_RHEL_8_STIG_V2R7  | R7      | 01 Apr 2026    |
| RHEL 9  | U_RHEL_9_STIG_V2R8  | R8      | 01 Apr 2026    |
| RHEL 10 | U_RHEL_10_STIG_V1R1 | R1      | 26 Feb 2026    |

The findings this role applies are the subset of each benchmark that is
satisfied through `authselect` features or through the PAM template files. This
includes the `pam_faillock`, `pam_pwquality`, and `pam_unix` (sha512, and on
RHEL 9 the `rounds=100000` requirement) settings in `system-auth` and
`password-auth`, the smartcard PAM configuration on RHEL 8, and the
`without-nullok` and smartcard features.

Requirements that are part of the same benchmarks but are **out of scope** for
this role, because DISA's own fix text places them in other files, include:
faillock parameter values (`/etc/security/faillock.conf`), all password
complexity values (`/etc/security/pwquality.conf`), password history values
(`/etc/security/pwhistory.conf`), smartcard certificate auth
(`/etc/sssd/sssd.conf`), SELinux faillock contexts, audit rules, `dconf`
policy, and the `su`/`sudo` PAM files. These must be handled by other roles in
your STIG pipeline.

## To do

- **Add RHEL 10 support.** The RHEL 10 V1R1 benchmark has been analyzed and the
  in-scope requirements are known: they match RHEL 9 at the requirement level,
  including `rounds=100000` on the `pam_unix` lines in `system-auth` and
  `password-auth`, with no PAM-stack smartcard requirement (smartcard auth
  moves to `sssd.conf`). To finish RHEL 10 support: widen the OS assert in
  `tasks/main.yml` to include `'10'`, update the `fail_msg`, and add
  `files/system-auth.el10` and `files/password-auth.el10` (identical to the
  `.el9` files; no `smartcard-auth.el10` is needed). This is intentionally
  deferred until RHEL 10 is rolled out as a managed platform.

## License

MIT

## Author Information

Created by themoosefulcorp in personal time, with personal resources.
Contributions and forks welcome, updates and maintenance not guaranteed.