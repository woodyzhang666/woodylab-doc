
## SEC

## PEI


`EFI_SEC_PEI_HAND_OFF` structure holds information about
PEI core's operating environment, such as the size of location of
temporary RAM, the stack location and BFV location.
```c
typedef struct _EFI_SEC_PEI_HAND_OFF {
  UINT16    DataSize;		/* Size of the data structure. */

  VOID      *BootFirmwareVolumeBase;
  UINTN     BootFirmwareVolumeSize;

  VOID      *TemporaryRamBase;
  UINTN     TemporaryRamSize;

  VOID     *PeiTemporaryRamBase;
  UINTN    PeiTemporaryRamSize;

  VOID     *StackBase;
  UINTN    StackSize;
} EFI_SEC_PEI_HAND_OFF;
```
- `BootFirmwareVolumeBase`, `BootFirmwareVolumeSize`
  Points to the first byte of the boot firmware volume,
  which the PEI Dispatcher should search for
  PEI modules.
  
- `TemporaryRamBase`, `TemporaryRamSize`
	the temporary ram which is safe to use.

- `PeiTemporaryRamBase`, `PeiTemporaryRamSize`
  Temporary RAM available for use by the PEI Foundation. The area
  described by `PeiTemporaryRamBase` and `PeiTemporaryRamSize`
  must not extend outside beyond the area described by
  `TemporaryRamBase` and `TemporaryRamSize`. This area should not
  overlap with the area reported by `StackBase` and `StackSize`.
  
- `StackBase`, `StackSize`
  This are may be part of the memory described by `TemporaryRamBase` and
  `TemporaryRamSize` or may be an entirely separate area.
  

SEC may also pass a list of PPI descriptions for PEI to get platform
information. PPI is a method whose usage is predefined ans identified by
GUID.
```c
typedef struct {
  UINTN       Flags;
  
  EFI_GUID    *Guid;

  VOID        *Ppi;
} EFI_PEI_PPI_DESCRIPTOR;
```
- Flags
	This field is a set of flags describing the characteristics of this imported table entry.
	All flags are defined as EFI_PEI_PPI_DESCRIPTOR_***, which can also be combined into one.

	- `EFI_PEI_PPI_DESCRIPTOR_PIC`
	- `EFI_PEI_PPI_DESCRIPTOR_PPI`
		The descriptor is a PPI.
	- `EFI_PEI_PPI_DESCRIPTOR_NOTIFY_CALLBACK`
	- `EFI_PEI_PPI_DESCRIPTOR_NOTIFY_DISPATCH`
	- `EFI_PEI_PPI_DESCRIPTOR_NOTIFY_TYPES`
	- `EFI_PEI_PPI_DESCRIPTOR_TERMINATE_LIST`
		Set on the last PPI in a list.

- Guid
	The address of the EFI_GUID that names the interface.

- Ppi
	A pointer to the PPI. The Guid determines what it is and how it is used.


