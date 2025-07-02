# TDX Module 源代码

## 调试打印

### 调试输出目标
* 目前支持的调试输出目标有三个：
  * Trace Buffer
  * 串口
  * 外部 Buffer
* src/common/debug/tdx_debug.h
```cpp
/**
 * @enum print_target_e
 *
 * @brief Enum for choosing the trace target
 */
typedef enum
{
    TARGET_TRACE_BUFFER,
    TARGET_SERIAL_PORT,
    TARGET_EXTERNAL_BUFFER
} print_target_e;
```
* 例如，用 `SEAMCALL(0xFE) RCX=0 RDX=1` 配置输出目标为 *串口*

### 调试配置
* `TDDEBUGCONFIG_LEAF` 这个 seamcall leaf 在 TDX Module spec 里并没提到
* include/tdx_api_defs.h
```cpp
/**< Enum for SEAMCALL leaves opcodes */
typedef enum
{
    TDH_VP_ENTER_LEAF             = 0,
    TDH_MNG_ADDCX_LEAF            = 1,
...
#ifdef DEBUGFEATURE_TDX_DBG_TRACE
    ,TDDEBUGCONFIG_LEAF = 0xFE
#endif
} SEAMCALL_LEAVES_OPCODES;
```
* 用 `TDDEBUGCONFIG_LEAF` 这个 leaf 可以使能 TDX Module 的调试配置
* src/vmm_dispatcher/tdx_vmm_dispatcher.c
```cpp
void tdx_vmm_dispatcher(void)
{
...
    // switch over leaf opcodes
    switch (leaf_opcode)
    {
#ifdef DEBUGFEATURE_TDX_DBG_TRACE
    case TDDEBUGCONFIG_LEAF:
    {
        local_data->vmm_regs.rax = td_debug_config(local_data->vmm_regs.rcx, local_data->vmm_regs.rdx,
                                                   local_data->vmm_regs.r8);
        break;
    }
#endif
    }
...
}
```
* `td_debug_config()` 的三个参数分别对应到 seamcall 的三个寄存器
  * `leaf`：`rcx`，指定以下功能：
    * `0`：设置调试信息的输出目标，`payload` 指定输出目标
    * `1`：Dump 内部的 buffer 到 VMM 提供的 buffer，`payload` 外部 buffer 的 HPA，dump `second_payload` 倒数的消息条数
    * `2`：设置一个 emergency buffer（`payload`），当 `ud2` 指令产生的 trace dump 时填充
    * `3`：设置打印的严重级别（`payload`）
  * `payload`：`rdx`
  * `second_payload`：`r8`
* 在 module 遇到严重关闭错误的情况下，内部 buffer 无法使用 SEAMCALL 读取，因此应在调用之前配置外部 emergency buffer：
  * `SEAMCALL(0xFE) RCX=2 RDX=(External memory buffer physical address)`
  * 当 module 遇到任何严重错误时，内部 buffer 的内容将自动复制到预先配置的内存缓冲区
* src/common/debug/tdx_debug.c
```cpp
uint64_t td_debug_config(uint64_t leaf, uint64_t payload, uint64_t second_payload)
{   //获取 TDX module per package 的全局数据的 debug_control 域
    debug_control_t* p_ctl = &(get_global_data()->debug_control);
    uint64_t retval = 0;
    //如果打印控制还未初始化，则初始化
    if (!p_ctl->initialized)
    {
        init_debug_control();
    }

    if (leaf == 0) // Set debug print target
    {
        print_target_e print_target = (print_target_e)payload;
        debug_message_t* target_buffer = NULL;

        if (print_target == TARGET_EXTERNAL_BUFFER)
        {   //如果输出目标是外部的 buffer，第三个参数的值需对齐到 256
            // Check alignment
            if (second_payload % MAX_PRINT_LENGTH)
            {
                return TDX_OPERAND_INVALID;
            }
            //输出目标是外部提供的 buffer，即第三个参数
            target_buffer = (debug_message_t*)second_payload;
        }
        else if (print_target == TARGET_TRACE_BUFFER)
        {   //输出目标是 TDX module per package 的全局数据中的 4096 条消息的 trace ring buffer
            target_buffer = get_global_data()->trace_buffer;
        }
        else if (print_target != TARGET_SERIAL_PORT) // Default case
        {   //最后一个可能如果不是串口就不对了
            return TDX_OPERAND_INVALID;
        }

        // Commit changes
        acquire_mutex_lock_or_wait(&(p_ctl->trace_buffer_lock));

        p_ctl->print_target = print_target;  //记录打印的目标到调试控制结构
        p_ctl->trace_buffer = target_buffer; //记录打印的 buffer 到调试控制结构

        // Reset all counters
        p_ctl->buffer_writer_pos = TRACE_BUFFER_FIRST_MESSAGE; //重置写者的位置
        p_ctl->buffer_reader_pos = TRACE_BUFFER_FIRST_MESSAGE; //重置读者的位置
        p_ctl->msg_num = 0; //重置消息数目

        release_mutex_lock(&(p_ctl->trace_buffer_lock));
    }
    else if (leaf == 1) // Dump internal buffer to given VMM buffer
    {
        retval = dump_print_buffer_to_vmm_memory(payload, (uint32_t)second_payload);
    }
    else if (leaf == 2) // Set an emergency buffer to be filled with the trace dump on ud2
    {
        p_ctl->emergency_buffer = payload;
    }
    else if (leaf == 3) // Set printing severity
    {
        p_ctl->print_severity = payload;
    }

    return retval;
}
```

