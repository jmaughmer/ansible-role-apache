# ansible-role-apache

Ansible role that installs and hardens Apache httpd on RHEL 8, 9 and 10 (and compatible: AlmaLinux, Rocky, Oracle Linux) from the AppStream repository, with optional PHP-FPM.

Features:

- httpd and mod_ssl from AppStream, with a default TLS virtual host, plain HTTP redirected to HTTPS, and extra name-based TLS virtual hosts.
- TLS from one of three sources: a self-signed CA and server certificate the role creates, certificates you supply with their keys in Ansible Vault, or Let's Encrypt through certbot. TLS 1.2+ only, with ciphers following the system crypto policy (`PROFILE=SYSTEM`).
- Hardening based on the DISA Apache Server 2.4 UNIX STIG: `ServerTokens Prod`, no signature, no TRACE, no ETags, request size and time limits, unneeded modules off (autoindex, userdir, info, status, DAV, CGI, Lua), the welcome page removed, security headers, HSTS, `HttpOnly`/`Secure` cookies, and no access to dot files, backups or VCS directories.
- Optional PHP-FPM from AppStream (PHP 8.2 module stream on EL8/EL9), with hardened `php.ini` settings.
- A place for other roles to plug in: configuration snippets, an include directory inside the default TLS vhost, and handlers they can notify.
- SELinux port and file labeling and booleans, a fapolicyd trust file for web content deployed outside RPMs, and FIPS checks.
- Molecule tests with podman on EL8, EL9 and EL10.