Pei Core private data structure instance. A structure defined in stack as
there is no heap available?.
```c
struct _PEI_CORE_INSTANCE {
  UINTN                             Signature;	/* 'PeiC' */

  EFI_PEI_SERVICES                  *Ps;	/* points to the copy on stack */
  PEI_PPI_DATABASE                  PpiData;

  ///
  /// The count of FVs which contains FFS and could be dispatched by PeiCore.
  ///
  UINTN                             FvCount;

  ///
  /// The max count of FVs which contains FFS and could be dispatched by PeiCore.
  ///
  UINTN                             MaxFvCount;

  ///
  /// Pointer to the buffer with the MaxFvCount number of entries.
  /// Each entry is for one FV which contains FFS and could be dispatched by PeiCore.
  ///
  PEI_CORE_FV_HANDLE                *Fv;

  ///
  /// Pointer to the buffer with the MaxUnknownFvInfoCount number of entries.
  /// Each entry is for one FV which could not be dispatched by PeiCore.
  ///
  PEI_CORE_UNKNOW_FORMAT_FV_INFO    *UnknownFvInfo;
  UINTN                             MaxUnknownFvInfoCount;
  UINTN                             UnknownFvInfoCount;

  ///
  /// Pointer to the buffer FvFileHandlers in PEI_CORE_FV_HANDLE specified by CurrentPeimFvCount.
  ///
  EFI_PEI_FILE_HANDLE               *CurrentFvFileHandles;
  UINTN                             AprioriCount;
  UINTN                             CurrentPeimFvCount;
  UINTN                             CurrentPeimCount;
  EFI_PEI_FILE_HANDLE               CurrentFileHandle;
  BOOLEAN                           PeimNeedingDispatch;
  BOOLEAN                           PeimDispatchOnThisPass;
  BOOLEAN                           PeimDispatcherReenter;
  EFI_PEI_HOB_POINTERS              HobList;
  BOOLEAN                           SwitchStackSignal;
  BOOLEAN                           PeiMemoryInstalled;
  VOID                              *CpuIo;
  EFI_PEI_SECURITY2_PPI             *PrivateSecurityPpi;
  EFI_PEI_SERVICES                  ServiceTableShadow;	/* a copy on stack */
  EFI_PEI_PPI_DESCRIPTOR            *XipLoadFile;
  EFI_PHYSICAL_ADDRESS              PhysicalMemoryBegin;
  UINT64                            PhysicalMemoryLength;
  EFI_PHYSICAL_ADDRESS              FreePhysicalMemoryTop;
  UINTN                             HeapOffset;
  BOOLEAN                           HeapOffsetPositive;
  UINTN                             StackOffset;
  BOOLEAN                           StackOffsetPositive;
  //
  // Information for migrating memory pages allocated in pre-memory phase.
  //
  HOLE_MEMORY_DATA                  MemoryPages;
  PEICORE_FUNCTION_POINTER          ShadowedPeiCore;
  CACHE_SECTION_DATA                CacheSection;
  //
  // For Loading modules at fixed address feature to cache the top address below which the
  // Runtime code, boot time code and PEI memory will be placed. Please note that the offset between this field
  // and Ps should not be changed since maybe user could get this top address by using the offset to Ps.
  //
  EFI_PHYSICAL_ADDRESS              LoadModuleAtFixAddressTopAddress;
  //
  // The field is define for Loading modules at fixed address feature to tracker the PEI code
  // memory range usage. It is a bit mapped array in which every bit indicates the corresponding memory page
  // available or not.
  //
  UINT64                            *PeiCodeMemoryRangeUsageBitMap;
  //
  // This field points to the shadowed image read function
  //
  PE_COFF_LOADER_READ_FILE          ShadowedImageRead;

  UINTN                             TempPeimCount;

  //
  // Pointer to the temp buffer with the TempPeimCount number of entries.
  //
  EFI_PEI_FILE_HANDLE               *TempFileHandles;
  //
  // Pointer to the temp buffer with the TempPeimCount number of entries.
  //
  EFI_GUID                          *TempFileGuid;

  //
  // Temp Memory Range is not covered by PeiTempMem and Stack.
  // Those Memory Range will be migrated into physical memory.
  //
  HOLE_MEMORY_DATA                  HoleData[HOLE_MAX_NUMBER];
};
typedef struct _PEI_CORE_INSTANCE PEI_CORE_INSTANCE;
```
- `ServiceTableShadow`, `Ps`
	The `PEI_CORE_INSTANCE` is a structure on stack. The `EFI_PEI_SERVICES`
	needs to be copy to a modifiable memory.
	`Ps` is the pointer of active `EFI_PEI_SERVICES`. Address of `Ps` (`&Ps`)
	is stored to architectural registers so that PEIM could access it easily.
	For ARM platforms, it's stored in TPID register.

- `HobList`
	Track the Hob pointers. The start of Hob list is the base of PEI
	temporary ram.

- `PeiMemoryInstalled`
	Whether memory is initialized by PEIM


