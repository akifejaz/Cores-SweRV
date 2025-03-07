# SPDX-License-Identifier: Apache-2.0
# Copyright 2020 Western Digital Corporation or its affiliates.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
CONF_PARAMS = -set iccm_enable
# Check for RV_ROOT
ifeq (,$(wildcard ${RV_ROOT}/configs/swerv.config))
$(error env var RV_ROOT does not point to a valid dir! Exiting!)
endif

TEST_CFLAGS = -g -O3 -funroll-all-loops
ABI = -mabi=ilp32 -march=rv32imc

# Allow snapshot override
target = default
snapshot = $(target)

# Allow tool override
SWERV_CONFIG = ${RV_ROOT}/configs/swerv.config
IRUN = xrun
VCS = vcs
VERILATOR = verilator
VLOG = qverilog
RIVIERA = riviera
GCC_PREFIX = riscv64-unknown-elf
BUILD_DIR = snapshots/${snapshot}
TBDIR = ${RV_ROOT}/testbench

# Define default test name
TEST = hello_world

# Define default test directory
TEST_DIR = $(TBDIR)/asm
HEX_DIR = $(TBDIR)/hex
ifneq (,$(wildcard $(TBDIR)/tests/$(TEST)))
        TEST_DIR = $(TBDIR)/tests/$(TEST)
endif

ifdef debug
 DEBUG_PLUS = +dumpon
 IRUN_DEBUG = -access +rc
 IRUN_DEBUG_RUN = -input ${RV_ROOT}/testbench/input.tcl
 VCS_DEBUG = -debug_access
 VERILATOR_DEBUG = --trace
 RIVIERA_DEBUG = +access +r
endif

# provide specific link file
ifeq (,$(wildcard $(TEST_DIR)/$(TEST).ld))
	LINK = $(TBDIR)/link.ld
else
	LINK = $(TEST_DIR)/$(TEST).ld
endif

OFILES = $(TEST).o

-include $(TEST_DIR)/$(TEST).mki

VPATH = $(TEST_DIR) $(BUILD_DIR) $(TBDIR)
TBFILES = $(TBDIR)/tb_top.sv $(TBDIR)/ahb_sif.sv

defines = $(BUILD_DIR)/common_defines.vh ${RV_ROOT}/design/include/swerv_types.sv
includes = -I${RV_ROOT}/design/include -I${RV_ROOT}/design/lib  -I${BUILD_DIR}

# CFLAGS for verilator generated Makefiles. Without -std=c++11 it complains for `auto` variables
CFLAGS += "-std=c++11"
# Optimization for better performance; alternative is nothing for slower runtime (faster compiles)
# -O2 for faster runtime (slower compiles), or -O for balance.
VERILATOR_MAKE_FLAGS = OPT_FAST="-Os"

# Targets
all: clean verilator

clean:
	rm -rf *.log *.s *.hex *.dis *.tbl irun* vcs* simv* *.map snapshots swerv* \
	verilator* *.exe obj* *.o ucli.key vc_hdrs.h csrc *.csv \
	work dataset.asdb  library.cfg

# If define files do not exist, then run swerv.config.
${BUILD_DIR}/defines.h :
	BUILD_PATH=${BUILD_DIR} ${SWERV_CONFIG} -target=$(target) $(CONF_PARAMS)

##################### Verilog Builds #####################################

verilator-build: ${TBFILES} ${BUILD_DIR}/defines.h test_tb_top.cpp
	echo '`undef ASSERT_ON' >> ${BUILD_DIR}/common_defines.vh
	$(VERILATOR) --cc -CFLAGS ${CFLAGS} $(defines) $(includes) \
                -Wno-UNOPTFLAT \
                -I${RV_ROOT}/testbench \
		-f ${RV_ROOT}/testbench/flist \
                ${TBFILES} \
                --top-module tb_top -exe test_tb_top.cpp --autoflush $(VERILATOR_DEBUG)
	cp ${RV_ROOT}/testbench/test_tb_top.cpp obj_dir
	$(MAKE) -j -C obj_dir/ -f Vtb_top.mk $(VERILATOR_MAKE_FLAGS)
	touch verilator-build

vcs-build: ${TBFILES} ${BUILD_DIR}/defines.h
	$(VCS) -full64 -assert svaext -sverilog +error+500 \
                +incdir+${RV_ROOT}/design/lib \
                +incdir+${RV_ROOT}/design/include \
		+incdir+${BUILD_DIR}  +libext+.v\
		 $(defines) -f ${RV_ROOT}/testbench/flist\
		 ${TBFILES} \
		 -l vcs_compile.log
	touch vcs-build