### 调试控制结构
* Trace Buffer 由 `TRACE_BUFFER_SIZE`（4096）条消息构成，每条消息长度为 `MAX_PRINT_LENGTH`（256） 字节，第一个消息槽用于存储消息槽的使用信息
* src/common/debug/tdx_debug.h
```cpp
#define TRACE_BUFFER_FIRST_MESSAGE  1                // Message at index 0 is used to store additional info
#define TRACE_BUFFER_SIZE           (4 * 1024)       // 4096 messages

#define MAX_PRINT_LENGTH            256
```
* 调试打印控制结构体 `struct debug_control_s`
```cpp
typedef struct debug_message_s {
    char message[MAX_PRINT_LENGTH];
} debug_message_t;

/**
 * @struct debug_control_t
 *
 * @brief Holds the configuration for printing/debug output
 */
typedef struct debug_control_s {
    uint8_t          com_port_lock;     //串口打印锁
    uint16_t         com_port_addr;     //串口打印的 I/O port 地址
    print_target_e   print_target;      //打印的目标
    bool_t           initialized;       //打印控制初始化的标志
    uint8_t          trace_buffer_lock; //锁
    debug_message_t* trace_buffer;      //打印的 buffer
    uint32_t         msg_num;           //消息的数目
    uint32_t         buffer_writer_pos; //写者的位置
    uint32_t         buffer_reader_pos; //读者的位置
    uint64_t         emergency_buffer;  //紧急 buffer 的位置，用于 ud2 指令造成的 trace dump
    uint64_t         print_severity;    //打印的严重级别
} debug_control_t;
```
* TDX module **per-package** 的全局数据中包含一个 `4096` 条消息的 `trace_buffer[TRACE_BUFFER_SIZE]`，作为 ring buffer 使用
  * 每条消息有 `256` 字节（`MAX_PRINT_LENGTH`）
* src/common/data_structures/tdx_global_data.h
```cpp
/**
 * @struct tdx_module_local_t
 *
 * @brief Holds the global per-package data
 *
 */
typedef struct tdx_module_global_s
{
...
#ifdef DEBUGFEATURE_TDX_DBG_TRACE
    debug_control_t debug_control;
    debug_message_t trace_buffer[TRACE_BUFFER_SIZE];
#endif

} tdx_module_global_t;
```