`EFI_PEI_SERVICES` is a collection of functions whose implementation is
provided by the PEI Foundation. These services fall into various classes,
including the following:
- Managing the boot mode
- Allocating both early and permanent memory
- Supporting the Firmware File System (FFS)
- Abstracting the PPI database abstraction
- Creating Hand-Off Blocks (HOBs).
```c
struct _EFI_PEI_SERVICES {
  ///
  /// The table header for the PEI Services Table.
  ///
  EFI_TABLE_HEADER                  Hdr;

  //
  // PPI Functions
  //
  EFI_PEI_INSTALL_PPI               InstallPpi;
  EFI_PEI_REINSTALL_PPI             ReInstallPpi;
  EFI_PEI_LOCATE_PPI                LocatePpi;
  EFI_PEI_NOTIFY_PPI                NotifyPpi;

  //
  // Boot Mode Functions
  //
  EFI_PEI_GET_BOOT_MODE             GetBootMode;
  EFI_PEI_SET_BOOT_MODE             SetBootMode;

  //
  // HOB Functions
  //
  EFI_PEI_GET_HOB_LIST              GetHobList;
  EFI_PEI_CREATE_HOB                CreateHob;

  //
  // Firmware Volume Functions
  //
  EFI_PEI_FFS_FIND_NEXT_VOLUME2     FfsFindNextVolume;
  EFI_PEI_FFS_FIND_NEXT_FILE2       FfsFindNextFile;
  EFI_PEI_FFS_FIND_SECTION_DATA2    FfsFindSectionData;

  //
  // PEI Memory Functions
  //
  EFI_PEI_INSTALL_PEI_MEMORY        InstallPeiMemory;
  EFI_PEI_ALLOCATE_PAGES            AllocatePages;
  EFI_PEI_ALLOCATE_POOL             AllocatePool;
  EFI_PEI_COPY_MEM                  CopyMem;
  EFI_PEI_SET_MEM                   SetMem;

  //
  // Status Code
  //
  EFI_PEI_REPORT_STATUS_CODE        ReportStatusCode;

  //
  // Reset
  //
  EFI_PEI_RESET_SYSTEM              ResetSystem;

  //
  // (the following interfaces are installed by publishing PEIM)
  // I/O Abstractions
  //
  EFI_PEI_CPU_IO_PPI                *CpuIo;
  EFI_PEI_PCI_CFG2_PPI              *PciCfg;

  //
  // Future Installed Services
  //
  EFI_PEI_FFS_FIND_BY_NAME          FfsFindFileByName;
  EFI_PEI_FFS_GET_FILE_INFO         FfsGetFileInfo;
  EFI_PEI_FFS_GET_VOLUME_INFO       FfsGetVolumeInfo;
  EFI_PEI_REGISTER_FOR_SHADOW       RegisterForShadow;
  EFI_PEI_FFS_FIND_SECTION_DATA3    FindSectionData3;
  EFI_PEI_FFS_GET_FILE_INFO2        FfsGetFileInfo2;
  EFI_PEI_RESET2_SYSTEM             ResetSystem2;
  EFI_PEI_FREE_PAGES                FreePages;
};
typedef struct _EFI_PEI_SERVICES EFI_PEI_SERVICES;
```

```c
```

##### Hob

```c
typedef union {
  EFI_HOB_GENERIC_HEADER                 *Header;
  EFI_HOB_HANDOFF_INFO_TABLE             *HandoffInformationTable;
  EFI_HOB_MEMORY_ALLOCATION              *MemoryAllocation;
  EFI_HOB_MEMORY_ALLOCATION_BSP_STORE    *MemoryAllocationBspStore;
  EFI_HOB_MEMORY_ALLOCATION_STACK        *MemoryAllocationStack;
  EFI_HOB_MEMORY_ALLOCATION_MODULE       *MemoryAllocationModule;
  EFI_HOB_RESOURCE_DESCRIPTOR            *ResourceDescriptor;
  EFI_HOB_GUID_TYPE                      *Guid;
  EFI_HOB_FIRMWARE_VOLUME                *FirmwareVolume;
  EFI_HOB_FIRMWARE_VOLUME2               *FirmwareVolume2;
  EFI_HOB_FIRMWARE_VOLUME3               *FirmwareVolume3;
  EFI_HOB_CPU                            *Cpu;
  EFI_HOB_MEMORY_POOL                    *Pool;
  EFI_HOB_UEFI_CAPSULE                   *Capsule;
  UINT8                                  *Raw;
} EFI_PEI_HOB_POINTERS;
```
Hob list is put at the bottom of PEI temporary ram.

