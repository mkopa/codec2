#
# stm32f4 Codec2 test programs
#
# CMake configuration contributed by Richard Shaw (KF5OIM)
# Please report questions, comments, problems, or patches to the freetel
# mailing list: https://lists.sourceforge.net/lists/listinfo/freetel-codec2
#
set(ARM_GCC_BIN "" CACHE STRING "Path to the bin directory of your arm-eabi-none-gcc (optional)")
project(stm32f4 C ASM)

if(CMAKE_CROSSCOMPILING)
    message(STATUS "We are cross compiling...")
else()
    message(STATUS "Performing standard host build...")
endif()

cmake_minimum_required(VERSION 2.8)

include(cmake/gencodebooks.cmake)

#
# Prevent in-source builds
# If an in-source build is attempted, you will still need to clean up a few
# files manually.
#
set(CMAKE_DISABLE_SOURCE_CHANGES ON)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
  message(FATAL_ERROR "In-source builds in ${CMAKE_BINARY_DIR} are not "
   "allowed, please remove ./CMakeCache.txt and ./CMakeFiles/, create a "
   "separate build directory and run cmake from there.")
endif("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")

#include(cmake/STM32_Toolchain.cmake)

###################################################

set(FLOAT_TYPE "hard" CACHE STRING "Floating point: defaults to hard.")
set(CMAKE_TOOLCHAIN_FILE "../stm32/cmake/STM32_Toolchain.cmake" CACHE STRING "Toolchain defs")

###################################################

# Set default C flags.
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -std=gnu11 -mlittle-endian -mthumb -mthumb-interwork --specs=nano.specs -u_printf_float -mcpu=cortex-m4 -ffunction-sections -fdata-sections -O3")

add_definitions(-DSTM32F40_41xxx -DCORTEX_M4 -D__EMBEDDED__)
add_definitions(-DFREEDV_MODE_EN_DEFAULT=0 -DFREEDV_MODE_1600_EN=1 -DFREEDV_MODE_700D_EN=1 -DCODEC2_MODE_EN_DEFAULT=0 -DCODEC2_MODE_1300_EN=1 -DCODEC2_MODE_700C_EN=1)

if(FLOAT_TYPE STREQUAL "hard")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsingle-precision-constant -Wdouble-promotion -mfpu=fpv4-sp-d16 -mfloat-abi=hard -D__FPU_PRESENT=1 -D__FPU_USED=1")
    #CFLAGS += -fsingle-precision-constant
else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msoft-float")
endif()

option(VERIFY_OPT "Enable this for dump files to help verify optimization" OFF)
if(VERIFY_OPT)
    add_definitions(-DDUMP)
endif()

#option(SEMIHOSTING "Enable for standard arm semihosting." OFF)
#if(SEMIHOSTING)
#    link_libraries(rdimon)
#    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --specs=rdimon.specs")
#else()
#    link_libraries(nosys)
#    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -specs=nosys.specs")
#endif()

# Set default build type
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Debug")
endif()
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    link_libraries(g m)
else()
    link_libraries(c m)
endif()

# Setup defaults that can't be set in the toolchain file
set(CMAKE_EXE_LINKER_FLAGS "-u_init -T${CMAKE_SOURCE_DIR}/stm32_flash.ld -Xlinker --gc-sections")
set(CMAKE_EXECUTABLE_SUFFIX_C ".elf")
set(CMAKE_EXECUTABLE_SUFFIX_CXX ".elf")
set(CMAKE_EXECUTABLE_SUFFIX_ASM ".elf")
set(CMAKE_ASM_FLAGS "${CMAKE_C_FLAGS} -x assembler-with-cpp")


# Check build flags

message(STATUS "CMAKE_C_FLAGS: " ${CMAKE_C_FLAGS})
message(STATUS "CMAKE_ASM_FLAGS: " ${CMAKE_ASM_FLAGS})


###################################################

# STM32F4 Standard Peripheral Library

include(cmake/STM32_Lib.cmake)

###################################################

