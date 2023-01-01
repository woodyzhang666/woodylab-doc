# Arm Trusted Firmware

## Build

There are many build macros configurable via `make` command line. The default
values are set in `make_helpers/defaults.mk`.

## Features

### Branch Protection

There are 4 kinds of branch protection methods for aarch64: 
- standard
	Enable both BTI and PAUTH.
- pac-ret
	Use only PAUTH
- pac-ret+leaf
- bti

### RME
### GENERATE_COT
Chain Of Trust

### DEBUGFS
### DECRYPTION_SUPPORT
### SPD
Secure Payload Dispatcher, part of BL31. Under services/spd/.
AA64 only.
- opteed

### SUPPORT_STACK_MEMTAG
Depends on ARMv8.5 MTE.

### Realm Management Extension

## Drivers

#### Console

```c
typedef struct console {
    struct console *next;	/* linked onto console_list after registration */
    /*
     * Only the low 32 bits are used. The type is u_register_t to align the
     * fields of the struct to 64 bits in AArch64 and 32 bits in AArch32
     */
    u_register_t flags;

    int (*const putc)(int character, struct console *console);
    int (*const getc)(struct console *console);
    void (*const flush)(struct console *console);

    uintptr_t base;
    /* Additional private driver data may follow here. */
} console_t;
```
- flags
	- CONSOLE_FLAG_BOOT
	- CONSOLE_FLAG_RUNTIME
	- CONSOLE_FLAG_CRASH
	- CONSOLE_FLAG_TRANSLATE_CRLF

```c
```

## BL2

Parameter Header embedded in other data structures.
```c
typedef struct param_header {
    uint8_t type;       /* type of the structure */
    uint8_t version;    /* version of this structure */
    uint16_t size;      /* size of this structure in bytes */
    uint32_t attr;      /* attributes: unused bits SBZ */
} param_header_t;
```
- `type`
	PARAM_EP
		Entrypoint
	PARAM_IMAGE_BINARY
	PARAM_BL31
	PARAM_BL_LOAD_INFO
	PARAM_BL_PARAMS
	PARAM_PSCI_LIB_ARGS
	PARAM_SP_IMAGE_BOOT_INFO

```c
typedef struct entry_point_info {
    param_header_t h;
    uintptr_t pc;
    uint32_t spsr;
#ifdef __aarch64__
    aapcs64_params_t args;
#else
    uintptr_t lr_svc;
    aapcs32_params_t args;
#endif
} entry_point_info_t;
```

```c
typedef struct image_info {
    param_header_t h;
    uintptr_t image_base;   /* physical address of base of image */
    uint32_t image_size;    /* bytes read from image file */
    uint32_t image_max_size;
} image_info_t;
```

Parameters for a BL
```c
typedef struct bl_params_node {
    unsigned int image_id;
    image_info_t *image_info;
    entry_point_info_t *ep_info;
    struct bl_params_node *next_params_info;
} bl_params_node_t;

typedef struct bl_params {
    param_header_t h;
    bl_params_node_t *head;
} bl_params_t;
```

```c
typedef struct image_desc {
    /* Contains unique image id for the image. */
    unsigned int image_id;
    /*
     * This member contains Image state information.
     * Refer IMAGE_STATE_XXX defined above.
     */
    unsigned int state;
    uint32_t copied_size;   /* image size copied in blocks */
    image_info_t image_info;
    entry_point_info_t ep_info;
} image_desc_t;
```

```c
typedef struct bl_load_info_node {
    unsigned int image_id;
    image_info_t *image_info;
    struct bl_load_info_node *next_load_info;
} bl_load_info_node_t;

typedef struct bl_load_info {
    param_header_t h;
    bl_load_info_node_t *head;
} bl_load_info_t;
```

#### Flow

BL2 starts from `bl2_entrypoint` with arguments saved in r0-r3 from previous
bootloader.
In most platforms, Bootrom code loads BL2.

- `bl2_entrypoint`
	- `el3_entrypoint_common`
		handle reset path, cpu initialization, sets up reset vector and
		prepares c environment.
	- `bl2_el3_setup`
		- `bl2_el3_early_platform_setup`
		- `bl2_el3_plat_arch_setup`
	- `bl2_main`

```c
```

```c
```
