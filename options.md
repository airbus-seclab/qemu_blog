# A deep dive into QEMU: Giving and adding options

In this blog post we will see how we can give QEMU access to a file in
the host that represents the code flash backend through the `-drive`
option. We will also detail how to add new options to the QEMU command
line.

## A block backend for the code flash

The CPIOM board we are simulating has a code flash memory that holds
the bootloader and the operating system firmware. It is the first
software components executed by the CPU and must be available from the
boot.

Obviously, the strategy is to have bootloader and OS firmware stored
in a file representing the code flash memory layout, and expose it to
QEMU as a block device backend. We can use the `-drive` option:

```shell
-drive file=/path/to/flash.code,format=raw,id=mxfc-flash-code
```

The QEMU API will connect the host file `/path/to/flash.code` to a QOM
device whose `id` is `mxfc-flash-code`, not to be confused with a QOM
device type we use to define when implementing new devices.

The flash component emulation code will make use of the QEMU block
device API to look for this given `id`:

```c
#define MXFC_FLASH_BLK_NAME "mxfc-flash-code"

blk = blk_by_name(MXFC_FLASH_BLK_NAME);
if (blk == NULL) {
    cpiom_error("no flash code block backend found\n");
}

/* We look into block backend API to access "-drive=" option. We
 * should use blk_pread(), blk_pwrite() to access data.  Here we
 * directly mmap() the file into memory for faster access.
 */
BlockDriverState *bs = blk_bs(blk);

fd = open(bs->filename, O_RDONLY);
if (fd < 0) {
    cpiom_error("can't open block backend: %s\n", bs->filename);
}
```

And that's it ! We got our file access. The last step is to expose the
file content thanks to the QEMU memory region API. QEMU is able to
populate RAM memory regions from file descriptors:

```c
memory_region_init_ram_from_fd(&mem, owner, MXFC_FLASH_BLK_NAME"-mmap",
                               size, false, fd, &error_fatal);
```


## Adding a new option


### Option definition

At some point, you may want to add specific option to the QEMU command
line. The options definition is located at
[`qemu-options.hx`](https://github.com/qemu/qemu/blob/v4.2.0/qemu-options.hx). Below
is an example `-gustave <jsonfile>` option:

```shell
DEF("gustave", HAS_ARG, QEMU_OPTION_gustave, \
    "-gustave file   GUSTAVE fuzzer JSON configuration filename\n", QEMU_ARCH_ALL)
STEXI
@item -gustave @var{file}
@findex -gustave
GUSTAVE fuzzer JSON configuration filename.
ETEXI
```

This file is used by the build system to generate `qemu-options.def`
before building the qemu image. The previous `gustave` option looks
like:

```shell
DEF("gustave", HAS_ARG, QEMU_OPTION_gustave, \
"-gustave file   GUSTAVE fuzzer JSON configuration filename\n", QEMU_ARCH_ALL)
```

### Option registering

The options parser is located at
[vl.c](https://github.com/qemu/qemu/blob/v4.2.0/vl.c) which is QEMU
main C file. Options are regrouped in `QemuOptsList`:

```c
static QemuOptsList qemu_gustave_opts = {
    .name = "gustave",
    .implied_opt_name = "gustave",
    .head = QTAILQ_HEAD_INITIALIZER(qemu_gustave_opts.head),
    .merge_lists = true,
    .desc = {
        { /* end of list */ }
    },
};
```

And you add these new options with: `qemu_add_opts(&qemu_gustave_opts);` in QEMU [`main`](https://github.com/qemu/qemu/blob/v4.2.0/vl.c#L2880).


## Option parsing

As you might imagine, QEMU provides you with a complete API to handle
your options. You can navigate through the linked list of a specific
`QemuOptsList`, but you can also look for option by string such as:

```c
const char *fname;
fname = qemu_opt_get(qemu_find_opts_singleton("gustave"), "gustave");
```

It's just that simple. By the way our option takes a JSON file name
and QEMU comes with a
[`JSON`](https://github.com/qemu/qemu/blob/v4.2.0/qobject/qjson.c)
parser easily backed by
[`QDict`](https://github.com/qemu/qemu/blob/v4.2.0/include/qapi/qmp/qdict.h)
objects if you want to have a look at. Just give file content to:

```c
obj  = qobject_from_json(content, NULL);
dict = qobject_to(QDict, obj);

obj   = qdict_get(dict, "key");
value = qnum_get_uint(qobject_to(QNum, obj));
```
