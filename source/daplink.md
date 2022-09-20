# DAPLink

## Build System

DAPLink uses "project-generator" as its build system, which parses yaml files
to get sources and toolchain definitions and replace the placeholders in 
corresponding makefile template. The quick build command is
`progen generate -f <yaml> -t make_gcc_arm <proj> -b`. If `-f <yaml>` is not
given, ${curdir}/projects.yaml is used. If `<proj>` is not given, all prjects
list in yaml are built.

The root yaml usually consists of at least two dicts:
`settings` and `projects`.

- `projects`
	 A list of dicts containing supported project and a related yaml files
	 tree. i.e. a recursive list. A list of yaml files for some specific
	 function constructs a module. A module can also include one or more
	 modules.

- `settings`
	Used to fill a ProjectSettings object. Supported subitems:

	- `export_dir`
		Output dir sets here. Could be a format string.

	- `root`
		Root directory of the project. defaults to cwd.

	- `tools`
		Give "gcc", "uvision" or "iar" paths or "template" path.
		For example:
		````
		settings:
		  tools:
		    gcc:
			  path:
			    - /path/to/toolchain/
		````

In DAPlink projects.yaml, 
- `modules`
	lists common sets of `.yaml` groups. A module can be included by a project
	or another module. `modules` are used to modularize sources and
	configurations.


Generator class: Created from a projects yaml.
- `projects_dict`
	The dict parsed from root yaml
- `workspaces`
- `settings`
	The ProjectSettings object filled from "settings" of root yaml

For more details about parsing .yaml, see `project_generator` code and
`https://github.com/project-generator/project_generator/wiki/Getting_started`

DAPLink firmware consists of two binaries: a bootloader and an usb interface.
The bootloader usually locates at the beginning of ROM. It initializes the usb
slave as MSC(Mass Storage Class) formated as FAT fs. User could write bin file
to flash a new interface binary or modify settings through exported plain text
files in it. 

In normal bootup, the bootloader loads the interface binary if available,
otherwise it enters bootloader mode and works as a MSC. User could force enter
bootloader mode by asserting nRST pin to flash new interface binary.

Both bootloader and interface sources are divided into common files and board
specific files. The common files are grouped into module `bl` and `if`. They
include the same tools module, i.e. compilation, linking flags and scripts.

```
    tools: &module_tools
        - records/tools/gcc_arm.yaml
        - records/tools/armcc.yaml
        - records/tools/armclang.yaml
        - records/tools/version.yaml
    bl: &module_bl
        - *module_tools
        - records/usb/usb-core.yaml
        - records/usb/usb-msc.yaml
        - records/daplink/bootloader.yaml
        - records/daplink/drag-n-drop.yaml
        - records/daplink/settings.yaml
        - records/daplink/settings_rom_stub.yaml
        - records/daplink/target_board.yaml
        - records/rtos/rtos-none.yaml
    if: &module_if
        - *module_tools
        - records/usb/usb-core.yaml
        - records/usb/usb-hid.yaml
        - records/usb/usb-msc.yaml
        - records/usb/usb-cdc.yaml
        - records/usb/usb-webusb.yaml
        - records/usb/usb-winusb.yaml
        - records/daplink/cmsis-dap.yaml
        - records/daplink/drag-n-drop.yaml
        - records/daplink/usb2uart.yaml
        - records/daplink/settings.yaml
        - records/daplink/settings_rom.yaml
        - records/daplink/interface.yaml
        - records/daplink/target_family.yaml
        - records/daplink/target_board.yaml
```

As shown above, bootloader only creates a usb msc interface.
Note:
- Common `if` module excludes usb bulk support. It can be included to other
	modules if required.
- `records/usb/usb-core.yaml` includes usb core in `source/usb/usbd_core.c`.
	It coopearated with hal specific usb implementations.
- Other `records/usb/usb-xxx.yaml` defines `XXX_ENDPOINT` and includes sources
	under `source/usb/xxx`.

### Linker Script

As we know that binaries for cortex-m starts with interrupt vectors which must
be placed at chip-specific address. Bootloader is just like the normal binary.
But Interface image should be put elsewhere and bootloader jumps there after
setting vbar properly.

Bootloader and Interface both use following linker script, but they have
different macro definitions. Macros are defined in board specific header
`daplink_addr.h`.

