# Nuke built-in rules.
.SUFFIXES:

# Delete the target of a failed recipe.
.DELETE_ON_ERROR:

# This is the name that our final executable will have.
# Change as needed.
override OUTPUT := efi-template

# Install prefix; /usr/local is a good, standard default pick.
PREFIX := /usr/local

# Default user QEMU flags. These are appended to the QEMU command calls.
QEMUFLAGS := -m 2G
QEMU_MACHINE := -M q35
QEMU_CPU :=
# Uncomment this line and comment out/remove the xHCI/USB keyboard/mouse
# block below if the program does not support xHCI/USB and wants legacy
# PS/2 keyboard and mouse instead.
#QEMU_DEVICES :=
QEMU_DEVICES := \
    -device qemu-xhci \
    -device usb-kbd \
    -device usb-mouse

override QEMUFLAGS := \
    $(QEMU_MACHINE) \
    $(QEMU_CPU) \
    $(QEMU_DEVICES) \
    $(QEMUFLAGS)
override QEMU_UEFI_FW := \
    -drive if=pflash,unit=0,format=raw,file=edk2-ovmf-bins/ovmf-code-x86_64.fd,readonly=on

# User controllable toolchain and toolchain prefix.
TOOLCHAIN :=
TOOLCHAIN_PREFIX :=
ifneq ($(TOOLCHAIN),)
    ifeq ($(TOOLCHAIN_PREFIX),)
        TOOLCHAIN_PREFIX := $(TOOLCHAIN)-
    endif
endif

# User controllable C compiler command.
ifneq ($(TOOLCHAIN_PREFIX),)
    CC := $(TOOLCHAIN_PREFIX)gcc
else
    CC := cc
endif

# User controllable objcopy command.
OBJCOPY := $(TOOLCHAIN_PREFIX)objcopy

# Defaults overrides for variables if using "llvm" as toolchain.
ifeq ($(TOOLCHAIN),llvm)
    CC := clang
endif

# User controllable C flags.
CFLAGS := -g -O2 -pipe

# User controllable C preprocessor flags. We set none by default.
CPPFLAGS :=

# User controllable nasm flags.
NASMFLAGS := -g

# User controllable linker flags. We set none by default.
LDFLAGS :=

# The goals other than cleaning and uninstalling, which need neither the
# dependencies nor a toolchain.
override BUILD_GOALS := $(filter-out clean distclean uninstall,$(or $(MAKECMDGOALS),all))

# Ensure the dependencies have been obtained, and set up the toolchain
# helper, which records the flags as given, before the internal ones.
ifneq ($(BUILD_GOALS),)
    ifeq ($(wildcard .deps-obtained),)
        $(error Please run the ./get-deps script first)
    endif
    override TCH_ARCH := x86_64
    override TCH_TARGET_MACRO := __x86_64__
    override TCH_CLANG_TARGET := x86_64-unknown-none-elf
    override TCH_SELECTORS := TOOLCHAIN TOOLCHAIN_PREFIX
    override TCH_CROSS_HINT := use TOOLCHAIN=llvm or a cross compiler for x86_64 with TOOLCHAIN_PREFIX
    include gmake-toolchain-helper/init.mk
endif

# Internal C flags that should not be changed by the user.
override CFLAGS += \
    -Wall \
    -Wextra \
    -std=gnu11 \
    -ffreestanding \
    -fno-builtin \
    -fno-exceptions \
    -fno-asynchronous-unwind-tables \
    -fno-unwind-tables \
    -fno-common \
    -fno-strict-aliasing \
    -fno-stack-protector \
    -fno-stack-check \
    -fshort-wchar \
    -fPIE \
    -fvisibility=hidden \
    -ffunction-sections \
    -fdata-sections \
    -m64 \
    -march=x86-64 \
    -mno-mmx \
    -mno-sse \
    -mno-red-zone

# Internal C preprocessor flags that should not be changed by the user.
override CPPFLAGS := \
    -nostdinc \
    -include hidden.h \
    -I src \
    -I picoefi/include \
    -isystem freestanding-c-hdrs/include \
    $(CPPFLAGS) \
    -MD \
    -MP

# Internal nasm flags that should not be changed by the user.
override NASMFLAGS := \
    -f elf64 \
    $(patsubst -g,-g -F dwarf,$(NASMFLAGS)) \
    -i src/ \
    -Wall

# Internal linker flags that should not be changed by the user.
override LDFLAGS += \
    -nostdlib \
    -Wl,-pie \
    -Wl,-z,text \
    -Wl,-z,max-page-size=0x1000 \
    -Wl,-z,noexecstack \
    -Wl,--gc-sections \
    -Wl,--build-id=none \
    -Wl,--hash-style=gnu \
    -Wl,-T,picoefi/src/x86_64/link_script.lds \
    -Wl,-m,elf_x86_64

# Check the toolchain with the internal flags, which also makes Clang target
# the architecture and link with LLD unless LDFLAGS picks another linker.
ifneq ($(BUILD_GOALS),)
    include gmake-toolchain-helper/check.mk
endif

# Flags that depend on the toolchain. When nothing is to be built, the
# helper is not included: its macros expand to nothing, and TCH_CC_IS_CLANG
# is empty.
override CFLAGS += \
    $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-fno-stack-clash-protection)
override CFLAGS += \
    $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-fcf-protection=none)

# GCC needs this flag for interrupt handlers and Clang below 3.9 has no
# name for it, where only a long double could reach the x87.
override CFLAGS += \
    $(call TCH_DRIVER_HAS_COMPILE_FLAG,CC,-mno-80387)