### 调试信息 Dump 函数
* `count_messages()` 数一数 buffer 里的消息数目
```cpp
static inline uint32_t count_messages(debug_control_t* p_ctl)
{
    uint32_t reader_pos = p_ctl->buffer_reader_pos;
    uint32_t writer_pos = p_ctl->buffer_writer_pos;

    uint32_t count = 0;

    while (reader_pos != writer_pos)
    {
        reader_pos++; //读者追赶写者
        count++;      //同时计数
        //如果读者位置到 buffer 末尾了
        if (reader_pos >= TRACE_BUFFER_SIZE)
        {   //回绕到 buffer 的开头
            reader_pos = TRACE_BUFFER_FIRST_MESSAGE;
        }
    }
    //追上了，返回计数
    return count;
}
```
* `get_advanced_reader_pos()` 返回当前读者位置 `current_pos` 往前的 `advance_num` 个位置，要处理 ring buffer 回绕的问题
```cpp
static inline uint32_t get_advanced_reader_pos(uint32_t current_pos, uint32_t advance_num)
{
    uint32_t reader_pos = current_pos;

    for (uint32_t i = 0; i < advance_num; i++)
    {
        reader_pos++;
        //如果读者位置到 buffer 末尾了
        if (reader_pos >= TRACE_BUFFER_SIZE)
        {   //回绕到 buffer 的开头
            reader_pos = TRACE_BUFFER_FIRST_MESSAGE;
        }
    }
    //返回往前后的读者位置位置
    return reader_pos;
}
```
* `dump_print_buffer_to_vmm_memory()` 从 trace buffer 里 dump 倒数 `num_of_messages_from_the_end` 条数据到 VMM 提供的 `hpa`，但 **不会** 消费掉消息
* `dump_message_to_vmm_memory()` 从第一个参数的 buffer 里往第二个参数提供的 buffer 里 dump 字符，长度由第三个参数指定，最后一个参数 `fillnull = true` 表示每条消息的末尾换行和最后的 `\0` 也会处理，返回实际 dump 的字节数
```cpp
uint32_t dump_print_buffer_to_vmm_memory(uint64_t hpa, uint32_t num_of_messages_from_the_end)
{
    debug_control_t* p_ctl = &(get_global_data()->debug_control);

    acquire_mutex_lock_or_wait(&(p_ctl->trace_buffer_lock));

    uint32_t vmm_buf_pos = 0;
    uint32_t message_count = count_messages(p_ctl); //当前 buffer 中的消息数

    uint32_t reader_pos = p_ctl->buffer_reader_pos; //当前读者的位置

    if (p_ctl->trace_buffer == NULL)
    {
        return 0;
    }
    //如果 buffer 里消息数多于要 dump 的消息数，需要计算一下 dump 的起始位置；否则全部 dump 出来
    if ((num_of_messages_from_the_end != 0) && (message_count > num_of_messages_from_the_end))
    {   //比如从 100 消息里读倒数 20 条，那么应该从 100 - 20 = 80 条开始读起，reader_pos 就是这个起始位置
        // Advance the reader position to (message_count - num_of_messages_from_the_end)
        // To read the last num_of_messages_from_the_end number of messages
        reader_pos = get_advanced_reader_pos(reader_pos, message_count - num_of_messages_from_the_end);
    }
    //读者追赶写者
    while (reader_pos != p_ctl->buffer_writer_pos)
    {   //该条消息数组的指针
        char* msg_buf_ptr = p_ctl->trace_buffer[reader_pos].message;
        //每次往 VMM 提供的 buffer 里 dump MAX_PRINT_LENGTH（256）个字符，返回实际 dump 的字节数
        vmm_buf_pos += dump_message_to_vmm_memory(msg_buf_ptr, hpa + vmm_buf_pos, MAX_PRINT_LENGTH, true);
        //dump 了一条数据，读者往前走一条
        reader_pos = get_advanced_reader_pos(reader_pos, 1);
    }

    release_mutex_lock(&(p_ctl->trace_buffer_lock));
    //返回总共往 VMM 提供的 buffer 里 dump 的字节数
    return vmm_buf_pos;
}
```

### 核心打印函数 `tdx_print()`
* 核心打印函数 `tdx_print()` 会根据调试打印控制结构决定往哪个目标输出数据
  * 往 trace buffer 或者外部 buffer 时打印会产生数据
  * 注意：当打印的目标是串口时，不会生产数据
