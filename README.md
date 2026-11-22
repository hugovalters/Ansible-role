# Ansible Role: ValtersIT Scanner

Deploy and run ValtersIT CVE scanners across your fleet with Ansible —
Linux packages, Docker, WordPress and every language ecosystem — and
collect the results on the controller. Open source, auditable,
privacy-first — powered by the ValtersIT CVE API.

> Part of the ValtersIT platform — CVE & Threat-IP data you can query or run yourself.
> Explore both API and scanners: https://www.valtersit.com/platform/
> Scanner plans & pricing: https://www.valtersit.com/cve/pricing/#scanners

## What it does

On each managed host, the role:

1. checks that Python 3.7+ is available;
2. downloads one ValtersIT scanner (two Python files, standard library
   only) into `/opt/valtersit/<scanner>`, at the ref you pin;
3. installs your authorised-use confirmation and the API key into
   `/var/lib/valtersit` (directory `0700`, files `0600`, never logged);
4. optionally installs a daily cron job — the API key is read from the
   `0600` env file at run time, so it never appears in the crontab;
5. runs a scan, prints the tail of its output, saves it on the host and
   fetches it to `valtersit-results/<host>-<scanner>.txt` on the
   controller.

Supported scanners: `linux-cve-scanner`, `docker-cve-scanner`,
`wordpress-cve-scanner`, `python-cve-scanner`, `nodejs-cve-scanner`,
`npm-audit-scanner`, `ruby-cve-scanner`, `dotnet-cve-scanner`,
`laravel-cve-scanner`, `magento-cve-scanner`, `kubernetes-cve-scanner`.

## Your data stays yours

- **What we receive:** component name and version only (e.g. `openssl
  3.0.13-0ubuntu3.4`), plus an opaque system identifier per host — a
  one-way hash of local machine signals (machine-id, disk UUID, CPU count)
  that cannot be reversed to any of them.
- **What we never receive:** your hosts' IP addresses, hostnames, MAC
  addresses, Ansible inventory, variables, credentials, or the contents of
  any file. Nothing outside of `name` + `version` (+ that opaque identifier)
  ever leaves the scanner.
- **What the role does not do:** it installs no packages, opens no ports,
  starts no daemons and changes nothing outside `/opt/valtersit`,
  `/var/lib/valtersit`, the optional crontab entry and its log file. Every
  task that touches the API key or consent record uses `no_log`.
- **Audit it yourself.** The role is ~200 lines of YAML; each scanner it
  downloads is two Python files you can read in its own repository.

## Requirements

- Ansible 2.14+ (`ansible-core`) on the controller.
- Python 3.7+ on the managed hosts (already present on every mainstream
  Linux distribution, and needed by Ansible itself).
- Outbound HTTPS from the hosts to `gitlab.com` (download) and
  `api.valtersit.com` (scan).

## Install

```bash
# requirements.yml
roles:
  - name: valtersit.valtersit_scanner
    src: https://gitlab.com/valtersit/ansible-role.git
    scm: git
    version: main        # pin a commit SHA
```

```bash
ansible-galaxy role install -r requirements.yml
```

## Set-up

### 1. Confirm authorised use once, locally

Scanners refuse to run non-interactively until a person has read the
authorised-use notice and typed `YES`. Do it once, on any machine, for each
scanner you will deploy:

```bash
git clone https://gitlab.com/valtersit/linux-cve-scanner /tmp/lcs
python3 /tmp/lcs/valtersit-linux-scan.py      # read the notice, type YES
cat ~/.valtersit/consent.json
```

### 2. Put the secrets in Ansible Vault

```bash
ansible-vault create group_vars/all/valtersit.yml
```

```yaml
valtersit_api_key: "vit_v1_your_key_here"
valtersit_consent_json: |
  {"linux-package-scanner": {"confirmed": true, "confirmed_at": "2026-11-18T18:00:00Z",
   "attestation_version": "v2", "source": "scanner-cli", "client_version": "1.2.0"}}
```

(Paste the real contents of your `consent.json`.)

## Example playbooks

Every server, OS packages, daily cron:

```yaml
- hosts: all
  become: true
  roles:
    - role: valtersit.valtersit_scanner
      vars:
        valtersit_scanner: linux-cve-scanner
        valtersit_scanner_ref: 5355e03
        valtersit_cron_enabled: true
```

