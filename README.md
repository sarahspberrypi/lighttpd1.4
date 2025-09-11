# Hermit Port of Lighttpd 1.4

## Build
Build this port within [the Hermit container](https://github.com/hermit-os/hermit-gcc). 
This port relies on cmake and can be build with the following command. 
```bash
cmake -S . -B build-hermit/ --toolchain path/to/x86_64-hermit.cmake -DWITH_PCRE2=OFF -DWITH_LIBDEFLATE=OFF -DWITH_ZLIB=OFF -DBUILD_STATIC=ON

cmake --build build-hermit/
```

## Hermit features necessary
The necessary Hermit features needed for networking and using the server correctly are:
`acpi,dhcpv4,smp,tcp,pci,mman,virtio-net,newlib`
Optionally, `strace` can be included for userspace tracing.

## Toolchain file

The `x86_64-hermit.cmake` toolchain file should look a little like this and assumes that the `libhermit.a` file is located in the `/mnt` directory of the container:
```cmake
set(CMAKE_SYSTEM_NAME Hermit)
set(CMAKE_SYSTEM_PROCESSOR x86_64)

set(CMAKE_C_COMPILER x86_64-hermit-gcc)
set(CMAKE_CXX_COMPILER x86_64-hermit-g++)

# Needed to pass CMake's compiler test during build system generation
set(CMAKE_EXE_LINKER_FLAGS_INIT "-L/mnt/ -fpie -pie -static-pie") # /mnt contains libhermit.a

# Disable shared library building for Hermit
set(BUILD_SHARED_LIBS OFF CACHE BOOL "Build shared libraries" FORCE)

# Additional platform-specific flags
# Hermit requires static linking and specific compiler flags
set(CMAKE_C_FLAGS_INIT "-static-pie")
set(CMAKE_CXX_FLAGS_INIT "-static-pie")

set(CMAKE_CROSSCOMPILING TRUE)

set(WITH_XATTR OFF)
set(WITH_MYSQL OFF)
set(WITH_PGSQL OFF)
set(WITH_DBI OFF)
set(WITH_BORINGSSL OFF)
set(WITH_GNUTLS OFF)
set(WITH_MBEDTLS OFF)
set(WITH_NSS OFF)
set(WITH_OPENSSL OFF)
set(WITH_WOLFSSL OFF)
set(WITH_NETTLE OFF)
set(WITH_PCRE2 OFF)
set(WITH_PCRE OFF)
set(WITH_WEBDAV_PROPS OFF)
set(WITH_WEBDAV_LOCKS OFF)
set(WITH_BROTLI OFF)
set(WITH_BZIP OFF)
set(WITH_ZLIB OFF)
set(WITH_ZSTD OFF)
set(WITH_KRB5 OFF)
set(WITH_LDAP OFF)
set(WITH_PAM OFF)
set(WITH_LUA OFF)
set(WITH_LUA_VERSION OFF)
set(WITH_FAM OFF)
set(WITH_LIBDEFLATE OFF)
set(WITH_LIBUNWIND OFF)
set(WITH_MAXMINDDB OFF)
set(WITH_SASL OFF)
set(WITH_XXHASH OFF)
```


## Example lighttpd config
```bash
# disable these to not have logging for each request and response
debug.log-request-handling = "enable" 
debug.log-request-header = "enable"
debug.log-response-header = "enable"

server.modules += ("mod_status")

server.document-root       = "path/to/document-root"
server.port = 9975   

index-file.names = (
	"test.html",
)

status.status-url = "/server-status"
```

## Mounting the filesystem containing configuration and sites
Virtiofsd is used to mount the files to Hermit, this can be done like this:
```bash
virtiofsd \
    --socket-path=/tmp/vhostqemu \
    --shared-dir ./path/to/mounted/directory \
    --announce-submounts \
    --sandbox none \
    --seccomp none \
    --inode-file-handles=never &
```

**Attention** This implementation is somewhat not working correctly as mentioned in [this issue](https://github.com/hermit-os/kernel/issues/1680). A quick fix is either causing an interrupt (e.g., hitting a button on your keyboard) during runtime when working with 1 vCPU or using multiple vCPUs.

## Running the server
Usernet via virtio-net is used to connect the server and reach it. When using QEMU, this command starts a Hermit lighttpd instance which can then be accessed in the browser:
```bash
qemu-system-x86_64 -smp 1 \
    -m 1G \
    -device isa-debug-exit,iobase=0xf4,iosize=0x04 \
    -display none -serial stdio \
    -chardev socket,id=char0,path=/tmp/vhostqemu \
    -device vhost-user-fs-pci,queue-size=1024,packed=on,chardev=char0,tag="name-for-shared-dir" \
    -object memory-backend-file,id=mem,size=1G,mem-path=/dev/shm,share=on \
    -numa node,memdev=mem \
    -kernel path/to/loader \
    -initrd path/to/hermit-lighttpd \
    -global virtio-mmio.force-legacy=off \
    -netdev user,id=net0,hostfwd=tcp::9975-:9975,hostfwd=udp::9975-:9975,net=192.168.76.0/24,dhcpstart=192.168.76.9 \
    -device virtio-net-pci,netdev=net0,disable-legacy=on,packed=on,mq=on
    -append "-- -D -f path/to/lighttpd.conf"
```