```cpp
//src/common/debug/tdx_debug.c
tdx_print()
   if (!p_ctl->initialized)
   -> init_debug_control()
   -> common_printf()
         if ((p_ctl->print_target == TARGET_TRACE_BUFFER) || (p_ctl->print_target == TARGET_EXTERNAL_BUFFER))
         -> print_to_buffer()
         else
         -> print_to_serial()
               //src/common/debug/serial_port.c
            -> log_to_com_port()
```
* src/common/debug/tdx_debug.c
```cpp
static void print_to_buffer(debug_control_t* p_ctl, char* print_buf, uint32_t print_len)
{
    // lock the memory buffer to make sure messages are not mixed
    acquire_mutex_lock_or_wait(&(p_ctl->trace_buffer_lock));
    //打印到 buffer，buffer 自然不能为空
    if (p_ctl->trace_buffer == NULL)
    {
        goto EXIT;
    }
    //target_debug_message 指向当前写者的消息槽
    debug_message_t* target_debug_message = &p_ctl->trace_buffer[p_ctl->buffer_writer_pos];

    if (p_ctl->print_target == TARGET_EXTERNAL_BUFFER) //如果目标是外部的 buffer
    {   //common_printf 栈上的 print_buf 的内容 dump 到 trace_buffer 的消息槽
        dump_message_to_vmm_memory(print_buf, (uint64_t)target_debug_message, print_len, true);
        //更新第 0 个消息槽中的打印 buffer 中的信息
        print_buffer_info_t* print_buffer_info = map_pa(&p_ctl->trace_buffer[0], TDX_RANGE_RW);
        print_buffer_info->last_message_idx = p_ctl->buffer_writer_pos;
        print_buffer_info->last_absolute_addr = (uint64_t)target_debug_message;
        print_buffer_info->total_printed_messages = p_ctl->msg_num + 1;
        free_la(print_buffer_info);
    }
    else if (p_ctl->print_target == TARGET_TRACE_BUFFER) // TARGET_TRACE_BUFFER
    {   //如果目标是 TDX module 内部的 trace buffer，只需按字符拷贝即可
        // Copying the message
        uint32_t i;
        for (i = 0; print_buf[i] != 0 && i < print_len && i < MAX_PRINT_LENGTH; ++i)
        {
            target_debug_message->message[i] = print_buf[i];
        }

        target_debug_message->message[i] = '\0';
    }
    else // In case we switched to serial port printing in between, trace buffer will be invalid
    {
        goto EXIT;
    }
    //写者往前走
    p_ctl->buffer_writer_pos++;
    //如果写者到了消息槽的末尾，重置到消息槽的开头并跳过第 0 个槽
    if (p_ctl->buffer_writer_pos >= TRACE_BUFFER_SIZE)
    {
        p_ctl->buffer_writer_pos = TRACE_BUFFER_FIRST_MESSAGE;
    }
    //如果写者追上了读者
    if (p_ctl->buffer_reader_pos == p_ctl->buffer_writer_pos)
    {   //推动读者往前走。如果读者到了消息槽的末尾，重置到消息槽的开头并跳过第 0 个槽
        p_ctl->buffer_reader_pos++;
        if (p_ctl->buffer_reader_pos >= TRACE_BUFFER_SIZE)
        {
            p_ctl->buffer_reader_pos = TRACE_BUFFER_FIRST_MESSAGE;
        }
    }
    //消息总数加一
    p_ctl->msg_num++;

EXIT:

    release_mutex_lock(&(p_ctl->trace_buffer_lock));
}
```
* `log_to_com_port()` 就是用 port I/O 指令往串口写，辅以流控
* 综上分析，三种调试输出的方式优劣如下：
1. 把输出目标定为外部 buffer 时，`dump_message_to_vmm_memory()` 和 `print_to_buffer()` 都会有内存的分配和释放和页表的建立的动作，开销是比较大的，应该避免使用这种方式
2. 串口方式可能作为启动调试方面会比较有用，但可能会有等待流控的开销。这是默认的打印方式，见 `init_debug_control()`
3. 最常规的方式应该还是通过 VMM 发出 seamcall `TDDEBUGCONFIG_LEAF` leaf `2` 指定 dump 一定数量的消息数，对 TDX Module 的影响应该是最小的

## 关于页面操作输入信息的检查
* TDX module 关于内存操作的大小是有规定的，比如：
  * `TDH.MEM.PAGE.ACCEPT` 的页大小是 `4k`、`2M`
  * `TDH.MEM.PAGE.ADD` 的页大小是 `4k`
  * `TDH.MEM.PAGE.AUG` 的页大小是 `4k`、`2M`
  * `TDH.MEM.PAGE.PROMOTE` 的页大小是 `4k`、`2M` 或者 `1G`
  * src/common/x86_defs/x86_defs.h