# Macro for elf->bin
macro(elf2bin target)
    add_custom_command(TARGET ${target}
    POST_BUILD COMMAND ${CMAKE_OBJCOPY} -O binary ${target}.elf ${target}.bin
    COMMENT "Creating binary for ${target}")
    set_source_files_properties(${target}.bin PROPERTIES GENERATED TRUE)
    set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" APPEND PROPERTY
    ADDITIONAL_MAKE_CLEAN_FILES ${target}.bin)
endmacro()

# Macro for elf->bin
macro(elf2dfu target)
    add_custom_command(TARGET ${target}
            POST_BUILD COMMAND ${CMAKE_OBJCOPY} -O ihex ${target}.elf ${target}.hex && ${CMAKE_SOURCE_DIR}/support/hex2dfu.py ${target}.hex ${target}.dfu
    COMMENT "Creating dfu file for ${target}")
    set_source_files_properties(${target}.bin PROPERTIES GENERATED TRUE)
    set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" APPEND PROPERTY
    ADDITIONAL_MAKE_CLEAN_FILES ${target}.hex ${target}.dfu)
endmacro()

# This macro just adds generation of a map file with the same name as the executable and .map suffix
# to the linker command line. This works in older Cmake version (versions >= 3.13 have target_link_options)
# it should be a one to one replacement for add_executable
macro(add_mapped_executable target)
    add_executable(${target} ${ARGN})
    target_link_libraries(${target} "-Wl,-Map=$<TARGET_PROPERTY:NAME>.map")
    set_source_files_properties(${target}.map PROPERTIES GENERATED TRUE)
    set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" APPEND PROPERTY
    ADDITIONAL_MAKE_CLEAN_FILES ${target}.map)
endmacro()

include(CTest)
include_directories(../src ../unittest inc ${PROJECT_BINARY_DIR})

add_subdirectory(unittest/src)



# Codec 2

# Output path is such that #include <codec2/version.h> in codec2.h works
set(CODEC2_VERSION_PATH "${PROJECT_BINARY_DIR}/codec2")
configure_file ("${PROJECT_SOURCE_DIR}/../cmake/version.h.in"
                "${CODEC2_VERSION_PATH}/version.h" )


set(CODEC2_SRC ../src)
set(CODEC2_GEN_CODEBOOK_SRC ../build/src)
set(CODEC2_SRCS
${CODEC2_SRC}/lpc.c
${CODEC2_SRC}/nlp.c
${CODEC2_SRC}/postfilter.c
${CODEC2_SRC}/sine.c
${CODEC2_SRC}/codec2.c
${CODEC2_SRC}/codec2_fft.c
${CODEC2_SRC}/gp_interleaver.c
${CODEC2_SRC}/interldpc.c
${CODEC2_SRC}/kiss_fft.c
${CODEC2_SRC}/kiss_fftr.c
${CODEC2_SRC}/interp.c
${CODEC2_SRC}/lsp.c
${CODEC2_SRC}/ofdm.c
${CODEC2_SRC}/phase.c
${CODEC2_SRC}/quantise.c
${CODEC2_SRC}/pack.c
${CODEC2_SRC}/dump.c
${CODEC2_SRC}/cohpsk.c
${CODEC2_SRC}/fdmdv.c
${CODEC2_SRC}/freedv_api.c
${CODEC2_SRC}/freedv_1600.c
${CODEC2_SRC}/freedv_700.c
${CODEC2_SRC}/freedv_2020.c
${CODEC2_SRC}/freedv_fsk.c
${CODEC2_SRC}/filter.c
${CODEC2_SRC}/varicode.c
${CODEC2_SRC}/golay23.c
${CODEC2_SRC}/freedv_data_channel.c
${CODEC2_SRC}/newamp1.c
${CODEC2_SRC}/mbest.c
${CODEC2_SRC}/HRA_112_112.c
${CODEC2_SRC}/linreg.c
${CODEC2_SRC}/mpdecode_core.c
${CODEC2_SRC}/phi0.c
${CODEC2_SRC}/HRAb_396_504.c
codebook.c
codebookd.c
codebookjvm.c
codebookge.c
codebooknewamp1_energy.c
codebooknewamp1.c
)