irun-build: ${TBFILES} ${BUILD_DIR}/defines.h
	$(IRUN) -64bit -elaborate $(IRUN_DEBUG) -q -sv -sysv  -nowarn CUVIHR -xmlibdirpath . -xmlibdirname swerv.build \
		-incdir ${RV_ROOT}/design/lib -incdir ${RV_ROOT}/design/include -incdir ${BUILD_DIR} -vlog_ext +.vh+.h\
		$(defines)  -f ${RV_ROOT}/testbench/flist\
		-top tb_top  ${TBFILES} -I${RV_ROOT}/testbench \
		 -elaborate  -snapshot $(snapshot) $(profile)
	touch irun-build

riviera-build: ${TBFILES} ${BUILD_DIR}/defines.h
	vlib work
	vlog -work work \
		+incdir+${RV_ROOT}/design/lib \
		+incdir+${RV_ROOT}/design/include \
		+incdir+${BUILD_DIR} +libext+.v $(defines) \
		-f ${RV_ROOT}/testbench/flist \
		${TBFILES}
	touch riviera-build


##################### Simulation Runs #####################################
riscof_compliance:
	riscof run --env $(RV_ROOT)/compliance/riscv-arch-test/riscv-test-suite/env/ \
	           --suite $(RV_ROOT)/compliance/riscv-arch-test/riscv-test-suite/rv32i_m/
verilator: program.hex verilator-build
	./obj_dir/Vtb_top ${DEBUG_PLUS}

irun: program.hex irun-build
	$(IRUN) -64bit +lic_queue -licqueue -status -xmlibdirpath . -xmlibdirname swerv.build \
		 -snapshot ${snapshot} -r ${snapshot} $(IRUN_DEBUG_RUN) $(profile)

vcs: program.hex vcs-build
	./simv $(DEBUG_PLUS) +vcs+lic+wait -l vcs.log

vlog: program.hex ${TBFILES} ${BUILD_DIR}/defines.h
	$(VLOG) -l vlog.log -sv -mfcu +incdir+${BUILD_DIR}+${RV_ROOT}/design/include+${RV_ROOT}/design/lib\
        $(defines) -f ${RV_ROOT}/testbench/flist ${TBFILES} -R +nowarn3829 ${DEBUG_PLUS}

riviera: program.hex riviera-build
	vsim -c -lib work ${DEBUG_PLUS} ${RIVIERA_DEBUG} tb_top -do "run -all; exit" -l riviera.log



##################### Test Build #####################################


ifeq ($(shell which $(GCC_PREFIX)-gcc 2> /dev/null),)
program.hex: ${BUILD_DIR}/defines.h
	@echo " !!! No $(GCC_PREFIX)-gcc in path, using canned hex files !!"
	cp ${HEX_DIR}/$(TEST).hex program.hex
else
ifneq (,$(wildcard $(TEST_DIR)/$(TEST).makefile))
program.hex:
	$(MAKE) -f $(TEST_DIR)/$(TEST).makefile
else
program.hex: $(OFILES) $(LINK)
	@echo Building $(TEST)
	$(GCC_PREFIX)-gcc $(ABI) -Wl,-Map=$(TEST).map -lgcc -T$(LINK) -o $(TEST).exe $(OFILES) -nostartfiles  $(TEST_LIBS)
	$(GCC_PREFIX)-objcopy -O verilog $(TEST).exe program.hex
	$(GCC_PREFIX)-objdump -S  $(TEST).exe > $(TEST).dis
	@echo Completed building $(TEST)

%.o : %.s ${BUILD_DIR}/defines.h
	$(GCC_PREFIX)-cpp -I${BUILD_DIR}  $<  > $*.cpp.s
	$(GCC_PREFIX)-as $(ABI) $*.cpp.s -o $@


%.o : %.c ${BUILD_DIR}/defines.h
	$(GCC_PREFIX)-gcc -I${BUILD_DIR} ${TEST_CFLAGS} ${ABI} -nostdlib -c $< -o $@
endif
endif

help:
	@echo Make sure the environment variable RV_ROOT is set.
	@echo Possible targets: verilator vcs irun vlog riviera help clean all verilator-build irun-build vcs-build riviera-build program.hex

.PHONY: help clean verilator vcs irun vlog riviera