```c
#include "daplink_addr.h"
#include "daplink_defaults.h"

/* Entry Point */
ENTRY(Reset_Handler)

HEAP_SIZE  = DEFINED(__heap_size__)  ? __heap_size__  : DAPLINK_HEAP_SIZE;
STACK_SIZE = DEFINED(__stack_size__) ? __stack_size__ : DAPLINK_STACK_SIZE;

/* Specify the memory areas */
MEMORY
{
  m_interrupts          (RX)  : ORIGIN = DAPLINK_ROM_APP_START, LENGTH = 0x400
  m_text                (RX)  : ORIGIN = DAPLINK_ROM_APP_START + 0x400, LENGTH = DAPLINK_ROM_APP_SIZE - 0x400
  m_cfgrom              (RW)  : ORIGIN = DAPLINK_ROM_CONFIG_USER_START, LENGTH = DAPLINK_ROM_CONFIG_USER_SIZE
  m_data                (RW)  : ORIGIN = DAPLINK_RAM_APP_START, LENGTH = DAPLINK_RAM_APP_SIZE
  m_cfgram              (RW)  : ORIGIN = DAPLINK_RAM_SHARED_START, LENGTH = DAPLINK_RAM_SHARED_SIZE
}

/* Define output sections */
SECTIONS
{
  /* The startup code goes first into internal flash */
  .interrupts :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector))     /* Startup code */
    FILL(0xffffffff)
    . = ALIGN(4);
    . += LENGTH(m_interrupts) - (. - ORIGIN(m_interrupts)); /* pad out to end of m_interrupts */
  } > m_interrupts

  /* The program code and other data goes into internal flash */
  .text :
  {
    . = ALIGN(4);
    *(.text)                 /* .text sections (code) */
    *(.text*)                /* .text* sections (code) */
    *(.rodata)               /* .rodata sections (constants, strings, etc.) */
    *(.rodata*)              /* .rodata* sections (constants, strings, etc.) */
    *(.glue_7)               /* glue arm to thumb code */
    *(.glue_7t)              /* glue thumb to arm code */
    *(.eh_frame)
    KEEP (*(.init))
    KEEP (*(.fini))
    . = ALIGN(4);
  } > m_text

  .ARM.extab :
  {
    *(.ARM.extab* .gnu.linkonce.armextab.*)
  } > m_text

  .ARM :
  {
    __exidx_start = .;
    *(.ARM.exidx*)
    __exidx_end = .;
  } > m_text

 .ctors :
  {
    __CTOR_LIST__ = .;
    /* gcc uses crtbegin.o to find the start of
       the constructors, so we make sure it is
       first.  Because this is a wildcard, it
       doesn't matter if the user does not
       actually link against crtbegin.o; the
       linker won't look for a file to match a
       wildcard.  The wildcard also means that it
       doesn't matter which directory crtbegin.o
       is in.  */
    KEEP (*crtbegin.o(.ctors))
    KEEP (*crtbegin?.o(.ctors))
    /* We don't want to include the .ctor section from
       from the crtend.o file until after the sorted ctors.
       The .ctor section from the crtend file contains the
       end of ctors marker and it must be last */
    KEEP (*(EXCLUDE_FILE(*crtend?.o *crtend.o) .ctors))
    KEEP (*(SORT(.ctors.*)))
    KEEP (*(.ctors))
    __CTOR_END__ = .;
  } > m_text

  .dtors :
  {
    __DTOR_LIST__ = .;
    KEEP (*crtbegin.o(.dtors))
    KEEP (*crtbegin?.o(.dtors))
    KEEP (*(EXCLUDE_FILE(*crtend?.o *crtend.o) .dtors))
    KEEP (*(SORT(.dtors.*)))
    KEEP (*(.dtors))
    __DTOR_END__ = .;
  } > m_text

  .preinit_array :
  {
    PROVIDE_HIDDEN (__preinit_array_start = .);
    KEEP (*(.preinit_array*))
    PROVIDE_HIDDEN (__preinit_array_end = .);
  } > m_text

  .init_array :
  {
    PROVIDE_HIDDEN (__init_array_start = .);
    KEEP (*(SORT(.init_array.*)))
    KEEP (*(.init_array*))
    PROVIDE_HIDDEN (__init_array_end = .);
  } > m_text

  .fini_array :
  {
    PROVIDE_HIDDEN (__fini_array_start = .);
    KEEP (*(SORT(.fini_array.*)))
    KEEP (*(.fini_array*))
    PROVIDE_HIDDEN (__fini_array_end = .);
  } > m_text

  __etext = .;    /* define a global symbol at end of code */
  __DATA_ROM = .; /* Symbol is used by startup for data initialization */

  .data : AT(__DATA_ROM)
  {
    . = ALIGN(4);
    __DATA_RAM = .;
    __data_start__ = .;      /* create a global symbol at data start */
    *(.data)                 /* .data sections */
    *(.data*)                /* .data* sections */
    KEEP(*(.jcr*))
    . = ALIGN(4);
    __data_end__ = .;        /* define a global symbol at data end */
  } > m_data

  __DATA_END = __DATA_ROM + (__data_end__ - __data_start__);

  /* fill .text out to the end of the app */
  .fill __DATA_END :
  {
    FILL(0xffffffff)
    . = ALIGN(4);
    . += DAPLINK_ROM_APP_START + DAPLINK_ROM_APP_SIZE - __DATA_END - 4;
    /* Need some contents in this section or it won't be copied to bin or hex. The CRC will
     * be placed here by post_build_script.py. */
    LONG(0x55555555)
  } > m_text

  text_end = ORIGIN(m_text) + LENGTH(m_text);
  ASSERT(__DATA_END <= text_end, "region m_text overflowed with text and data")

  .cfgrom (NOLOAD) :
  {
    *(cfgrom)
  } > m_cfgrom

  /* Uninitialized data section */
  .bss :
  {
    /* This is used by the startup in order to initialize the .bss section */
    . = ALIGN(4);
    __START_BSS = .;
    __bss_start__ = .;
    *(.bss)
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
    __bss_end__ = .;
    __END_BSS = .;
  } > m_data

  .heap :
  {
    . = ALIGN(8);
    __end__ = .;
    PROVIDE(end = .);
    __HeapBase = .;
    . += HEAP_SIZE;
    __HeapLimit = .;
    __heap_limit = .; /* Add for _sbrk */
  } > m_data

  .stack :
  {
    . = ALIGN(8);
    . += STACK_SIZE;
  } > m_data

  .cfgram (NOLOAD) :
  {
    *(cfgram)
  } > m_cfgram

  /* Initializes stack on the end of block */
  __StackTop   = ORIGIN(m_data) + LENGTH(m_data);
  __StackLimit = __StackTop - STACK_SIZE;
  PROVIDE(__stack = __StackTop);

  .ARM.attributes 0 : { *(.ARM.attributes) }

  ASSERT(__StackLimit >= __HeapLimit, "region overflowed with stack and heap")
}
```

