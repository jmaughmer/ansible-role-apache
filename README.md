# ansible-role-apache

Ansible role that installs and hardens Apache httpd on RHEL 8, 9 and 10 (and compatible: AlmaLinux, Rocky, Oracle Linux) from the AppStream repository, with optional PHP-FPM.

Features:

- httpd and mod_ssl from AppStream, with a default TLS virtual host, plain HTTP redirected to HTTPS, and extra name-based TLS virtual hosts.
- TLS with a supplied certificate or a self-signed CA and server certificate the role creates. TLS 1.2+ only, with ciphers following the system crypto policy (`PROFILE=SYSTEM`).
- Hardening based on the DISA Apache Server 2.4 UNIX STIG: `ServerTokens Prod`, no signature, no TRACE, no ETags, request size and time limits, unneeded modules off (autoindex, userdir, info, status, DAV, CGI, Lua), the welcome page removed, security headers, HSTS, `HttpOnly`/`Secure` cookies, and no access to dot files, backups or VCS directories.
- Optional PHP-FPM from AppStream (PHP 8.2 module stream on EL8/EL9), with hardened `php.ini` settings.
- A place for other roles to plug in: configuration snippets, an include directory inside the default TLS vhost, and handlers they can notify.
- SELinux port and file labeling and booleans, a fapolicyd trust file for web content deployed outside RPMs, and FIPS checks.
- Molecule tests with podman on EL8, EL9 and EL10.

