TARGET     = snander
PKG        = $(TARGET)
VERSION    = 1.7.9-1

CC        ?= gcc
# Detect clang-based MinGW toolchains (llvm-mingw); clang differs from GCC
# in a few flags (no -posix, no -rpath with lld, stricter callback warnings).
CC_IS_CLANG := $(shell $(CC) --version 2>/dev/null | grep -q clang && echo yes)
STRIP     ?= strip
INSTALL   ?= install
WINDRES   ?= windres
PREFIX    ?= /usr
BINDIR    ?= $(PREFIX)/bin
CFLAGS    ?= -std=gnu99 -Wall -O2 -D_FILE_OFFSET_BITS=64
LDFLAGS   ?= -pthread
RES       ?= res.bin

EEPROM_SUPPORT = yes

SRCS = flashcmd_api.c spi_controller.c spi_nand_flash.c spi_nor_flash.c ch347_spi.c timer.c main.c

ifdef LIBS_BASE
CFLAGS += -I$(LIBS_BASE)/include
ifeq ($(CC_IS_CLANG),)
LDFLAGS += -L$(LIBS_BASE)/lib -Wl,-rpath -Wl,$(LIBS_BASE)/lib
else
LDFLAGS += -L$(LIBS_BASE)/lib
endif
endif

ifeq ($(CONFIG_STATIC), yes)
LDFLAGS += -static
endif

ifeq ($(EEPROM_SUPPORT), yes)
CFLAGS += -DEEPROM_SUPPORT
SRCS += i2c_eeprom.c spi_eeprom.c bitbang_microwire.c mw_eeprom.c ch341a_gpio.c ch347_i2c.c
endif

ifeq ($(TARGET_OS), MinGW)
EXEC_SUFFIX := .exe
SRCS += $(RES)
# GCC MinGW uses -posix for the posix threading model; clang (llvm-mingw) rejects it.
ifeq ($(CC_IS_CLANG),)
CFLAGS += -posix
else
# The source passes plain callbacks to libusb (stdcall on Windows): clang treats
# this as an error while GCC treats it as a warning. Downgrade it for clang.
CFLAGS += -Wno-incompatible-function-pointer-types
endif
CFLAGS += -Dffs=__builtin_ffs
CFLAGS += -D__USE_MINGW_ANSI_STDIO=1
endif

ifeq ($(TARGET_OS), Darwin)
CFLAGS += -Wno-gnu-designator
LDFLAGS += -lobjc -Wl,-framework,IOKit -Wl,-framework,CoreFoundation -Wl,-framework,Security
endif

$(PKG): $(TARGET)$(EXEC_SUFFIX)

$(TARGET)$(EXEC_SUFFIX): $(SRCS)
	$(CC) $(CFLAGS) $(SRCS) $(LDFLAGS) -lusb-1.0 -o $@

$(RES):
	$(WINDRES) -i $(TARGET).rc -o $@

clean: 
	rm -f $(TARGET)$(EXEC_SUFFIX) $(RES)

strip: $(TARGET)$(EXEC_SUFFIX)
	$(STRIP) $(TARGET)$(EXEC_SUFFIX)

install: $(TARGET)
	mkdir -p $(DESTDIR)$(BINDIR)
	mkdir -p $(DESTDIR)/etc/udev/rules.d
	$(INSTALL) -m 0755 $(TARGET) $(DESTDIR)$(BINDIR)
	$(INSTALL) -m 0664 40-persistent-ch347.rules $(DESTDIR)/etc/udev/rules.d/40-persistent-ch347.rules

install-udev-rule:
	cp 40-persistent-ch347.rules /etc/udev/rules.d/
	udevadm control --reload-rules
.PHONY: clean install-udev-rule

debian/changelog:
	dch --create -v $(VERSION) --package $(PKG)

deb:
	dpkg-buildpackage -b -us -uc
.PHONY: install debian/changelog deb
