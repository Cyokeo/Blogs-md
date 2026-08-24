## reference
- https://blog.csdn.net/dengjin20104042056/article/details/137828704
- [partitions](https://opensource.rock-chips.com/wiki_Partitions)
- [Boot Option](https://opensource.rock-chips.com/wiki_Boot_option#Boot_introduce)
- [GPT partition](https://github.com/rockchip-linux/u-boot/blob/android/doc/README.gpt)
- [Build Uboot](https://opensource.rock-chips.com/wiki_U-Boot#Build_U-Boot)
- 重要参考：
	- 1.  [Rockchip RK3399 - 引导流程和准备工作](https://www.cnblogs.com/zyly/p/17380243.html "发布于 2023-05-07 21:45")
	- 2. [Rockchip RK3399 - 官方固件方式加载uboot](https://www.cnblogs.com/zyly/p/17403323.html "发布于 2023-05-15 22:16")
	- 3. [Rockchip RK3399 - TPL/SPL方式加载uboot](https://www.cnblogs.com/zyly/p/17389525.html "发布于 2023-05-11 01:02")

## 总结
1. 注意USB-TTL连接的TX/RX别接反了

## 1.5. Boot mode

Firefly-RK3399 has three startup modes:

- Normal mode
    
- Loader mode
    
- MaskRom mode

### 1.5.1. Normal mode

Normal mode is the Normal startup process. Each component loads in turn and enters the system normally.

### 1.5.2. Loader mode

In Loader mode, the bootloader will enter the upgrade state, waiting for the host command for firmware upgrade, etc.

### 1.5.3. MaskRom mode

MaskRom mode is used for system repair when the bootloader is damaged.

In general, there is no need to enter `MaskRom mode`. Only when the bootloader verification fails (the IDB block cannot be read, or the bootloader is damaged), the BootRom code will enter this mode. At this time, the BootRom code waits for the host to transmit the bootloader code through the USB interface, load and run it. When the board becomes bricked and cannot start or upgrade the program normally, you can also manually enter the `MaskRom mode`.

# loader image
```c
// uboot-rockchip/tools/rockchip/boot_merger.h
#define TAG 0x544F4F42
#define MERGER_VERSION 0x01030000
/* rk3399 chip info: {0x33333043, 0x32303136, 0x30313138, 0x56313030} - 330B20160118V100 */

#define MAX_NAME_LEN 20
typedef struct {
	uint8_t size;
	rk_entry_type type;
	uint16_t name[MAX_NAME_LEN];
	uint32_t dataOffset;
	uint32_t dataSize;
	uint32_t dataDelay;
} rk_boot_entry;
#pragma pack()

typedef enum {
	ENTRY_471 =1,
	ENTRY_472 =2,
	ENTRY_LOADER =4,
} rk_entry_type;

typedef struct {
	uint32_t tag;   // 0x544F4F42
	uint16_t size;  // sizeof(rk_boot_header);
	uint32_t version;
	uint32_t mergerVersion;
	rk_time releaseTime;
	uint32_t chipType;      // "RK330C" -> "330C" -> 0x33333043
	uint8_t code471Num;     // = 1 -> *ddr_*.bin
	uint32_t code471Offset; // sizeof(rk_boot_header)
	uint8_t code471Size;    // sizeof(rk_boot_entry);
	uint8_t code472Num;     // = 1 -> *usbplug*.bin
	uint32_t code472Offset; // hdr->code471Offset + gOpts.code471Num * hdr->code471Size;
	uint8_t code472Size;
	uint8_t loaderNum;      // = 2 -> *ddr*.bin and *miniloadr*.bin
	uint32_t loaderOffset;  // hdr->code472Offset + gOpts.code472Num * hdr->code472Size;
	uint8_t loaderSize;     // sizeof(rk_boot_entry);
	
	uint8_t signFlag; // = 0
	uint8_t rc4Flag;  // enable RC4 for IDB data(both ddr and preloader)
	uint8_t reserved[BOOT_RESERVED_SIZE];
} rk_boot_header;
```

# trust image
> BL31*.elf
> bl32*.bin
```c
// scripts/atf.sh --ini RK3399MINIALL.ini --sha 3[CONFIG_TRUST_SHA_MODE] --rsa 2[CONFIG_TRUST_RSA_MODE] --size 2048[CONFIG_TRUST_SIZE_KB] 2[CONFIG_TRUST_NUM]

// trust_merge

typedef struct {
	bool sec;
	uint32_t id;
	char path[MAX_LINE_LEN];
	uint32_t addr;
	uint32_t offset;
	uint32_t size;
	uint32_t align_size;
} bl_entry_t;

// 如果BL3*是elf文件，则将elf中LOAD的load段信息转化为bl_entry_t，且entry->addr为elf文件中的信息
// 如果是bin文件，则entry->addr为ini文件中指定的地址信息

// entry->path为ini文件中指定的PATH信息

typedef struct {
	uint32_t tag;  // = "BL3X"
	uint32_t version;
	uint32_t flags;  // sha << 0 | rsa << 4
	uint32_t size;
	uint32_t reserved[4];
	uint32_t RSA_N[64];
	uint32_t RSA_E[64];
	uint32_t RSA_C[64];
} TRUST_HEADER, *PTRUST_HEADER;

typedef struct {
	uint32_t HashData[8];// 每个trust bin hash256 gist
	uint32_t LoadAddr;   // = bl_entry->addr
	uint32_t LoadSize;   // = bl_entry->align_size >> 9;
	uint32_t reserved[2];
} COMPONENT_DATA, *PCOMPONENT_DATA;

typedef struct {
	uint32_t ComponentID;
	uint32_t StorageAddr;  // 在最终文件的存储偏移 >> 9
	uint32_t ImageSize;
	uint32_t reserved;
} TRUST_COMPONENT, *PTRUST_COMPONENT;

{
	// -- Trust Headr
	// -- n*COMPONENT_DATA [size = sizeof COMPONENT_DATA]
	// -- trust bin sign [SIGNATURE] [size = 256]
	// -- TRUST_COMPONENT*n
	// . = 2048
}
// trust bins


```

# u-boot image
```c
// scripts/uboot.sh --load CONFIG_SYS_TEXT_BASE[=0x00200000] --size ${CONFIG_UBOOT_SIZE_KB=2048} ${CONFIG_UBOOT_NUM=2}
// ../rkbin/tools/loaderimage --pack --uboot u-boot.bin uboot.img ${LOAD_ADDR} ${SIZE}



```