```cpp
/**
 *   @brief Definition of GPA EPT entry level
 */
typedef enum {
    LVL_PT      = 0,    // EPT 4KB leaf entry level
    LVL_PD      = 1,    // EPT page directory or 2MB leaf entry level
    LVL_PDPT    = 2,    // EPT page table directory or 1GB leaf entry level
    LVL_PML4    = 3,    // EPT Page map entry level 4
    LVL_PML5    = 4,    // EPT Page map entry level 5
    LVL_MAX     = 5,
} ept_level_t;
```
* 检查如下：
```cpp
 1 src/td_dispatcher/vm_exits/tdg_mem_page_accept.c|149| <<tdg_mem_page_accept>> if (!verify_page_info_input(gpa_mappings, LVL_PT, LVL_PD))
 2 src/td_dispatcher/vm_exits/tdg_mem_page_attr_wr.c|222| <<tdg_mem_page_attr_wr>> if (!verify_page_info_input(gpa_mappings, LVL_PT, LVL_PDPT))
 3 src/vmm_dispatcher/api_calls/tdh_mem_page_add.c|101| <<tdh_mem_page_add>> if (!verify_page_info_input(gpa_mappings, LVL_PT, LVL_PT))
 4 src/vmm_dispatcher/api_calls/tdh_mem_page_aug.c|92| <<tdh_mem_page_aug>> if (!verify_page_info_input(gpa_mappings, LVL_PT, LVL_PD))
 5 src/vmm_dispatcher/api_calls/tdh_mem_page_demote.c|139| <<tdh_mem_page_demote>> if (!verify_page_info_input(gpa_mappings, LVL_PD, LVL_PDPT))
 6 src/vmm_dispatcher/api_calls/tdh_mem_page_promote.c|236| <<tdh_mem_page_promote>> if (!verify_page_info_input(gpa_mappings, LVL_PD, LVL_PDPT))
 7 src/vmm_dispatcher/api_calls/tdh_mem_page_relocate.c|92| <<tdh_mem_page_relocate>> if (!verify_page_info_input(gpa_mappings, LVL_PT, LVL_PT))
 8 src/vmm_dispatcher/api_calls/tdh_mem_page_remove.c|87| <<tdh_mem_page_remove>> if (!verify_page_info_input(gpa_mappings, LVL_PT, LVL_PDPT))
 9 src/vmm_dispatcher/api_calls/tdh_mem_range_block.c|157| <<tdh_mem_range_block>> if (!verify_page_info_input(sept_level_and_gpa, LVL_PT, tdcs_ptr->executions_ctl_fields.eptp.fields.ept_pwl))
10 src/vmm_dispatcher/api_calls/tdh_mem_range_unblock.c|88| <<tdh_mem_range_unblock>> if (!verify_page_info_input(gpa_mappings, LVL_PT, tdcs_ptr->executions_ctl_fields.eptp.fields.ept_pwl))
11 src/vmm_dispatcher/api_calls/tdh_mem_sept_add.c|351| <<tdh_mem_sept_add>> if (!verify_page_info_input(gpa_mappings, LVL_PD, tdcs_ptr->executions_ctl_fields.eptp.fields.ept_pwl))
12 src/vmm_dispatcher/api_calls/tdh_mem_sept_rd.c|97| <<tdh_mem_sept_rd>> if (!verify_page_info_input(gpa_mappings, LVL_PT, tdcs_ptr->executions_ctl_fields.eptp.fields.ept_pwl))
13 src/vmm_dispatcher/api_calls/tdh_mem_sept_remove.c|104| <<tdh_mem_sept_remove>> if (!verify_page_info_input(gpa_mappings, LVL_PD, tdcs_ptr->executions_ctl_fields.eptp.fields.ept_pwl))
```
* 输入页面的信息的检查函数 `verify_page_info_input()` 确保 GPA 不在保留区域，页面级别符合 Seamcall 的规定，以及 GPA 需和它对应的级别页对齐
  * common/helpers/helpers.c
```cpp
bool_t verify_page_info_input(page_info_api_input_t gpa_page_info, ept_level_t min_level, ept_level_t max_level)
{
    // Verify that GPA mapping input reserved fields equal zero
    if (!is_reserved_zero_in_mappings(gpa_page_info))
    {
        TDX_ERROR("Reserved fields in GPA mappings are not zero\n");
        return false;
    }

    // Verify mapping level input is valid
    if (!((gpa_page_info.level >= min_level) && (gpa_page_info.level <= max_level)))
    {
        TDX_ERROR("Input GPA level (=%d) is not valid\n", gpa_page_info.level);
        return false;
    }

    // Check the page GPA is page aligned
    if (!is_gpa_aligned(gpa_page_info))
    {
        TDX_ERROR("Page GPA 0x%llx is not page aligned\n", gpa_page_info.raw);
        return false;
    }

    return true;
}
```

## SEPT 状态查询表

* 几个宏定义如下
```cpp
#define MAX_SEAMCALL_LEAF 128     //目前最多可以支持 128 个 SEAMCALL
#define MAX_TDCALL_LEAF 32        //目前最多可以支持 32 个 TDCALL
#define MAX_SEPT_STATE_ENC 128    //SEPT state encode 一共占用了 7 bits，可以表示 128 个状态
#define MAX_L2_SEPT_STATE_ENC 16
#define NUM_SEPT_STATES 16        //目前定义了 16 个 SPET state
```
* 下面这个数组（表）定义了具体哪个 SEAMCALL 支持哪些 SEPT states
  * include/auto_gen/sept_state_lookup.c
