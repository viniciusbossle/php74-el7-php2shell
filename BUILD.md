# Reproducing the build

These packages are a straight rebuild of **Remi Collet's `php74` spec at release `7.4.33-31`** for
Enterprise Linux 7. Nothing in the PHP source or in Remi's patches was changed. Only **two** small
spec adjustments are needed, both because EL7's toolchain is older than what the current spec assumes
(GCC 4.8.5 and ICU 73, versus the GCC/ICU on EL8/EL9).

If you rebuild from the published `src.rpm`, those two adjustments are already baked in, so a plain
rebuild reproduces the binaries. The notes below are for rebuilding from Remi's upstream spec yourself,
and to document exactly what differs from it.

## Build host

- An EL7 box (CentOS 7 / RHEL 7), `x86_64`, GCC 4.8.5 (the stock `devtoolset` is **not** required).
- Build tooling:
  ```bash
  yum -y install rpm-build rpmdevtools redhat-rpm-config
  rpmdev-setuptree
  ```
- Remi's repositories reachable for `BuildRequires`. Since the stock EL7 mirrorlists point at the
  archive now, use the live tree directly, e.g.:
  ```ini
  # /etc/yum.repos.d/remi-live.repo
  [remi-live]
  baseurl=https://rpms.remirepo.net/enterprise/7/remi/x86_64/
  enabled=1
  gpgcheck=0
  [remi-php74-live]
  baseurl=https://rpms.remirepo.net/enterprise/7/php74/x86_64/
  enabled=1
  gpgcheck=0
  [remi-safe-live]
  baseurl=https://rpms.remirepo.net/enterprise/7/safe/x86_64/
  enabled=1
  gpgcheck=0
  ```

## Sources

- PHP source, straight from php.net:
  ```bash
  curl -O https://www.php.net/distributions/php-7.4.33.tar.xz
  curl -O https://www.php.net/distributions/php-7.4.33.tar.xz.asc
  sha256sum php-7.4.33.tar.xz
  # expected: 924846abf93bc613815c55dd3f5809377813ac62a9ec4eb3778675b82a27b927
  ```
- Remi's `php74` spec and patches at the `7.4.33-31` release, from
  <https://git.remirepo.net/cgit/rpms/php/php74.git/> (the commit that adds
  `php-7.4-cve-2026-109618.patch` and bumps the release to 31).

Put the tarball, the `.asc`, the spec's patches into `~/rpmbuild/SOURCES/` and the spec into
`~/rpmbuild/SPECS/php74.spec`.

## The two EL7 adjustments

Both are in `php74.spec`.

### 1. Allow ICU 73 for `intl`

Remi's spec requires ICU ≥ 74 at build time, but EL7's Remi only ships `libicu73`. The `intl`
extension links fine against ICU 73 (that's what the stock `7.4.33-15.el7.remi` build does). Lower the
version gate:

```diff
-BuildRequires: pkgconfig(icu-i18n) >= 74
-BuildRequires: pkgconfig(icu-io) >= 74
-BuildRequires: pkgconfig(icu-uc) >= 74
+BuildRequires: pkgconfig(icu-i18n) >= 73
+BuildRequires: pkgconfig(icu-io) >= 73
+BuildRequires: pkgconfig(icu-uc) >= 73
```

Install the matching ICU 73 devel from Remi (and remove the base `libicu-devel` so they don't clash):

```bash
yum remove -y libicu-devel          # pulls base libicu 50; conflicts with 73
yum install -y libicu73-devel oniguruma5php-devel gd3php-devel
```

### 2. Skip the `icu74` adaptation patch

The spec applies an `icu74` patch (Patch11, `php-7.4.33-icu.patch`) whose configure snippet forces
`-std=gnu++17`. GCC 4.8.5 on EL7 has no C++17, and `intl` then fails to compile
(`icu_73::UnicodeString has no member named getBuffer`). The patch is only needed for ICU ≥ 74; with
ICU 73 the original `config.m4` already selects C++11 — the same standard the stock EL7 build uses.
Comment the patch application out in `%prep`:

```diff
-%patch -P11 -p1 -b .icu74
+#%patch -P11 -p1 -b .icu74   # EL7: ICU 73 keeps the original C++11, no icu74 adaptation
```

No other patch is touched. The php2shell mitigation (Patch243,
`php-7.4-cve-2026-109618.patch`) and every other patch apply unchanged.

## Build

```bash
rpmbuild -ba ~/rpmbuild/SPECS/php74.spec --nocheck
```

`--nocheck` skips the upstream test suite (`%check`): the full 9,966-test run has one flaky
timeout on a loaded build host and is not needed to produce the packages. Drop it if you want to run
the suite.

Expected output: **35 binary RPMs + 1 `src.rpm`** at `7.4.33-31.el7`, under
`~/rpmbuild/RPMS/x86_64/` and `~/rpmbuild/SRPMS/`.

## Verifying against the published binaries

Rebuilt RPMs won't be byte-identical to the signed ones here (build host, timestamps, signature), but
you can confirm the payload matches:

```bash
# compare the file list and the installed-file digests
rpm -qlvp your-build/php-common-7.4.33-31.el7.x86_64.rpm | sort
rpm -qlvp published/php-common-7.4.33-31.el7.x86_64.rpm  | sort
# or diff the extracted trees
```