set(STM32F4_ADAC_SRCS
src/stm32f4_adc.c 
src/stm32f4_dac.c 
../src/codec2_fifo.c
)
add_library(stm32f4_adac STATIC ${STM32F4_ADAC_SRCS})

add_library(codec2 STATIC ${CODEC2_SRCS})
add_library(codec2_prof STATIC ${CODEC2_SRCS})
target_compile_definitions(codec2_prof PRIVATE PROFILE)

set(SYSTEM_SRCS
src/system_stm32f4xx.c
src/startup_stm32f4xx.s
src/gdb_stdio.c
)

add_library(sm1000base STATIC src/sm1000_leds_switches.c src/debugblinky.c ${SYSTEM_SRCS})

set(PROFILE_SYSTEM_SRCS
src/stm32f4_machdep.c
)

list(APPEND PROFILE_SYSTEM_SRCS ${SYSTEM_SRCS})

set(FFT_TEST_SRCS 
src/fft_test.c
../src/kiss_fft.c
)

add_mapped_executable(fft_test ${FFT_TEST_SRCS} ${PROFILE_SYSTEM_SRCS})
target_link_libraries(fft_test stm32f4 CMSIS)
elf2bin(fft_test)


# Rule for programming the SM1000
#%.pgm: %.bin
#	$(SUDO) dfu-util -d 0483:df11 -c 1 -i 0 -a 0 -s 0x08000000 -D $<
####################################################

set(CODEC2_PROFILE_SRCS
src/codec2_profile.c
)

add_mapped_executable(codec2_profile ${CODEC2_PROFILE_SRCS} ${PROFILE_SYSTEM_SRCS})
target_link_libraries(codec2_profile codec2_prof stm32f4 CMSIS)
target_compile_definitions(codec2_profile PRIVATE PROFILE)
elf2bin(codec2_profile)

set(DAC_UT_SRCS
src/dac_ut.c
)

add_mapped_executable(dac_ut ${DAC_UT_SRCS})
target_link_libraries(dac_ut stm32f4_adac stm32f4 sm1000base)
elf2bin(dac_ut)

set(FAST_DAC_UT_SRCS
src/fast_dac_ut.c
../src/codec2_fifo.c
src/iir_duc.c
src/stm32f4_dacduc.c
src/debugblinky.c
)

add_mapped_executable(fast_dac_ut ${FAST_DAC_UT_SRCS} ${SYSTEM_SRCS})
target_link_libraries(fast_dac_ut stm32f4)
target_compile_options(fast_dac_ut PRIVATE "-O3")
elf2bin(fast_dac_ut)

set(ADCDAC_UT_SRCS
src/adcdac_ut.c
)

add_mapped_executable(adcdac_ut ${ADCDAC_UT_SRCS} ${STM32F4_ADAC_SRCS} ${SYSTEM_SRCS})
target_link_libraries(adcdac_ut stm32f4 sm1000base)
target_compile_options(adcdac_ut PRIVATE "-O0")
elf2bin(adcdac_ut)


set(DAC_PLAY_SRCS
src/dac_play.c
../src/codec2_fifo.c
src/stm32f4_dac.c
)

add_mapped_executable(dac_play ${DAC_PLAY_SRCS} ${SYSTEM_SRCS})
target_link_libraries(dac_play stm32f4 sm1000base)
target_compile_options(dac_play PRIVATE "-O0")
elf2bin(dac_play)

set(ADC_REC_SRCS
src/adc_rec.c
)

add_mapped_executable(adc_rec ${ADC_REC_SRCS})
target_link_libraries(adc_rec stm32f4_adac stm32f4 sm1000base)
elf2bin(adc_rec)

set(ADC_SD_SRCS
src/adc_sd.c
)

add_mapped_executable(adc_sd ${ADC_SD_SRCS})
target_link_libraries(adc_sd stm32f4_adac stm32f4 sm1000base)
elf2bin(adc_sd)

set(PWM_UT_SRCS
src/stm32f4_pwm.c
)

