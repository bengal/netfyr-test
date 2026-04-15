# SPEC-502: RPM Packaging for Fedora

## What

Create an RPM spec file (`netfyr.spec`) that builds and packages netfyr as RPMs for Fedora. The spec file produces two packages:

1. **`netfyr`** — the CLI binary, man pages, default configuration directories, and a LICENSE file.
2. **`netfyr-daemon`** — the daemon binary, systemd service and socket units, and the state directory.

The spec file follows Fedora packaging guidelines and builds using `cargo` via the `%cargo_build` / `%cargo_install` macros from `rust-packaging`.

## Why

RPM packaging is the standard distribution mechanism for Fedora. A proper spec file allows netfyr to be built in Fedora's build system (Koji/mock), installed via `dnf`, and potentially submitted to Fedora as an official package. This also ensures all runtime dependencies (shared libraries, systemd) are declared, the binaries are installed to the correct FHS paths, and the man pages and systemd units are properly owned.

## User interaction

```bash
# Build the RPM locally using the build script
./scripts/build-rpm.sh
# or manually:
rpmbuild -ba netfyr.spec

# Install
sudo dnf install ./x86_64/netfyr-0.1.0-1.fc42.x86_64.rpm
sudo dnf install ./x86_64/netfyr-daemon-0.1.0-1.fc42.x86_64.rpm

# Verify installation
rpm -ql netfyr
/usr/bin/netfyr
/usr/share/man/man1/netfyr.1.gz
/usr/share/man/man1/netfyr-apply.1.gz
/usr/share/man/man1/netfyr-query.1.gz
/usr/share/man/man5/netfyr.yaml.5.gz
/usr/share/man/man7/netfyr-examples.7.gz
/usr/share/doc/netfyr/examples/policies/bare-ethernet.yaml
/etc/netfyr/policies
/usr/share/licenses/netfyr/LICENSE

rpm -ql netfyr-daemon
/usr/bin/netfyr-daemon
/usr/lib/systemd/system/netfyr.service
/usr/lib/systemd/system/netfyr.socket
/usr/share/licenses/netfyr-daemon/LICENSE

# Use it
netfyr apply /etc/netfyr/policies/
man netfyr

# Start the daemon
sudo systemctl start netfyr
```

## Implementation details

- Crate: (root workspace)
- Files:
  - `netfyr.spec` — RPM spec file at workspace root
  - `scripts/build-rpm.sh` — script to create source tarball and build the RPM
  - `dist/netfyr.service` — systemd service unit for the daemon
  - `dist/netfyr.socket` — systemd socket unit for Varlink
- Dependencies (external crates): none (packaging only)

### RPM spec file (`netfyr.spec`)

