
## <b>Nx_WebServer application description</b>

This application provides an example of Azure RTOS NetX Duo stack usage on STM32F769I-DISCO board, it shows how to develop Web HTTP server based application.
The application is designed to load files and static/dynamic web pages stored in SD card using a Web HTTP server, the code provides all required features to build a compliant Web HTTP Server.
The main entry function tx_application_define() is called by ThreadX during kernel start, at this stage, all NetXDuo and FileX resources are created.

 + A NX_PACKET_POOL **AppPool** is allocated
 + A NX_IP instance **IpInstance** using that pool is initialized
 + A NX_PACKET_POOL **WebServerPool** is allocated
 + A NX_WEB_HTTP_SERVER **HTTPServer** instance using that pool is initialized
 + The ARP, ICMP and protocols (TCP and UDP) are enabled for the NX_IP instance
 + A DHCP client is created.

 The application then creates 2 threads with different priorities:

 + **AppMainThread** (priority 10, PreemtionThreashold 10) : created with the TX_AUTO_START flag to start automatically.
 + **AppWebServerThread** (priority 5, PreemtionThreashold 5) : created with the TX_DONT_START flag to be started later.

The **AppMainThread** starts and perform the following actions:

  + starts the DHCP client
  + waits for the IP address resolution
  + resumes the **AppWebServerThread**

The **AppWebServerThread**, once started:

The user's browser can load a static web page, download a zip folder or watch a video that are stored in SD card.
In order to open simultaneous sessions for the server, you should define the number of simultaneous sessions NX_WEB_HTTP_SERVER_SESSION_MAX in "nx_user.h".

#### <b>Expected success behavior</b>

When an SD card is inserted into the STM32F769I-DISCO SD card reader and the board is powered up and connected to DHCP enabled Ethernet network.
when the Web HTTP server is successfully started the files can be loaded on the web browser after entring the url http://@IP/file_name.
An example web page is provided for testing the application that can be found under "NetXDuo/Nx_WebServer/Web_Content/dashboard.html".

#### <b>Error behaviors</b>

If the WEB HTTP server is not successfully started, this message "HTTP WEB Server successfully started" is printed on the HyperTerminal.
In case of other errors, the Web HTTP server does not operate as designed (Files stored in the SD card are not loaded in the web browser).

#### <b>Assumptions if any</b>

The uSD card must be plugged before starting the application.
The board must be in a DHCP Ethernet network.

#### <b>Known limitations</b>

  - Hotplug is not implemented for this example, that is, the SD card is expected to be inserted before application running.
  - The packet pool is not optimized. It can be less than that by reducing NX_PACKET_POOL_SIZE in file "app_netxduo.h" and NX_APP_MEM_POOL_SIZE in file "app_azure_rtos_config.h". This update can decrease NetXDuo performance.

### <b>Notes</b>

 1.  If the user code size exceeds the DTCM-RAM size or starts from internal cacheable memories (SRAM1 and SRAM2), that is shared between several processors,
      then it is highly recommended to enable the CPU cache and maintain its coherence at application level.
      The address and the size of cacheable buffers (shared between CPU and other masters) must be properly updated to be aligned to cache line size (32 bytes).

