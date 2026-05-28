
# SONAME

In Linux, the dynamic linker identifies shared libraries primarily by their SONAME (`DT_SONAME`). In practice, SONAME acts as the logical identity of the library inside the process.

This leads to an important behavior:

Suppose we have:

```text
./temp/libB
./temp2/libB
./temp2/libA
```

where `libA` depends on `libB`.

Assume:

* `libA` has `RUNPATH=$ORIGIN`
* both `libB` copies have the same SONAME
* both libraries export the same symbols

At first glance, it may seem that loading `./temp2/libA` should resolve `libB` from `./temp2/libB` because of `$ORIGIN`.

However, this is not guaranteed.

If:

1. `./temp/libB` is loaded first
2. then `./temp2/libB` is loaded
3. then `./temp2/libA` is loaded

then `libA` may still bind against the already loaded `./temp/libB`.

This happens because the linker first checks already loaded libraries matching the required dependency identity (SONAME/scope rules) before resolving through `RUNPATH`.

As a result:

* copying libraries into different directories is not sufficient isolation
* `RUNPATH` alone does not guarantee dependency separation
* ELF dependency resolution is process-global unless isolated with mechanisms such as `dlmopen`

This behavior was experimentally confirmed using duplicated library trees and loader tracing (`LD_DEBUG=libs,bindings`).

# Debug
Here are ways to run the program with ld debug in order to see what library is getting loaded and unloaded. Might help a lot while working with plugins
```sh
LD_DEBUG=bindings,libs ./main 2>&1 | grep process_param_get
LD_DEBUG=libs,files,bindings ./main > out.txt 2>&1
```

Here is an example ld output which shows a bug
```txt
24900: binding file ./temp2/autopilot_deps/libautopilot.so [0] to ./temp/autopilot_deps/libvt45.so [0]: normal symbol process_param_get'
```