### Config Settings

As we could see in linker script, there are two section cfgrom and cfgram for
config settings. Both contain configuration values accessed through vfs files.

- cfgram is retained after jumping from bootloader to interface or reset from
interface to bootloader.

```c
typedef struct __attribute__((__packed__)) cfg_ram {
    uint32_t key;               // Magic key to indicate a valid record
    uint16_t size;              // Offset of the last member from the start

    uint8_t hold_in_bl;
    char assert_file_name[64 + 1];
    uint16_t assert_line;
    uint8_t assert_source;

    // Additional debug information on faults
    uint8_t  valid_dumps;
    uint32_t hexdump[ALLOWED_HEXDUMP];  //Alignments checked

    // Disable msd support
    uint8_t disable_msd;

    //Add new entries from here
    uint8_t page_erase_enable;
} cfg_ram_t;
```
- `hold_in_bl`
	This is checked before bl jumping into interface. It set, stay in bl mode.

- cfgrom

## Bootloader

Bootloader is an individual small binary booted naturally. Bootloader checks
whether it can boot the Interface. Usually it can if Interface binary is ok.
But user could force it staying bootloader mode by pulling a button(pin) or
setting `hold_in_bl` in cfgram (last time).

In bootloader mode, it provides a usb msc device from which user could flash
Interface binary.
There is no os in bootloader mode to save resources.

```c
```

## USB Protocol

##### String Descriptor
```c
static uint32_t host_id[4];		/* usually universal unique chip id */
static uint32_t target_id[4];
static uint32_t hic_id = DAPLINK_HIC_ID;

static uint32_t crc_bootloader;
static uint32_t crc_interface;
static uint32_t crc_config_user;

// Strings
static char string_unique_id[48 + 1];	/* board_id + family_id + host_id + hic_id */
static char string_mac[12 + 1];
static char string_board_id[4 + 1];
static char string_family_id[4 + 1];
static char string_host_id[32 + 1];		/* host id in hex32 mode */
static char string_target_id[32 + 1];
static char string_hic_id[8 + 1];		/* hex32 mode */
static char string_version[4 + 1];

/* usb string desc(type 3) reported to master */
static char usb_desc_unique_id[2 + sizeof(string_unique_id) * 2];
```

##### Config Descriptor

`USBD_ConfigDescriptor`, `USBD_ConfigDescriptor_HS` contains the config
descriptor followed by multiple interface descriptors and their endpoints
descriptors.

```c
    const U8 start_desc[] = {
        /* Configuration 1 */
        USB_CONFIGUARTION_DESC_SIZE,                // bLength
        USB_CONFIGURATION_DESCRIPTOR_TYPE,          // bDescriptorType
        WBVAL(USBD_WTOTALLENGTH_MAX),               // wTotalLength
        USBD_IF_NUM_MAX,                            // bNumInterfaces
        0x01,                                       // bConfigurationValue: 0x01 is used to select this configuration
        0x00,                                       // iConfiguration: no string to describe this configuration
        USBD_CFGDESC_BMATTRIBUTES |                 // bmAttributes
        (USBD_POWER << 6),
        USBD_CFGDESC_BMAXPOWER                      // bMaxPower, device power consumption
    };
```