Used by [ansible-role-phpmyadmin](https://github.com/jmaughmer/ansible-role-phpmyadmin).

## Requirements

- Ansible core 2.15 or newer. RHEL 8 targets need ansible-core 2.16 or older on the controller, because 2.17+ no longer supports EL8's Python 3.6.
- Collections from `requirements.yml`: `ansible.posix`, `community.general` and `community.crypto`.

## Quick start

```yaml
- hosts: web_servers
  become: true
  roles:
    - role: ansible-role-apache
      vars:
        apache_server_name: web01.corp.example.com
        apache_php_enabled: true
        apache_php_packages: [php-mysqlnd, php-mbstring]
        apache_selinux_booleans:
          httpd_can_network_connect_db: true
```

## TLS

```yaml
# Self-signed (default): a private CA plus a server certificate, RSA 3072 / SHA-256.
# SANs cover apache_server_name, the short hostname, localhost and every apache_vhosts name.
apache_tls_source: selfsigned

# Supplied: PEM content (from Vault) or files on the controller. Put the
# intermediate chain in the certificate file after the server certificate.
apache_tls_source: provided
apache_tls_cert_content: "{{ vault_web_cert }}"
apache_tls_key_content: "{{ vault_web_key }}"
```

Files go to `/etc/pki/httpd` (`server.crt`, `server.key` mode 0600, `ca.crt`). The role writes its own `/etc/httpd/conf.d/ssl.conf` in place of the one from mod_ssl; it is a `noreplace` config file, so package updates leave it alone.

## Adding applications

The default TLS virtual host serves `apache_document_root` for `apache_server_name`. Applications can attach to it in three ways:

1. **Server-wide snippets:** `apache_conf_snippets: [{ name: myapp, content: "Alias /myapp /usr/share/myapp ..." }]` writes `/etc/httpd/conf.d/myapp.conf`. Port 80 only redirects, so aliases are only reachable over HTTPS.
2. **Inside the default TLS vhost:** drop a `.conf` file into `/etc/httpd/conf.d/default-ssl.d/`.
3. **Their own vhost:** add an entry to `apache_vhosts` (`server_name`, `server_aliases`, `document_root`, `tls_cert_path`, `tls_key_path`, `extra`).

Another role in the same play can write its own files and `notify: reload httpd` (or `restart httpd`, `restart php-fpm`). The role runs `httpd -t` before starting or reloading.

The default vhost is the first `*:443` vhost Apache reads. A file in `conf.d` that sorts before `ssl.conf` and defines its own `*:443` vhost will take that place.

## Hardening

`apache_stig_enabled: true` (the default) applies request limits and cookie flags. The other settings in the table have their own variables:

| Setting | Variable | Default |
|---|---|---|
| `ServerTokens` / `ServerSignature` | `apache_server_tokens` / `apache_server_signature` | `Prod` / `Off` |
| `TraceEnable` | `apache_trace_enable` | `Off` |
| `FileETag` | `apache_file_etag` | `None` |
| Timeouts | `apache_timeout`, `apache_keepalive_timeout`, `apache_request_read_timeout` | 60 s, 5 s, mod_reqtimeout limits |
| Request limits | `apache_limit_request_*` | 10 MB body, 100 fields, 8190 bytes |
| Disabled modules | `apache_disabled_modules` | autoindex, userdir, info, status, dav*, cgi*, lua |
| Blanked package confs | `apache_blank_confs` | welcome, autoindex, userdir |
| Security headers | `apache_security_headers` | nosniff, SAMEORIGIN, referrer and permissions policies |
| HSTS | `apache_hsts_max_age` | 1 year |
| TLS protocols | `apache_tls_protocol` | `all -SSLv3 -TLSv1 -TLSv1.1` |
| Cookie flags | `apache_cookie_flags` | adds `HttpOnly` and `Secure` |

The document root gets `Options None` and `AllowOverride None`. Logs use a `combined_tls` format that adds the request time and the TLS protocol and cipher.

## PHP

`apache_php_enabled: true` installs `php-fpm`, `php-cli` and `php-common` plus `apache_php_packages`, from the `php:8.2` module stream on EL8/EL9 (`apache_php_module_stream`) or the stock PHP on EL10. The package's `/etc/httpd/conf.d/php.conf` hands `.php` files to the `www` pool. `/etc/php.d/99-hardening.ini` turns off `expose_php`, error display and URL includes, and sets strict, secure session cookies. Add your own settings with `apache_php_ini`.

## SELinux

With SELinux enforcing, the role labels non-standard listener ports as `http_port_t`, labels document roots outside `/var/www` as `httpd_sys_content_t`, and runs `restorecon`. Set booleans with `apache_selinux_booleans` (for example `httpd_can_network_connect_db` when PHP talks to a database over TCP), and add writable paths with `apache_selinux_fcontexts` (`httpd_sys_rw_content_t`).

## fapolicyd

fapolicyd trusts RPM-installed files, and its default language rules deny interpreters (php-fpm included) from opening untrusted script files. That means a PHP application deployed from a tarball or git checkout will not run until it is trusted. List its directories in `apache_fapolicyd_trusted_dirs` (or files in `apache_fapolicyd_trusted_files`). The role writes each file's path, size and SHA-256 to `/etc/fapolicyd/trust.d/apache` and refreshes fapolicyd. Re-run the role after updating the application.

## FIPS

TLS follows the system crypto policy, and self-signed certificates use RSA 3072 and SHA-256. `apache_fips_required: true` fails the run when the host is not in FIPS mode, and `apache_crypto_policy` sets the system policy.

## Variables

See [`defaults/main.yml`](defaults/main.yml) for every variable with comments.

Tags: `apache_install`, `apache_tls`, `apache_config`, `apache_php`, `apache_security`, `apache_selinux`, `apache_fapolicyd`, `apache_fips`, `apache_firewall`, `apache_service`.

## Testing

Tests run locally with podman; there is no CI. On the AlmaLinux 10 WSL distribution (or any EL10 host):

```bash
scripts/test.sh            # lint, default (EL9 + EL10), el8
```

The repository directory must be named `ansible-role-apache`. SELinux, FIPS and firewalld can't be exercised in containers.

### One-time AlmaLinux 10 WSL setup

1. Turn on systemd in the distro. Put this in `/etc/wsl.conf`, run `wsl --shutdown` from Windows, then reopen the distro:

   ```ini
   [boot]
   systemd=true
   ```

2. Install the tools: `sudo dnf install -y podman python3 python3-pip git`.
3. Set up rootless podman for your user:

   ```bash
   grep -q "^$USER:" /etc/subuid || sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 "$USER"
   sudo loginctl enable-linger "$USER"
   podman system migrate
   podman info          # should succeed and show cgroupVersion: v2
   ```

4. Clone the repositories side by side under your Linux home directory (for example `~/src/`), not under `/mnt/c`. The Windows mount is slow and does not keep Linux file permissions.

`scripts/test.sh` checks for systemd, cgroup v2, subordinate IDs and a user session before it starts, and prints the fix for anything missing. If rootless systemd containers still won't start, run `sudo scripts/test.sh` to use rootful podman.

## License

MIT