add_mapped_executable(pwm_ut ${PWM_UT_SRCS})
target_link_libraries(pwm_ut stm32f4 sm1000base)
elf2bin(pwm_ut)

# ------------------------------------------------

set(USART_UT_SRCS
src/stm32f4_usart.c
src/usart_ut.c
)

add_mapped_executable(usart_ut ${USART_UT_SRCS})
target_link_libraries(usart_ut stm32f4 sm1000base)
elf2bin(usart_ut)

# ------------------------------------------------


set(POWER_UT_SRCS
src/power_ut.c
)

list(APPEND POWER_UT_SRCS ${CODEC2_SRCS})

# stm32/src/power_ut.c:124:5: error: too few arguments to function 'adc_open'
# stm32/src/power_ut.c:125:5: error: too few arguments to function 'dac_open'
#add_mapped_executable(power_ut ${POWER_UT_SRCS})
#target_link_libraries(power_ut stm32f4_adac stm32f4 sm1000base)

set(USB_VCP
usb_conf/usb_bsp.c
usb_conf/usbd_desc.c
usb_conf/usbd_usr.c
usb_lib/cdc/usbd_cdc_core.c
usb_lib/cdc/usbd_cdc_vcp.c
usb_lib/core/usbd_core.c
usb_lib/core/usbd_ioreq.c
usb_lib/core/usbd_req.c
usb_lib/otg/usb_core.c
usb_lib/otg/usb_dcd.c
usb_lib/otg/usb_dcd_int.c)

set(USB_VCP_UT
src/usb_vcp_ut.c
src/stm32f4_usb_vcp.c
)

list(APPEND USB_VCP_UT ${USB_VCP})

add_definitions(-DUSE_USB_OTG_FS -DUSE_ULPI_PHY)
include_directories(usb_conf usb_lib/cdc usb_lib/core usb_lib/otg)

add_mapped_executable(usb_vcp_ut ${USB_VCP_UT})
target_link_libraries(usb_vcp_ut stm32f4 sm1000base)
elf2bin(usb_vcp_ut)

set(ADC_REC_USB_SRCS
src/adc_rec_usb.c
src/stm32f4_usb_vcp.c
)

add_mapped_executable(adc_rec_usb ${ADC_REC_USB_SRCS} ${USB_VCP})
target_link_libraries(adc_rec_usb stm32f4_adac stm32f4 sm1000base)
elf2bin(adc_rec_usb)


set(FDMDV_PROFILE_SRCS
src/fdmdv_profile.c
)

list(APPEND FDMDV_PROFILE_SRCS ${CODEC2_SRCS} ${SYSTEM_SRCS})

# Doesn't compile because M_PITCH is not defined but M_PITCH_S is.
#add_mapped_executable(fdmdv_profile ${FDMDV_PROFILE_SRCS})
#target_link_libraries(fdmdv_profile stm32f4)
#target_compile_definitions(fdmdv_profile PRIVATE PROFILE)

set(SM1000_LEDS_SWITCHES_UT_SRCS
src/sm1000_leds_switches_ut.c
src/sm1000_leds_switches.c
)

add_mapped_executable(sm1000_leds_switches_ut ${SM1000_LEDS_SWITCHES_UT_SRCS})
target_link_libraries(sm1000_leds_switches_ut stm32f4 sm1000base)
elf2bin(sm1000_leds_switches_ut)


set(SM1000_SRCS
src/sm1000_main.c
src/tone.c
src/sfx.c
src/sounds.c
src/morse.c
src/menu.c
src/tot.c
src/sm1000_leds_switches.c
../src/codec2_fifo.c
src/debugblinky.c
src/stm32f4_vrom.c
src/stm32f4_usart.c
src/memtools.c
)

list(APPEND SM1000_SRCS ${CODEC2_SRCS})


add_mapped_executable(sm1000v4 ${SM1000_SRCS} ${SYSTEM_SRCS})
target_link_libraries(sm1000v4 stm32f4_adac stm32f4 CMSIS)
target_compile_options(sm1000v4 PRIVATE "-O3")
elf2bin(sm1000v4)
elf2dfu(sm1000v4)