##### Interface Descriptors

```c
#define MSC_DESC                                                                                            \
/* Interface, Alternate Setting 0, MSC Class */                                                             \
  USB_INTERFACE_DESC_SIZE,              /* bLength */                                                       \
  USB_INTERFACE_DESCRIPTOR_TYPE,        /* bDescriptorType */                                               \
  0x00,                                 /* bInterfaceNumber USBD_MSC_IF_NUM*/                               \
  0x00,                                 /* bAlternateSetting */                                             \
  0x02,                                 /* bNumEndpoints */                                                 \
  USB_DEVICE_CLASS_STORAGE,             /* bInterfaceClass */                                               \
  MSC_SUBCLASS_SCSI,                    /* bInterfaceSubClass */                                            \
  MSC_PROTOCOL_BULK_ONLY,               /* bInterfaceProtocol */                                            \
  USBD_MSC_IF_STR_NUM,                  /* iInterface */

#define MSC_EP                          /* MSC Endpoints for Low-speed/Full-speed */                        \
  /* Endpoint, EP Bulk IN */                                                                                \
  USB_ENDPOINT_DESC_SIZE,               /* bLength */                                                       \
  USB_ENDPOINT_DESCRIPTOR_TYPE,         /* bDescriptorType */                                               \
  USB_ENDPOINT_IN(USBD_MSC_EP_BULKIN),  /* bEndpointAddress */                                              \
  USB_ENDPOINT_TYPE_BULK,               /* bmAttributes */                                                  \
  WBVAL(USBD_MSC_WMAXPACKETSIZE),       /* wMaxPacketSize */                                                \
  0x00,                                 /* bInterval: ignore for Bulk transfer */                           \
                                                                                                            \
  /* Endpoint, EP Bulk OUT */                                                                               \
  USB_ENDPOINT_DESC_SIZE,               /* bLength */                                                       \
  USB_ENDPOINT_DESCRIPTOR_TYPE,         /* bDescriptorType */                                               \
  USB_ENDPOINT_OUT(USBD_MSC_EP_BULKOUT),/* bEndpointAddress */                                              \
  USB_ENDPOINT_TYPE_BULK,               /* bmAttributes */                                                  \
  WBVAL(USBD_MSC_WMAXPACKETSIZE),       /* wMaxPacketSize */                                                \
  0x00,                                 /* bInterval: ignore for Bulk transfer */

#define MSC_EP_HS                       /* MSC Endpoints for High-speed */                                  \
  /* Endpoint, EP Bulk IN */                                                                                \
  USB_ENDPOINT_DESC_SIZE,               /* bLength */                                                       \
  USB_ENDPOINT_DESCRIPTOR_TYPE,         /* bDescriptorType */                                               \
  USB_ENDPOINT_IN(USBD_MSC_EP_BULKIN),  /* bEndpointAddress */                                              \
  USB_ENDPOINT_TYPE_BULK,               /* bmAttributes */                                                  \
  WBVAL(USBD_MSC_HS_WMAXPACKETSIZE),    /* wMaxPacketSize */                                                \
  USBD_MSC_HS_BINTERVAL,                /* bInterval */                                                     \
                                                                                                            \
  /* Endpoint, EP Bulk OUT */                                                                               \
  USB_ENDPOINT_DESC_SIZE,               /* bLength */                                                       \
  USB_ENDPOINT_DESCRIPTOR_TYPE,         /* bDescriptorType */                                               \
  USB_ENDPOINT_OUT(USBD_MSC_EP_BULKOUT),/* bEndpointAddress */                                              \
  USB_ENDPOINT_TYPE_BULK,               /* bmAttributes */                                                  \
  WBVAL(USBD_MSC_HS_WMAXPACKETSIZE),    /* wMaxPacketSize */                                                \
  USBD_MSC_HS_BINTERVAL,                /* bInterval */
```

usb core data
```c
U16 USBD_DeviceStatus;
U8 USBD_DeviceAddress;
U8 USBD_Configuration;
U32 USBD_EndPointMask;
U32 USBD_EndPointHalt;
U32 USBD_EndPointStall;          /* EP must stay stalled */
U8 USBD_NumInterfaces;
U8 USBD_HighSpeed;
U8 USBD_ZLP;

USBD_EP_DATA USBD_EP0Data;
USB_SETUP_PACKET USBD_SetupPacket;
```



```c
```
