# Maintainer: Red Arch Engineer
pkgbase=linux-red-arch-kernel
pkgname=(linux-red-arch-kernel linux-red-arch-kernel-headers)
pkgver=6.12.37
pkgrel=1
pkgdesc="An even better kernel stitched for Red arch/Arch-based distros"
arch=(x86_64)
url="https://www.kernel.org"
license=(GPL2)
depends=(coreutils kmod initramfs)
makedepends=(
  bc pahole cpio perl tar xz git
  clang lld llvm
  xmlto docbook-xsl python
)
options=(!strip)

source=(
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-${pkgver}.tar.xz"
)
sha256sums=(
  'SKIP'
)

prepare() {
  cd "linux-${pkgver}"

  # 1. Apply BORE Scheduler Patch using the exact stable 6.12.37 patch path
  msg2 "Applying BORE scheduler patch..."
  patch -Np1 < "$startdir/bore-scheduler-1.4.12/patches/stable/0001-linux6.12.37-bore-7.0.0.patch"

  # 2. Safely target KBUILD_CFLAGS for -O3 optimization without breaking host tools
  msg2 "Safely patching KBUILD_CFLAGS to enforce -O3..."
  sed -i 's/KBUILD_CFLAGS\s*+= -O2/KBUILD_CFLAGS += -O3/g' Makefile

  # 3. Use your currently working running kernel config as the infallible hardware base
  msg2 "Injecting active hardware config to ensure all drivers (NVMe, Btrfs, Wi-Fi) work..."
  zcat /proc/config.gz > .config

  # 4. Apply Maximum Performance, Zero-Latency Kconfig Flags & Custom Localversion
  msg2 "Applying Maximum Performance & Zero-Latency Kconfig Flags..."
  
  # Set distinct local version to avoid module directory collisions
  ./scripts/config --set-str LOCALVERSION "-red-arch"

  # Compiler & LTO
  ./scripts/config --enable LTO_CLANG_THIN
  ./scripts/config --disable LTO_NONE
  ./scripts/config --disable CFI_CLANG

  # Scheduling, Preemption, RCU Boosting & Latency Elimination
  ./scripts/config --enable SCHED_BORE
  ./scripts/config --enable SCHED_CLASS_EXT
  ./scripts/config --enable BPF_SYSCALL
  ./scripts/config --enable BPF_JIT
  ./scripts/config --enable BPF_JIT_ALWAYS_ON
  ./scripts/config --enable PREEMPT
  ./scripts/config --disable PREEMPT_VOLUNTARY
  ./scripts/config --disable PREEMPT_NONE
  ./scripts/config --enable HZ_1000
  ./scripts/config --set-val HZ 1000
  ./scripts/config --enable NO_HZ_IDLE
  ./scripts/config --enable RCU_BOOST
  ./scripts/config --set-val RCU_BOOST_DELAY 500

  # Lightning-Fast ZSTD Kernel Boot Compression
  ./scripts/config --disable KERNEL_GZIP
  ./scripts/config --disable KERNEL_XZ
  ./scripts/config --enable KERNEL_ZSTD

  # Default CPU Frequency Governor -> Performance
  ./scripts/config --enable CPU_FREQ_DEFAULT_GOV_PERFORMANCE
  ./scripts/config --disable CPU_FREQ_DEFAULT_GOV_SCHEDUTIL
  ./scripts/config --disable CPU_FREQ_DEFAULT_GOV_POWERSAVE

  # Memory Management (THP Always, MGLRU, ZRAM + ZSTD)
  ./scripts/config --enable TRANSPARENT_HUGEPAGE
  ./scripts/config --enable TRANSPARENT_HUGEPAGE_ALWAYS
  ./scripts/config --disable TRANSPARENT_HUGEPAGE_MADVISE
  ./scripts/config --enable LRU_GEN
  ./scripts/config --enable LRU_GEN_ENABLED
  ./scripts/config --module ZRAM
  ./scripts/config --enable ZRAM_DEF_COMP_ZSTD
  ./scripts/config --enable ZSTD_COMPRESS

  # Block I/O & Networking (Kyber/Deadline WBT, TCP BBR + Fair Queue Default)
  ./scripts/config --enable BLK_WBT
  ./scripts/config --enable BLK_WBT_MQ
  ./scripts/config --enable MQ_IOSCHED_KYBER
  ./scripts/config --enable MQ_IOSCHED_DEADLINE
  ./scripts/config --enable TCP_CONG_ADVANCED
  ./scripts/config --enable TCP_CONG_BBR
  ./scripts/config --enable DEFAULT_BBR
  ./scripts/config --set-str DEFAULT_TCP_CONG "bbr"
  ./scripts/config --enable NET_SCH_FQ
  ./scripts/config --enable DEFAULT_FQ
  ./scripts/config --set-str DEFAULT_NET_SCH "fq"

  # Extreme Bloat, Debug & Latency Overhead Removal
  ./scripts/config --disable DEBUG_KERNEL
  ./scripts/config --disable DEBUG_INFO
  ./scripts/config --enable DEBUG_INFO_NONE
  ./scripts/config --disable SCHED_DEBUG
  ./scripts/config --disable SLUB_DEBUG
  ./scripts/config --disable FTRACE
  ./scripts/config --disable FUNCTION_TRACER
  ./scripts/config --disable STACK_TRACER
  ./scripts/config --disable CPU_MITIGATIONS
  ./scripts/config --disable DEBUG_PREEMPT
  ./scripts/config --disable LATENCYTOP
  ./scripts/config --disable DEBUG_OBJECTS

  make LLVM=1 LLVM_IAS=1 olddefconfig
  cat .config > ../kernel.config
}

build() {
  cd "linux-${pkgver}"
  
  export KCFLAGS="-march=x86-64-v2 -mtune=generic"
  export KCPPFLAGS="-march=x86-64-v2 -mtune=generic"
  export MAKEFLAGS="-j$(nproc)"

  msg2 "Compiling Red-Arch-Kernel with Clang ThinLTO and -O3..."
  make LLVM=1 LLVM_IAS=1 all
}

package_linux-red-arch-kernel() {
  pkgdesc="An even better kernel stitched for Red arch/Arch-based distros"
  depends=(coreutils kmod initramfs)
  optdepends=('wireless-regdb: to set the correct wireless channels')
  provides=(linux=$pkgver)

  cd "linux-${pkgver}"
  
  local kernver="$(make kernelrelease)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  msg2 "Installing kernel image..."
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  msg2 "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  rm -f "$modulesdir"/{source,build}

  msg2 "Installing boot configuration..."
  install -Dm644 ../kernel.config "$modulesdir/config"
}

package_linux-red-arch-kernel-headers() {
  pkgdesc="Header files and scripts for building modules for Red-Arch-Kernel"
  depends=(pahole)

  cd "linux-${pkgver}"
  local kernver="$(make kernelrelease)"
  local srcdir="$pkgdir/usr/src/linux-$kernver"

  msg2 "Installing complete kernel headers for DKMS support..."
  install -dm755 "$srcdir"
  
  cp Makefile Module.symvers System.map vmlinux.h 2>/dev/null || true
  cp -r include arch/x86/include scripts "$srcdir/"
  
  if [ -d tools/objtool ]; then
    install -dm755 "$srcdir/tools/objtool"
    cp -r tools/objtool "$srcdir/tools/"
  fi
  
  cp .config "$srcdir/"
  
  find "$srcdir" -type f -name "*.o" -delete

  install -dm755 "$pkgdir/usr/lib/modules/$kernver"
  ln -sr "$srcdir" "$pkgdir/usr/lib/modules/$kernver/build"
}