```c
typedef struct {
  UINT16    HobType;
  UINT16    HobLength;
  UINT32    Reserved; /// This field must always be set to zero.
} EFI_HOB_GENERIC_HEADER;
```
- HobType
	EFI_HOB_TYPE_HANDOFF              
		This hob passes boot mode and memory info to next phase.
	EFI_HOB_TYPE_MEMORY_ALLOCATION    
	EFI_HOB_TYPE_RESOURCE_DESCRIPTOR  
	EFI_HOB_TYPE_GUID_EXTENSION       
	EFI_HOB_TYPE_FV                   
	EFI_HOB_TYPE_CPU                  
	EFI_HOB_TYPE_MEMORY_POOL          
	EFI_HOB_TYPE_FV2                  
	EFI_HOB_TYPE_LOAD_PEIM_UNUSED     
	EFI_HOB_TYPE_UEFI_CAPSULE         
	EFI_HOB_TYPE_FV3                  
	EFI_HOB_TYPE_UNUSED               
	EFI_HOB_TYPE_END_OF_HOB_LIST      

```c
typedef struct {
  EFI_HOB_GENERIC_HEADER    Header;
  UINT32                    Version;
  EFI_BOOT_MODE             BootMode;
  EFI_PHYSICAL_ADDRESS      EfiMemoryTop;
  EFI_PHYSICAL_ADDRESS      EfiMemoryBottom;

  EFI_PHYSICAL_ADDRESS      EfiFreeMemoryTop;
  EFI_PHYSICAL_ADDRESS      EfiFreeMemoryBottom;
  EFI_PHYSICAL_ADDRESS      EfiEndOfHobList;
} EFI_HOB_HANDOFF_INFO_TABLE;
```
- BootMode
  The system boot mode as determined during the HOB producer phase.

- EfiMemoryTop, EfiMemoryBottom
	memory that is allocated for use by the HOB producer phase. This address
	must be 4-KB aligned to meet page restrictions of UEFI.
  
- EfiFreeMemoryTop, EfiFreeMemoryBottom
  free memory that is currently available for use by the HOB producer phase.
  Memory occupied by Hobs are carved out. Everytime a new hob is created, the
  bottom of free memory steps upwards.
  
- EfiEndOfHobList
	Address of the last Hob of type `EFI_HOB_TYPE_END_OF_HOB_LIST`.
	The last Hob will be reset when creating new Hob or Hoblist.

```c
typedef struct {
  ///
  /// A GUID that defines the memory allocation region's type and purpose, as well as
  /// other fields within the memory allocation HOB. This GUID is used to define the
  /// additional data within the HOB that may be present for the memory allocation HOB.
  /// Type EFI_GUID is defined in InstallProtocolInterface() in the UEFI 2.0
  /// specification.
  ///
  EFI_GUID                Name;

  ///
  /// The base address of memory allocated by this HOB. Type
  /// EFI_PHYSICAL_ADDRESS is defined in AllocatePages() in the UEFI 2.0
  /// specification.
  ///
  EFI_PHYSICAL_ADDRESS    MemoryBaseAddress;

  ///
  /// The length in bytes of memory allocated by this HOB.
  ///
  UINT64                  MemoryLength;

  ///
  /// Defines the type of memory allocated by this HOB. The memory type definition
  /// follows the EFI_MEMORY_TYPE definition. Type EFI_MEMORY_TYPE is defined
  /// in AllocatePages() in the UEFI 2.0 specification.
  ///
  EFI_MEMORY_TYPE         MemoryType;

  ///
  /// Padding for Itanium processor family
  ///
  UINT8                   Reserved[4];
} EFI_HOB_MEMORY_ALLOCATION_HEADER;

typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_MEMORY_ALLOCATION.
  ///
  EFI_HOB_GENERIC_HEADER              Header;
  ///
  /// An instance of the EFI_HOB_MEMORY_ALLOCATION_HEADER that describes the
  /// various attributes of the logical memory allocation.
  ///
  EFI_HOB_MEMORY_ALLOCATION_HEADER    AllocDescriptor;
  //
  // Additional data pertaining to the "Name" Guid memory
  // may go here.
  //
} EFI_HOB_MEMORY_ALLOCATION;
```