# Older GCCs need this for ms_abi calls. Clang has no such flag.
ifneq ($(TCH_CC_IS_CLANG),1)
    override CFLAGS += \
        -maccumulate-outgoing-args
endif

# Pack relative relocations where the linker can, as RELR tables take a
# fraction of the room. LLD before 15 only knows --pack-dyn-relocs.
ifneq ($(call TCH_LINKER_HAS_Z_FLAG,pack-relative-relocs),)
    override LDFLAGS += \
        -Wl,-z,pack-relative-relocs
else
    ifneq ($(call TCH_LINKER_HAS_FLAG,--pack-dyn-relocs),)
        override LDFLAGS += \
            -Wl,--pack-dyn-relocs=relr
    endif
endif

# Tell the compiler as well if it takes the flag, as it otherwise passes
# its own default on to the linker.
override LDFLAGS += \
    $(call TCH_DRIVER_HAS_LINK_FLAG,-pie)

# Use "find" to glob all *.c, *.S, and *.asm files in the tree and obtain the
# object and header dependency file names.
override SRCFILES := $(shell find -L src freestanding-lgrt/src picoefi/src/x86_64 -type f 2>/dev/null | LC_ALL=C sort)
override CFILES := $(filter %.c,$(SRCFILES))
override ASFILES := $(filter %.S,$(SRCFILES))
override NASMFILES := $(filter %.asm,$(SRCFILES))
override OBJ := $(addprefix obj/,$(CFILES:.c=.c.o) $(ASFILES:.S=.S.o) $(NASMFILES:.asm=.asm.o))
override HEADER_DEPS := $(addprefix obj/,$(CFILES:.c=.c.d) $(ASFILES:.S=.S.d) $(NASMFILES:.asm=.asm.d))

# Default target. This must come first, before header dependencies.
.PHONY: all
all: bin/$(OUTPUT).efi

# Include header dependencies.
-include $(HEADER_DEPS)

# Each kind of file depends on records of the variables it is built with,
# named after them, which the toolchain helper remakes when they change.
override TCH_RECORDS := $(addprefix obj/,CC OBJCOPY CFLAGS CPPFLAGS LDFLAGS NASMFLAGS)
ifneq ($(BUILD_GOALS),)
    include gmake-toolchain-helper/records.mk
endif

# Rule to convert the final ELF executable to a .EFI PE executable.
bin/$(OUTPUT).efi: bin/$(OUTPUT) GNUmakefile obj/OBJCOPY
	mkdir -p "$(dir $@)"
	$(OBJCOPY) -O binary $< $@

# Link rules for the final executable.
bin/$(OUTPUT): GNUmakefile obj/CC obj/CFLAGS obj/LDFLAGS picoefi/src/x86_64/link_script.lds $(OBJ)
	mkdir -p "$(dir $@)"
	$(CC) $(CFLAGS) $(LDFLAGS) $(OBJ) -o $@

# The compiler may emit calls to the memory functions and the compiler
# runtime that LTO does not account for, so never build those with LTO.
obj/src/memory.c.o obj/freestanding-lgrt/src/freestanding-lgrt.c.o: override CFLAGS += -fno-lto

# Compilation rules for *.c files.
obj/%.c.o: %.c GNUmakefile obj/CC obj/CFLAGS obj/CPPFLAGS
	mkdir -p "$(dir $@)"
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

# Compilation rules for *.S files.
obj/%.S.o: %.S GNUmakefile obj/CC obj/CFLAGS obj/CPPFLAGS
	mkdir -p "$(dir $@)"
	$(CC) $(CFLAGS) $(CPPFLAGS) -c $< -o $@

# Compilation rules for *.asm (nasm) files.
obj/%.asm.o: %.asm GNUmakefile obj/NASMFLAGS
	mkdir -p "$(dir $@)"
	nasm $(NASMFLAGS) -MD $(@:.o=.d) -MP $< -o $@

# Rules to download the UEFI firmware for testing.
.INTERMEDIATE: edk2-ovmf-bins.tar.gz
edk2-ovmf-bins.tar.gz:
	curl -fL -o $@ https://github.com/osdev0/edk2-ovmf-stable-bins/releases/latest/download/edk2-ovmf-bins.tar.gz

edk2-ovmf-bins: edk2-ovmf-bins.tar.gz
	rm -rf edk2-ovmf-bins
	gunzip < edk2-ovmf-bins.tar.gz | tar -xf -

# Rules for running our executable in QEMU.
.PHONY: run
run: all edk2-ovmf-bins
	mkdir -p boot/EFI/BOOT
	cp bin/$(OUTPUT).efi boot/EFI/BOOT/BOOTX64.EFI
	qemu-system-x86_64 \
		-drive file=fat:rw:boot \
		$(QEMUFLAGS) \
		$(QEMU_UEFI_FW)
	rm -rf boot

# Remove object files and the final executable.
.PHONY: clean
clean:
	rm -rf bin obj

# Remove everything built and generated including downloaded dependencies.
.PHONY: distclean
distclean: clean
	rm -rf .deps-obtained .cache compile_commands.json freestanding-c-hdrs freestanding-lgrt gmake-toolchain-helper picoefi edk2-ovmf-bins edk2-ovmf-bins.tar.gz

# Install the final built executable to its final on-root location. Not
# install(1), which takes other arguments on illumos and Solaris.
.PHONY: install
install: all
	mkdir -p "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)"
	cp bin/$(OUTPUT).efi "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/"
	chmod 644 "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/$(OUTPUT).efi"

# Try to undo whatever the "install" target did.
.PHONY: uninstall
uninstall:
	rm -f "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)/$(OUTPUT).efi"
	rmdir "$(DESTDIR)$(PREFIX)/share/$(OUTPUT)" 2>/dev/null || :
