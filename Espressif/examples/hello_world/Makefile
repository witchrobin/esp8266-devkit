#############################################################
#
# Root Level Makefile
#
# (c) by CHERTS <sleuthhound@gmail.com>
#
#############################################################

BUILD_BASE	= build
FW_BASE		= firmware

# Base directory for the compiler
XTENSA_TOOLS_ROOT ?= c:/Espressif/xtensa-lx106-elf/bin

# base directory of the ESP8266 SDK package, absolute
SDK_BASE	?= c:/Espressif/ESP8266_SDK
SDK_TOOLS	?= c:/Espressif/utils

# esptool path and port
ESPTOOL ?= $(SDK_TOOLS)/esptool.exe
ESPPORT ?= COM2
# old - use c:/Espressif/utils/esptool.exe util
# new - use c:/Espressif/utils/esptool-ck.exe util
ESPTOOLMODE ?= new

# SPI_SPEED = 40, 26, 20, 80
SPI_SPEED ?= 40
# SPI_MODE: qio, qout, dio, dout
SPI_MODE ?= qio
# SPI_SIZE: 512K, 256K, 1M, 2M, 4M
SPI_SIZE ?= 4M

ifeq ($(ESPTOOLMODE), new)
    esptoolmode = new
else
    esptoolmode = old
endif

ifeq ($(SPI_SPEED), 26)
	flashimageoptions = -ff 26m
else
    ifeq ($(SPI_SPEED), 20)
        flashimageoptions = -ff 20m
    else
        ifeq ($(SPI_SPEED), 80)
		flashimageoptions = -ff 80m
        else
		flashimageoptions = -ff 40m
        endif
    endif
endif

ifeq ($(SPI_MODE), qout)
	flashimageoptions += -fm qout
else
    ifeq ($(SPI_MODE), dio)
	flashimageoptions += -fm dio
    else
        ifeq ($(SPI_MODE), dout)
		flashimageoptions += -fm dout
        else
		flashimageoptions += -fm qio
        endif
    endif
endif

# flash larger than 1024KB only use 1024KB to storage user1.bin and user2.bin
ifeq ($(SPI_SIZE), 256K)
	flashimageoptions += -fs 2m
else
    ifeq ($(SPI_SIZE), 1M)
	flashimageoptions += -fs 8m
    else
        ifeq ($(SPI_SIZE), 2M)
		flashimageoptions += -fs 16m
        else
            ifeq ($(SPI_SIZE), 4M)
		flashimageoptions += -fs 32m
            else
		flashimageoptions += -fs 4m
            endif
        endif
    endif
endif

# name for the target project
TARGET = app

# which modules (subdirectories) of the project to include in compiling
MODULES	= driver user
EXTRA_INCDIR = include $(SDK_BASE)/../extra/include

# libraries used in this project, mainly provided by the SDK
LIBS = c gcc hal phy pp net80211 lwip wpa main

# compiler flags using during compilation of source files
CFLAGS = -Os -g -O2 -Wpointer-arith -Wundef -Werror -Wl,-EL -fno-inline-functions -nostdlib -mlongcalls -mtext-section-literals -mno-serialize-volatile -D__ets__ -DICACHE_FLASH

# linker flags used to generate the main object file
LDFLAGS = -nostdlib -Wl,--no-check-sections -u call_user_start -Wl,-static

# linker script used for the above linkier step
LD_SCRIPT = eagle.app.v6.ld

ifeq ($(esptoolmode), new)
    ESPTOOL	= $(SDK_TOOLS)/esptool-ck.exe
endif

# various paths from the SDK used in this project
SDK_LIBDIR	= lib
SDK_LDDIR	= ld
SDK_INCDIR	= include include/json

# select which tools to use as compiler, librarian and linker
CC		:= $(XTENSA_TOOLS_ROOT)/xtensa-lx106-elf-gcc
AR		:= $(XTENSA_TOOLS_ROOT)/xtensa-lx106-elf-ar
LD		:= $(XTENSA_TOOLS_ROOT)/xtensa-lx106-elf-gcc
OBJCOPY := $(XTENSA_TOOLS_ROOT)/xtensa-lx106-elf-objcopy
OBJDUMP := $(XTENSA_TOOLS_ROOT)/xtensa-lx106-elf-objdump

# no user configurable options below here
SRC_DIR		:= $(MODULES)
BUILD_DIR	:= $(addprefix $(BUILD_BASE)/,$(MODULES))

SDK_LIBDIR	:= $(addprefix $(SDK_BASE)/,$(SDK_LIBDIR))
SDK_INCDIR	:= $(addprefix -I$(SDK_BASE)/,$(SDK_INCDIR))