Describes the memory stack that is produced by the HOB producer
phase and upon which all post-memory-installed executable
content in the HOB producer phase is executing.
```c
typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_MEMORY_ALLOCATION.
  ///
  EFI_HOB_GENERIC_HEADER              Header;
  ///
  /// An instance of the EFI_HOB_MEMORY_ALLOCATION_HEADER that describes the
  /// various attributes of the logical memory allocation.
  ///
  EFI_HOB_MEMORY_ALLOCATION_HEADER    AllocDescriptor;
} EFI_HOB_MEMORY_ALLOCATION_STACK;

///
/// Defines the location of the boot-strap
/// processor (BSP) BSPStore ("Backing Store Pointer Store").
/// This HOB is valid for the Itanium processor family only
/// register overflow store.
///
typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_MEMORY_ALLOCATION.
  ///
  EFI_HOB_GENERIC_HEADER              Header;
  ///
  /// An instance of the EFI_HOB_MEMORY_ALLOCATION_HEADER that describes the
  /// various attributes of the logical memory allocation.
  ///
  EFI_HOB_MEMORY_ALLOCATION_HEADER    AllocDescriptor;
} EFI_HOB_MEMORY_ALLOCATION_BSP_STORE;

///
/// Defines the location and entry point of the HOB consumer phase.
///
typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_MEMORY_ALLOCATION.
  ///
  EFI_HOB_GENERIC_HEADER              Header;
  ///
  /// An instance of the EFI_HOB_MEMORY_ALLOCATION_HEADER that describes the
  /// various attributes of the logical memory allocation.
  ///
  EFI_HOB_MEMORY_ALLOCATION_HEADER    MemoryAllocationHeader;
  ///
  /// The GUID specifying the values of the firmware file system name
  /// that contains the HOB consumer phase component.
  ///
  EFI_GUID                            ModuleName;
  ///
  /// The address of the memory-mapped firmware volume
  /// that contains the HOB consumer phase firmware file.
  ///
  EFI_PHYSICAL_ADDRESS                EntryPoint;
} EFI_HOB_MEMORY_ALLOCATION_MODULE;
```

```c
typedef struct {
  EFI_HOB_GENERIC_HEADER         Header;	/* EFI_HOB_TYPE_RESOURCE_DESCRIPTOR */
  ///
  /// A GUID representing the owner of the resource. This GUID is used by HOB
  /// consumer phase components to correlate device ownership of a resource.
  ///
  EFI_GUID                       Owner;

  EFI_RESOURCE_TYPE              ResourceType;
  EFI_RESOURCE_ATTRIBUTE_TYPE    ResourceAttribute;

  EFI_PHYSICAL_ADDRESS           PhysicalStart;
  UINT64                         ResourceLength;
} EFI_HOB_RESOURCE_DESCRIPTOR;
```
- ResourceType
	- `EFI_RESOURCE_SYSTEM_MEMORY`
	- `EFI_RESOURCE_MEMORY_MAPPED_IO`
	- `EFI_RESOURCE_IO`
	- `EFI_RESOURCE_FIRMWARE_DEVICE`
	- `EFI_RESOURCE_MEMORY_MAPPED_IO_PORT`
	- `EFI_RESOURCE_MEMORY_RESERVED`
	- `EFI_RESOURCE_IO_RESERVED`