Web servers, WordPress site and the Docker images next to it:

```yaml
- hosts: webservers
  become: true
  tasks:
    - ansible.builtin.include_role:
        name: valtersit.valtersit_scanner
      vars:
        valtersit_scanner: wordpress-cve-scanner
        valtersit_scan_args: ["--path", "/var/www/html"]
        valtersit_cron_enabled: true

    - ansible.builtin.include_role:
        name: valtersit.valtersit_scanner
      vars:
        valtersit_scanner: docker-cve-scanner
```

Deploy the cron job only, without scanning during the play:

```yaml
      vars:
        valtersit_run_scan: false
        valtersit_cron_enabled: true
```

## Role variables

| Variable | Default | Meaning |
|---|---|---|
| `valtersit_scanner` | `linux-cve-scanner` | Scanner to deploy and run |
| `valtersit_scan_args` | `[]` | Extra scanner arguments, as a list |
| `valtersit_scanner_ref` | `main` | Scanner branch/tag/commit — pin a SHA in production |
| `valtersit_api_key` | `""` | ValtersIT API key — **Vault** |
| `valtersit_consent_json` | `""` | Your `consent.json` contents — **Vault** |
| `valtersit_system_token` | `""` | Stable identifier for hosts without `/etc/machine-id` (`openssl rand -hex 16`) |
| `valtersit_install_dir` | `/opt/valtersit/<scanner>` | Where the scanner files go |
| `valtersit_state_dir` | `/var/lib/valtersit` | Consent, env file, last output |
| `valtersit_python` | `python3` | Interpreter on the host |
| `valtersit_run_scan` | `true` | Run a scan during the play |
| `valtersit_fetch_results` | `true` | Fetch the output to the controller |
| `valtersit_results_dir` | `{{ playbook_dir }}/valtersit-results` | Where fetched output lands |
| `valtersit_cron_enabled` | `false` | Install (`true`) or remove (`false`) the daily cron job |
| `valtersit_cron_hour` / `_minute` | `3` / `30` | Cron schedule |
| `valtersit_cron_user` | `root` | Crontab owner |
| `valtersit_log_file` | `/var/log/valtersit-<scanner>.log` | Cron output log |

## Sample run

```
TASK [valtersit.valtersit_scanner : Run the scan] ******************************
ok: [web-01]

TASK [valtersit.valtersit_scanner : Show the scan result] **********************
ok: [web-01] => {
    "msg": [
        " 1203 item(s) detected. No IP address, hostname, MAC address, file",
        " contents, or credentials were read or transmitted by this scan.",
        "[+] Scan submitted. 1000 component(s) sent for matching.",
        "{",
        "  \"matched_cves\": 3,",
        ...
    ]
}

TASK [valtersit.valtersit_scanner : Fetch the scan output to the controller] ***
changed: [web-01]
```

The play fails on a host whose scan did not complete (e.g. an invalid API
key or an unreachable API), after the output has been saved and fetched —
so `--limit @retry` style re-runs only touch the hosts that need it.

## Authorised use only

Run these scanners only on hosts you own, or for which you have explicit
written authorisation to assess. `valtersit_consent_json` carries the
confirmation you personally gave after reading this notice:

```
------------------------------------------------------------------------------
 ValtersIT Scanner — Authorised Use Confirmation
------------------------------------------------------------------------------
 Run this tool ONLY against systems you own, or for which you have explicit
 written authorisation to assess.

 This scan sends only component name + version, plus an opaque system
 identifier (a one-way hash of local machine signals such as machine-id
 and disk UUID, so repeat scans are recognised as the same system), to
 the ValtersIT API — no IP address, hostname, MAC address, or file
 contents. See the Privacy Policy: https://www.valtersit.com/privacy

 Type YES to confirm you are authorised to scan this system and continue.
 Anything else aborts without scanning.
------------------------------------------------------------------------------
```

Licensed under Apache-2.0 (see `LICENSE` and `NOTICE`); provided without
warranty of any kind. "ValtersIT" and the ValtersIT logo are trademarks of
Valters Capital, SIA and are not licensed for use. Ansible is a trademark
of Red Hat, Inc.; this project is not affiliated with or endorsed by it.

## Questions or a bug?

Please **don't** open GitHub/GitLab issues — I don't monitor them.
Reach me through the contact form instead: https://www.valtersit.com/contacts/

Maintained by Hugo Valters