```cpp
const bool_t seamcall_sept_state_lookup[MAX_SEAMCALL_LEAF][NUM_SEPT_STATES] = {
    [2] = {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 },
    [3] = {1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 },
    [5] = {0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1 },
    [6] = {1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 },
    [7] = {0, 0, 1, 0, 1, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0 },
    [12] = {0, 0, 0, 0, 1, 0, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0 },
    [14] = {0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0 },
    [15] = {0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0 },
    [16] = {0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 },
    [23] = {0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 },
    [25] = {1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1 },
    [29] = {0, 0, 0, 0, 1, 1, 1, 0, 0, 0, 1, 1, 1, 0, 0, 0 },
    [30] = {0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 },
    [39] = {0, 0, 0, 1, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0 },
    [65] = {0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1, 0 },
    [66] = {0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 0, 0, 0, 1, 1, 1 },
    [68] = {0, 0, 0, 0, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1 },
    [75] = {0, 0, 0, 0, 0, 0, 1, 1, 0, 1, 0, 0, 1, 1, 0, 1 },
    [83] = {1, 1, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0 }
};
```
* 例如，SEAMCALL `TDH_MEM_RANGE_BLOCK_LEAF = 7` 支持的 SEPT states 为 `{0, 0, 1, 0, 1, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0 }`，即仅允许 `TDH_MEM_RANGE_BLOCK` 作用于以下 `state = 1` 的 SEPT entry
```cpp
[ 0] FREE = 0
[ 1] REMOVED = 0
[ 2] NL_MAPPED = 1
[ 3] NL_BLOCKED = 0
[ 4] MAPPED = 1
[ 5] BLOCKED = 0
[ 6] BLOCKEDW = 1
[ 7] EXPORTED_BLOCKEDW = 0
[ 8] EXPORTED_DIRTY = 0
[ 9] EXPORTED_DIRTY_BLOCKEDW = 0
[10] PENDING = 1
[11] PENDING_BLOCKED = 0
[12] PENDING_BLOCKEDW = 1
[13] PENDING_EXPORTED_BLOCKEDW = 0
[14] PENDING_EXPORTED_DIRTY = 0
[15] PENDING_EXPORTED_DIRTY_BLOCKEDW = 0
```
* SEPT state encode 一共占用了 7 bits 合并编码如下：
  * include/auto_gen/sept_state_lookup.h
```cpp
// State encoding for SEPT states - Bits PS:D

#define SEPT_STATE_FREE_ENCODING 0xe
#define SEPT_STATE_REMOVED_ENCODING 0xc
#define SEPT_STATE_NL_MAPPED_ENCODING 0x0
#define SEPT_STATE_NL_BLOCKED_ENCODING 0x8
#define SEPT_STATE_MAPPED_ENCODING 0x60
#define SEPT_STATE_BLOCKED_ENCODING 0x68
#define SEPT_STATE_BLOCKEDW_ENCODING 0x64
#define SEPT_STATE_EXPORTED_BLOCKEDW_ENCODING 0x66
#define SEPT_STATE_EXPORTED_DIRTY_ENCODING 0x63
#define SEPT_STATE_EXPORTED_DIRTY_BLOCKEDW_ENCODING 0x67
#define SEPT_STATE_PENDING_ENCODING 0x70
#define SEPT_STATE_PENDING_BLOCKED_ENCODING 0x78
#define SEPT_STATE_PENDING_BLOCKEDW_ENCODING 0x74
#define SEPT_STATE_PENDING_EXPORTED_BLOCKEDW_ENCODING 0x76
#define SEPT_STATE_PENDING_EXPORTED_DIRTY_ENCODING 0x73
#define SEPT_STATE_PENDING_EXPORTED_DIRTY_BLOCKEDW_ENCODING 0x77
#define SEPT_STATE_L2_FREE_ENCODING 0x0
#define SEPT_STATE_L2_NL_MAPPED_ENCODING 0x8
#define SEPT_STATE_L2_NL_BLOCKED_ENCODING 0x1
#define SEPT_STATE_L2_MAPPED_ENCODING 0x6
#define SEPT_STATE_L2_BLOCKED_ENCODING 0x7
```
* 这个是将 SEPT entry 里的 bits 转为 state encode 的宏
  * src/common/memory_handlers/sept_manager.h
```cpp
#define SEPT_CONVERT_TO_ENCODING(ept_entry)  ( ((uint64_t)(ept_entry).state_encoding.state_encoding_0) |   \
                                               (((uint64_t)(ept_entry).state_encoding.state_encoding_1_4) << 1ULL) |  \
                                               (((uint64_t)(ept_entry).state_encoding.state_encoding_5_6) << 5ULL) )
```
* 还有另一个数组（表），因为 SEPT state encode 一共占用了 `7` bits，可以表示 `128` 个状态，所以这个数组映射了 `128` 个 state encode 到 `16` 个 states 的映射关系，以及其他关系。
  * include/auto_gen/sept_state_lookup.c
