# RPM Packaging Lab: Ubuntu + Docker + CentOS Stream 10

A practical record of building, installing, verifying, and uninstalling
a simple RPM package.

## 1. Pull the CentOS image

Run on your Ubuntu host:

``` bash
docker pull quay.io/centos/centos:stream10
```

## 2. Start the container

``` bash
docker run -it --name rpm-lab quay.io/centos/centos:stream10 bash
```

The following commands run inside the container. To return to it later:

``` bash
docker start -ai rpm-lab
```

## 3. Install build tools

``` bash
dnf install -y rpm-build gcc make tar
```

## 4. Create the RPM build tree

``` bash
mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
```

  Directory     Purpose
  ------------- -------------------------------------
  `BUILD`       Unpacked source and build workspace
  `BUILDROOT`   Temporary staging filesystem
  `RPMS`        Binary RPM packages
  `SOURCES`     Source archives
  `SPECS`       RPM spec files
  `SRPMS`       Source RPM packages

## 5. Create the C application

``` bash
mkdir -p ~/myapp-1.0
cd ~/myapp-1.0
```

Create `main.c`:

``` c
#include <stdio.h>

int main(void)
{
    printf("Hello from CentOS RPM package!\\n");
    return 0;
}
```

Create `Makefile`:

``` makefile
all:
    gcc -Wall -Wextra main.c -o myapp

install:
    install -D -m 755 myapp $(DESTDIR)/usr/bin/myapp

clean:
    rm -f myapp
```

**Makefile note:** The command lines beneath each target must begin with
a real tab character.

Test the program:

``` bash
make
./myapp
```

Expected output:

``` text
Hello from CentOS RPM package!
```

## 6. Create the source archive

``` bash
cd ~
tar -czvf ~/rpmbuild/SOURCES/myapp-1.0.tar.gz myapp-1.0
```

Check that the archive has the expected top-level directory:

``` bash
tar -tzf ~/rpmbuild/SOURCES/myapp-1.0.tar.gz
```

It should contain `myapp-1.0/main.c` and `myapp-1.0/Makefile`.

## 7. Create the spec file

``` bash
vi ~/rpmbuild/SPECS/myapp.spec
```

Use:

``` spec
Name:           myapp
Version:        1.0
Release:        1%{?dist}
%global debug_package %{nil}
Summary:        My first RPM package

License:        MIT
Source0:        %{name}-%{version}.tar.gz

BuildRequires:  gcc
BuildRequires:  make

%description
My first RPM package containing a simple C application.

%prep
%setup -q

%build
make

%install
mkdir -p %{buildroot}/usr/bin
install -m 755 myapp %{buildroot}/usr/bin/myapp

%files
/usr/bin/myapp

%changelog
* Sun Nov 01 2026 Your Name <you@example.com> - 1.0-1
- Initial RPM package
```

Replace the changelog name and email with your own. Update the date if
you create the package on a different day.

### What the spec sections mean

-   `Name`, `Version`, `Release`: package identity.
-   `Source0`: source archive used by the build.
-   `BuildRequires`: tools required to compile the application.
-   `%prep`: unpacks the source archive.
-   `%build`: compiles the application using `make`.
-   `%install`: copies the compiled binary into the temporary build
    root.
-   `%files`: declares which staged files belong to the RPM.
-   `%global debug_package %{nil}`: disables automatic debug subpackage
    generation for this small exercise. It addresses the empty
    debug-source file-list error encountered in this environment; it is
    not required for every RPM.

## 8. Build the RPM

``` bash
rpmbuild -ba ~/rpmbuild/SPECS/myapp.spec
```

`-ba` builds both the binary RPM and source RPM.

List the binary RPM:

``` bash
ls -lh ~/rpmbuild/RPMS/$(uname -m)/
```

On an x86_64 system, its name should resemble:

``` text
myapp-1.0-1.el10.x86_64.rpm
```

List the source RPM:

``` bash
ls -lh ~/rpmbuild/SRPMS/
```

## 9. Inspect the package

List files contained in the RPM:

``` bash
rpm -qpl ~/rpmbuild/RPMS/$(uname -m)/myapp-1.0-1*.rpm
```

Expected:

``` text
/usr/bin/myapp
```

Inspect package metadata:

``` bash
rpm -qpi ~/rpmbuild/RPMS/$(uname -m)/myapp-1.0-1*.rpm
```

## 10. Install the RPM

``` bash
dnf install -y ~/rpmbuild/RPMS/$(uname -m)/myapp-1.0-1*.rpm
```

DNF installs the local package and resolves its dependencies.

## 11. Run and verify the application

``` bash
myapp
```

Expected output:

``` text
Hello from CentOS RPM package!
```

Verify package installation:

``` bash
rpm -q myapp
```

List package-owned files:

``` bash
rpm -ql myapp
```

Show installed package metadata:

``` bash
rpm -qi myapp
```

## 12. Uninstall the RPM

``` bash
dnf remove -y myapp
```

Verify removal:

``` bash
rpm -q myapp
command -v myapp
```

The package query should report that `myapp` is not installed, and
`command -v` should no longer find the executable.

## 13. Useful RPM commands

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `rpm -qa`                           List installed RPM packages

  `rpm -q myapp`                      Check whether `myapp` is installed

  `rpm -qi myapp`                     Show installed package information

  `rpm -ql myapp`                     List files owned by an installed
                                      package

  `rpm -qpl package.rpm`              List files inside an RPM file

  `rpm -qpi package.rpm`              Show metadata from an RPM file

  `rpm -qp --scripts package.rpm`     Display package scriptlets

  `dnf install ./package.rpm`         Install a local RPM and resolve
                                      dependencies

  `dnf remove myapp`                  Uninstall a package
  -----------------------------------------------------------------------

## 14. Key concepts

### Build-time compilation

This spec compiles the program while building the RPM:

1.  `%prep` unpacks the source.
2.  `%build` compiles the program.
3.  `%install` stages the binary under `%{buildroot}`.
4.  `%files` records the binary as package-owned.
5.  Installing the RPM places it at `/usr/bin/myapp`.

The target machine does not need to compile the C source during
installation.

### Build root

`%{buildroot}` is a temporary staging directory used only while creating
the package. A staged path such as:

``` text
%{buildroot}/usr/bin/myapp
```

becomes `/usr/bin/myapp` when the RPM is installed.

### Build dependencies

`BuildRequires` declares tools needed to build the RPM. It does not mean
those tools are automatically installed on every target machine. RPM
also detects shared-library requirements of the compiled executable.

------------------------------------------------------------------------

**Outcome:** A binary RPM and source RPM were built on CentOS Stream 10;
the binary RPM was installed, tested, queried, and removed using DNF.