```specfile
Name:           netfyr
Version:        0.1.0
Release:        1%{?dist}
Summary:        Declarative Linux network configuration

License:        MIT
URL:            https://github.com/netfyr/netfyr
Source0:        %{name}-%{version}.tar.gz
Source1:        %{name}-%{version}-vendor.tar.gz

ExclusiveArch:  %{rust_arches}

BuildRequires:  cargo >= 1.86
BuildRequires:  rust >= 1.86
BuildRequires:  rust-packaging >= 25
BuildRequires:  systemd-rpm-macros

%description
Netfyr is a declarative, policy-based network configuration tool for Linux.
It reads YAML policy files, reconciles them into a desired network state,
diffs against the running system, and applies changes via rtnetlink.

%package daemon
Summary:        Netfyr daemon for dynamic network configuration
Requires:       %{name} = %{version}-%{release}
Requires:       systemd

%description daemon
The netfyr daemon manages dynamic network configuration including DHCPv4.
It listens on a Varlink socket and accepts policy submissions from the
netfyr CLI. Required only when using DHCP or other dynamic factories.

%prep
%autosetup -n %{name}-%{version}
tar xf %{SOURCE1}
%cargo_prep -v vendor

%build
%cargo_build

# Generate man pages (use "cargo run -p xtask", not the xtask alias,
# because %%cargo_prep may overwrite .cargo/config.toml)
cargo run -p xtask -- man

%install
# Install CLI binary
install -Dpm 0755 target/release/netfyr %{buildroot}%{_bindir}/netfyr

# Install daemon binary
install -Dpm 0755 target/release/netfyr-daemon %{buildroot}%{_bindir}/netfyr-daemon

# Install man pages
install -d %{buildroot}%{_mandir}/man1
install -Dpm 0644 man/*.1 %{buildroot}%{_mandir}/man1/
install -d %{buildroot}%{_mandir}/man5
install -Dpm 0644 man/netfyr.yaml.5 %{buildroot}%{_mandir}/man5/
install -d %{buildroot}%{_mandir}/man7
install -Dpm 0644 man/netfyr-examples.7 %{buildroot}%{_mandir}/man7/

# Install systemd units
install -Dpm 0644 dist/netfyr.service %{buildroot}%{_unitdir}/netfyr.service
install -Dpm 0644 dist/netfyr.socket %{buildroot}%{_unitdir}/netfyr.socket

# Create config directories
install -d %{buildroot}%{_sysconfdir}/netfyr/policies

# Install example files
install -d %{buildroot}%{_docdir}/%{name}/examples/policies
install -pm 0644 examples/policies/*.yaml %{buildroot}%{_docdir}/%{name}/examples/policies/

# Note: LICENSE is handled by %%license in %%files — do NOT install it manually
# to avoid a conflict with the %%license directive.

%check
# Smoke-test: verify the built binaries are functional
target/release/netfyr --help > /dev/null
target/release/netfyr-daemon --help > /dev/null

%post daemon
%systemd_post netfyr.service netfyr.socket

%preun daemon
%systemd_preun netfyr.service netfyr.socket

%postun daemon
%systemd_postun_with_restart netfyr.service netfyr.socket

%files
%license LICENSE
%{_bindir}/netfyr
%{_mandir}/man1/netfyr.1*
%{_mandir}/man1/netfyr-apply.1*
%{_mandir}/man1/netfyr-query.1*
%{_mandir}/man5/netfyr.yaml.5*
%{_mandir}/man7/netfyr-examples.7*
%dir %{_sysconfdir}/netfyr
%dir %{_sysconfdir}/netfyr/policies
%{_docdir}/%{name}/examples/policies/

%files daemon
%license LICENSE
%{_bindir}/netfyr-daemon
%{_unitdir}/netfyr.service
%{_unitdir}/netfyr.socket
```

### Systemd service unit (`dist/netfyr.service`)

```ini
[Unit]
Description=Netfyr declarative network configuration daemon
After=network-pre.target
Before=network.target
Wants=network-pre.target

[Service]
Type=notify
ExecStart=/usr/bin/netfyr-daemon
Restart=on-failure
RuntimeDirectory=netfyr
StateDirectory=netfyr

[Install]
WantedBy=multi-user.target
```

### Systemd socket unit (`dist/netfyr.socket`)

```ini
[Unit]
Description=Netfyr Varlink socket

[Socket]
ListenStream=/run/netfyr/netfyr.sock
SocketMode=0666

[Install]
WantedBy=sockets.target
```

### Build and runtime dependencies

**Build-time:**
- `cargo` and `rust` >= 1.86 (minimum Rust version for the project)
- `rust-packaging` >= 25 (provides `%cargo_build`, `%cargo_install`, and `%rust_arches` macros)
- `systemd-rpm-macros` (provides `%systemd_post`, `%systemd_preun`, `%systemd_postun_with_restart`, `%_unitdir`)

**Run-time:**
- `netfyr` package: no runtime dependencies beyond glibc
- `netfyr-daemon` package: requires `netfyr` (same version) and `systemd`

### FHS paths

| Item | Path | Package |
|---|---|---|
| CLI binary | `/usr/bin/netfyr` | netfyr |
| Daemon binary | `/usr/bin/netfyr-daemon` | netfyr-daemon |
| Man pages (section 1) | `/usr/share/man/man1/netfyr*.1.gz` | netfyr |
| Man page (section 5) | `/usr/share/man/man5/netfyr.yaml.5.gz` | netfyr |
| Man page (section 7) | `/usr/share/man/man7/netfyr-examples.7.gz` | netfyr |
| Example files | `/usr/share/doc/netfyr/examples/` | netfyr |
| Systemd service | `/usr/lib/systemd/system/netfyr.service` | netfyr-daemon |
| Systemd socket | `/usr/lib/systemd/system/netfyr.socket` | netfyr-daemon |
| Policy dir | `/etc/netfyr/policies/` | netfyr |
| Runtime state | `/run/netfyr/` (created by systemd RuntimeDirectory) | netfyr-daemon |
| Persisted policies | `/var/lib/netfyr/` (created by systemd StateDirectory) | netfyr-daemon |
| License | `/usr/share/licenses/netfyr/LICENSE` | both |

