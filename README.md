# HeapX

HeapX is a fast & strong **Memory Manager** primarily aimed at embedded world.

# Key features

- Manage up to 32 memory regions
- Several initialization strategies (allow on-demand hardware init/deinit)
- Optimized realloc with several strategies either
- Manage allocations by size affinities
- Optionnal advanced usage statistics
- Optionnal real-time memory checks

# History

HeapX has been primarily developed to replace *FreeRTOS heap5.c* (_and all the heapX.c serie_) running on a ST microcontroller with two internal memory regions, plus an external memory chip.

The first requirement was to power up external ram chips on the first allocation and to shut them down when the last allocaton is freed.
Then, because applications/firmwares are all diffetent and because memory fragmentation can be an issue, having several intialization, alloction & reallocation strategies can make the difference.
Finally, every C/C++ developper has already faced memory related bugs and we all know they are often hard-to-find bugs. Being able to activate deep memory checks running on every malloc/realloc/free call to allow early detection of buffer underruns/overruns and other _mad pointer behaviors_ was also a strong requirement.

# Usage
## Installation
### Get the sources
Get the **HeapX** folder and add the 3 following files to your project
```
HeapAllocator.cpp
HeapAllocator.h
HeapAllocatorConf_template.h
```
Copy or rename the file `HeapAllocatorConf_template.h` to `HeapAllocatorConf.h`... Et voila!
_The **VS** folder contains unit test projects for Visual Studio 2017 and are not required._
### Compile
HeapX is written in full C++99 for the robustness of the language.
It has been compiled with GCC 6.X (-Wall -Wextra) and Visual Studio 2017 (-Wall) and does not generate any warning polution.
## Configuration
Configuring the allocator is quite easy:
### Configuring the Allocator
Define a global structure named `HeapXMemoryConfiguration`, in the global namespace, of type `HeapX::AllocatorDefinition`.
```cpp
static HeapX::AllocatorDefinition HeapXMemoryConfiguration =
{
  &InitNotifier,
  &DeinitNotifier,
  &ErrorHandler,
  MyMemoryRegions
};
```
The 3 fist fields are function pointers and can be set to null to disable the corresponding calls:
* The first field is called once when the allocator is initializing its context.
* The second field may be called when the allocator deinitialize it's last region if the configuration is relevant.
* The third is the Error handler pointer. It is strongly recommended to provide a function, and even more, to catch all the calls with a breakpoint, logs or whatever. You should be suprised to catch errors where you think your code is clean!

The last field must point to an array of `HeapX::RegionDefinition` structures:
```cpp
static RegionDefinition MyMemoryRegions[] =
{
  { (void*)0x10000000,  64 << 10, nullptr, nullptr, InitializationStrategy::Static, AllocationAffinity::Small, ReallocationStrategy::Default },
  { (void*)0x20000000, 192 << 10, nullptr, nullptr, InitializationStrategy::DynamicOnce, AllocationAffinity::Medium, ReallocationStrategy::Default },
  { (void*)0xD0000000,   8 << 20, &Init, &Deinit, InitializationStrategy::DynamicFull, AllocationAffinity::None, ReallocationStrategy::Default },
  { nullptr }
};
```
This array provides the allocator with the memory regions along with their configuration.
The array *MUST* end with a null Region definition (nullptr/NULL in the first field).

The `RegionDefinition` is defined as following:
```cpp
  struct RegionDefinition
  {
    void*                  StartAddress;    //!< Start address of the Memory Region
    unsigned int           Size;            //!< Size in byte of the Memory Region
    RegionInitializer      Initializer;     //!< Memory region initializer callback
    RegionDeinitializer    Deinitializer;   //!< Memory region deinitializer callback
    InitializationStrategy Initializations; //!< Initialization/Deinitialization strategy
    AllocationAffinity     Affinities;      //!< Size Affinities (small/medium/large)
    ReallocationStrategy   Reallocations;   //!< Reallocation strategy
  };
```

Lets see all fields in details.
#### .StartAddress :
Hold the _address_ of the memory region.

Required Alignments:
- 8 bytes on 32bit systems with `HEAPX_CHECK` < 2
- 16 bytes on 32bit systems with `HEAPX_CHECK` == 2 or 64bits systems with `HEAPX_CHECK` < 2
- 32 bytes on 64bit systems with `HEAPX_CHECK` == 2

If the alignment condition is not meet, the allocator will take the next aligned address as the new region start address.
Addressing whole memory banks is not an issue as they are aligned by nature. But if you use a static array as a memory region (like does the heap4/5 in FreeRTOS for example) you must take care of the alignment (use pragma or attributes to align) or you will loose some bytes at the bottom of the memory region.

