# Custom QEMU RISC-V Virt Machine Linux firmware build

## Assumptions

The following assumptions are valid during this tutorial:
- You're probably familiar with concepts like "Linux from scratch", because
  this is essentially a guide about building your own RISC-V linux firmware.
  I never followed the entire "Linux from scratch" guide book but I used
  some tips on it to tweak some parts.
- You're ready for troubleshooting many problems on your own - this is the real deal...
  sadly, while this guide works perfectly as I write it now, it might stop work tomorrow.
- Imagination is your best friend here. While I showed a minimal set of shell commands to get
  to a working state, you can customize things further (in kernel, userspace, etc).
- I compiled everything on my Archlinux machine, but technically this should work on any
  Linux distribution, provided you installed the needed utilities on your host machine.

With that being said, let's get this to work!

## Install basic toolchain

On my Arch-based Linux system I did:
```sh
pacman -S gcc make flex bison autoconf automake
```

Other Linux distributions' users should find these packages easily in their package manager.

## Cross compile musl-backed GCC for RISC-V

Start by entering the build environment directory.

Then start by compiling the toolchain:
```sh
git clone https://github.com/supercomputer7/musl-cross-make.git
pushd musl-cross-make
TARGET=riscv64-linux-musl make
export OUTPUT=output/
TARGET=riscv64-linux-musl make install
popd

mkdir -p rootfs/lib/
cp -r musl-cross-make/output/riscv64-linux-musl/lib/* rootfs/lib/
```

And apply a new file to preserve the following toolchain settings:
```sh
cat > use_toolchain <<EOF
export ENV_DIR=$(pwd)/
export MUSL_ENV_DIR="$(pwd)/musl-cross-make/output/"
export CFLAGS="-I$ENV_DIR/rootfs/usr/include"
export LDFLAGS="-L$ENV_DIR/rootfs/lib/ -L$ENV_DIR/rootfs/usr/lib/ -L$ENV_DIR/rootfs/usr/local/lib/" 
export AR=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-ar
export AS=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-as 
export CXX=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-cpp 
export CC=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-cc
export LD=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-ld 
export NM=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-gcc-nm
export RANLIB=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-gcc-ranlib 
export STRIP=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-strip 
export OBJCOPY=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-objcopy 
export OBJDUMP=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-objdump 
export READELF=$(pwd)/musl-cross-make/output/bin/riscv64-linux-musl-readelf
export PATH=$(pwd)/musl-cross-make/output/bin/:$PATH
EOF
```

## rootfs directory creation

If for some reason you don't have the rootfs directory already:
```sh
mkdir -p $ENV_DIR/rootfs/
```

## Kernel compilation

Use the toolchain settings if not used already:
```sh
source use_toolchain
```

Then start by downloading and compiling the kernel:
```sh
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.6.4.tar.xz
tar xvf linux-6.6.4.tar.xz
pushd linux-6.6.4
make ARCH=riscv CROSS_COMPILE=riscv64-linux-musl- defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-linux-musl- -j $(nproc)
make O=$ENV_DIR/linux-6.6.4-headers headers_install
popd
```

Then install the headers in the appropriate directories:
```sh
mkdir -p $ENV_DIR/rootfs/usr/include
cp -r $ENV_DIR/linux-6.6.4-headers/usr/include/* $ENV_DIR/rootfs/usr/include/
cp -r $ENV_DIR/linux-6.6.4-headers/usr/include/* $ENV_DIR/musl-cross-make/output/riscv64-linux-musl/include/
```

## Basic rootfs creation (with simple init program)