Used by [ansible-role-phpmyadmin](https://github.com/jmaughmer/ansible-role-phpmyadmin).

## Requirements

- Ansible core 2.15 or newer. RHEL 8 targets need ansible-core 2.16 or older on the controller, because 2.17+ no longer supports EL8's Python 3.6.
- Collections from `requirements.yml`: `ansible.posix`, `community.general` and `community.crypto`.
- Packages come from AppStream, except certbot, which comes from EPEL. The role only enables EPEL with `apache_tls_source: letsencrypt` (`apache_epel_manage: false` if you already have EPEL or a mirror of it). RHEL hosts may also need the CodeReady Builder repository for certbot's dependencies.

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

`apache_tls_source` picks where certificates come from: `selfsigned` (the default), `provided` or `letsencrypt`.

Whichever you pick, the default virtual host uses `apache_tls_cert_path` and `apache_tls_key_path` (`/etc/pki/httpd/server.crt` and `server.key`), so other roles can rely on those paths. With `provided` and `letsencrypt` they are links to the real files.

The role writes its own `/etc/httpd/conf.d/ssl.conf` in place of the one from mod_ssl; it is a `noreplace` config file, so package updates leave it alone.

### Self-signed

```yaml
# A private CA plus a server certificate, RSA 3072 / SHA-256. SANs cover
# apache_server_name, the short hostname, localhost and every apache_vhosts name.
apache_tls_source: selfsigned
```

Files go to `/etc/pki/httpd` (`server.crt`, `server.key` mode 0600, `ca.crt`, `ca.key` mode 0600). Browsers and clients need to trust `ca.crt`.

### Supplied certificates

```yaml
apache_tls_source: provided
ssl_certificates:
  - name: "{{ ansible_fqdn }}"
    certificate:
      content: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
    key:
      content: !vault |
        $ANSIBLE_VAULT;1.1;AES256
        ...
```

`name` is also the file name: each entry becomes `/etc/pki/httpd/<name>.crt` (mode 0644) and `<name>.key` (mode 0600). Put any intermediate certificates in `certificate.content`, after the server certificate. Recent ansible-core versions print a deprecation warning for `ansible_fqdn`; `ansible_facts['fqdn']` gives the same name without it.

Encrypt each key with Ansible Vault. This prints the `content: !vault |` block to paste under `key:`:

```bash
ansible-vault encrypt_string --vault-password-file ~/.vault-pass --stdin-name content < server.key
```

A vault-encrypted vars file works too, with `content: "{{ vault_web_key }}"`. Either way, the role keeps keys out of its output, including with `-vvv` and `--diff`.

Before it writes anything, the role checks every entry: the certificate and key are valid PEM, the key has no passphrase (httpd can't prompt for one), the key belongs to the certificate, and the certificate hasn't expired. If an entry fails, the run stops with the existing files and the running httpd untouched.

Each virtual host uses the first of these that applies:

| Virtual host | Certificate |
|---|---|
| Default | `apache_tls_default_certificate`, then the entry named `apache_server_name`, then the first entry |
| `apache_vhosts` entry | `tls_cert_path` and `tls_key_path`, then `ssl_certificate: <name>`, then the entry named after its `server_name`, then the default certificate |

Removing an entry doesn't delete its files.

### Let's Encrypt

```yaml
apache_tls_source: letsencrypt
apache_letsencrypt_terms_agreed: true   # accepts the CA's subscriber agreement
apache_letsencrypt_email: netops@example.com
# One certificate for all of these. Defaults to apache_server_name plus every apache_vhosts name.
apache_letsencrypt_domains: [ipam.example.com, ipam.example.net]
```

certbot gets the certificate with the HTTP-01 challenge, using `apache_letsencrypt_webroot` (`/var/www/letsencrypt`), which every virtual host serves at `/.well-known/acme-challenge/`. `certbot-renew.timer` renews it, and httpd reloads after each renewal. RSA 3072 is the default; for ECDSA, set `apache_letsencrypt_key_type: ecdsa`.

- **Every name needs public DNS, and port 80 must be reachable from the internet.** Internal-only hosts can't pass the challenge, so use `provided` for those. Wildcards aren't possible with this challenge.
- **The first run starts on a self-signed certificate.** httpd needs a certificate to start and has to be running to get one. Once httpd is up, the role requests the real certificate and switches to it. If the request fails, the run fails and httpd stays up on the self-signed certificate.
- **Try it on staging first.** Let's Encrypt has rate limits, so set `apache_letsencrypt_server: https://acme-staging-v02.api.letsencrypt.org/directory` while testing. Staging certificates aren't trusted by browsers.
- **Let's Encrypt isn't an approved DoD PKI**, so sites bound by the STIG should use `provided`.

Extra arguments for `certbot certonly` go in `apache_letsencrypt_extra_args`.

### Changing source

Change `apache_tls_source` and re-run. The links at `apache_tls_cert_path` and `apache_tls_key_path` are replaced. Supplied files and Let's Encrypt files stay on disk, so you can switch back, and so does the self-signed `ca.key`; delete them yourself if you no longer want them on the host.

`ssl_certificates` replaces `apache_tls_cert_content`, `apache_tls_key_content`, `apache_tls_ca_content` and the `apache_tls_*_src` variables, which no longer exist. A run with `apache_tls_source: provided` and no `ssl_certificates` fails and says so.

## Adding applications

The default TLS virtual host serves `apache_document_root` for every name no other vhost claims. That includes `apache_server_name`, unless a vhost uses that name. Applications can attach to it in three ways:

1. **Server-wide snippets:** `apache_conf_snippets: [{ name: myapp, content: "Alias /myapp /usr/share/myapp ..." }]` writes `/etc/httpd/conf.d/myapp.conf`. Port 80 only redirects, so aliases are only reachable over HTTPS.
2. **Inside the default TLS vhost:** drop a `.conf` file into `/etc/httpd/conf.d/default-ssl.d/`.
3. **Their own vhost:** add an entry to `apache_vhosts` (`server_name`, `server_aliases`, `document_root`, `tls_cert_path`, `tls_key_path`, `ssl_certificate`, `extra`).

Another role in the same play can write its own files and `notify: reload httpd` (or `restart httpd`, `restart php-fpm`). The role runs `httpd -t` before starting or reloading.

The default vhost is the first `*:443` vhost Apache reads. A file in `conf.d` that sorts before `ssl.conf` and defines its own `*:443` vhost will take that place.

Its `ServerName` is `default.invalid`, a reserved name, so other vhosts can use any real name, including `apache_server_name`. mod_ssl logs an `AH01909` warning about this at startup because the certificate can't list that name. You can ignore it.

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

With SELinux enforcing, the role labels non-standard listener ports as `http_port_t`, labels document roots and the Let's Encrypt webroot outside `/var/www` as `httpd_sys_content_t`, and runs `restorecon`. Set booleans with `apache_selinux_booleans` (for example `httpd_can_network_connect_db` when PHP talks to a database over TCP), and add writable paths with `apache_selinux_fcontexts` (`httpd_sys_rw_content_t`).

## fapolicyd

fapolicyd trusts RPM-installed files, and its default language rules deny interpreters (php-fpm included) from opening untrusted script files. That means a PHP application deployed from a tarball or git checkout will not run until it is trusted. List its directories in `apache_fapolicyd_trusted_dirs` (or files in `apache_fapolicyd_trusted_files`). The role writes each file's path, size and SHA-256 to `/etc/fapolicyd/trust.d/apache` and refreshes fapolicyd. Re-run the role after updating the application.

## FIPS

TLS follows the system crypto policy, and self-signed certificates use RSA 3072 and SHA-256. `apache_fips_required: true` fails the run when the host is not in FIPS mode, and `apache_crypto_policy` sets the system policy. certbot hasn't been tested in FIPS mode. Supplied keys and certificates must use FIPS-approved algorithms, which the role doesn't check.

## Variables

See [`defaults/main.yml`](defaults/main.yml) for every variable with comments.

Tags: `apache_install`, `apache_tls`, `apache_letsencrypt`, `apache_config`, `apache_php`, `apache_security`, `apache_selinux`, `apache_fapolicyd`, `apache_fips`, `apache_firewall`, `apache_service`.

## Testing

Tests run locally with podman; there is no CI. On the AlmaLinux 10 WSL distribution (or any EL10 host):

```bash
scripts/test.sh            # everything below
scripts/test.sh lint default el8                   # lint, then self-signed on EL9 + EL10 and on EL8
scripts/test.sh provided provided-el8              # supplied certificates
scripts/test.sh letsencrypt letsencrypt-el8        # Let's Encrypt, against Pebble
```

The repository directory must be named `ansible-role-apache`. SELinux, FIPS and firewalld can't be exercised in containers.

The `provided` scenarios use throwaway certificates, with test keys vaulted using the password in `molecule/provided/vault-password`. The script fails a run if a private key shows up in its output. The `letsencrypt` scenarios download [Pebble](https://github.com/letsencrypt/pebble), Let's Encrypt's test CA, and run it inside the container, so they never contact Let's Encrypt.

## License

MIT
