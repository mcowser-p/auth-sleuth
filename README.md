# auth-sleuth 🕵️

SSSD and login-chain troubleshooting as one Ansible role. When a user
can't log in, the cause hides somewhere along
**DNS → domain controllers → time → realm/keytab → sssd → nsswitch →
identity → PAM** — auth-sleuth walks the whole chain, prints a report,
and (only when you explicitly ask) fixes what it found.

```bash
ansible-galaxy collection install mcowser_p.auth_sleuth
```

```yaml
- hosts: web01
  become: true
  roles:
    - role: mcowser_p.auth_sleuth.auth_sleuth
      vars:
        auth_sleuth_domain: ads.example.com
        auth_sleuth_test_user: svc.probe
        auth_sleuth_expected_groups: [web01-app_restricted]
```

## The check catalog

| Layer | Checks |
| --- | --- |
| network | domain A record, `_ldap`/`_kerberos`/`_gc` SRV records, **per-DC per-port reachability** (88, 389, 445, 464, 636, **3268/3269 Global Catalog** — a blocked GC = "user resolves but groups are incomplete"), NTP sync, clock skew vs the Kerberos tolerance |
| join | realm membership matches the expected domain, host keytab present, opt-in live `kinit -k` |
| sssd | service active+enabled, sssd.conf perms/domain stanza, `simple_allow_groups` covers the expected groups (the `realm permit` state), `sssctl domain-status` online, recent log errors, `sss` in nsswitch |
| identity | live `getent` of every expected group and the test user, membership verification |
| pam | `pam_group.so` hook in pam.d/sshd, expected `group.conf` entries (declarative_access line format), local groups exist, **authselect-drift correlation** (entries present + hook gone = the known authselect apply-changes caveat), opt-in `pamtester` account probe, sshd up |

Every check lands in `auth_sleuth_result.checks` as
`{id, layer, status: ok|warn|fail|skipped, detail, remediation}`.
Checks the role can't perform degrade to `skipped` with a hint — never
to silent success. `auth_sleuth_fail_on: fail|warn|never` decides
whether findings fail the play; `auth_sleuth_report_path` writes the
full YAML report.

## Remediation: explicit, gated, verified

```yaml
auth_sleuth_remediate: [fix_sssd_conf_perms, reinsert_pam_group, clear_cache]
```

There is no `all`. A fix runs **only** if you listed it **and** its
check actually failed; afterwards the whole check suite re-runs so the
report shows the post-fix state. Available ids: `clear_cache`,
`restart_sssd`, `enable_sssd`, `restart_sshd`, `fix_sssd_conf_perms`,
`reinsert_pam_group`, `add_group_conf_entries`, `fix_nsswitch`,
`sync_time`, `terminate_user_sessions`. Credential-requiring actions
(realm join, creating AD groups) are never automated — the report gives
the manual command instead.

## Configurable for any organization

Domain, DCs, ports, expected groups, pam_group entry shapes, and every
file path are variables with neutral defaults — see
[examples/org-config.yml](examples/org-config.yml) for a filled-in
profile matching the
[declarative_access](https://github.com/mcowser-p/ansible-declarative-access)
conventions (local-principal mode included: leave
`auth_sleuth_domain` empty and everything domain-dependent skips
cleanly).

## License

Apache-2.0