Run the following commands to create a basic rootfs layout:
```sh
mkdir -p $ENV_DIR/rootfs/{bin,dev,etc,home,mnt,proc,sys,usr}

mkdir -p $ENV_DIR/rootfs/usr/local/lib/

ln -s /usr/local/lib/ $ENV_DIR/rootfs/lib
ln -s /usr/local/lib/ $ENV_DIR/rootfs/usr/lib

cp $ENV_DIR/musl-cross-make/build/local/riscv64-linux-musl/obj_sysroot/lib/* $ENV_DIR/rootfs/usr/local/lib/
chmod +x $ENV_DIR/rootfs/usr/local/lib/libc.so
ln -sf /lib/libc.so $ENV_DIR/rootfs/usr/local/lib/ld-musl-riscv64.so.1

mkdir -p $ENV_DIR/rootfs/etc/init.d/

cat > $ENV_DIR/rootfs/etc/init.d/rcS <<EOF
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys

mkdir -p /dev/pts
mount -t devpts devpts /dev/pts

ip link set eth0 up
ip addr add dev eth0 192.168.76.9/24
ip r add default via 192.168.76.2

exec /bin/sh
EOF

chmod +x $ENV_DIR/rootfs/etc/init.d/rcS
mkdir -p $ENV_DIR/rootfs/sbin
ln -sf /linuxrc $ENV_DIR/rootfs/sbin/init
```

## Build minimal set of util-linux binaries

Use the toolchain settings if not used already:
```sh
source use_toolchain
```

Then compile:
```sh
git clone https://github.com/util-linux/util-linux.git --depth 1
pushd util-linux

./autogen.sh
./configure --host=riscv64-linux-musl --target=riscv64-linux-musl --prefix=$ENV_DIR/rootfs/usr/local --with-sysroot=$ENV_DIR/rootfs/ --without-systemd --without-btrfs --without-udev --without-ncursesw --without-tinfo --without-readline --without-cap-ng --without-libz --without-libmagic --without-python --disable-widechar

make -j$(nproc)
sudo make install
popd
```

## Busybox static compilation

Use the toolchain settings if not used already:
```sh
source use_toolchain
```

Busybox should be compiled easily with the following commands:
```sh
git clone https://git.busybox.net/busybox --depth 1
pushd busybox
CROSS_COMPILE=riscv64-linux-musl- make defconfig
EXTRA_LDFLAGS='-static' CROSS_COMPILE=riscv64-linux-musl- make -j $(nproc) install
popd
cp -r busybox/_install/* $ENV_DIR/rootfs/
```


## Optional: Build native libraries for RISC-V binutils programs

Use the toolchain settings if not used already:
```sh
source use_toolchain
```

Then compile:
```sh
wget https://ftp.gnu.org/gnu/binutils/binutils-2.41.tar.xz
tar xvf binutils-2.41.tar.xz

pushd binutils-2.41
./configure --with-sysroot=/ --with-build-sysroot=$MUSL_ENV_DIR/ --host=riscv64-linux-musl --target=riscv64-linux-musl --disable-nls --disable-werror --prefix=$ENV_DIR/rootfs/usr/

make -j$(nproc)
sudo make install
popd
```

## Optional: Create SSH daemon with Golang SSH gliderlabs

You might completely skip this and go over to the `Compile & install dropbear` section
if you want a more standard SSH server.

Use the toolchain settings if not used already:
```sh
source use_toolchain
```

You will need to acquire a Golang compiler.
You can do this by running the following commands:
```sh
wget https://go.dev/dl/go1.21.4.linux-amd64.tar.gz
mkdir golang-toolchain
tar -C golang-toolchain -xzf go1.21.4.linux-amd64.tar.gz
```

Then clone the Golang SSH gliderlabs repository:

```sh
git clone https://github.com/gliderlabs/ssh.git
```

You may want to tweak the `_examples/ssh-pty/pty.go` with your preferences (such as listening port, prefered shell binary).
I recommend adding the `PATH=/usr/local/bin:/usr/local/sbin:/usr/bin:/sbin:/bin` string to the execution environment in the SSH server code.

Then, run the following commands:
```sh
export GOOS=linux
export GOARCH=riscv64
export CGO_ENABLED=1
export CC=riscv64-linux-musl-gcc
pushd ssh
$ENV_DIR/golang-toolchain/go/bin/go get github.com/creack/pty
$ENV_DIR/golang-toolchain/go/bin/go build _examples/ssh-pty/pty.go
popd
```

Then copy the program to the rootfs:
```sh
cp -f $ENV_DIR/ssh/pty $ENV_DIR/rootfs/bin/go-sshd
chmod +x $ENV_DIR/rootfs/bin/go-sshd
ln -sf /bin/go-sshd $ENV_DIR/rootfs/bin/sshd
```

