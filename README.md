# Poky With Flatpak Support

This repository contains a fairly recent clone of [poky]
(http://git.yoctoproject.org/cgit/cgit.cgi/poky/) with [meta-flatpak]
(http://github.com/klihub/meta-flatpak) from [IoT RefKit]
(http://github.com/intel/intel-iot-refkit) added as a git submodule.
meta-flatpak has been slightly adjusted to work on top of poky without
any extra bits from refkit.

The changes in meta-flatpak have been kept minimal, aiming to achieve
a basic working setup while minimizing the diff to the original
refkit-specific branch. Therefore there are occasional oddities left in
there that are originally refkit-specific but do interfere with poky.


## Cloning, Setup, And Building

Let's clone the repository a build some flatpak-enabled images.

```
mandark test $ git clone http://git@github.com/klihub/poky-flatpak.git
mandark test $ cd poky-flatpak
mandark poky-flatpak $ git submodule init
mandark poky-flatpak $ git submodule update --recursive
mandark poky-flatpak $ . ./oe-init-build-env
mandark build $ bitbake core-image-flatpak-runtime core-image-flatpak-sdk qemu-helper-native
```

If everything goes fine you should have, in addition to the built
images, a set of GPG keys GPG keyring, two flatpak repositories and
apache configuration file for them.

```
mandark build $ ls -ls
...
 4 drwxrwxr-x   8 kli kli  4096 Mar  6 13:40 core-image-flatpak-runtime.flatpak
 4 -rw-rw-r--   1 kli kli   309 Mar  6 13:40 core-image-flatpak-runtime.flatpak.http.conf
 4 drwxrwxr-x   8 kli kli  4096 Mar  6 13:42 core-image-flatpak-sdk.flatpak
 4 -rw-rw-r--   1 kli kli   297 Mar  6 13:42 core-image-flatpak-sdk.flatpak.http.conf
24 drwxrwxr-x   4 kli kli 20480 Mar  3 16:57 downloads
 4 drwx------   3 kli kli  4096 Mar  2 18:33 gpg
 4 -rw-rw-r--   1 kli kli   163 Mar  2 18:33 gpg.trustdb
 4 -rw-rw-r--   1 kli kli   273 Mar  2 18:33 poky-signing.cfg
 4 -rw-rw-r--   1 kli kli  1171 Mar  2 18:33 poky-signing.pub
 4 -rw-------   1 kli kli  1244 Mar  2 18:33 poky-signing.sec
...
```


## Installing the Flatpak SDK Runtime

To build flatpaks we first need to pull in the flatpak SDK runtime with
the toolchain for our flatpak-enabled image. To be able to do so we need
to expose our flatpak repository over HTTP. The build generates http
configuration fragments that should be usable as such for apache2 on
fedora. If you use another distro or HTTP server do whatever is needed
to achieve the same result.

```
# for Fedora
mandark build $ cp core-image-flatpak-sdk.flatpak.http.conf /etc/httpd/conf.d
mandark build $ sudo systemctl restart httpd
# add flatpak repo
mandark build $ flatpak remote-add --gpg-import=poky-signing.pub core-sdk http://127.0.0.1/flatpak/core-image-sdk/sdk/
mandark build $ flatpak install --runtime core-sdk iot.poky.BaseSdk
```

If everything went fine, you should be able to see our newly installed
SDK runtime among the installed flatpak runtimes.

```
mandark build $ flatpak list
...
iot.poky.BaseSdk
...
```


## Building Some Flatpaks

Let's build a some test flatpaks for our image. The build steps are the
normal configure/build/install steps, they just need to be run under
flatpak build and the SDK runtime for the target. Here is a trivial
script that can be used to automate the repetitive steps:

```
mandark build $ mkdir apps
mandark apps $ cp ~/build-app.sh .
mandark apps $ cat build-app.sh
#!/bin/bash

app="$1"
target="$2"

fatal () {
    echo "fatal error: $*"
    exit 1
}


if [ -z "$app" -o -z "$target" ]; then
    fatal "usage: $0 <app> {poky|refkit}"
fi

if [ ! -d $target/build ]; then
    fatal "No directory $target/build found"
fi

if [ ! -d $app ]; then
    fatal "No directory $app found"
fi

build=$target/build/$app
src=$app
repo=$target/flatpak.repo

set -e

rm -fr $build
mkdir -p $build
flatpak build-init $build \
    org.$target.$app iot.$target.BaseSdk iot.$target.BasePlatform \
    current

echo "* Building $app for $target..."

(cd $src
 flatpak build ../$build ./configure --prefix=/app && \
    flatpak build ../$build make && \
    flatpak build ../$build make install)

echo "* Finalizing build of $app for $target..."
flatpak build-finish $build --command=$app --filesystem=home && \
    vi $build/metadata

echo "* Exporting build of $app for $target to repo $repo..."
flatpak build-export \
    --gpg-homedir=$target/gpg --gpg-sign=$target-signing@key \
    $repo $build
```

Se let's clone some quasirandom packages and try to build them as flatpaks.

```
mandark apps $ git clone http://github.com/vim/vim.git
mandark apps $ git clone https://git.savannah.gnu.org/git/bash.git
...
mandark apps $ mkdir -p poky/build
mandark apps $ ln -s ../gpg poky
mandark apps $ ln -s ../poky-signing.pub poky
mandark apps $ ./build-app.sh vim poky
...
# Once the build is done, before exporting to the flatpak repository,
# we'll get $EDITOR open to make any changes we see fit to the flatpak
# metadata for vim. Let's edit it, make sure command is set to vim,
# and let's allow it access to the full filesystem by changing the
# script-default 'home' to 'host'. The end result should look something
# like the following.  Save the changes and exit $EDITOR to let the
# script export vim as a flatpak.
mandark apps $ cat poky/build/vim/metadata
[Application]
name=org.poky.vim
runtime=iot.poky.BasePlatform/x86_64/current
sdk=iot.poky.BaseSdk/x86_64/current
command=vim

[Context]
filesystems=host
# Repeat the same steps for bash if you wish to test that one as well...
```


## Export Flatpak App Repository

The application repository needs to be exported over HTTP just like we
did with the SDK repository earlier.

```
# Let's create a HTTP config fragment. Again this is for Fedora/Apache2.
# Do whatever is required by your distro.
mandark apps $ cat core-image-apps.flatpak.http.conf
Alias "/flatpak/core-image/apps/" "<your pwd for apps dir>/poky/flatpak.repo/"

<Directory <your pwd for apps dir>/poky/flatpak.repo>
    Options Indexes FollowSymLinks
    Require all granted
</Directory>

mandark apps $ sudo cp core-image-apps.flatpak.http.conf /etc/httpd/conf.d
mandark apps $ sudo systemctl restart httpd
```

## Booting and Testing

Let's boot our test image with qemu. You may want to grant a bit more
generous amount of memory to qemu than the defautl 256M. 1G should be
plenty...

```
mandark apps $ cd ..
mandark build $ sed -e 's/-m 256/-m 1024/g' -i tmp/deploy/images/genericx86-64/core-image-flatpak-runtime-genericx86-64.qemuboot.conf
mand
mandark build $ runqemu tmp/deploy/images/genericx86-64/core-image-flatpak-runtime-genericx86-64.qemuboot.conf
```

At this point it is a good idea to make sure qemu can access your HTTP server:
```
mandark build $ sudo iptables -t filter -I INPUT -i 'tap+' -j ACCEPT
```

Let's copy the necessary stuff over to our running qemu instance, then
log in and finish off testing there.

```
mandark build $ cd apps
mandark apps $ scp poky/poky-signing.pub root@192.168.7.2:
mandark apps $ ssh root@192.168.7.2
root@genericx86-64:~# flatpak remote-add --gpg-import=poky-signing.pub poky-apps http://192.168.7.1/flatpak/core-image/apps/
root@genericx86-64:~# flatpak install poky-apps org.poky.vim
...
root@genericx86-64:~# flatpak list
org.poky.vim
root@genericx86-64:~# flatpak run org.poky.vim test-file
...tapety-tap, write some test input, save and exit...
root@genericx86-64:~# cat test-file
The quick brown fox jumps over the lazy dog.
```


## Droppping Into An Image With Flatpak

You can also use flatpak to drop into a shell in an image you built. If
you want to do more complex stuff, such as cross-compile some test code
for your device, use the SDK runtime image (BaseSdk). Otherwise use the
normal runtime image (BasePlatform).

So let's first pull in the BasePlatform (runtime) image. We need to export
our corresponding flatpak repository to be able to do that.

```
mandark build $ cd apps
mandark apps $ sudo cp core-image-flatpak-runtime.flatpak.http.conf /etc/httpd/conf.d
mandark apps $ sudo systemctl restart httpd
mandark apps $ flatpak remote-add --gpg-import=poky/poky-signing.pub core-image-runtime http://127.0.0.1/flatpak/core-image/runtime/
mandark apps $ flatpak install core-image-runtime iot.poky.BasePlatform
```

Let's create a dummy flatpak application that will simply start the default
shell (/usr/bin/sh) from the image for us. This is not strictly necessary
if we have at least one flatpak application installed for our runtime but
it makes things a bit simpler.

```
mandark apps $ mkdir dummy-shell
mandark apps $ cat > dummy-shell/configure
#!/bin/sh
exit 0
^D
mandark apps $ chmod a+x dummy-shell/configure
mandark apps $ cat > dummy-shell/Makefile
all:
install:
clean:
%:
^D
mandark apps $ ./build-app.sh dummy-shell poky
# ... change the command metadata to sh ...
mandark apps $ cat poky/build/dummy-shell/metadata
[Application]
name=org.poky.dummy-shell
runtime=iot.poky.BasePlatform/x86_64/current
sdk=iot.poky.BaseSdk/x86_64/current
command=sh

[Context]
filesystems=home;
```

Armed with the dummy app in our repository, let's pull it in
and try to drop in to a shell in our image.

```
mandark apps $ flatpak install core-image-apps org.poky.dummy-shell
mandark apps $ flatpak run org.poky.dummy-shell
cat /etc/os-release
...
ID="poky"
NAME="Poky (Yocto Project Reference Distro)"
VERSION="2.2+snapshot-20170302 (master)"
VERSION_ID="2.2-snapshot-20170302"
PRETTY_NAME="Poky (Yocto Project Reference Distro)
2.2+snapshot-20170302 (master)"

exit
```

Now, since we have the SDK runtime installed as well, we can now
easily drop into a shell running from the SDK runtime image where
we have the full toolchain available.

```
mandark apps $ flatpak run --devel org.poky.dummy-shell
mandark apps $ echo $SHLVL
3
mandark apps $ cat /etc/os-release
...
ID="poky"
NAME="Poky (Yocto Project Reference Distro)"
VERSION="2.2+snapshot-20170302 (master)"
VERSION_ID="2.2-snapshot-20170302"
PRETTY_NAME="Poky (Yocto Project Reference Distro)
2.2+snapshot-20170302 (master)"
mandark apps $ gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-poky-linux/6.3.0/lto-wrapper
Target: x86_64-poky-linux
Configured with:
../../../../../../work-shared/gcc-6.3.0-r0/gcc-6.3.0/configure
--build=x86_64-linux --host=x86_64-poky-linux
--target=x86_64-poky-linux --prefix=/usr --exec_prefix=/usr
--bindir=/usr/bin --sbindir=/usr/sbin --libexecdir=/usr/libexec
--datadir=/usr/share --sysconfdir=/etc --sharedstatedir=/com
--localstatedir=/var --libdir=/usr/lib --includedir=/usr/include
--oldincludedir=/usr/include --infodir=/usr/share/info
--mandir=/usr/share/man --disable-silent-rules
--disable-dependency-tracking
--with-libtool-sysroot=/storage0/src/users/kli/work/IoT/refkit/poky/poky-flatpak/build/tmp/work/core2-64-poky-linux/gcc/6.3.0-r0/recipe-sysroot
--with-gnu-ld --enable-shared --enable-languages=c,c++
--enable-threads=posix --enable-multilib --enable-c99
--enable-long-long --enable-symvers=gnu --enable-libstdcxx-pch
--program-prefix=x86_64-poky-linux- --without-local-prefix
--enable-lto --enable-libssp --enable-libitm --disable-bootstrap
--disable-libmudflap --with-system-zlib --with-linker-hash-style=gnu
--enable-linker-build-id --with-ppl=no --with-cloog=no
--enable-checking=release --enable-cheaders=c_global --without-isl
--with-sysroot=/
--with-build-sysroot=/storage0/src/users/kli/work/IoT/refkit/poky/poky-flatpak/build/tmp/work/core2-64-poky-linux/gcc/6.3.0-r0/recipe-sysroot
--with-gxx-include-dir=/usr/include/c++/6.3.0
--without-long-double-128 --disable-static --enable-nls
--enable-initfini-array --enable-__cxa_atexit
Thread model: posix
gcc version 6.3.0 (GCC)
```

