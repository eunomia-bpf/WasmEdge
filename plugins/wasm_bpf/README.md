# wasm_bpf Plugin

This plugin added six host functions that give you Wasm application access to eBPF.

Six functions are listed here. And all of them are in the module `wasm_bpf`, if you loaded this plugin.

```c
/// lookup a bpf map fd by name.
i32 wasm_bpf_map_fd_by_name(u64 obj, u32 name);
/// detach and close a bpf program.
i32 wasm_close_bpf_object(u64 obj);
/// CO-RE load a bpf object into the kernel.
u64 wasm_load_bpf_object(u32 obj_buf, u32 obj_buf_sz);
/// attach a bpf program to a kernel hook.
i32 wasm_attach_bpf_program(u64 obj, u32 name,
                            u32 attach_target);
/// poll a bpf buffer, and call a wasm callback indicated by sample_func.
/// the first time to call this function will open and create a bpf buffer.
i32 wasm_bpf_buffer_poll(u64 program, i32 fd, u32 sample_func,
                         u32 ctx, u32 data, i32 max_size,
                         i32 timeout_ms);
/// lookup, update, delete, and get_next_key operations on a bpf map.
i32 wasm_bpf_map_operate(u64 fd, i32 cmd, u32 key, u32 value,
                         u32 next_key, u64 flags);
```

- `iXX` denotes signed integer with `XX` bits
- `uXX` denotes unsigned integer with `XX` bits

## How to compile this plugin

### Install dependencies

See the <https://wasmedge.org/book/en/contribute/build_from_src/linux.html> for how to build `WasmEdge` from source.

```sh
apt update && apt install pkg-config libelf1 libelf-dev zlib1g-dev
```

#### libbpf

This plugin requires `libbpf >= 1.2`

