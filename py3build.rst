Python 3 build instructions
==============================================================================

At present time we're using the 3.9 line.  This will be updated soon.

We're making a build that's statically linked to OpenSSL (latest
1.1.1w at time of writing), so we don't have to distribute a library
dependency.

Fortunately, both OpenSSL and Python builds are straightforward.

.. contents::


Build Platform
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are some caveats with the build systems needed, to get a Python
that runs on all of our systems.  The build host needs extra packages
installed, and we have to make different builds to cover all our
operating system variations.

Currently (or recently retired) builds with the following variants and
package name suffix:

==================  ======  ======= ========
distro              suffix  status  arch
==================  ======  ======= ========
centos6             rhel6   extant  amd64
u14,u16,centos7     u14     defunct amd64
u18                 u18     extant  aarch64
u20                 u20     extant  amd64
u22                 u22     extant  amd64
==================  ======  ======= ========

**TODO** TEST u22 built actually runs on u20, this require vernius


Development Packages
--------------------

First, all the build systems need several development packages
installed::

    $ sudo apt install \
      {lib{bz2,gdbm{,-compat},sqlite3,readline,lzma,ffi},zlib1g,uuid}-dev

These are so the standard ``import`` libraries will all build.
Otherwise, it will build, but missing key features and not all standard
``import`` statements will work with the resulting Python.


Don't build NIS
---------------

We don't want NIS support because it requires a lot of shared libraries,
it's deprecated in glibc, and it's removed from Python in 3.12.
Unfortunately the ``setup.py`` autodetects it based on header file pnd
library presence, with no override available, and no related configure
options.  We cannot remove ``libtirpc3``, ``libtirpc-dev``, or ``glibc``
which includes files it searches for.  So best to just edit
``detect_nis()`` and have it simply do::

    self.missing.append('nis')
    return

This will ensure it doesn't get built.


Platform Variants for Dynloads
------------------------------

Even though the main Python executable works fine on all our systems,
some loadable modules (found in python install ``lib-dynload/``
directory) have additional external shared library dependencies, so some
imports won't work unless Python is built on that particular platform.

Excluding ones that don't change across OS versions, we have a list of
loadable modules with extra dependencies looking like this::

    $ for file in `find /opt/python-3.9.18 -name '*.so'`; do
          echo $file; ldd $file; echo
      done \
      | grep -vE \
          -e linux-vdso \
          -e ld-linux-x86 \
          -e 'lib(c|pthread|m|dl|crypt|freebl3|rt|uuid|nsl).so' \
      | awk '/^.opt.python/ {
          deps = "";
          libline = $0;
          while (getline) {
              if ($1 ~ /^lib/) {
                  deps = deps" "$1;
                  continue;
              } else if (length(deps)) {
                  printf("%s:%s\n",
                         gensub("^.*/(.+).cpython.*$", "\\1", 1, libline),
                         deps);
              };
              next;
          }
      }' \
      | sort \
      | column -t

    binascii:       libz.so.1
    _bz2:           libbz2.so.1.0
    _ctypes:        libffi.so.8
    _curses:        libncursesw.so.6     libtinfo.so.6
    _curses_panel:  libpanelw.so.6       libncursesw.so.6  libtinfo.so.6
    _dbm:           libgdbm_compat.so.4  libgdbm.so.6
    _gdbm:          libgdbm.so.6
    _lzma:          liblzma.so.5
    readline:       libreadline.so.8     libtinfo.so.6
    _sqlite3:       libsqlite3.so.0
    zlib:           libz.so.1

We don't want all those imports to fail, so we have to build different
versions (see table above), which differ in the above set of shared
libraries.

The Python main executable itself actually works on all of them; its
link dependencies end up looking like this::

  $ ldd $(readlink -f $(which python3)) | awk '{print $1}'

  linux-vdso.so.1
  libm.so.6
  libc.so.6
  /lib64/ld-linux-x86-64.so.2

without any OpenSSL dependency.  We need to build and maintain OpenSSL
as well on the build system for the link (see below), but the 1.1
maintenance line that we use changes infrequently.

Also note, on Ubuntu platforms, only one system needs to build OpenSSL,
and this library works on all OS versions linked statically, it works
fine.  Only Python needs a different build on each host system.


OpenSSL 1.1 build
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

get the source::

    $ git clone -q https://github.com/openssl/openssl
    $ cd openssl
    $ # or git fetch/pull if refreshing