### Source tarball

The `Source0` tarball is created from the git repository with:

```bash
git archive --format=tar.gz --prefix=netfyr-0.1.0/ -o netfyr-0.1.0.tar.gz HEAD
```

This includes the full workspace (all crates, xtask, Cargo.lock).

### Build script (`scripts/build-rpm.sh`)

A shell script automates the tarball creation and RPM build. It extracts the name and version from the spec file so they stay in sync.

```bash
#!/bin/bash
set -euo pipefail

SPEC="netfyr.spec"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
REPO_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"

cd "$REPO_ROOT"

# Extract Name and Version from the spec file
NAME=$(grep '^Name:' "$SPEC" | awk '{print $2}')
VERSION=$(grep '^Version:' "$SPEC" | awk '{print $2}')

# Create rpmbuild directory structure
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}

# Create source tarball from git
echo "Creating source tarball ${NAME}-${VERSION}.tar.gz ..."
git archive --format=tar.gz \
    --prefix="${NAME}-${VERSION}/" \
    -o ~/rpmbuild/SOURCES/"${NAME}-${VERSION}.tar.gz" \
    HEAD

# Create vendor tarball (%cargo_prep forces offline mode, so dependencies
# must be vendored — but vendor/ is gitignored and never committed)
echo "Creating vendor tarball ${NAME}-${VERSION}-vendor.tar.gz ..."
cargo vendor vendor
tar czf ~/rpmbuild/SOURCES/"${NAME}-${VERSION}-vendor.tar.gz" vendor/

# Copy the spec file
cp "$SPEC" ~/rpmbuild/SPECS/

# Build the RPM
echo "Building RPM ..."
rpmbuild -ba ~/rpmbuild/SPECS/"$SPEC"

echo ""
echo "Build complete. RPMs:"
find ~/rpmbuild/RPMS/ -name "${NAME}*.rpm" -type f 2>/dev/null
find ~/rpmbuild/SRPMS/ -name "${NAME}*.src.rpm" -type f 2>/dev/null
```

The script must be executable (`chmod +x scripts/build-rpm.sh`).

## Depends on

- SPEC-001 (Workspace setup — crate structure)
- SPEC-301 (CLI apply — binary entry point)
- SPEC-403 (Daemon — daemon binary and systemd units)
- SPEC-501 (Man pages — man page generation)
- SPEC-503 (YAML reference man page — netfyr.yaml(5))

## Acceptance criteria