- `ResourceAttribute`
	- `EFI_RESOURCE_ATTRIBUTE_PRESENT`
	- `EFI_RESOURCE_ATTRIBUTE_INITIALIZED`
	- `EFI_RESOURCE_ATTRIBUTE_TESTED`
	- `EFI_RESOURCE_ATTRIBUTE_READ_PROTECTED`
	- `EFI_RESOURCE_ATTRIBUTE_WRITE_PROTECTED`
	- `EFI_RESOURCE_ATTRIBUTE_EXECUTION_PROTECTED`
	- `EFI_RESOURCE_ATTRIBUTE_PERSISTENT`

	- `EFI_RESOURCE_ATTRIBUTE_SINGLE_BIT_ECC`
	- `EFI_RESOURCE_ATTRIBUTE_MULTIPLE_BIT_ECC`
	- `EFI_RESOURCE_ATTRIBUTE_ECC_RESERVED_1`
	- `EFI_RESOURCE_ATTRIBUTE_ECC_RESERVED_2`
	- `EFI_RESOURCE_ATTRIBUTE_UNCACHEABLE`
	- `EFI_RESOURCE_ATTRIBUTE_WRITE_COMBINEABLE`
	- `EFI_RESOURCE_ATTRIBUTE_WRITE_THROUGH_CACHEABLE`
	- `EFI_RESOURCE_ATTRIBUTE_WRITE_BACK_CACHEABLE`
	- `EFI_RESOURCE_ATTRIBUTE_16_BIT_IO`
	- `EFI_RESOURCE_ATTRIBUTE_32_BIT_IO`
	- `EFI_RESOURCE_ATTRIBUTE_64_BIT_IO`
	- `EFI_RESOURCE_ATTRIBUTE_UNCACHED_EXPORTED`
	- `EFI_RESOURCE_ATTRIBUTE_READ_PROTECTABLE`
	- `EFI_RESOURCE_ATTRIBUTE_WRITE_PROTECTABLE`
	- `EFI_RESOURCE_ATTRIBUTE_EXECUTION_PROTECTABLE`
	- `EFI_RESOURCE_ATTRIBUTE_PERSISTABLE`
	
	- `EFI_RESOURCE_ATTRIBUTE_READ_ONLY_PROTECTED`
	- `EFI_RESOURCE_ATTRIBUTE_READ_ONLY_PROTECTABLE`

	- `EFI_RESOURCE_ATTRIBUTE_MORE_RELIABLE`

```c
///
/// Allows writers of executable content in the HOB producer phase to
/// maintain and manage HOBs with specific GUID.
///
typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_GUID_EXTENSION.
  ///
  EFI_HOB_GENERIC_HEADER    Header;
  ///
  /// A GUID that defines the contents of this HOB.
  ///
  EFI_GUID                  Name;
  //
  // Guid specific data goes here
  //
} EFI_HOB_GUID_TYPE;
```

```c
///
/// Details the location of firmware volumes that contain firmware files.
///
typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_FV.
  ///
  EFI_HOB_GENERIC_HEADER    Header;
  ///
  /// The physical memory-mapped base address of the firmware volume.
  ///
  EFI_PHYSICAL_ADDRESS      BaseAddress;
  ///
  /// The length in bytes of the firmware volume.
  ///
  UINT64                    Length;
} EFI_HOB_FIRMWARE_VOLUME;

///
/// Details the location of a firmware volume that was extracted
/// from a file within another firmware volume.
///
typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_FV2.
  ///
  EFI_HOB_GENERIC_HEADER    Header;
  ///
  /// The physical memory-mapped base address of the firmware volume.
  ///
  EFI_PHYSICAL_ADDRESS      BaseAddress;
  ///
  /// The length in bytes of the firmware volume.
  ///
  UINT64                    Length;
  ///
  /// The name of the firmware volume.
  ///
  EFI_GUID                  FvName;
  ///
  /// The name of the firmware file that contained this firmware volume.
  ///
  EFI_GUID                  FileName;
} EFI_HOB_FIRMWARE_VOLUME2;

///
/// Details the location of a firmware volume that was extracted
/// from a file within another firmware volume.
///
typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_FV3.
  ///
  EFI_HOB_GENERIC_HEADER    Header;
  ///
  /// The physical memory-mapped base address of the firmware volume.
  ///
  EFI_PHYSICAL_ADDRESS      BaseAddress;
  ///
  /// The length in bytes of the firmware volume.
  ///
  UINT64                    Length;
  ///
  /// The authentication status.
  ///
  UINT32                    AuthenticationStatus;
  ///
  /// TRUE if the FV was extracted as a file within another firmware volume.
  /// FALSE otherwise.
  ///
  BOOLEAN                   ExtractedFv;
  ///
  /// The name of the firmware volume.
  /// Valid only if IsExtractedFv is TRUE.
  ///
  EFI_GUID                  FvName;
  ///
  /// The name of the firmware file that contained this firmware volume.
  /// Valid only if IsExtractedFv is TRUE.
  ///
  EFI_GUID                  FileName;
} EFI_HOB_FIRMWARE_VOLUME3;
```

