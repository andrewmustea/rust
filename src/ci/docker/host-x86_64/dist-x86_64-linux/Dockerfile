# We document platform support for minimum glibc 2.17 and kernel 3.2.
# CentOS 7 has headers for kernel 3.10, but that's fine as long as we don't
# actually use newer APIs in rustc or std without a fallback. It's more
# important that we match glibc for ELF symbol versioning.
FROM centos:7

WORKDIR /build

RUN yum upgrade -y && \
    yum install -y epel-release && \
    yum install -y \
      automake \
      bzip2 \
      file \
      gcc \
      gcc-c++ \
      git \
      glibc-devel.i686 \
      glibc-devel.x86_64 \
      libedit-devel \
      libstdc++-devel.i686 \
      libstdc++-devel.x86_64 \
      make \
      ncurses-devel \
      openssl-devel \
      patch \
      perl \
      pkgconfig \
      python3 \
      unzip \
      wget \
      xz \
      zlib-devel.i686 \
      zlib-devel.x86_64 \
      && yum clean all

RUN mkdir -p /rustroot/bin

ENV PATH=/rustroot/bin:$PATH
ENV LD_LIBRARY_PATH=/rustroot/lib64:/rustroot/lib32:/rustroot/lib
ENV PKG_CONFIG_PATH=/rustroot/lib/pkgconfig
WORKDIR /tmp
RUN mkdir /home/user
COPY host-x86_64/dist-x86_64-linux/shared.sh /tmp/

# Need at least GCC 5.1 to compile LLVM nowadays
COPY host-x86_64/dist-x86_64-linux/build-gcc.sh /tmp/
RUN ./build-gcc.sh && yum remove -y gcc gcc-c++

# LLVM 17 needs cmake 3.20 or higher.
COPY scripts/cmake.sh /tmp/
RUN ./cmake.sh

# Now build LLVM+Clang, afterwards configuring further compilations to use the
# clang/clang++ compilers.
COPY host-x86_64/dist-x86_64-linux/build-clang.sh /tmp/
RUN ./build-clang.sh
ENV CC=clang CXX=clang++

# rustc-perf version from 2023-05-30
# Should also be changed in the opt-dist tool for other environments.
ENV PERF_COMMIT 8b2ac3042e1ff2c0074455a0a3618adef97156b1
RUN curl -LS -o perf.zip https://ci-mirrors.rust-lang.org/rustc/rustc-perf-$PERF_COMMIT.zip && \
    unzip perf.zip && \
    mv rustc-perf-$PERF_COMMIT rustc-perf && \
    rm perf.zip

COPY scripts/sccache.sh /scripts/
RUN sh /scripts/sccache.sh

ENV PGO_HOST=x86_64-unknown-linux-gnu

ENV HOSTS=x86_64-unknown-linux-gnu

ENV RUST_CONFIGURE_ARGS \
      --enable-full-tools \
      --enable-sanitizers \
      --enable-profiler \
      --enable-compiler-docs \
      --set target.x86_64-unknown-linux-gnu.linker=clang \
      --set target.x86_64-unknown-linux-gnu.ar=/rustroot/bin/llvm-ar \
      --set target.x86_64-unknown-linux-gnu.ranlib=/rustroot/bin/llvm-ranlib \
      --set llvm.thin-lto=true \
      --set llvm.ninja=false \
      --set rust.jemalloc \
      --set rust.use-lld=true \
      --set rust.lto=thin

ENV SCRIPT python3 ../x.py build --set rust.debug=true opt-dist && \
    ./build/$HOSTS/stage0-tools-bin/opt-dist linux-ci -- python3 ../x.py dist \
    --host $HOSTS --target $HOSTS \
    --include-default-paths \
    build-manifest bootstrap
ENV CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=clang

# This is the only builder which will create source tarballs
ENV DIST_SRC 1

# When we build cargo in this container, we don't want it to use the system
# libcurl, instead it should compile its own.
ENV LIBCURL_NO_PKG_CONFIG 1

ENV DIST_REQUIRE_ALL_TOOLS 1
