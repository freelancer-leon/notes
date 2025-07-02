# 如何给 TD 注入错误

## 原理简述
* 往 TD guest 中精准注入错误里会涉及到以下几个小程序和工具：
  * `run_in_vm.static`：运行在 TD VM 中的程序，负责分配内存和消费 poison。
  * Trace event：Linux kernel 自带的事件跟踪机制，捕捉建立 `GPA -> HPA` 映射关系的事件。
  * `write_private_mem.sh`：运行在 host 中的脚本，根据捕捉到的事件找到将要注入错误的 `GPA` 对应的 `HPA`。
  * `devmem2`：运行在 host 侧的程序，由 `write_private_mem.sh` 脚本调用，往 private memory 中写入内容导致 private memory 内存损坏。

## 详细步骤
0. Host kernel 需确保 `CONFIG_STRICT_DEVMEM` 选项 **关闭**，否则 `devmem2` 无法操作 `/dev/mem`。
   * 该选项几乎不会在正式产品中启用，这里做实验将其开启。
   * 开启该选项使得用户态程序可以直接读写物理内存，从而绕过 UPM/GMEM 的限制。
1. Host kernel 添加如下命令行选项：
   ```sh
   grubby --update-kernel /boot/vmlinuz-`uname -r` --args='nopat crashkernel=512M'
   ```
   * 由于要使用 `devmem2` 工具直接写物理内存，host kernel 最好禁用 PAT，因此添加 `nopat` 选项。
   * 如果 host kernel 没有打上 TDX `#MC` 相关的 patch 可能会 panic，因此添加 `crashkernel=512M` 给 kexec 预留足够的内存，并且部署 kexec/kdump。
   * TDX `#MC` 处理相关的 patch 为：
     * KVM: TDX: Fixup fatal #MC happens in TD guest that delivered as SMI
     * KVM: TDX: Skip #MSMI induced #MC handling and handle it as a user #MC
     * KVM: TDX: Clear memory poisoned bit on memory failure
     * [x86/mce: Mask out non-address bits from machine check bank](https://git.kernel.org/pub/scm/linux/kernel/git/tip/tip.git/commit/?id=8a01ec97dc066009dd89e43bfcf55644f2dd6d19)（commit 8a01ec97dc06）

2. Host kernel 需开启以下 trace event，错误注入脚本 `write_private_mem.sh` 依赖该 event 在 trace buffer 中的输出来找到注错的物理地址：
   ```sh
   echo 1 > /sys/kernel/debug/tracing/events/kvmmmu/kvm_mmu_set_spte/enable
   ```

3. 将 `run_in_vm.c` 源代码编译成可执行文件，此处我将其静态编译成名为 `run_in_vm.static` 的文件

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <assert.h>

#define DEFAULT_SLEEP (60)
#define DEFAULT_SIZE (4096)

#define _mm_clflushopt(addr) \
  asm volatile(".byte 0x66; clflush %0" : \
      "+m" (*(volatile char *)(addr)));

/*
 * get information about address from /proc/self/pagemap
 */
unsigned long long vtop(unsigned long long addr)
{
  static int pagesize;
  unsigned long long pinfo;
  long offset;
  int fd;

  if (pagesize == 0)
    pagesize = getpagesize();

  offset = addr / pagesize * (sizeof pinfo);

  fd = open("/proc/self/pagemap", O_RDONLY);
  if (fd == -1) {
    perror("pagemap");
    exit(1);
  }

  if (pread(fd, &pinfo, sizeof pinfo, offset) != sizeof pinfo) {
    perror("pagemap");
    exit(1);
  }

  close(fd);

  if ((pinfo & (1ull << 63)) == 0) {
    printf("page not present\n");
    return ~0ull;
  }

  return ((pinfo & 0x007fffffffffffffull) * pagesize) + (addr & (pagesize - 1));
}

void malloc_free(unsigned int sec, size_t size)
{
  void *ptr = malloc(size);

  assert(ptr != NULL);
  printf("malloc(%ld): GVA: %p GPA: %p\n", size, ptr, (void *)vtop((unsigned long long)ptr));

  // Step1
  printf("sleep %d sec for error injection...\n", sec);
  sleep(sec);

  // Step2
  memset(ptr, 0xab, size);
  _mm_clflushopt(ptr);

  // Step3
  useconds_t usec = 10000;
  printf("sleep %d usec before reading malloced buffer...\n", usec);
  usleep(usec);

  // Step4 - consume poison
  sec = *(unsigned int *)ptr;

  printf("!!! should never reach here, expected to be killed by recovery code :%p=%x\n", ptr, sec);
  free(ptr);
}

int main(int argc, char **argv)
{
  unsigned int sec = DEFAULT_SLEEP;
  size_t size = DEFAULT_SIZE;

  if (argc > 1)
    sec = atol(argv[1]);
  if (argc > 2)
    size = atol(argv[2]);

  malloc_free(sec, size);

  return 0;
}
```

4. 将 `run_in_vm.static` 文件复制到 TD VM 中运行，得到输出如下：

   ```sh
   [root@INTELTDX ~]# ./run_in_vm.static
   malloc(4096): GVA: 0x15d9610 GPA: 0x1151b610
   sleep 60 sec for error injection...
   sleep 10000 usec before reading malloced buffer...
   ```
   * 我们需特别关注的是类似 `GPA: 0x1151b610` 这样的信息，它就是我们要注入错误的地址的 GPA，后面的步骤需要该信息作为输入。
   * 该程序会等待 `60` 秒的时间，让 host 侧有机会去精准地写入期望的 TD VM 使用的物理地址。
     * *等待时间* 可以传入第一个参数进行修改，默认是 `60` 秒。
     * *分配的内存的大小* 可以传入第二个参数进行修改，默认是 `4096` 字节。

5. 将 `devmem2.c` 编译为 `devmen2` 可执行文件。
   * Github 上的 [devmem2](https://github.com/radii/devmem2) 程序有 bug，修改后如下。
```cpp
/*
 * devmem2.c: Simple program to read/write from/to any location in memory.
 *
 *  Copyright (C) 2000, Jan-Derk Bakker (jdb@lartmaker.nl)
 *
 *
 * This software has been developed for the LART computing board
 * (http://www.lart.tudelft.nl/). The development has been sponsored by
 * the Mobile MultiMedia Communications (http://www.mmc.tudelft.nl/)
 * and Ubiquitous Communications (http://www.ubicom.tudelft.nl/)
 * projects.
 *
 *
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation; either version 2 of the License, or
 * (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program; if not, write to the Free Software
 * Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA
 *
 */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <signal.h>
#include <fcntl.h>
#include <ctype.h>
#include <termios.h>
#include <sys/types.h>
#include <sys/mman.h>

#define FATAL do { fprintf(stderr, "Error at line %d, file %s (%d) [%s]\n", \
  __LINE__, __FILE__, errno, strerror(errno)); exit(1); } while(0)