```c
///
/// Describes processor information, such as address space and I/O space capabilities.
///
typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_CPU.
  ///
  EFI_HOB_GENERIC_HEADER    Header;
  ///
  /// Identifies the maximum physical memory addressability of the processor.
  ///
  UINT8                     SizeOfMemorySpace;
  ///
  /// Identifies the maximum physical I/O addressability of the processor.
  ///
  UINT8                     SizeOfIoSpace;
  ///
  /// This field will always be set to zero.
  ///
  UINT8                     Reserved[6];
} EFI_HOB_CPU;
```

```c
///
/// Describes pool memory allocations.
///
typedef struct {
  ///
  /// The HOB generic header. Header.HobType = EFI_HOB_TYPE_MEMORY_POOL.
  ///
  EFI_HOB_GENERIC_HEADER    Header;
} EFI_HOB_MEMORY_POOL;
```

```c
///
/// Each UEFI capsule HOB details the location of a UEFI capsule. It includes a base address and length
/// which is based upon memory blocks with a EFI_CAPSULE_HEADER and the associated
/// CapsuleImageSize-based payloads. These HOB's shall be created by the PEI PI firmware
/// sometime after the UEFI UpdateCapsule service invocation with the
/// CAPSULE_FLAGS_POPULATE_SYSTEM_TABLE flag set in the EFI_CAPSULE_HEADER.
///
typedef struct {
  ///
  /// The HOB generic header where Header.HobType = EFI_HOB_TYPE_UEFI_CAPSULE.
  ///
  EFI_HOB_GENERIC_HEADER    Header;

  ///
  /// The physical memory-mapped base address of an UEFI capsule. This value is set to
  /// point to the base of the contiguous memory of the UEFI capsule.
  /// The length of the contiguous memory in bytes.
  ///
  EFI_PHYSICAL_ADDRESS      BaseAddress;
  UINT64                    Length;
} EFI_HOB_UEFI_CAPSULE;
```
  
 
 
 


```c
```

## File Format

## Porting

### ARM

#### Library Interfaces

##### ArmPlatformLib

```c
VOID ArmPlatformPeiBootAction (VOID);
```
This function is actually the first function called by the PrePi
or PrePeiCore modules. It allows to retrieve arguments passed to
the UEFI firmware through the CPU registers.

This function might be written into assembler as no stack are set
when the function is invoked.

```c
UINTN ArmPlatformIsPrimaryCore ( IN UINTN  MpId);
```
This function returns a non-zero value if the callee is the primary core.
The primary core is the core responsible to initialize the hardware and run UEFI.

```c
UINTN ArmPlatformGetCorePosition ( IN UINTN  MpId);
```
This function returns the core position from the position 0 in the processor.

```c
UINTN ArmPlatformGetPrimaryCoreMpId ( VOID);
```
This function returns the MpId of the primary core.

```c
EFI_BOOT_MODE ArmPlatformGetBootMode ( VOID);
```
This function returns the boot reason on the platform

```c
RETURN_STATUS ArmPlatformInitialize ( IN  UINTN  MpId);
```
This function is called by the ArmPlatformPkg/PrePi or ArmPlatformPkg/PlatformPei
in the PEI phase.

```c
VOID ArmPlatformGetVirtualMemoryMap ( OUT ARM_MEMORY_REGION_DESCRIPTOR  **VirtualMemoryMap);
```
This Virtual Memory Map is used by MemoryInitPei Module to initialize the MMU on your platform.

```c
VOID ArmPlatformGetPlatformPpiList (
  OUT UINTN                   *PpiListSize,
  OUT EFI_PEI_PPI_DESCRIPTOR  **PpiList
  );
```

```c
```
