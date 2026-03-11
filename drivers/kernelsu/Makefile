kernelsu-objs := ksu.o
kernelsu-objs += allowlist.o
kernelsu-objs += apk_sign.o
kernelsu-objs += sucompat.o
kernelsu-objs += throne_tracker.o
kernelsu-objs += core_hook.o
kernelsu-objs += ksud.o
kernelsu-objs += embed_ksud.o
kernelsu-objs += kernel_compat.o

kernelsu-objs += selinux/selinux.o
kernelsu-objs += selinux/sepolicy.o
kernelsu-objs += selinux/rules.o
ccflags-y += -I$(srctree)/security/selinux -I$(srctree)/security/selinux/include
ccflags-y += -I$(objtree)/security/selinux -include $(srctree)/include/uapi/asm-generic/errno.h

obj-$(CONFIG_KSU) += kernelsu.o

#1 compliant to last upstream kernel change as of 35d8162
LPATH := /usr/bin/env PATH="$$PATH":/usr/bin:/usr/local/bin
MDIR := $(dir $(abspath $(lastword $(MAKEFILE_LIST))))

# Check if this is a git repository
# Try to detect Git repo intelligently
GIT_ROOT := $(shell cd $(MDIR) && $(LPATH) git rev-parse --show-toplevel 2>/dev/null)
ifneq ($(GIT_ROOT),)
KERNEL_GIT_ROOT := $(shell cd $(srctree) && $(LPATH) git rev-parse --show-toplevel 2>/dev/null)
ifneq ($(GIT_ROOT),$(KERNEL_GIT_ROOT))
# Only set version if it's a different repo from kernel
$(shell cd $(GIT_ROOT) && [ -f .git/shallow ] && $(LPATH) git fetch --unshallow 2>/dev/null || true)
KSU_GIT_VERSION := $(shell cd $(GIT_ROOT) && $(LPATH) git rev-list --count HEAD 2>/dev/null)
KSU_GIT_TAG := $(shell cd $(GIT_ROOT) && $(LPATH) git describe --tags --abbrev=0 2>/dev/null)
KSU_GIT_VERSION_VALID := 1
$(info -- KernelSU-Next Git repo detected at: $(GIT_ROOT))
endif
endif

# Calculate version if git version is available
ifdef KSU_GIT_VERSION_VALID
# ksu_version: major * 30000 + git version for historical reasons
$(eval KSU_VERSION=$(shell expr 30000 + $(KSU_GIT_VERSION) + 119))
$(info -- KernelSU-Next version: $(KSU_VERSION))
ccflags-y += -DKSU_VERSION=$(KSU_VERSION)
else
# If there is no .git directory, use default version
$(warning "KSU_GIT_VERSION not defined! It is better to make KernelSU-Next a git repository!")
KSU_VERSION_FALLBACK := 32999
$(info -- KernelSU-Next version fallback: $(KSU_VERSION_FALLBACK))
ccflags-y += -DKSU_VERSION=$(KSU_VERSION_FALLBACK)
endif

ifdef KSU_GIT_VERSION_VALID
$(eval KSU_VERSION_TAG=$(KSU_GIT_TAG))
$(info -- KernelSU-Next tag: $(KSU_VERSION_TAG))
ccflags-y += -DKSU_VERSION_TAG=\"$(KSU_VERSION_TAG)\"
else
$(warning "KSU_VERSION_TAG not defined! It is better to make KernelSU-Next a git submodule!")
KSU_VERSION_TAG_FALLBACK := v3.1.0
$(info -- KernelSU-Next tag fallback: $(KSU_VERSION_TAG_FALLBACK))
ccflags-y += -DKSU_VERSION_TAG=\"$(KSU_VERSION_TAG_FALLBACK)\"
endif

ifndef KSU_EXPECTED_SIZE
KSU_EXPECTED_SIZE := 0x033b
endif

ifndef KSU_EXPECTED_HASH
KSU_EXPECTED_HASH := c371061b19d8c7d7d6133c6a9bafe198fa944e50c1b31c9d8daa8d7f1fc2d2d6
endif

ifdef KSU_MANAGER_PACKAGE
ccflags-y += -DKSU_MANAGER_PACKAGE=\"$(KSU_MANAGER_PACKAGE)\"
$(info -- KernelSU Manager package name: $(KSU_MANAGER_PACKAGE))
endif

$(info -- KernelSU Manager signature size: $(KSU_EXPECTED_SIZE))
$(info -- KernelSU Manager signature hash: $(KSU_EXPECTED_HASH))

ccflags-y += -DEXPECTED_SIZE=$(KSU_EXPECTED_SIZE)
ccflags-y += -DEXPECTED_HASH=\"$(KSU_EXPECTED_HASH)\"

# UL, look for "ext4_unregister_sysfs" on fs/ext4
ifeq ($(shell grep -q "^extern void ext4_unregister_sysfs" $(srctree)/fs/ext4/ext4.h 2>/dev/null; echo $$?),0)
ccflags-y += -DKSU_HAS_MODERN_EXT4
endif

# enable functions that use stuct filename when ksu_handle_execveat is found
ifeq ($(shell grep -q "ksu_handle_execveat" $(srctree)/fs/exec.c; echo $$?),0)
ccflags-y += -DKSU_USE_STRUCT_FILENAME
endif

ifeq ($(shell grep -q "ksu_handle_execveat" $(srctree)/arch/arm/kernel/sys_arm.c; echo $$?),0)
ccflags-y += -DKSU_USE_STRUCT_FILENAME
endif

ifneq ($(shell grep -q "get_cred_rcu" $(srctree)/include/linux/cred.h; echo $$?),0)
$(error You must backport get_cred_rcu - https://github.com/tiann/KernelSU/pull/2320#issuecomment-2564232958 )
endif

ifeq ($(shell grep -q "int path_umount" $(srctree)/fs/namespace.c; echo $$?),0)
$(info -- KernelSU/compat: path_umount found)
ccflags-y += -DKSU_HAS_PATH_UMOUNT
endif

ifeq ($(shell grep -q "strncpy_from_user_nofault" $(srctree)/include/linux/uaccess.h; echo $$?),0)
$(info -- KernelSU/compat: strncpy_from_user_nofault found)
ccflags-y += -DKSU_STRNCPY_FROM_USER_NOFAULT
endif

ifeq ($(shell grep -q "strncpy_from_unsafe_user" $(srctree)/include/linux/uaccess.h; echo $$?),0)
$(info -- KernelSU/compat: strncpy_from_unsafe_user found)
ccflags-y += -DKSU_STRNCPY_FROM_UNSAFE_USER
endif

ifeq ($(shell grep -q "ssize_t kernel_read" $(srctree)/fs/read_write.c; echo $$?),0)
$(info -- KernelSU/compat: newer kernel_read found)
ccflags-y += -DKSU_NEW_KERNEL_READ
endif

ifeq ($(shell grep "ssize_t kernel_write" $(srctree)/fs/read_write.c | grep -q "const void" ; echo $$?),0)
$(info -- KernelSU/compat: newer kernel_write found)
ccflags-y += -DKSU_NEW_KERNEL_WRITE
endif

ccflags-y += -Wno-implicit-function-declaration -Wno-strict-prototypes -Wno-int-conversion
ccflags-y += -Wno-declaration-after-statement -Wno-unused-function

# Keep a new line here!! Because someone may append config