#### .Size :
Size of the memory region, in byte. Must meet the same alignment requirements as for the StartAddress. If not, you will loose some bytes at the top of the memory region.
* Minimum region size: Alignment requirement x 4 (8 x 4 = 32 bytes on 32bit systems with `HEAPX_CHECK` < 2)
* Maximum region size: 256 MegaBytes on 32bit systems, and virtually 4 HexaBytes on 64bit systems.

#### .Initializer :
If not NULL/nullptr, the provided function will be called when the region is initialized.
This may happend once, or several times, depending of the region configuration. Or even not happend at all if no allocation falls into this region.

When called, the `StartAddress` and the `Size` are passed so that you can identify what region is being initialized.
That is, you can initialize the memory region:
- Fill it with zero
- Power-up the hardware (for external ram)
- ...

#### .Deinitializer :
If not NULL/nullptr, the provided function will be called when the region is deinitialized.
This may happend once, several times, or never, depending of the region configuration.

As for the `Initializer` you may be interested in powering down external RAM or doing whatever you need in such situation.

#### .Initializations :
This field sets the initialization and deinitialization behavior of the memory region.
It can be one of the following value from the `enum HeapX::InitializationStrategy` :

| Value | Behavior |
| ------ | ------ |
| **`DoNotUse`** | Disable this region. Useful only for debugging purpose or to deactivate a region programatically *before* the first allocation. |
| **`Static`** | The region is initialized once, when the allocator is initialized and build its internal context. The region is never deinitialized. |
| **`DynamicOnce`** | The region is initialized once, when the region is required to host its first allocation. Never deinitialized. |
| **`DynamicFull`** | Same as for `DynamicOnce`, but the region can be deinitialized when its last allocation is being freed. |
| **`Default`** | Can be used as the default value when the initialization strategy does not matter. Same as `DynamicOnce`. |

It is important to understand that the HeapX allocator initialize itself when the very first allocation is called (lazy-initialization).
The allocator reads the `HeapXMemoryConfiguration` global structure once and build its internal context. Then, `HeapXMemoryConfiguration` is no more used, and *changing its content has no effect*.
However, if all memory regions are configured as `DynamicFull` and the very last allocation is freed, the allocator calls the global deinitializer function, read from `HeapXMemoryConfiguration`.

Also, if you plan to build or change the configuration structure programmatically, beware of the static initializations. In C++, static classes might call malloc/calloc/realloc before your initialization code. In such cases, use the global intitialized to edit the region list.

#### .Affinities
It is common to see memory allocators to dispatch small, medium and large allocations in different regions. Doing this greatly decreases the memory fragmentation and helps reallocations also.
HeapX is no exception.
The field `Affinities` is a bitflag that makes the allocator selecting the right region for the right allocation size. You can choose multiple values by or-ing more than one `enum AllocationAffinity` (the `|` operator (bitwise OR) is defined)

| Value | Behavior |
| ------ | ------ |
| **`Small`** | This region will receive small allocations first. |
| **`Medium`** | This region will receive medium allocations first. |
| **`Large`** | This region will receive large allocations first. |
| **`Any`** | This region will receive allocations of any size. Its value is the combination of Small | Medium | Large. |
| **`None`** | This region has no affinity and will be selected as the last choice for allocations of any size. Its value is 0. |
| **`Default`** | Use `Default` when you have only one region, when affinities do not matter, or when you don't know what to choose! Equals to `Any`. |

When HeapX needs to allocate bytes, it start by qualifying the requested size: small, medium, or large.
Then, it looks for the first region with enough room and the affinity matching the requested size.
If nothing's found, it tries seeking for the first region with enough room and no affinity.
If still nothing, it tries the first region with enough room.

Default small/medium/large threshold values are:
* 64 bytes <= Small
* 64 bytes < Medium <= 32 Kbytes
* 32 KBytes < Large
Theses values can be adjusted using the `HeapXConfiguration.h`.

#### .Reallocations
Even if you are not familiar with the `realloc` function, lots of libraries are relying on. Sometimes because it is an all-in-one memory management function (yes, you can allocate, reallocate and free with `realloc`), but often because it's convenient to increase or decrease you in-memory objects or buffers with a all-in-one function hiding the process to the developper.

### Using the allocator: malloc/free and their friends
Every C Library expose the 4 following functions:
* void* malloc(size_t size);
* void* calloc(size_t elements, size_t size);
* void* realloc(void* ptr, size_t size);
* void free(void* ptr);

### Advanced configuration
### Samples & Real time cases