```gherkin
Feature: RPM packaging for Fedora

  Scenario: RPM spec file is valid
    Given the netfyr.spec file exists at the workspace root
    When the developer runs "rpmlint netfyr.spec"
    Then no errors are reported

  Scenario: Build script exists and is executable
    Given scripts/build-rpm.sh exists at the workspace root
    Then it has the executable bit set
    And it reads Name and Version from netfyr.spec

  Scenario: Source tarball can be created
    Given a clean git working tree
    When the developer runs "git archive --format=tar.gz --prefix=netfyr-0.1.0/ -o netfyr-0.1.0.tar.gz HEAD"
    Then a tarball is created containing the full workspace

  Scenario: RPM builds successfully with rpmbuild
    Given the source tarball is placed in ~/rpmbuild/SOURCES/
    When the developer runs "rpmbuild -ba netfyr.spec"
    Then the build completes without errors
    And a binary RPM for netfyr is produced under ~/rpmbuild/RPMS/
    And a binary RPM for netfyr-daemon is produced under ~/rpmbuild/RPMS/
    And a source RPM is produced under ~/rpmbuild/SRPMS/

  Scenario: RPM builds successfully in mock
    Given the source tarball and spec file
    When the developer runs "mock -r fedora-42-x86_64 --buildsrpm --spec netfyr.spec --sources ."
    And then "mock -r fedora-42-x86_64 --rebuild netfyr-0.1.0-1.fc42.src.rpm"
    Then the build succeeds
    And binary RPMs for both packages are produced

  Scenario: CLI RPM installs the binary to /usr/bin
    Given the netfyr RPM is installed
    When the user runs "rpm -ql netfyr"
    Then the output includes "/usr/bin/netfyr"
    And the binary is executable

  Scenario: CLI RPM installs man pages
    Given the netfyr RPM is installed
    When the user runs "man -w netfyr"
    Then the path points to /usr/share/man/man1/netfyr.1.gz
    And "man netfyr" renders the man page

  Scenario: Daemon RPM installs the daemon binary
    Given the netfyr-daemon RPM is installed
    When the user runs "rpm -ql netfyr-daemon"
    Then the output includes "/usr/bin/netfyr-daemon"

  Scenario: Daemon RPM installs systemd units
    Given the netfyr-daemon RPM is installed
    When the user runs "systemctl cat netfyr.service"
    Then the unit file content is displayed
    And ExecStart contains "/usr/bin/netfyr-daemon"
    And Type is "notify"

  Scenario: Daemon RPM installs socket unit
    Given the netfyr-daemon RPM is installed
    When the user runs "systemctl cat netfyr.socket"
    Then the socket unit is displayed
    And ListenStream is "/run/netfyr/netfyr.sock"

  Scenario: CLI RPM creates config directories
    Given the netfyr RPM is installed
    Then /etc/netfyr/policies/ exists and is owned by the netfyr package

  Scenario: Daemon RPM requires CLI package
    Given the netfyr-daemon RPM is built
    When the user runs "rpm -qR netfyr-daemon"
    Then netfyr is listed as a dependency
    And systemd is listed as a dependency

  Scenario: RPM uninstall cleans up
    Given the netfyr and netfyr-daemon RPMs are installed
    When the user runs "sudo dnf remove netfyr-daemon netfyr"
    Then /usr/bin/netfyr and /usr/bin/netfyr-daemon are removed
    And the systemd units are removed
    And /etc/netfyr/ is preserved (config files are not removed on uninstall)

  Scenario: Config directories survive upgrade
    Given the netfyr RPM is installed
    And the user has created /etc/netfyr/policies/my-server.yaml
    When the user upgrades to a newer netfyr RPM
    Then /etc/netfyr/policies/my-server.yaml is preserved

  Scenario: Build script produces installable RPMs end-to-end
    Given a clean git working tree with all changes committed
    When the developer runs "./scripts/build-rpm.sh"
    Then the script exits with code 0
    And the resulting RPMs can be installed with "sudo dnf install <rpm-path>"
    And "netfyr --help" works after installation
```

### Important implementation notes

The spec file **must** be validated by actually building the RPM during development (e.g. by running `./scripts/build-rpm.sh` or `rpmbuild -ba`). A spec file that has not been tested against a real build is not acceptable.

Common pitfalls to avoid:
- **Missing `%cargo_prep`**: The `%cargo_build` macro from `rust-packaging` requires `%cargo_prep` to have been called in `%prep`. Without it the build fails.
- **`cargo xtask` alias**: Do not use `cargo xtask man`. The `.cargo/config.toml` alias may not exist, and `%cargo_prep` may overwrite it. Use `cargo run -p xtask -- man` instead.
- **Double file installation**: Do not manually `install` the LICENSE file — the `%license LICENSE` directive in `%files` handles it.
- **Missing `%check`**: Always include a `%check` section that at minimum smoke-tests the built binaries.
- **Never commit vendored crates**: The `%cargo_prep` macro forces offline mode, so the spec uses `Source1` (a vendor tarball) and `%cargo_prep -v vendor`. The build script creates this tarball on the fly via `cargo vendor`. However, the `vendor/` directory itself (thousands of files, hundreds of megabytes) must **never** be committed to the repository. Add `/vendor` to `.gitignore`.
- **Macro expansion in comments**: RPM expands macros even inside comments. If you need to mention a macro name like `%cargo_install` in a comment, double the percent sign (`%%cargo_install`) to prevent expansion.
