Arduino ULP v1.0
==================
This guide explains how to setup Arduino to use ULP assembly files for your esp32 projects. Currently this guide is only geared for MacOS but will probably work with Linux. Windows is not supported yet but if you port it over let me know. Must have python 2.7 or higher installed which most likely you have if you use Arduino and esp. This is still beta and many things could go wrong so let me know if you encounter any issues.

Typically in Arduino you can compile assembly files using the '.S' extension. Using the ESP32 Arduino core framework these files would correspond to the Xtensa processors whose toolchain is incompatible with the ULP coprocessor. Luckily Arduino provides a fairly easy albeit not that flexible build framework using series of recipes. This guide extends the esp32 recipes for building the ULP assembly files. We will use the '.s' extensions for ULP assembly files which Arduino will let you create. I tried to keep the ulp build process the same as the esp-if framework with a few small modifications the users needs to do to compile in Arduino.

Setup Steps
===========
1. Download this repository -> "arduino_ulp".
2. Download the pre-compiled binutils-esp32ulp toolchain for Mac/Linux: https://github.com/espressif/binutils-esp32ulp/wiki.
3. Find your Arduino-esp32 core directory which Arduino IDE uses. Typically ../Arduino/hardware/esp32
4. In the "arduino_ulp" repository folder you downloaded, copy the folder 'ulp' to ../esp32/tools/sdk/include/ replacing the existing folder named 'ulp'."
5. In the 'arduino_ulp' repository folder you downloaded, copy the file "platform.txt" to ../esp32 replacing the one you have. If you want just remain the old "platform.txt" to save it if you want to revert back.
6. In the 'arduino_ulp' repository folder you downloaded, copy the 'ulp_example' folder to where Arduino saves or sketchs. 
7. Copy the binutils-esp32ulp toolchain you downloaded to ../esp32/tools.

Thats it, you now have all the files in place, lets look at very simple example to get you compiling ulp code now!

Example:
========
Open a blank Arduino sketch and copy and paste the code below into the that sketch.
```
#include "esp32/ulp.h"
// include ulp header you created
#include "ulp_main.h"

// Unlike the esp-idf always use these binary blob names
extern const uint8_t ulp_main_bin_start[] asm("_binary_ulp_main_bin_start");
extern const uint8_t ulp_main_bin_end[]   asm("_binary_ulp_main_bin_end");

static void init_run_ulp(uint32_t usec);

void setup() {
    Serial.begin(115200);
    delay(1000);
    init_run_ulp(100 * 1000); // 100 msec
}

void loop() {
    // ulp variables data is the lower 16 bits
    Serial.printf("ulp count: %i\n", ulp_count & 0xFFFF);
    delay(100);
}

static void init_run_ulp(uint32_t usec) {
    // initialize ulp variable
    ulp_count = 0;
    ulp_set_wakeup_period(0, usec);
    esp_err_t err = ulp_load_binary(0, ulp_main_bin_start, (ulp_main_bin_end - ulp_main_bin_start) / sizeof(uint32_t));
    // ulp coprocessor will run on its own now
    err = ulp_run((&ulp_entry - RTC_SLOW_MEM) / sizeof(uint32_t));
}
```

Create a new tab named <b>ulp.s</b>, take notice that the extension is a lower case 's'. Copy the code below into that ulp assembly file.
```
/* Define variables, which go into .bss section (zero-initialized data) */
    .bss
/* Store count value */
    .global count
count:
    .long 0

/* Code goes into .text section */
    .text
    .global entry
entry:
    move    r3, count
    ld      r0, r3, 0 
    add     r0, r0, 1
    st      r0, r3, 0
    halt
```

Create a new tab named <b>ulp_main.h</b>. This header allows your sketch to see global variables who's memory is allocated your ulp assembly file. This is in the SLOW RTC memory section. Copy the code below into that header file. As with the esp-idf you have to add 'ulp_' to the front of the variable name. Unlike esp-idf the name of this header is always this name.
```
/*
    Put your ULP globals here you want visibility
    for your sketch. Add "ulp_" to the beginning
    of the variable name and must be size 'uint32_t'
*/
#include "Arduino.h"

extern uint32_t ulp_entry;
extern uint32_t ulp_count;
```

Compile and run and you should see the variable "ulp_count" increment every 100 msecs.

Under the Hood
==============
...

Limitations
==============
...