```cpp
const sept_special_flags_t sept_special_flags_lookup[MAX_SEPT_STATE_ENC] = {
  [14] = {              //SEPT_STATE_FREE_ENCODING 0xe
    .public_state = 0,  //出错时在 out.rdx 中的 state & level 域看到的 state 部分，对应到 TDX module API spec，Table 3.21
    .live_export_allowed = 0,
    .paused_export_allowed = 0,
    .first_time_export_allowed = 0,
    .re_export_allowed = 0,
    .export_cancel_allowed = 0,
    .first_time_import_allowed = 1,
    .re_import_allowed = 0,
    .import_cancel_allowed = 0,
    .mapped_or_pending = 0,
    .any_exported = 0,
    .any_exported_and_dirty = 0,
    .any_exported_and_non_dirty = 0,
    .any_pending = 0,
    .any_pending_and_guest_acceptable = 0,
    .any_blocked = 0,
    .any_blockedw = 0,
    .guest_fully_accessible_leaf = 0,
    .tlb_tracking_required = 0,
    .guest_accessible_leaf = 0,
    .index = 0                  //state：FREE
  },
...
  [104] = {                     //SEPT_STATE_BLOCKED_ENCODING 0x68
    .public_state = 1,          //out.rdx 的 state & level 中的 state：表示 BLOCKED
    .live_export_allowed = 0,
    .paused_export_allowed = 0,
    .first_time_export_allowed = 0,
    .re_export_allowed = 0,
    .export_cancel_allowed = 0,
    .first_time_import_allowed = 0,
    .re_import_allowed = 0,
    .import_cancel_allowed = 0,
    .mapped_or_pending = 0,
    .any_exported = 0,
    .any_exported_and_dirty = 0,
    .any_exported_and_non_dirty = 0,
    .any_pending = 0,
    .any_pending_and_guest_acceptable = 0,
    .any_blocked = 1,
    .any_blockedw = 0,
    .guest_fully_accessible_leaf = 0,
    .tlb_tracking_required = 1,
    .guest_accessible_leaf = 0,
    .index = 5                   //state：BLOCKED
  },
...
}
```
* 比如对于 `FREE` 状态的 SPTE entry，`SEPT_STATE_FREE_ENCODING 0xe` 就是 `14`，那么它的 `.index = 0` 对应到第一个状态 `FREE`
* 比如对于 `BLOCKED` 状态的 SPTE entry，`SEPT_STATE_BLOCKED_ENCODING 0x68` 就是 `104`，那么它的 `.index = 5` 对应到第五个状态 `BLOCKED`
* `sept_state_is_seamcall_leaf_allowed()` 函数判断 SEAMCALL `current_leaf` 是否允许作用于 SEPT entry `ept_entry`
  * 它通过 `SEPT_CONVERT_TO_ENCODING(ept_entry)` 将 SEPT entry 中用于表示 state 的 7 bits 提取出来 `sept_state_enc`
  * 然后传给要求输入为 encode 好 state bits 的 `septe_state_encoding_is_seamcall_allowed()` 函数去判断
* `septe_state_encoding_is_seamcall_allowed()` 就利用 encode 过的 SPTE 状态和 `sept_special_flags_lookup[]`、`seamcall_sept_state_lookup[]` 数组来返回是否允许 SEAMCALL 操作
* src/common/memory_handlers/sept_manager.h
```cpp
_STATIC_INLINE_ bool_t sept_state_is_seamcall_leaf_allowed(seamcall_leaf_opcode_t current_leaf, ia32e_sept_t ept_entry)
{
    uint64_t sept_state_enc = SEPT_CONVERT_TO_ENCODING(ept_entry);
    return septe_state_encoding_is_seamcall_allowed(sept_state_enc, current_leaf);
}

_STATIC_INLINE_ bool_t septe_state_encoding_is_seamcall_allowed(uint64_t septe_state_enc, seamcall_leaf_opcode_t leaf_number)
{
    tdx_debug_assert(septe_state_enc < (MAX_SEPT_STATE_ENC));
    tdx_debug_assert((uint64_t)leaf_number < MAX_SEAMCALL_LEAF);

    uint32_t septe_state_enc_index = sept_special_flags_lookup[septe_state_enc].index;
    tdx_debug_assert(septe_state_enc_index < NUM_SEPT_STATES);

    bool_t is_seamcall_allowed = seamcall_sept_state_lookup[leaf_number][septe_state_enc_index];

    return is_seamcall_allowed;
}
```