checkout latest 1.1::

    $ git tag -l | grep -Pi '^openssl.1.1.\d[a-z]$' | sort -V | tail -1
    OpenSSL_1_1_1w

    $ git checkout OpenSSL_1_1_1w
    $ # "make distclean" if refreshing

configure (use ``/etc/pki/tls`` to share vendor config on rhel)::

    $ ./config \
      --prefix=/opt/openssl-1.1.1w \
      --openssldir=/etc/ssl no-shared

build::

    $ make -j`nproc`

install, inspect and package without needing to run as root::

    $ rm -rf /tmp/relocate
    $ mkdir /tmp/relocate
    $ make DESTDIR=/tmp/relocate install
    $ cd /tmp/relocate

inspect file tree, make sure not writing anywhere strange, then create
the archive::

    $ tar -caf ~/openssl-3.0.13_static_amd64_opt.tar.zst \
      --owner=0 --group=0 \
      *

install on the build system so python build can link the static library::

    $ sudo tar -C / -xapf ~/openssl-1.1.1w_static_amd64_opt.tar.zst

**build system note:**
the tarball will write openssl config files to /etc/ssl/, but the only
differences from older ubuntu defaults are:

- does not set RANDFILE anymore
- sets *CA:true* in *basicConstraints* to ``critical``
- sets *signer_digest* to ``sha256`` (new)
- changes *digests* from ``md5, sha1`` to ``sha1, sha256, sha384, sha512``

If the build system's ``openssl.cnf`` is customized (or the distro's
version shall remain untouched), only ``opt`` dir should be extracted
from the tarball, but the stock config seems to work fine.  The library
need only be installed on the build system.


Python 3.9 build
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Python build is straightforward on Ubuntu.


Prepare the build
-----------------

once the build system is selected, get the source::

    $ git clone -q https://github.com/python/cpython
    $ cd cpython
    $ # or "git fetch --all" if refreshing

checkout latest 3.8::

    $ git tag -l | grep -P '^v3.9.\d+$' | sort -V | tail -1
    v3.9.18

    $ git checkout v3.9.18
    $ # "make distclean" if refreshing

configure and build (replace version numbers)::

    $ sh configure \
        --prefix=/opt/python-3.9.18 \
        --enable-optimizations \
        --disable-shared \
        --disable-ipv6 \
        --with-openssl=/opt/openssl-1.1.1w \
        --with-ensurepip=install


Building on Ubuntu 18+
----------------------

The build on these platforms works without any trouble::

    $ make -j`nproc`


Installing Python in relocate root
----------------------------------

Installation is the same on all platforms: install, inspect and package
without needing to run as root::

    $ rm -rf /tmp/relocate
    $ mkdir /tmp/relocate
    $ make DESTDIR=/tmp/relocate install
    $ cd /tmp/relocate

Make sure to inspect the file tree, to verify it's not writing
anywhere/anything strange, since we will unarchive this from the root
filesystem directory as the root user.


Changing pip interpreter line
-----------------------------

We want pip to be more specific about its interpreter::

    $ for file in `find opt/*/bin/ -type f -executable`
      do echo "${file##*/}: $(file -b $file)"
      done | grep python.script

    pip3: a /opt/python-3.9.18/bin/python script, ASCII text executable
    pip3.9: a /opt/python-3.9.18/bin/python script, ASCII text executable

for some reason it just refers to ``bin/python`` which actually does not
exist.  Only ``python3`` and ``python3.9`` are present.  These happen to
match the specificity of interpreter expressed in the executable name so
we will use that and replace it in the interpreter line::

    $ for file in `find opt/*/bin/ -type f -executable`; do
          if file $file | grep -q python.script; then
              exe=${file##*/}
              ver=$(sed -r 's,^[^[:digit:]]+,,' <<< "$exe")
              sed -i "1 s,python\$,python$ver," $file
          fi
      done


Creating the archive
--------------------

Make the package, ensuring the user is stored user is root since we will
write it in /opt/ as a system package::

    $ tar -caf ~/python-3.9.18_staticssl-1.1.1w_amd64_opt_u$(
          lsb_release -r | awk '{print $2}' | awk -F . '{print $1}'
      ).tar.zst \
      --owner=0 --group=0 \
      *

Finally, extract it to the system location where it will reside (ie, in
``/opt``, which is the path embedded in the archive)::

    $ sudo tar -C / -xapf ~/python-3.9.18_staticssl-1.1.1w_amd64_opt_u$(
          lsb_release -r | awk '{print $2}' | awk -F . '{print $1}'
      ).tar.zst

.. todo: we can get rid of the first awk filter by using '-sr', but only
   once u18 is retired, as it only works on u20+

.
