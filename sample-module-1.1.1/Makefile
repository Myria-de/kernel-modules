export CROSS_COMPILE=
export ARCH=x86_64

# For building for the current running version of Linux
ifndef TARGET
TARGET		:= $(shell uname -r)
endif

TARGET_MODULE:=sample-module

# Secure boot
SIG_MOK	:=	$(shell mokutil --sb-state 2>/dev/null || echo "(unknown)")

# If we are running by kernel building system
ifneq ($(KERNELRELEASE),)
	$(TARGET_MODULE)-objs := char_example.o
	obj-m := $(TARGET_MODULE).o
# If we running without kernel build system
else
	BUILDSYSTEM_DIR:=/lib/modules/$(shell uname -r)/build
	PWD:=$(shell pwd)

# Directory below /lib/modules/$(TARGET)/kernel into which to install
# the module:
KERNEL_MODULES	:= /lib/modules/$(TARGET)
MODDESTDIR=$(KERNEL_MODULES)/misc

#SYSTEM_MAP	:= $(KERNEL_BUILD)/System.map
ifneq ("","$(wildcard /boot/System.map-$(TARGET))")
SYSTEM_MAP	:= /boot/System.map-$(TARGET)
else
# Arch
SYSTEM_MAP	:= /proc/kallsyms
endif

all : modules

modules:

# run kernel build system to make module
	$(MAKE) -C $(BUILDSYSTEM_DIR) M=$(PWD) modules
	
sign:
ifeq ($(SIG_MOK), SecureBoot enabled)
	@echo $(SIG_MOK)
	@echo "Signing module"
	sudo $(BUILDSYSTEM_DIR)/scripts/sign-file sha256 /var/lib/shim-signed/mok/MOK.priv /var/lib/shim-signed/mok/MOK.der $(TARGET_MODULE).ko
else
	@echo "Secure boot not enabled"	
endif

install: modules_install

modules_install:
ifeq ($(SIG_MOK), SecureBoot enabled)
	@echo $(SIG_MOK)
	@echo "Signing module"
	sudo $(BUILDSYSTEM_DIR)/scripts/sign-file sha256 /var/lib/shim-signed/mok/MOK.priv /var/lib/shim-signed/mok/MOK.der $(TARGET_MODULE).ko
else
	@echo "Secure boot not enabled"	
endif
	install -p -m 644 $(TARGET_MODULE).ko  $(MODDESTDIR)
	depmod -a -F $(SYSTEM_MAP) $(TARGET)
clean:
# run kernel build system to cleanup in current directory
	$(MAKE) -C $(BUILDSYSTEM_DIR) M=$(PWD) clean
load:
	sudo insmod ./$(TARGET_MODULE).ko

unload:
	sudo rmmod ./$(TARGET_MODULE).ko
endif
