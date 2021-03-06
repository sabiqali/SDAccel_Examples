.PHONY: help

help::
	$(ECHO) "Makefile Usage:"
	$(ECHO) "  make all TARGET=<sw_emu/hw_emu/hw> DEVICE=<FPGA platform>"
	$(ECHO) "      Command to generate the design for specified Target and Device."
	$(ECHO) ""
	$(ECHO) "  make clean "
	$(ECHO) "      Command to remove the generated non-hardware files."
	$(ECHO) ""
	$(ECHO) "  make cleanall"
	$(ECHO) "      Command to remove all the generated files."
	$(ECHO) ""
	$(ECHO) "  make check TARGET=<sw_emu/hw_emu/hw> DEVICE=<FPGA platform>"
	$(ECHO) "      Command to run application in emulation."
	$(ECHO) ""
	$(ECHO) "  make build TARGET=<sw_emu/hw_emu/hw> DEVICE=<FPGA platform>"
	$(ECHO) "      Command to build xclbin application."
	$(ECHO) ""
	$(ECHO) "  make run_nimbix DEVICE=<FPGA platform>"
	$(ECHO) "      Command to run application on Nimbix Cloud."
	$(ECHO) ""
	$(ECHO) "  make aws_build DEVICE=<FPGA platform>"
	$(ECHO) "      Command to build AWS xclbin application on AWS Cloud."
	$(ECHO) ""

# Points to Utility Directory
COMMON_REPO = ../../../
ABS_COMMON_REPO = $(shell readlink -f $(COMMON_REPO))

TARGETS := hw
TARGET := $(TARGETS)
DEVICE := $(DEVICES)
XCLBIN := ./xclbin

include ./utils.mk

DSA := $(call device2dsa, $(DEVICE))
BUILD_DIR := ./_x.$(TARGET).$(DSA)

BUILD_DIR_sketch_GOOD = $(BUILD_DIR)/sketch_GOOD
BUILD_DIR_sketch_BAD = $(BUILD_DIR)/sketch_BAD

CXX := $(XILINX_SDX)/bin/xcpp
XOCC := $(XILINX_SDX)/bin/xocc

#Include Libraries
include $(ABS_COMMON_REPO)/libs/opencl/opencl.mk
include $(ABS_COMMON_REPO)/libs/xcl2/xcl2.mk
include $(ABS_COMMON_REPO)/libs/bitmap/bitmap.mk
CXXFLAGS += $(xcl2_CXXFLAGS) $(bitmap_CXXFLAGS)
LDFLAGS += $(xcl2_LDFLAGS) $(bitmap_LDFLAGS)
HOST_SRCS += $(xcl2_SRCS) $(bitmap_SRCS)
CXXFLAGS += $(opencl_CXXFLAGS) -Wall -O0 -g -std=c++14
LDFLAGS += $(opencl_LDFLAGS)

HOST_SRCS += src/host.cpp

# Host compiler global settings
CXXFLAGS += -fmessage-length=0
LDFLAGS += -lrt -lstdc++ 

# Kernel compiler global settings
CLFLAGS += -t $(TARGET) --platform $(DEVICE) --save-temps 



EXECUTABLE = host
CMD_ARGS = $(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xclbin $(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xclbin ./data/car_128x128.bmp

EMCONFIG_DIR = $(XCLBIN)/$(DSA)

BINARY_CONTAINERS += $(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xclbin
BINARY_CONTAINER_sketch_GOOD_OBJS += $(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xo
BINARY_CONTAINERS += $(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xclbin
BINARY_CONTAINER_sketch_BAD_OBJS += $(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xo

CP = cp -rf
DATA = ./data

.PHONY: all clean cleanall docs emconfig
all: check-devices $(EXECUTABLE) $(BINARY_CONTAINERS) emconfig

.PHONY: exe
exe: $(EXECUTABLE)

.PHONY: build
build: $(BINARY_CONTAINERS)

# Building kernel
$(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xo: src/sketch.cpp
	mkdir -p $(XCLBIN)
	$(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_sketch_GOOD) -c -k sketch_GOOD -I'$(<D)' -o'$@' '$<'
$(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xo: src/sketch_bad.cpp
	mkdir -p $(XCLBIN)
	$(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_sketch_BAD) -c -k sketch_BAD -I'$(<D)' -o'$@' '$<'
$(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xclbin: $(BINARY_CONTAINER_sketch_GOOD_OBJS)
	mkdir -p $(XCLBIN)
	$(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_sketch_GOOD) -l $(LDCLFLAGS) --nk sketch_GOOD:1 -o'$@' $(+)
$(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xclbin: $(BINARY_CONTAINER_sketch_BAD_OBJS)
	mkdir -p $(XCLBIN)
	$(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_sketch_BAD) -l $(LDCLFLAGS) --nk sketch_BAD:1 -o'$@' $(+)

# Building Host
$(EXECUTABLE): check-xrt $(HOST_SRCS) $(HOST_HDRS)
	$(CXX) $(CXXFLAGS) $(HOST_SRCS) $(HOST_HDRS) -o '$@' $(LDFLAGS)

emconfig:$(EMCONFIG_DIR)/emconfig.json
$(EMCONFIG_DIR)/emconfig.json:
	emconfigutil --platform $(DEVICE) --od $(EMCONFIG_DIR)

check: all
ifeq ($(TARGET),$(filter $(TARGET),sw_emu hw_emu))
	$(CP) $(EMCONFIG_DIR)/emconfig.json .
	XCL_EMULATION_MODE=$(TARGET) ./$(EXECUTABLE) $(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xclbin $(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xclbin ./data/car_128x128.bmp
else
	 ./$(EXECUTABLE) $(XCLBIN)/sketch_GOOD.$(TARGET).$(DSA).xclbin $(XCLBIN)/sketch_BAD.$(TARGET).$(DSA).xclbin ./data/car_128x128.bmp
endif
	sdx_analyze profile -i profile_summary.csv -f html

run_nimbix: all
	$(COMMON_REPO)/utility/nimbix/run_nimbix.py $(EXECUTABLE) $(CMD_ARGS) $(DSA)

aws_build: check-aws_repo $(BINARY_CONTAINERS)
	$(COMMON_REPO)/utility/aws/run_aws.py $(BINARY_CONTAINERS)

# Cleaning stuff
clean:
	-$(RMDIR) $(EXECUTABLE) $(XCLBIN)/{*sw_emu*,*hw_emu*} 
	-$(RMDIR) profile_* TempConfig system_estimate.xtxt *.rpt *.csv 
	-$(RMDIR) src/*.ll _xocc_* .Xil emconfig.json dltmp* xmltmp* *.log *.jou *.wcfg *.wdb

cleanall: clean
	-$(RMDIR) $(XCLBIN)
	-$(RMDIR) _x.*
	-$(RMDIR) ./output.bmp 
