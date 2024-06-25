# Makefile for building dev/cuda kernels
# Collects all the make commands in one file but each file also
# has the compile and run commands in the header comments section.

BUILD_DIR = build
ifeq ($(OS), Windows_NT)
  $(shell if not exist $(BUILD_DIR) mkdir $(BUILD_DIR))
  REMOVE_BUILD_OBJECT_FILES := del $(BUILD_DIR)\*.obj
else
  $(shell mkdir -p $(BUILD_DIR))
  REMOVE_BUILD_OBJECT_FILES := rm -f $(BUILD_DIR)/*.o
endif

# AMD flags
ROCM_PATH ?= /opt/rocm
AMDGPU_TARGETS ?= $(shell $(ROCM_PATH)/llvm/bin/amdgpu-offload-arch)
HIPCC := $(shell which hipcc 2>/dev/null)
HIPIFY := $(shell which hipify-perl 2>/dev/null)
HIPCC_FLAGS = -O3 -march=native -I$(BUILD_DIR)
HIPCC_FLAGS += $(addprefix --offload-arch=,$(AMDGPU_TARGETS))
HIPCC_LDFLAGS = -lhipblas -lhipblaslt -lamdhip64

HIPCC_FLAGS += -I./build/hip
HIPCC_FLAGS += -I../../build/hip/llmc
HIPCC_FLAGS += -I../../llmc

HIPCC_FLAGS += -I/opt/rocm/include/hipblas
ifneq ($(filter gfx1100,$(AMDGPU_TARGETS)),)
  HIPCC_LDFLAGS += -ldevice_gemm_operations -lutility -ldevice_other_operations
else
  HIPCC_FLAGS += -DDISABLE_CK
endif
ifdef DISABLE_CK
  HIPCC_FLAGS += -DDISABLE_CK
endif
ifdef WAVEFRONTSIZE64
  HIPCC_FLAGS += -DWAVEFRONTSIZE64 -mwavefrontsize64
endif
ifdef CUMODE
  HIPCC_FLAGS += -mcumode
endif
ifneq ($(NO_MULTI_GPU), 1)
  ifeq ($(shell [ -d /usr/lib/x86_64-linux-gnu/openmpi/lib/ ] && [ -d /usr/lib/x86_64-linux-gnu/openmpi/include/ ] && echo "exists"), exists)
    HIPCC_FLAGS += -I/usr/lib/x86_64-linux-gnu/openmpi/include -DMULTI_GPU
    HIPCC_LDFLAGS += -L/usr/lib/x86_64-linux-gnu/openmpi/lib/ -lmpi -lrccl
  endif
endif
AMD_HEADERS = $(addprefix $(BUILD_DIR)/hip/,$(wildcard *h))

# Conditional inclusion of AMD targets
ifeq ($(HIPCC),)
    $(error ✗ hipcc not found, stopping GPU/AMD builds)
else
    $(info ✓ hipcc found, including GPU/AMD builds)
    HIPCC_FLAGS += -DBUILD_AMD
endif



# Build all targets
TARGETS = adamwamd attention_backwardamd attention_forwardamd classifier_fusedamd crossentropy_forwardamd crossentropy_softmax_backwardamd encoder_backwardamd encoder_forwardamd gelu_backwardamd gelu_forwardamd layernorm_backwardamd layernorm_forwardamd matmul_backwardamd matmul_backward_biasamd matmul_forwardamd nccl_all_reduceamd residual_forwardamd softmax_forwardamd trimat_forwardamd fused_residual_forwardamd global_normamd
all: $(TARGETS)


$(BUILD_DIR)/hip/%h: %h
	@mkdir -p $(dir $@)
	$(HIPIFY) -quiet-warnings $< -o $@

amd_headers: $(AMD_HEADERS)

$(BUILD_DIR)/hip/%.cu: %.cu
	@mkdir -p $(dir $@)
	$(HIPIFY) $< -o $@

%amd: $(BUILD_DIR)/hip/%.cu amd_headers
	$(HIPCC) $(HIPCC_FLAGS) $(PFLAGS) $< $(HIPCC_LDFLAGS) -o $@


clean:
	rm -f $(TARGETS)