set(FREEDV_TX_PROFILE_SRCS
src/freedv_tx_profile.c
)

add_mapped_executable(freedv_tx_profile ${FREEDV_TX_PROFILE_SRCS} ${PROFILE_SYSTEM_SRCS})
target_link_libraries(freedv_tx_profile codec2_prof stm32f4 CMSIS)
target_compile_definitions(freedv_tx_profile PRIVATE PROFILE)
elf2bin(freedv_tx_profile)


set(FREEDV_RX_PROFILE_SRCS
src/freedv_rx_profile.c
)

add_mapped_executable(freedv_rx_profile ${FREEDV_RX_PROFILE_SRCS} ${PROFILE_SYSTEM_SRCS})
target_link_libraries(freedv_rx_profile codec2_prof stm32f4 CMSIS)
#target_link_libraries(freedv_rx_base "-Wl,-Map=$<TARGET_PROPERTY:NAME>.map")
elf2bin(freedv_rx_profile)

set(FDMDV_DUMP_RT_SRCS
src/fdmdv_dump_rt.c
)

list(APPEND FDMDV_DUMP_RT_SRCS ${CODEC2_SRCS} ${SYSTEM_SRCS})

# Doesn't build due to FREEDV_NSAMPLES not being defined.
#add_mapped_executable(fdmdv_dump_rt ${FDMDV_DUMP_RT_SRCS})
#target_link_libraries(fdmdv_dump_rt stm32f4_adac stm32f4 sm1000base)
#target_compile_options(fdmdv_dump_rt PRIVATE "-O3")

# ---------------------------------------------------------------------------

set(TUNER_UT_SRCS
src/tuner_ut.c
src/iir_tuner.c
../src/fm.c)

# this needs to be compiled without the optimiser or ugly things happen
# would be nice to work out why as ISRs need to run fast

add_library(stm32f4_adc_tuner STATIC src/stm32f4_adc_tuner.c)
add_mapped_executable(tuner_ut ${TUNER_UT_SRCS} ${SYSTEM_SRCS})
target_compile_options(tuner_ut PRIVATE "-O3")
target_link_libraries(tuner_ut stm32f4_adc_tuner stm32f4_adac stm32f4 sm1000base)
elf2bin(tuner_ut)

# -----------------------------------------------------------------------------

set(ADC_SFDR_UT_SRCS
src/adc_sfdr_ut.c
src/iir_tuner.c
)

add_mapped_executable(adc_sfdr_ut ${ADC_SFDR_UT_SRCS})
target_link_libraries(adc_sfdr_ut stm32f4_adc_tuner stm32f4_adac stm32f4 sm1000base CMSIS)
elf2bin(adc_sfdr_ut)

# -----------------------------------------------------------------------------

#set(FM_LODUC_PLAY_SRCS
#src/fm_loduc_play.c
#gdb_stdio.c
#../src/codec2_fifo.c
#../src/fm.c
#src/debugblinky.c
#src/system_stm32f4xx.c
#src/startup_stm32f4xx.s
#)

#add_library(stm32f4_dacloduc STATIC src/stm32f4_dacloduc.c)

#add_mapped_executable(fm_loduc_play ${FM_LODUC_PLAY_SRCS})
#target_link_libraries(fm_loduc_play stm32f4_dacloduc)

# -----------------------------------------------------------------------------

set(SI5351_UT_SRCS
src/si5351_ut.c
src/new_i2c.c
src/si53xx.c
)

add_mapped_executable(si5351_ut ${SI5351_UT_SRCS})
target_link_libraries(si5351_ut stm32f4 sm1000base)
elf2bin(si5351_ut)

# -----------------------------------------------------------------------------

set(MCO_UT_SRCS
src/mco_ut.c
src/tm_stm32f4_mco_output.c
src/tm_stm32f4_gpio.c
)

add_mapped_executable(mco_ut ${MCO_UT_SRCS})
target_link_libraries(mco_ut stm32f4 sm1000base)
elf2bin(mco_ut)

# -----------------------------------------------------------------------------