#### <b>ThreadX usage hints</b>

  - ThreadX uses the Systick as time base, thus it is mandatory that the HAL uses a separate time base through the TIM IPs.
  - ThreadX is configured with 100 ticks/sec by default, this should be taken into account when using delays or timeouts at application. It is always possible to reconfigure it in the "tx_user.h", the "TX_TIMER_TICKS_PER_SECOND" define,but this should be reflected in "tx_initialize_low_level.s" file too.
  - ThreadX is disabling all interrupts during kernel start-up to avoid any unexpected behavior, therefore all system related calls (HAL, BSP) should be done either at the beginning of the application or inside the thread entry functions.
  - ThreadX offers the "tx_application_define()" function, that is automatically called by the tx_kernel_enter() API.
   It is highly recommended to use it to create all applications ThreadX related resources (threads, semaphores, memory pools...)  but it should not in any way contain a system API call (HAL or BSP).
  - Using dynamic memory allocation requires to apply some changes to the linker file.
   ThreadX needs to pass a pointer to the first free memory location in RAM to the tx_application_define() function,
   using the "first_unused_memory" argument.
   This require changes in the linker files to expose this memory location.
    + For EWARM add the following section into the .icf file:
     ```
	 place in RAM_region    { last section FREE_MEM };
	 ```
    + For MDK-ARM:
	```
    either define the RW_IRAM1 region in the ".sct" file
    or modify the line below in "tx_low_level_initilize.s to match the memory region being used
        LDR r1, =|Image$$RW_IRAM1$$ZI$$Limit|
	```
    + For STM32CubeIDE add the following section into the .ld file:
	``` 
    ._threadx_heap :
      {
         . = ALIGN(8);
         __RAM_segment_used_end__ = .;
         . = . + 64K;
         . = ALIGN(8);
       } >RAM AT> RAM
	``` 

       The simplest way to provide memory for ThreadX is to define a new section, see ._threadx_heap above.
       In the example above the ThreadX heap size is set to 64KBytes.
       The ._threadx_heap must be located between the .bss and the ._user_heap_stack sections in the linker script.	 
       Caution: Make sure that ThreadX does not need more than the provided heap memory (64KBytes in this example).	 
       Read more in STM32CubeIDE User Guide, chapter: "Linker script".
	  
    + The "tx_initialize_low_level.s" should be also modified to enable the "USE_DYNAMIC_MEMORY_ALLOCATION" flag.

#### <b>NetX Duo usage hints</b>

 - The NetXDuo application needs to allocate the <b> <i> NX_PACKET </i> </b> pool in a dedicated section that is  configured as below is an example of the section declaration for different IDEs.
   + For EWARM ".icf" file
   ```
   define symbol __ICFEDIT_region_NXDATA_start__  = 0x20062000;
   define symbol __ICFEDIT_region_NXDATA_end__   = 0x2007FFFF;
   define region NXAppPool_region  = mem:[from __ICFEDIT_region_NXDATA_start__ to __ICFEDIT_region_NXDATA_end__];
   place in NXAppPool_region { section .NXAppPoolSection};
   ```
   + For MDK-ARM
   ```
    RW_NXDriverSection 0x20062000 0x1A000  {
  *(.NetXPoolSection)
  }
   ```
   + For STM32CubeIDE ".ld" file
   ``` 
   .nx_section 0x20062000 (NOLOAD): {
     *(.NXAppPoolSection)
     } >RAM
   ```

  this section is then used in the <code> app_azure_rtos.c</code> file to force the <code>nx_byte_pool_buffer</code> allocation.

```
/* USER CODE BEGIN NX_Pool_Buffer */

#if defined ( __ICCARM__ ) /* IAR Compiler */
#pragma location = ".NetXPoolSection"

#elif defined ( __CC_ARM ) /* MDK ARM Compiler */
__attribute__((section(".NetXPoolSection")))

#elif defined ( __GNUC__ ) /* GNU Compiler */
__attribute__((section(".NetXPoolSection")))
#endif

/* USER CODE END NX_Pool_Buffer */
static UCHAR  nx_byte_pool_buffer[NX_APP_MEM_POOL_SIZE];
static TX_BYTE_POOL nx_app_byte_pool;
```
For more details about the MPU configuration please refer to the [AN4838](https://www.st.com/resource/en/application_note/dm00272912-managing-memory-protection-unit-in-stm32-mcus-stmicroelectronics.pdf)

### <b>Keywords</b>

RTOS, ThreadX, Network, NetxDuo, FileX, File ,SDMMC, UART


### <b>Hardware and Software environment</b>

  - This application runs on STM32F769xx devices
  - This application has been tested with STMicroelectronics STM32F769I-Discovery board MB1225 Revision B-02
    and can be easily tailored to any other supported device and development board.

  - This application uses USART1 to display logs, the hyperterminal configuration is as follows:
      - BaudRate = 115200 baud
      - Word Length = 8 Bits
      - Stop Bit = 1
      - Parity = None
      - Flow control = None

### <b>How to use it ?</b>

In order to make the program work, you must do the following :

 - Open your preferred toolchain
 - Rebuild all files and load your image into target memory
 - Run the application