## 地址翻译
* TD guest 通过 Secure EPT 的翻译对 private memory 的访问，确实无需借助软件将 HKID 编码到 SEPT 的 entry 中。如 TDX spec 所说，是由 CPU 在翻译的最后将 HKID 设置到 HPA 上的。
* MKTME spec 中所说的将 key ID 编码到 PT entry 中，让 MKTME engine 用指定的 key 去读写数据的方式依然是有效的。
  * 在 TD 的场景下，常见于 TDX module 代替 TD，用 TD 的 key 去读写其所属的数据的时候。
    * 例如：Host 通过 SEAMCALL `tdh_mem_page_add()` 将一个 page 添加到 TD 时，需要将源 page 中的内容复制至属于 TD 的目标 page，这个操作不需要经过 TD guest，但仍需用 TD 对应的 key 将数据写入。
* TDX module 里用的 `map_pa()` 不是恒等映射，而是在预先分配好的页表空间中创建的临时映射，返回的是一个逻辑地址，使用完之后是需要 free 的。
  * 但是映射的 PA 是 TDX module 完全可控的，如果给 `map_pa()` 传的第一个参数，即 `pa` 中嵌入了 HKID（见 `check_lock_and_map_explicit_private_4k_hpa()`），那么填充到页表条目里的 PA 就带上了（见 `fill_keyhole_pte()`），并且在访存时，MKTME 也会据此进行加解密。
* tdx-module/src/common/helpers/helpers.c
```cpp
api_error_type check_lock_and_map_explicit_private_4k_hpa(
        pa_t hpa,
        uint64_t operand_id,
        tdr_t* tdr_p,
        mapping_type_t mapping_type,
        lock_type_t lock_type,
        page_type_t expected_pt,
        pamt_block_t* pamt_block,
        pamt_entry_t** pamt_entry,
        bool_t* is_locked,
        void**         la
        )
{
    api_error_type errc;

    errc = check_and_lock_explicit_4k_private_hpa( hpa, operand_id,
             lock_type, expected_pt, pamt_block, pamt_entry, is_locked);
    if (errc != TDX_SUCCESS)
    {
        return errc;
    }

    pa_t hpa_with_hkid = assign_hkid_to_hpa(tdr_p, hpa);

    *la = map_pa((void*)hpa_with_hkid.full_pa, mapping_type);

    return TDX_SUCCESS;
}
```
* tdx-module/src/vmm_dispatcher/api_calls/tdh_mem_page_add.c
```cpp
api_error_type tdh_mem_page_add(page_info_api_input_t gpa_page_info,
                           uint64_t target_tdr_pa,
                           uint64_t target_page_pa,
                           uint64_t source_page_pa)
{
...
    //该函数会给目标页面 td_page_pa 创建临时映射 td_page_ptr，并且在 PA 中嵌入 TD 的 HKID
    // Check, lock and map the new TD page
    return_val = check_lock_and_map_explicit_private_4k_hpa(td_page_pa,
                                                            OPERAND_ID_R8,
                                                            tdr_ptr,
                                                            TDX_RANGE_RW,
                                                            TDX_LOCK_EXCLUSIVE,
                                                            PT_NDA,
                                                            &td_page_pamt_block,
                                                            &td_page_pamt_entry_ptr,
                                                            &td_page_locked_flag,
                                                            (void**)&td_page_ptr);
...
    //源页面是 shared 的，因此它创建的临时映射并不需要嵌入 HKID
    // Map the source page
    source_page_ptr = map_pa((void*)source_pa.raw, TDX_RANGE_RO);
    //TDX module 代 TD guest 进行数据复制时，用的是 TD 的 key，借助的就是之前嵌入在临时映射的页表项中的 HKID
    // Copy the source image to the target TD page, using the TD’s ephemeral
    // private HKID and direct writes (MOVDIR64B)
    cache_aligned_copy_direct((uint64_t)source_page_ptr, (uint64_t)td_page_ptr, TDX_PAGE_SIZE_IN_BYTES);
    //对于提供给 TD guest 使用的 SEPT 页表项无需嵌入 HKID，CPU PMH 会在翻译的最后自动把 HKID 加入到 HPA 中
    // Update the parent EPT entry with the new TD page HPA and SEPT_PRESENT state
    sept_set_leaf_unlocked_entry(page_sept_entry_ptr, SEPT_PERMISSIONS_RWX, td_page_pa, SEPT_STATE_MAPPED_MASK);
...
EXIT:
    // Release all acquired locks and free keyhole mappings
    if (td_page_locked_flag)
    {
        pamt_unwalk(td_page_pa, td_page_pamt_block, td_page_pamt_entry_ptr, TDX_LOCK_EXCLUSIVE, PT_4KB);
        free_la(td_page_ptr);
    }
    if (page_sept_entry_ptr != NULL)
    {
        free_la(page_sept_entry_ptr);
    }
    if (source_page_ptr != NULL)
    {
        free_la(source_page_ptr);
    }
...
}
```