Follow [https://github.com/libbpf/libbpf#building-libbpf](https://github.com/libbpf/libbpf#building-libbpf) to build and install `libbpf`.

### Build `wasm_bpf` plug-in

Run the following commands at the root of the `WasmEdge` project:

- Note: It's important to set `WASMEDGE_PLUGIN_WASM_BPF` to `TRUE` in the command line. This toggle controls the build of `wasm_bpf` plugin.

```sh
cmake -DWASMEDGE_PLUGIN_WASM_BPF:BOOL=TRUE -B ./build -G "Unix Makefiles" -DWASMEDGE_LINK_PLUGINS_STATIC=true
cmake --build ./build --config Release --target install -j
```

## How to use this plugin

You can either download the examples or build them by yourself.

### Download the examples

```sh
wget https://eunomia-bpf.github.io/wasm-bpf/examples/runqlat/runqlat.wasm
```

### build the examples

Examples of wasm-bpf programs can be found in [wasm-bpf](https://github.com/eunomia-bpf/wasm-bpf/tree/main/examples) repo. You can build them by running the following commands:

```sh
# install the wasi-sdk if you don't have it
wget https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-17/wasi-sdk-17.0-linux.tar.gz
tar -zxf wasi-sdk-17.0-linux.tar.gz
sudo mkdir -p /opt/wasi-sdk/ && sudo mv wasi-sdk-17.0/* /opt/wasi-sdk/

# build the examples
git clone https://github.com/eunomia-bpf/wasm-bpf
cd wasm-bpf/examples
git submodule update --init --recursive

# for example, build the execve example
cd execve && make
```

All examples are:

```console
$ ls
bootstrap  execve  go-execve  go-lsm  lsm   opensnoop runqlat  rust-bootstrap  sockfilter  sockops
```

### run the examples

After building, you can find the plug-in `./build/plugins/wasm_bpf/libwasmedgePluginWasmBpf.so` and the WasmEdge CLI tool at `./build/tools/wasmedge/wasmedge`.

Set `WASMEDGE_PLUGIN_PATH=./build/plugins/wasm_bpf/` and run wasmedge:

```console
# WASMEDGE_PLUGIN_PATH=./build/plugins/wasm_bpf/ ./build/tools/wasmedge/wasmedge execve.wasm 

[289150] node -> /bin/sh -c which ps 
[289151] sh -> which ps 
[289152] node -> /bin/sh -c /usr/bin/ps -ax -o pid=,ppid=,pcpu=,pmem=,c 
[289153] sh -> /usr/bin/ps -ax -o pid=,ppid=,pcpu=,pmem=,command= 
[289154] node -> /bin/sh -c "/root/.vscode-server-insiders/bin/96a795cc 
[289155] sh -> /root/.vscode-server-insiders/bin/96a795cc0 245632 245678 289148 
[289156] cpuUsage.sh -> sed -n s/^cpu\s//p /proc/stat 
[289157] cpuUsage.sh -> cat /proc/245632/stat 
[289158] cpuUsage.sh -> cat /proc/245678/stat 
[289159] cpuUsage.sh -> cat /proc/289148/stat 
[289160] cpuUsage.sh -> sleep 1 
^C
```

## Run wasm-bpf plugin with podman

Use our prebuild container image:

```console
$ docker run --privileged --rm -it ghcr.io/eunomia-bpf/wasm-podman 
# podman --runtime /usr/local/bin/crun run --privileged  --rm -it --platform=wasi/wasm -v /runqlat.wasm:/runqlat.wasm  -v /libwasmedgePluginWasmBpf.so:/libwasmedgePluginWasmBpf.so -v /libbpf.so:/libbpf.so -v /usr/lib/x86_64-linux-gnu/:/usr/lib/x86_64-linux-gnu/ docker.io/wasmedge/example-wasi:latest /runqlat.wasm
Trying to pull docker.io/wasmedge/example-wasi:latest...
Getting image source signatures
Copying blob 5cf93dcbdcd8 skipped: already exists  
Copying config 332aed9d05 done  
Writing manifest to image destination
Storing signatures

Tracing run queue latency... Hit Ctrl-C to end.

     usecs               : count    distribution
         0 -> 1          : 51       |*******************                     |
         2 -> 3          : 74       |****************************            |
         4 -> 7          : 103      |****************************************|
         8 -> 15         : 63       |************************                |
        16 -> 31         : 38       |**************                          |
        32 -> 63         : 11       |****                                    |
        64 -> 127        : 3        |*                                       |
```

Note: in some cases, it will report `Error: OCI runtime error: crun-wasm: the requested cgroup controller pids is not available`. Retry the command will temporary solve this issue.

## Build Mannually

See the privious doc to build the plugin.

Install crun:

```sh
apt install -y make git gcc build-essential pkgconf libtool \
    libsystemd-dev libprotobuf-c-dev libcap-dev libseccomp-dev libyajl-dev \
    go-md2man libtool autoconf python3 automake podman
podman system reset
```

Clone and run crun

```sh
git clone https://github.com/eunomia-bpf/crun
cd crun && git checkout enable_plugin
./autogen.sh
./configure --with-wasmedge
make -j && make install

# in WasmEdge dir
cp build/plugins/wasm_bpf/*.so /
wget https://eunomia-bpf.github.io/wasm-bpf/examples/runqlat/runqlat.wasm
podman --runtime /usr/local/bin/crun run --privileged  --rm -it --platform=wasi/wasm -v /runqlat.wasm:/runqlat.wasm  -v /libwasmedgePluginWasmBpf.so:/libwasmedgePluginWasmBpf.so -v /libbpf.so:/libbpf.so -v /usr/lib/x86_64-linux-gnu/:/usr/lib/x86_64-linux-gnu/ docker.io/wasmedge/example-wasi:latest /runqlat.wasm

# kill
podman ps
podman kill xxx 
```

Dockerfile

```Dockerfile
FROM scratch
# Add the TensorFlow lite library as the WASI-NN plugin dependency.
ADD libbpf.so /
# Add WasmEdge WASI-NN Plugin
ADD libwasmedgePluginWasmBpf.so /

# This is for the container entry, give a hint to users that they will need to spec>
CMD ["/runqlat.wasm"]  
```

refer to: <https://github.com/hydai/wasm-container-images/tree/master/tflite-bird-v1>