#define MAP_SIZE 4096UL
#define MAP_MASK (MAP_SIZE - 1)

void show_usage(char *program)
{
	fprintf(stderr, "\nUsage:\t%s { address } [ type [ data ] ]\n"
		"\taddress : memory address to act upon\n"
		"\ttype    : access operation type : [b]yte, [h]alfword, [w]ord, [l]ong\n"
		"\tdata    : data to be written\n\n",
		program);
		exit(1);
}

unsigned long read_mem(int access_type, void *virt_addr)
{
	unsigned long read_result = 0;

	switch (access_type) {
		case 'b':
			read_result = *((unsigned char *) virt_addr);
			break;
		case 'h':
			read_result = *((unsigned short *) virt_addr);
			break;
		case 'w':
			read_result = *((unsigned int *) virt_addr);
			break;
		case 'l':
			read_result = *((unsigned long *) virt_addr);
			break;
		default:
			fprintf(stderr, "Illegal data type '%c'.\n", access_type);
			exit(2);
	}

	return read_result;
}

int main(int argc, char **argv)
{
	int fd;
	void *map_base, *virt_addr;
	unsigned long read_result, writeval;
	off_t target;
	int access_type = 'w';

	if (argc < 2)
		show_usage(argv[0]);

	target = strtoul(argv[1], 0, 0);

	if (argc > 2)
		access_type = tolower(argv[2][0]);

	if ((fd = open("/dev/mem", argv[3] ? (O_RDWR | O_SYNC) : (O_RDONLY | O_SYNC)))
		 == -1) FATAL;
	printf("/dev/mem opened.\n");
	fflush(stdout);

	/* Map one page */
	map_base = mmap(0, MAP_SIZE,
		argv[3] ? (PROT_READ | PROT_WRITE) : PROT_READ,
		MAP_SHARED, fd, target & ~MAP_MASK);
	if (map_base == MAP_FAILED) FATAL;
	printf("Memory mapped at address %p.\n", map_base);
	fflush(stdout);

  virt_addr = map_base + (target & MAP_MASK);

  /* Read memory */
  if (argc <= 3) {
    read_result = read_mem(access_type, virt_addr);
    printf("Value at address 0x%lX (%p): 0x%lX\n", target, virt_addr, read_result);
    fflush(stdout);
  }

  /* Write memory */
	if (argc > 3) {
		writeval = strtoul(argv[3], 0, 0);
		switch (access_type) {
			case 'b':
				*((unsigned char *) virt_addr) = writeval;
				break;
			case 'h':
				*((unsigned short *) virt_addr) = writeval;
				break;
			case 'w':
				*((unsigned int *) virt_addr) = writeval;
				break;
			case 'l':
				*((unsigned long *) virt_addr) = writeval;
		}
		printf("Written 0x%lX\n", writeval);
		fflush(stdout);
	}

	if (munmap(map_base, MAP_SIZE) == -1) FATAL;
	close(fd);
	return 0;
}
```

6. Host 侧准备好 `write_private_mem.sh` 脚本，内容如下：
   * `devmem2` 程序需放在在同一目录
```sh
#!/usr/bin/env bash
set -ue