SRC			:= $(foreach sdir,$(SRC_DIR),$(wildcard $(sdir)/*.c))
OBJ			:= $(patsubst %.c,$(BUILD_BASE)/%.o,$(SRC))
LIBS		:= $(addprefix -l,$(LIBS))
APP_AR		:= $(addprefix $(BUILD_BASE)/,$(TARGET)_app.a)
TARGET_OUT	:= $(addprefix $(BUILD_BASE)/,$(TARGET).out)

LD_SCRIPT	:= $(addprefix -T$(SDK_BASE)/$(SDK_LDDIR)/,$(LD_SCRIPT))

INCDIR			:= $(addprefix -I,$(SRC_DIR))
EXTRA_INCDIR	:= $(addprefix -I,$(EXTRA_INCDIR))
MODULE_INCDIR	:= $(addsuffix /include,$(INCDIR))

V ?= $(VERBOSE)
ifeq ("$(V)","1")
Q :=
vecho := @true
else
Q := @
vecho := @echo
endif

vpath %.c $(SRC_DIR)

define compile-objects
$1/%.o: %.c
	$(vecho) "CC $$<"
	$(Q) $(CC) $(INCDIR) $(MODULE_INCDIR) $(EXTRA_INCDIR) $(SDK_INCDIR) $(CFLAGS)  -c $$< -o $$@
endef

.PHONY: all checkdirs clean flash flashinit flashonefile rebuild

all: checkdirs $(TARGET_OUT)

$(TARGET_OUT): $(APP_AR)
	$(vecho) "LD $@"
	$(Q) $(LD) -L$(SDK_LIBDIR) $(LD_SCRIPT) $(LDFLAGS) -Wl,--start-group $(LIBS) $(APP_AR) -Wl,--end-group -o $@
	$(vecho) "------------------------------------------------------------------------------"
	$(vecho) "Section info:"
	$(Q) $(OBJDUMP) -h -j .data -j .rodata -j .bss -j .text -j .irom0.text $@
	$(vecho) "------------------------------------------------------------------------------"
	$(vecho) "Section info:"
	$(Q) $(SDK_TOOLS)/memanalyzer.exe $(OBJDUMP).exe $@
	$(vecho) "------------------------------------------------------------------------------"
	$(Q) $(ESPTOOL) -eo $@ -bo $(FW_BASE)/eagle.flash.bin -bm $(SPI_MODE) -bz $(SPI_SIZE) -bf $(SPI_SPEED) -bs .text -bs .data -bs .rodata -bc -ec -eo $@ -es .irom0.text $(FW_BASE)/eagle.irom0text.bin -ec
	$(vecho) "Generate eagle.flash.bin and eagle.irom0text.bin successully in folder $(FW_BASE)."
	$(vecho) "eagle.flash.bin-------->0x00000"
	$(vecho) "eagle.irom0text.bin---->0x40000"
	$(vecho) "Done"

$(APP_AR): $(OBJ)
	$(vecho) "AR $@"
	$(Q) $(AR) cru $@ $^

checkdirs: $(BUILD_DIR) $(FW_BASE)

$(BUILD_DIR):
	$(Q) mkdir -p $@

$(FW_BASE):
	$(Q) mkdir -p $@

flashonefile: all
	$(SDK_TOOLS)/gen_flashbin.exe $(FW_BASE)/eagle.flash.bin $(FW_BASE)/eagle.irom0text.bin 0x40000
	$(Q) mv eagle.app.flash.bin $(FW_BASE)/
	$(vecho) "Generate eagle.app.flash.bin successully in folder $(FW_BASE)."
	$(vecho) "eagle.app.flash.bin-------->0x00000"
ifeq ($(esptoolmode), new)
	$(ESPTOOL) -cp $(ESPPORT) -cd ck -cb 256000 -ca 0x00000 -cf $(FW_BASE)/eagle.app.flash.bin -v
else
	$(ESPTOOL) -p $(ESPPORT) -b 256000 write_flash $(flashimageoptions) 0x00000 $(FW_BASE)/eagle.app.flash.bin
endif

flash: all
ifeq ($(esptoolmode), new)
	$(ESPTOOL) -cp $(ESPPORT) -cd ck -ca 0x00000 -cf $(FW_BASE)/eagle.flash.bin -v
	$(ESPTOOL) -cp $(ESPPORT) -cd ck -cb 256000 -ca 0x40000 -cf $(FW_BASE)/eagle.irom0text.bin -v
else
	$(ESPTOOL) -p $(ESPPORT) -b 256000 write_flash $(flashimageoptions) 0x00000 $(FW_BASE)/eagle.flash.bin 0x40000 $(FW_BASE)/eagle.irom0text.bin
endif

flashinit:
	$(vecho) "Flash init data:"
	$(vecho) "Clear old settings (EEP area):"
	$(vecho) "clear_eep.bin-------->0x79000"
	$(vecho) "Default config (Clear SDK settings):"
	$(vecho) "blank.bin-------->0x7E000"
	$(vecho) "esp_init_data_default.bin-------->0x7C000"
ifeq ($(esptoolmode), new)
	$(ESPTOOL) -cp $(ESPPORT) -cd ck -ca 0x7c000 -cf $(SDK_BASE)/bin/esp_init_data_default.bin -v
	$(ESPTOOL) -cp $(ESPPORT) -cd ck -ca 0x7e000 -cf $(SDK_BASE)/bin/blank.bin -v
else
	$(ESPTOOL) -p $(ESPPORT) -b $(ESPBAUD) write_flash $(flashimageoptions) 0x79000 $(SDK_BASE)/bin/clear_eep.bin 0x7c000 $(SDK_BASE)/bin/esp_init_data_default.bin 0x7e000 $(SDK_BASE)/bin/blank.bin
endif

rebuild: clean all

clean:
	$(Q) rm -f $(APP_AR)
	$(Q) rm -f $(TARGET_OUT)
	$(Q) rm -rf $(BUILD_DIR)
	$(Q) rm -rf $(BUILD_BASE)
	$(Q) rm -rf $(FW_BASE)

$(foreach bdir,$(BUILD_DIR),$(eval $(call compile-objects,$(bdir))))