Please note that you can invoke execution of this in `/etc/init.d/rcS` in the rootfs image.
Also, `devpts` must be mounted (as the example in this guide shows) on `/dev/pts` to ensure
proper PTY functionality.

In addition to that, please note that on each reboot you must remove the host key from your known hosts file.
Otherwise, you can connect with ssh with something like this:
```sh
ssh -o "StrictHostKeyChecking=no" -p 2222 root@127.0.0.1
```

If this is undesirable, you might need to consider other SSH servers. Please look
at the `Compile & install dropbear` section then.

## Optional: Compile & install dropbear

You might completely skip the `Create SSH daemon with Golang SSH gliderlabs` section
and install dropbear instead.
You should consider adding `dropbear` to the `/etc/init.d/rcS` file.

### Compile & install zlib

Dropbear depends on `zlib`, so we need to compile and install it first:
```sh
wget https://www.zlib.net/zlib-1.3.tar.gz
tar xvf zlib-1.3.tar.gz
pushd zlib-1.3
export LDSHARED="$CC -shared -Wl,-soname,libz.so"
./configure --prefix=$ENV_DIR/rootfs/usr/local

make -j$(nproc)
sudo make install
popd
```

### Compile & Install dropbear

Compile and install dropbear:
```sh
wget https://mirror.dropbear.nl/mirror/releases/dropbear-2022.82.tar.bz2
tar xvf dropbear-2022.82.tar.bz2

pushd dropbear-2022.82
./configure --host=riscv64-linux-musl --prefix=$ENV_DIR/rootfs/usr/local --disable-zlib --disable-utmp --disable-wtmp --disable-lastlog --disable-syslog

make -j$(nproc)
sudo make install
popd
```

### Before rootfs image updating - install necessary hooks

Update the rootfs image. We need to generate an account directory for root 
and ensure `dropbear` can generate host keys on first run:
```sh
mkdir -p $ENV_DIR/rootfs/root
chmod 600 $ENV_DIR/rootfs/root

mkdir -p $ENV_DIR/rootfs/etc/dropbear

sed -i '4s/^/mount -o remount,rw \/\n/' $ENV_DIR/rootfs/etc/init.d/rcS

echo "root:root:0:0::/root:/bin/sh" > $ENV_DIR/rootfs/etc/passwd
```

Generate a root password:
```sh
openssl passwd -6 -salt xyz PASSWORD
```

Then generate the `/etc/shadow` file, with the password hash:
```sh
echo 'root:PUT_HASH_HERE:19699::::::' > $ENV_DIR/rootfs/etc/shadow
sudo chmod 600 $ENV_DIR/rootfs/etc/shadow
```

You might also want to change `19699` to something else (this value is for the last password change in days since January 1, 1970).

### Install hostkey files

After invoking the QEMU VM instance, simply run `/usr/local/sbin/dropbear -R`, then wait.

## Update the rootfs image

```sh
rm -f $ENV_DIR/rootfs.img
mke2fs -d $ENV_DIR/rootfs/ -L '' -N 0 -O ^64bit -m 5 -r 1 -t ext2 rootfs.img 3G
```

Please note that you might need to use `sudo` if you generate a rootfs image
with `/root` being created.

# Boot!

```sh
qemu-system-riscv64 -nographic -machine virt \
                    -kernel linux-6.6.4/arch/riscv/boot/Image \
                    -append "console=ttyS0 root=/dev/vda" \
                    -m 2048 \
                    -device e1000e,netdev=net0 \
                    -netdev user,id=net0,hostfwd=tcp::2222-:22,net=192.168.76.0/24,dhcpstart=192.168.76.9 \
                    -drive file=$ENV_DIR/rootfs.img,format=raw,id=hd0 \
                    -device virtio-blk-device,drive=hd0
```

# Sources & Credits

[1] Build basic Linux+Busybox QEMU RISC-V image - https://github.com/darchr/riscv-full-system/blob/main/qemu.md

[2] Optional Nice & Dumb DHCP client I used during writing of this guide, just to get the allocated IP and the router IP from the DHCP server, on the screen - https://github.com/anryko/daft-dhcp-client

[3] Cross-compile GCC toolchain with Musl-libc - https://github.com/richfelker/musl-cross-make
