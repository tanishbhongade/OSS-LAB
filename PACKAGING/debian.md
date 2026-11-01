# Build a C Application During `.deb` Installation

## Aim

Create a Debian package (`.deb`) that contains C source code and
compiles it on the target Linux machine when the package is installed.

> **How it works:** The package contains the source code and a
> `postinst` script. Debian unpacks the package, installs its
> dependencies, and then runs `postinst` during package configuration.
> The script runs `make` to compile and install the executable.

## Requirements

Use a Debian-based Linux system such as Ubuntu or Debian.

Install the packaging tools:

``` bash
sudo apt update
sudo apt install dpkg-dev build-essential
```

`build-essential` provides GCC, Make, and other basic build tools. The
target machine also needs these tools; the package below declares
`build-essential` as a dependency so APT can install it.

## Step 1: Create the package directory structure

Run:

``` bash
mkdir -p myapp/DEBIAN
mkdir -p myapp/usr/src/myapp
```

The final structure will be:

``` text
myapp/
├── DEBIAN/
│   ├── control
│   ├── postinst
│   └── postrm
└── usr/
    └── src/
        └── myapp/
            ├── main.c
            └── Makefile
```

## Step 2: Create the control file

Create `myapp/DEBIAN/control`:

``` text
Package: myapp
Version: 1.0
Section: utils
Priority: optional
Architecture: amd64
Maintainer: Your Name <you@example.com>
Depends: build-essential
Description: Example application compiled during installation
 A simple C program compiled on the target machine when this package is installed.
```

Notes:

-   `Package` is the package name.
-   `Version` is the package version.
-   `Architecture: amd64` is for 64-bit x86 systems. Use the appropriate
    architecture for your target, such as `arm64`, if needed. Do not use
    `all` for this example because it generates a machine-specific
    executable.
-   `Depends: build-essential` asks APT to install the compiler and
    build tools before configuring this package.
-   Replace the maintainer name and email with your own details.

## Step 3: Create the C source file

Create `myapp/usr/src/myapp/main.c`:

``` c
#include <stdio.h>

int main(void)
{
    printf("Hello from a DEB package!\\n");
    return 0;
}
```

## Step 4: Create the Makefile

Create `myapp/usr/src/myapp/Makefile`:

``` makefile
all:
    gcc -Wall -Wextra main.c -o myapp

install:
    install -D -m 755 myapp $(DESTDIR)/usr/bin/myapp

clean:
    rm -f myapp
```

**Important:** The command lines under `all`, `install`, and `clean`
must begin with a real TAB character, not spaces.

## Step 5: Create the `postinst` script

`postinst` runs when Debian configures the package, after its files have
been unpacked and its dependencies have been installed.

Create `myapp/DEBIAN/postinst`:

``` sh
#!/bin/sh
set -e

echo "Building myapp from source..."

cd /usr/src/myapp
make
make install

echo "myapp build and installation completed."

exit 0
```

Make the script executable:

``` bash
chmod 755 myapp/DEBIAN/postinst
```

The script runs as root during package configuration. Therefore,
`make install` can install the executable into `/usr/bin`.

## Step 6: Create a removal script

The executable is generated during installation, so it is not listed as
a file originally unpacked from the `.deb`. Add a `postrm` script to
remove it when the package is removed.

Create `myapp/DEBIAN/postrm`:

``` sh
#!/bin/sh
set -e

if [ "$1" = "remove" ] || [ "$1" = "purge" ]; then
    rm -f /usr/bin/myapp
fi

exit 0
```

Make it executable:

``` bash
chmod 755 myapp/DEBIAN/postrm
```

## Step 7: Build the `.deb` package

Run this command from the directory containing the `myapp` folder:

``` bash
dpkg-deb --build myapp
```

It creates:

``` text
myapp.deb
```

## Step 8: Inspect the package

View package metadata:

``` bash
dpkg-deb --info myapp.deb
```

View the files included in the package:

``` bash
dpkg-deb --contents myapp.deb
```

## Step 9: Install the package

Run:

``` bash
sudo apt install ./myapp.deb
```

APT installs the declared dependency (`build-essential`) if it is not
already installed. It then unpacks and configures `myapp`. During
configuration, `postinst` runs and compiles the C source code.

The installation output should include messages similar to:

``` text
Building myapp from source...
gcc -Wall -Wextra main.c -o myapp
myapp build and installation completed.
```

## Step 10: Test the application

Run:

``` bash
myapp
```

Expected output:

``` text
Hello from a DEB package!
```

Check package installation status:

``` bash
dpkg -l myapp
```

## Step 11: Remove the package

Run:

``` bash
sudo apt remove myapp
```

The `postrm` script removes the generated `/usr/bin/myapp` executable.

## Installation sequence to remember for the examination

1.  User runs `sudo apt install ./myapp.deb`.
2.  APT installs the package dependencies.
3.  Debian unpacks the package files.
4.  Debian configures the package.
5.  The `postinst` script runs.
6.  `postinst` changes to `/usr/src/myapp`.
7.  `make` runs GCC and creates the executable.
8.  `make install` copies the executable to `/usr/bin/myapp`.
9.  Package configuration completes.

## Important points

-   `postinst` means **post-installation configuration script**. It is
    run during package configuration, not at the moment the `.deb` file
    is created.

-   The source code and Makefile are included in the `.deb`; compilation
    happens on the target machine.

-   The target machine needs a compiler and build tools. The `Depends`
    field ensures APT installs `build-essential` when installing through
    APT.

-   If compilation fails, package configuration fails. Check the error,
    correct the problem, and retry configuration with:

    ``` bash
    sudo dpkg --configure -a
    ```

-   This is a simple educational demonstration. For production Debian
    packages, compiling software in a maintainer script is generally
    discouraged. Prefer building the executable when preparing the
    package, or use an appropriate DKMS workflow for kernel modules.

-   The example assumes an `amd64` target. Build and test for the
    architecture you actually intend to support.

## Quick revision: important commands

``` bash
# Install tools
sudo apt update
sudo apt install dpkg-dev build-essential

# Create package directories
mkdir -p myapp/DEBIAN
mkdir -p myapp/usr/src/myapp

# Set maintainer script permissions
chmod 755 myapp/DEBIAN/postinst
chmod 755 myapp/DEBIAN/postrm

# Build the package
dpkg-deb --build myapp

# Inspect the package
dpkg-deb --info myapp.deb
dpkg-deb --contents myapp.deb

# Install and test
sudo apt install ./myapp.deb
myapp

# Remove
sudo apt remove myapp
```

**Key examination concept:** The `.deb` contains source code, and the
`postinst` script invokes the compiler during package configuration. The
generated executable is then installed on the target system.