[ `id -u` -ne "0" ] && echo "Must run with root privilege" && exit 0
[ "$#" -ne "1" ] && echo -e "Usage: ${0##.*/} GPA \n\tGPA - Guest Physical Address" && exit 0

DEBUG_FS=`cat /proc/mounts | grep debugfs | cut -d ' ' -f2 | head -1`
TRACE_FILE="${DEBUG_FS}/tracing/trace"

# Enable the trace event
echo 1 > /sys/kernel/debug/tracing/events/kvmmmu/kvm_mmu_set_spte/enable

let "GPA = $1 & 0xfffffffffffff000"
let "offset = $1 & 0xfff"

# Make sure GPA is hex format leading with '0x'
GPA=$(printf "0x%x" $GPA)
let "GFN = $GPA >> 12"

# Remove the leading '0x'
GFN=$(printf "%x" $GFN)
echo "GPA=$GPA GFN=$GFN"

PADDR="$(grep -P "gfn $GFN spte [[:xdigit:]]+" $TRACE_FILE)"
echo $PADDR
PADDR=$(echo ${PADDR} | grep -Po "gfn $GFN spte [[:xdigit:]]+" | cut -d" " -f 4)

# Only 51~12 bits are valid, 11~0 bits are flags
PADDR=$((16#$PADDR & 0xFFFFFFFFFF000))
PADDR=$(printf "0x%x" $PADDR)
[ -z "$PADDR" ] && echo "failed to find physical address for virtual address $1" && exit 1

let "PADDR |= $offset"
PADDR=$(printf "0x%x" $PADDR)
echo "PADDR=$PADDR"

./devmem2 $PADDR w

printf "Start to inject physical address 0x%x ...\n" $PADDR
echo "Start doing error injection!"

./devmem2 $PADDR w 0xcdcdcdcd

echo "Done"
```

7. Host 侧需在 guest 等待注入错误的期间（`60` 秒）运行脚本 `write_private_mem.sh` 去破坏（写入）TD 的内存，脚本需要第 4 步中 `GPA: 0x1151b610` 的这个信息作为输入。运行示例如下：
   ```sh
   ./write_private_mem.sh 0x1151b610
   GPA=0x1151b000 GFN=1151b
        tdxvm-13694-13714   [065] ....   807.856265: kvm_mmu_set_spte: gfn 1151b spte 86000080a5669b37 (rwxu) level 1 at 80b34fc8d8
   PADDR=0x80a5669610
   /dev/mem opened.
   Memory mapped at address 0x7fc75016c000.
   Value at address 0x80A5669610 (0x7fc75016c610): 0x0
   Start to inject physical address 0x80a5669610 ...
   Start doing error injection!
   /dev/mem opened.
   Memory mapped at address 0x7fd2ff2d0000.
   Written 0xCDCDCDCD
   Done
   ```
   * **注意：** 有时 trace event 可能会捕捉不到 `GPA -> HPA` 的映射建立的信息而导致脚本运行出错，可以尝试以下几种方法避免错误：
     * 终止 `run_in_vm.static` 后再重新运行，有可能这次分配的 `GPA -> HPA` 的映射信息在 trace buffer 中；
     * 减小分配给 TD VM 的内存大小；
     * 增加 trace buffer 大小，然后重启 TD VM。

8. Guest 等待注入错误的倒计时结束后，因 `run_in_vm.static` 访问被污染的物理地址触发 `#MC`。
   * Host kernel 如果已经打上 TDX `#MC` 相关的 patch，会将错误页面从 Buddy system 中隔离，而不会 panic。
   ```c
   [  896.156739] kvm_intel: kvm [13696]: TD exit 0xe000060400000000, 0 hkid 0x41 hkid pa 0x8200000000000
   [  896.157136] {1}[Hardware Error]: Hardware error from APEI Generic Hardware Error Source: 0
   [  896.157139] {1}[Hardware Error]: event severity: recoverable
   [  896.157140] {1}[Hardware Error]:  Error 0, type: recoverable
   [  896.157141] {1}[Hardware Error]:  fru_text: Socket1
   [  896.157142] {1}[Hardware Error]:   section_type: general processor error
   [  896.157143] {1}[Hardware Error]:   processor_type: 0, IA32/X64
   [  896.157144] {1}[Hardware Error]:   processor_isa: 2, X64
   [  896.157145] {1}[Hardware Error]:   error_type: 0x08
   [  896.157146] {1}[Hardware Error]:   micro-architectural error
   [  896.157147] {1}[Hardware Error]:   operation: 0, unknown or generic
   [  896.157148] {1}[Hardware Error]:   version_info: 0x00000000000c06f2
   [  896.157149] {1}[Hardware Error]:   processor_id: 0x000000000000008c
   [  896.162769] kvm_intel: kvm [13696]: TD exit 0x6000000500000006, 6 hkid 0x41 hkid pa 0x8200000000000
   [  896.173209] ------------[ cut here ]------------
   [  896.182892] mce: Uncorrected hardware memory error in user-access at 82080a5669600
   [  896.185484] kvm_intel: kvm [13696]: TD exit 0xe000060400000000, 0 hkid 0x41 hkid pa 0x8200000000000
   [  896.188816] WARNING: CPU: 49 PID: 13715 at arch/x86/kvm/vmx/tdx.c:4208 read_private_memory+0xa1/0xc0 [kvm_intel]
   [  896.188817] kvm_intel: cleared poisoned cache hkid 0x41 pa 0x80a5669600
   [  896.188819] Modules linked in:
   [  896.188820] mce: [Hardware Error]: Machine check events logged
   [  896.245297] kvm_intel: kvm [13696]: TD exit 0xe000060400000000, 0 hkid 0x41 hkid pa 0x8200000000000
   [  896.247264] kvm_intel: kvm [13696]: TD exit 0xe000060400000000, 0 hkid 0x41 hkid pa 0x8200000000000
   [  896.247266] kvm_intel: kvm [13696]: TD exit 0xe000060400000000, 0 hkid 0x41 hkid pa 0x8200000000000
   [  896.247402]  vhost_vsock vmw_vsock_virtio_transport_common vhost vhost_iotlb vsock snd_seq_dummy snd_hrtimer snd_seq snd_timer snd_seq_device snd
   [  896.248331] kvm_intel: kvm [13696]: TD exit 0xe000060400000000, 0 hkid 0x41 hkid pa 0x8200000000000
   [  896.248334] kvm_intel: kvm [13696]: TD exit 0xe000060400000000, 0 hkid 0x41 hkid pa 0x8200000000000
   [  896.383919]  soundcore rfkill sunrpc x86_pkg_temp_thermal intel_powerclamp coretemp kvm_intel kvm iTCO_wdt ipmi_ssif iTCO_vendor_support irqbypass pcspkr idxd isst_if_mbox_pci isst_if_mmio mei_me i2c_i801 isst_if_common idxd_bus i2c_smbus mei i2c_ismt acpi_ipmi ipmi_si ipmi_devintf ipmi_msghandler acpi_power_meter acpi_pad crct10dif_pclmul crc32_pclmul crc32c_intel nvme ghash_clmulni_intel megaraid_sas nvme_core wmidm_mod
   [  896.426053] CPU: 49 PID: 13715 Comm: tdxvm-13694 Kdump: loaded Tainted: G   M    W         5.10.60-tdx+ #8
   [  896.437368] Hardware name: Quanta Cloud Technology Inc. QuantaGrid D54Q-2U/S6Q-MB-MPS, BIOS 3B07.TEL2P1 02/20/2024
   [  896.449092] RIP: 0010:read_private_memory+0xa1/0xc0 [kvm_intel]
   [  896.455842] Code: 5d 41 5c c3 cc cc cc cc 0f 0b b8 01 01 00 00 be 01 03 00 00 48 89 3c 24 66 89 87 c1 e7 02 00 e8 15 7d 32 00 48 8b 3c 24 eb 94 <0f> 0b 31 d2 48 89 c6 bf 0c 00 00 00 e8 1e a0 00 00 b8 fb ff ff ff
   [  896.477023] RSP: 0018:ff5ef3695ed73a88 EFLAGS: 00010286
   [  896.482974] RAX: e000060400000000 RBX: 0000000000000001 RCX: 0000000000000000
   [  896.491076] RDX: 0000000000000000 RSI: 0000000000000000 RDI: 000000000000000c
   [  896.499177] RBP: 0000000000000000 R08: 0000000000000000 R09: 0000000000000000
   [  896.507277] R10: ffffffffc104a9e0 R11: 0000000000000000 R12: ff5ef3695ed73ae0
   [  896.515394] R13: ff32d35130cb3000 R14: ffffffffc107edc0 R15: 0000000000000000
   [  896.523511] FS:  00007f99cfffe640(0000) GS:ff32d38f9fa40000(0000) knlGS:0000000000000000
   [  896.532684] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
   [  896.539242] CR2: 00007f99cfff7fd0 CR3: 00000080e8b88002 CR4: 0000000000773ee0
   [  896.547349] PKRU: 55555554
   [  896.550503] Call Trace:
   [  896.553352]  ? read_private_memory+0xc0/0xc0 [kvm_intel]
   [  896.559417]  read_private_memory_unalign+0x4d/0xc0 [kvm_intel]
   [  896.566072]  tdx_access_guest_memory+0x16a/0x1d0 [kvm_intel]
   [  896.572534]  tdx_read_write_memory.constprop.0+0xba/0x1b0 [kvm_intel]
   [  896.579872]  ? __smp_call_single_queue+0x23/0x40
   [  896.585159]  ? ttwu_queue_wakelist+0xad/0xd0
   [  896.590057]  tdx_read_guest_memory+0x88/0xa0 [kvm_intel]
   [  896.596176]  kvm_arch_vm_ioctl+0x216/0xc80 [kvm]
   [  896.601476]  ? trace_hardirqs_on+0x2b/0xc0
   [  896.606176]  ? _raw_spin_unlock_irqrestore+0x22/0x30
   [  896.611862]  ? _raw_spin_unlock_irqrestore+0x22/0x30
   [  896.617527]  ? trace_hardirqs_on+0x2b/0xc0
   [  896.622226]  ? __wake_up_common_lock+0x8a/0xc0
   [  896.627330]  ? do_tty_write+0x195/0x250
   [  896.631746]  ? n_tty_poll+0x1f0/0x1f0
   [  896.635953]  kvm_vm_ioctl+0x334/0x770 [kvm]
   [  896.640759]  ? file_tty_write.constprop.0+0x98/0xc0
   [  896.646344]  ? new_sync_write+0x119/0x1b0
   [  896.650947]  ? vfs_write+0x152/0x280
   [  896.655062]  __x64_sys_ioctl+0x87/0xc0
   [  896.659387]  do_syscall_64+0x30/0x40
   [  896.663516]  entry_SYSCALL_64_after_hwframe+0x61/0xc6
   [  896.669270] RIP: 0033:0x7f9a6933957b
   [  896.673384] Code: ff ff ff 85 c0 79 9b 49 c7 c4 ff ff ff ff 5b 5d 4c 89 e0 41 5c c3 66 0f 1f 84 00 00 00 00 00 f3 0f 1e fa b8 10 00 00 00 0f 05 <48> 3d 01 f0 ff ff 73 01 c3 48 8b 0d 75 68 0f 00 f7 d8 64 89 01 48
   [  896.694576] RSP: 002b:00007f99cfff9df8 EFLAGS: 00000246 ORIG_RAX: 0000000000000010
   [  896.703186] RAX: ffffffffffffffda RBX: 00000000c018aecc RCX: 00007f9a6933957b
   [  896.711309] RDX: 00007f99cfff9e90 RSI: ffffffffc018aecc RDI: 000000000000000c
   [  896.719403] RBP: 000055f1ccd64090 R08: 0000000001000000 R09: 00007f9a5446e510
   [  896.727503] R10: 0000000000000002 R11: 0000000000000246 R12: 00007f99cfff9e90
   [  896.735626] R13: 00007f99cfffa06c R14: 0000000000000000 R15: 000055f1ccaf22a0
   [  896.743740] ---[ end trace 27a62be0b8fed9fb ]---
   [  896.749040] SEAMCALL[TDH_MEM_RD(12)] failed: Unknown SEAMCALL status code(0xe000060400000000)
   [  896.758747] SEAMCALL[TDH_MEM_RD(12)] failed: Unknown SEAMCALL status code(0xe000060400000000)
   [  896.758821] Memory failure: 0x80a5669: clean unevictable LRU page still referenced by 1 users
   [  896.768445] SEAMCALL[TDH_MEM_RD(12)] failed: Unknown SEAMCALL status code(0xe000060400000000)
   [  896.768455] SEAMCALL[TDH_MEM_RD(12)] failed: Unknown SEAMCALL status code(0xe000060400000000)
   [  896.778125] Memory failure: 0x80a5669: recovery action for clean unevictable LRU page: Failed
   [  896.787789] SEAMCALL[TDH_MEM_RD(12)] failed: Unknown SEAMCALL status code(0xe000060400000000)
   [  896.797458] mce: Memory error not recovered
   [  896.807124] SEAMCALL[TDH_MEM_RD(12)] failed: Unknown SEAMCALL status code(0xe000060400000000)
   ```
   * 而 TD VM 则会被 shutdown。
   ```c
   KVM internal error. Suberror: 4
   extra data[0]: 0xe000060400000000
   extra data[1]: 0x0000000000000031
   EAX=00000000 EBX=00000000 ECX=00000000 EDX=00000000
   ESI=00000000 EDI=00000000 EBP=00000000 ESP=00000000
   EIP=00000000 EFL=00000000 [-------] CPL=0 II=0 A20=1 SMM=0 HLT=0
   ES =0000 00000000 00000000 00008000
   CS =0000 00000000 00000000 00008000
   SS =0000 00000000 00000000 00008000
   DS =0000 00000000 00000000 00008000
   FS =0000 00000000 00000000 00008000
   GS =0000 00000000 00000000 00008000
   LDT=0000 00000000 00000000 00008000
   TR =0000 00000000 00000000 00008000
   GDT=     0000000000000000 00000000
   IDT=     0000000000000000 00000000
   CR0=00000000 CR2=0000000000000000 CR3=0000000000000000 CR4=00000000
   DR0=0000000000000000 DR1=0000000000000000 DR2=0000000000000000 DR3=0000000000000000
   DR6=00000000fffe07f0 DR7=0000000000000400
   EFER=0000000000000d01
   Code=KVM internal error. Suberror: 4
   extra data[0]: 0xe000060400000000
   extra data[1]: 0x0000000000000003
   KVM internal error. Suberror: 4
   extra data[0]: 0xe000060400000000
   extra data[1]: 0x00000000000000a5
   KVM internal error. Suberror: 4
   extra data[0]: 0xe000060400000000
   extra data[1]: 0x0000000000000032
   KVM internal error. Suberror: 4
   extra data[0]: 0xe000060400000000
   extra data[1]: 0x0000000000000004
   KVM internal error. Suberror: 4
   extra data[0]: 0xe000060400000000
   extra data[1]: 0x0000000000000002
   KVM internal error. Suberror: 4
   extra data[0]: 0xe000060400000000
   extra data[1]: 0x00000000000000a7
   <??> ?? ?? ?? ??./start-qemu.sh: line 108: 13696 Bus error  (core dumped)
   ```

### 再次注错
* 再次注入错误需要将 trace buffer 中的内容清除：
  ```sh
  echo > /sys/kernel/debug/tracing/trace
  ```
* 重启 TD VM，然后重复执行上面的第 4 步和第 7 步。