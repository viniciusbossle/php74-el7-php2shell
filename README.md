# php74-el7-php2shell

Unofficial rebuild of the **PHP 7.4** packages for **Enterprise Linux 7** (CentOS 7 / RHEL 7 /
AlmaLinux 7 / Rocky-compatible 7) that carries the fix for the **php2shell** vulnerability
(**CVE-2026-109618** and **CVE-2026-109619**), disclosed on 2026-10-21.

I still run a few legacy EL7 boxes on PHP 7.4 from [Remi Collet's repository](https://rpms.remirepo.net/).
CentOS 7 reached end-of-life in June 2024 and the EL7 `php74` channel is frozen at
`7.4.33-15.el7.remi` (June 2024). When php2shell dropped, the official fix landed in the maintained
PHP branches (8.5.12 / 8.4.27 / 8.3.36 / 8.2.35) and Remi shipped it for the branches he still builds —
but **there is no upstream build for PHP 7.4 on EL7**. So I backported it by rebuilding Remi's current
`php74` spec for EL7, and I'm sharing the result in case you're stuck on the same dead stack and need
breathing room while you plan a real upgrade.

---

## ⚠️ Read this first

- **This is a stopgap, not a fix.** PHP 7.4 and CentOS 7 are both end-of-life. The only real fix is
  to move off them. These packages buy you time; they are not a reason to stay.
- **Unofficial and unaffiliated.** This is a community rebuild. It is **not** endorsed by or affiliated
  with the PHP project, Remi Collet, Red Hat, or any distribution. Do not report bugs in these packages
  to any of them.
- **No warranty.** Provided as-is. You run them as root on your servers; verify before you trust (see
  [Verifying what you install](#verifying-what-you-install)). The full source (SRPM) and build recipe
  are published so you — or anyone — can rebuild and audit them.
- **Scope.** `x86_64` only. PHP `7.4.33`. EL7 only. If you are on PHP 8.x or EL8/EL9, use the official
  updates from your distro / Remi instead — you do not need this.

---

## What's in here

These RPMs rebuild Remi's **current** `php74` release (`7.4.33-31`) for EL7. Because the EL7 channel
was frozen at `-15` in June 2024, this is **more than just the php2shell patch** — it also includes
roughly two years of security backports that Remi applied to the branches he still builds but that were
never compiled for the frozen EL7 `php74` channel. The changelog in the packages lists every CVE
addressed between `-15` and `-31`.

The php2shell fix itself is Remi's "Minimal mitigation" patch for CVE-2026-109618 / CVE-2026-109619,
whose attack vector is PHP's `session.upload_progress` handling (CVSS v4 9.2). This README stays at the
advisory level and does not describe how to exploit it.

**35 packages**, same split as Remi's `php74`: `php`, `php-cli`, `php-common`, `php-fpm`, `php-gd`,
`php-intl`, `php-mbstring`, `php-mysqlnd`, `php-pdo`, `php-opcache`, `php-sodium`, `php-xml`, and the
rest of the extension set — plus `php-devel` and the `src.rpm`.

---

## Requirements

- An EL7 system (CentOS 7 / RHEL 7 / AlmaLinux 7 / etc.), `x86_64`.
- **Remi's repository already installed and reachable.** This overlay only ships the `php-*` packages;
  their runtime dependencies (`oniguruma5php`, `libicu73`, `gd3php`, `libsodium23`, …) still come from
  Remi. If you installed PHP 7.4 from Remi, you already have them.
- PHP 7.4 already installed from Remi (`7.4.33-15.el7.remi` is the expected baseline). This is an
  **update**, not a fresh PHP install.

> Since the stock Remi EL7 mirrorlists point at the archive now that EL7 is EOL, you may need to point
> Remi's `php74`/`remi`/`remi-safe` repos at the live tree
> (`https://rpms.remirepo.net/enterprise/7/<repo>/x86_64/`) or set `skip_if_unavailable=1` so yum can
> still resolve dependencies.

---

## Install

Pick whichever fits your setup. All three leave Remi in place as the dependency source.

### 1. As a yum repository (recommended)

Import the signing key and drop in the repo file, then update:

```bash
sudo rpm --import https://viniciusbossle.github.io/php74-el7-php2shell/RPM-GPG-KEY-php74-el7

sudo tee /etc/yum.repos.d/php74-el7-php2shell.repo >/dev/null <<'EOF'
[php74-el7-php2shell]
name=PHP 7.4 el7 php2shell rebuild
baseurl=https://viniciusbossle.github.io/php74-el7-php2shell/7/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://viniciusbossle.github.io/php74-el7-php2shell/RPM-GPG-KEY-php74-el7
EOF

sudo yum clean all
sudo yum update 'php*'
```

yum picks the highest version, so your installed `php-*` move from `7.4.33-15.el7.remi` up to the
rebuilt release automatically. Then restart whatever serves PHP:

```bash
sudo systemctl restart httpd      # mod_php
# or
sudo systemctl restart php-fpm
```

### 2. As a local mirror (air-gapped / fleet)

Download the RPMs from the [latest release](https://github.com/viniciusbossle/php74-el7-php2shell/releases/latest),
drop them in a directory, and build a local repo:

```bash
mkdir -p /srv/php74-el7-php2shell && cd /srv/php74-el7-php2shell
# copy the *.rpm here, then:
sudo yum install -y createrepo
createrepo .

sudo tee /etc/yum.repos.d/php74-local.repo >/dev/null <<'EOF'
[php74-local]
name=PHP 7.4 el7 php2shell (local)
baseurl=file:///srv/php74-el7-php2shell/
enabled=1
gpgcheck=0
EOF

sudo yum clean all && sudo yum update 'php*'
```

Serve that directory over HTTP from one host and point the rest of your fleet at it the same way.

### 3. Without any repo

Download the RPMs you need and let yum resolve dependencies against Remi:

```bash
# from a downloaded set:
sudo yum update ./php-*.rpm

# or straight from the hosted tree (only the packages you already have installed):
BASE=https://viniciusbossle.github.io/php74-el7-php2shell/7/x86_64
sudo yum update $BASE/php-common-7.4.33-31.el7.x86_64.rpm \
                $BASE/php-cli-7.4.33-31.el7.x86_64.rpm \
                $BASE/php-7.4.33-31.el7.x86_64.rpm
                # ... and the rest: rpm -qa 'php-*' tells you which
```

> **Do not** `yum downgrade 'php*'` with a wildcard if you have `php-pecl-*` extensions installed — the
> glob drags them in and yum may try to match them against the wrong PHP build. Downgrade the core
> packages explicitly (see [Rollback](#rollback)).

---

## Verifying what you install

Trust nothing by default. In order of effort:

1. **Checksums:** compare against [`SHA256SUMS`](./SHA256SUMS) in this repo.
   ```bash
   sha256sum -c SHA256SUMS
   ```
2. **Signature:** the packages are GPG-signed. Import the key and check — the fingerprint should be:

   ```
   2AD9 952F 6864 AC90 1C24  52E2 CEEB 3E17 0997 3444
   ```
   ```bash
   rpm --import https://viniciusbossle.github.io/php74-el7-php2shell/RPM-GPG-KEY-php74-el7
   rpm --checksig php-*.rpm        # should report: ... digests signatures OK
   ```
3. **Rebuild it yourself:** the `src.rpm` and the exact spec changes are published. Rebuilding from
   source and diffing against these binaries is the strongest check — see [BUILD.md](./BUILD.md).

---

## Configuration suggestions (defense in depth)

The patch closes the hole, but these settings reduce exposure to the whole class of
`session.upload_progress` abuse and are worth keeping regardless. They also matter if you **can't patch
right away**: applying the first one alone already neutralizes the known vector — that's what buys you
time to schedule the real upgrade instead of scrambling.

### PHP — disable the feature behind php2shell (the important one)

```ini
; /etc/php.d/99-harden.ini  (or your php.ini)
session.upload_progress.enabled = Off
```

If nothing in your app relies on upload-progress tracking (most apps don't), this directly removes the
attack surface.

### PHP — cap request/upload sizes

```ini
post_max_size = 16M
upload_max_filesize = 8M
```

Pick values your app actually needs. Smaller caps shrink what can be pushed through a single request.

### Web server — cap the request body at the edge

Apache (`mod_php`, or when proxying to php-fpm):

```apache
# /etc/httpd/conf.d/zz-harden.conf
LimitRequestBody 16777216   # 16 MB — mirror your post_max_size
```

nginx (in front of php-fpm):

```nginx
client_max_body_size 16m;
```

### Web server — block multipart uploads entirely (aggressive)

Only for a vhost/app that never handles file uploads. It's a blunt instrument — it breaks every
legitimate upload — so scope it to apps you're sure don't need it:

```apache
<If "%{CONTENT_TYPE} =~ m#multipart/#">
    Require all denied
</If>
```

Reload the web server after changing any of these. And keep it in perspective: **config hardening is a
seatbelt, not the repair. Patch to buy time, then upgrade.**

---

## Rollback

The rebuilt repo ships the original `7.4.33-15.el7.remi` packages too, so you can roll back entirely
within it. Downgrade the **core** packages explicitly — never with a `php*` wildcard if you have PECL
extensions installed:

```bash
sudo yum downgrade $(rpm -qa --qf '%{NAME}\n' | grep -E '^php(-|$)' | grep -v '^php-pecl')
sudo systemctl restart httpd   # or php-fpm
```

---

## Reproducing the build

Everything needed to rebuild these packages from Remi's spec is in [BUILD.md](./BUILD.md): the PHP
source tarball and its checksum, Remi's `php74` spec at release `-31`, and the three small EL7-specific
spec changes required to build with the older toolchain (GCC 4.8 / ICU 73) that ships with EL7.

---

## Credits

- **Remi Collet** — the entire `php74` packaging (spec, patches, and the php2shell mitigation patch)
  comes from his work at <https://rpms.remirepo.net/>. All I did was rebuild his current spec for the
  frozen EL7 channel. If these packages are useful to you, the credit for the actual security work is
  his.
- **The PHP project** — upstream source from <https://www.php.net/>.

## License

PHP is distributed under the [PHP License v3.01](https://www.php.net/license/). The packaging
(spec and patches) retains its upstream licensing from the Remi repository. This repository adds only
the build metadata and documentation, released under the same terms as the packaged software.
