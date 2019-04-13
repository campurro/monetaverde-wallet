#  Monetaverde wallet Graphical User Interface
#  Based on Bytecoin wallet
#
#  First updtate submodules
#  git submodule update --init --recursive
#  git submodule foreach git pull origin master
#

cmake_minimum_required(VERSION 3.1)

#Set build version
set(VERSION_MAJOR "1")
set(VERSION_MINOR "0.0")
set(VERSION "${VERSION_MAJOR}.${VERSION_MINOR}")
find_package(Git QUIET)
if(Git_FOUND OR GIT_FOUND)
    message(STATUS "Found Git: ${GIT_EXECUTABLE}")
    execute_process(COMMAND git rev-parse --abbrev-ref HEAD  # Get the current branch
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_BRANCH
        OUTPUT_STRIP_TRAILING_WHITESPACE)

    execute_process(COMMAND git log -1 --format=%h # Get latest commit hash
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE COMMIT_ID
        OUTPUT_STRIP_TRAILING_WHITESPACE)
else()
    message(STATUS "Git not found !")
    set(GIT_BRANCH "unknown")
    set(COMMIT_ID "unknown")
endif()

set(GIT_REVISION "${COMMIT_ID}-${CMAKE_BUILD_TYPE}")
foreach(p
        CMP0071 # 3.1: Let AUTOMOC and AUTOUIC process GENERATED files
        )
    if(POLICY ${p})
        cmake_policy(SET ${p} NEW)
    endif()
endforeach()

include(CryptoNoteWallet.cmake)

project(${CN_PROJECT_NAME})

set(CRYPTONOTE_LIB cryptoNote)

include_directories(${CMAKE_CURRENT_BINARY_DIR}
    src
    include
    cryptonote/external
    cryptonote/external/rocksdb/include/
    cryptonote/include
    cryptonote/src
    cryptonote/src/MultiHashing/xmrig
    cryptonote/src/MultiHashing/xmrig/3rdparty
    )

find_package(Qt5 REQUIRED
    Gui
    Widgets
    Network
    DBus
    )

include_directories(${Qt5DBus_INCLUDE_DIRS} cryptonote/external/rocksdb/include/ include src external "${CMAKE_CURRENT_BINARY_DIR}/version")

set(Boost_USE_STATIC_LIBS ON)
if(WIN32)
    set(Boost_USE_STATIC_RUNTIME OFF)
else(WIN32)
    set(Boost_USE_STATIC_RUNTIME ON)
endif(WIN32)

find_package(Boost 1.58 REQUIRED COMPONENTS date_time filesystem program_options regex serialization system thread chrono)
if ((${Boost_MAJOR_VERSION} EQUAL 1) AND (${Boost_MINOR_VERSION} EQUAL 54))
    message(SEND_ERROR "Boost version 1.58 is unsupported, more details are available here http://goo.gl/RrCFmA")
endif ()

include_directories(${Boost_INCLUDE_DIRS})
link_directories(${Boost_LIBRARY_DIRS})

add_definitions(-DGIT_REVISION="${GIT_REVISION}")

set(CMAKE_AUTOMOC ON)

###############
# Multi-hashing based on xmrig
###############
option(WITH_AEON            "CryptoNight-Lite support" ON)
option(WITH_SUMO            "CryptoNight-Heavy support" ON)
option(WITH_CN_PICO         "CryptoNight-Pico support" ON)
option(WITH_CN_GPU          "CryptoNight-GPU support" ON)
set(MultiHashing
    cryptonote/src/MultiHashing/multihashing.cpp
    cryptonote/src/MultiHashing/xmrig/Mem.cpp
    cryptonote/src/MultiHashing/xmrig/crypto/c_blake256.c
    cryptonote/src/MultiHashing/xmrig/crypto/c_groestl.c
    cryptonote/src/MultiHashing/xmrig/crypto/c_jh.c
    cryptonote/src/MultiHashing/xmrig/crypto/c_skein.c
    cryptonote/src/MultiHashing/xmrig/common/crypto/keccak.cpp
  )

if (WIN32)
  set(MultiHashing_OS
      cryptonote/src/MultiHashing/xmrig/Mem_win.cpp
  )
else ()
  set(MultiHashing_OS
      cryptonote/src/MultiHashing/xmrig/Mem_unix.cpp
  )
endif ()

if (CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64)$")
    set(MultiHashing "${MultiHashing}"
        cryptonote/src/MultiHashing/xmrig/crypto/cn_gpu_ssse3.cpp
        cryptonote/src/MultiHashing/xmrig/common/cpu/Cpu.cpp
        cryptonote/src/MultiHashing/xmrig/common/cpu/BasicCpuInfo.cpp
    )
endif ()

if (XMRIG_ARM)
    set(MultiHashing_OS "${MultiHashing_OS}" cryptonote/src/MultiHashing/xmrig/crypto/CryptoNight_arm.h)
else()
    set(MultiHashing_OS "${MultiHashing_OS}" cryptonote/src/MultiHashing/xmrig/crypto/CryptoNight_x86.h)
endif()
#################### CPU ###############################
if (CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64)$")
    add_definitions(/DRAPIDJSON_SSE2)
endif()

if (NOT ARM_TARGET)
    if (CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64|armv8-a)$")
        set(ARM_TARGET 8)
    elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "^(armv7|armv7f|armv7s|armv7k|armv7-a|armv7l)$")
        set(ARM_TARGET 7)
    endif()
endif()

if (ARM_TARGET AND ARM_TARGET GREATER 6)
    set(XMRIG_ARM     ON)
    set(WITH_LIBCPUID OFF)
    add_definitions(/DXMRIG_ARM)

    message(STATUS "Use ARM_TARGET=${ARM_TARGET} (${CMAKE_SYSTEM_PROCESSOR})")

    include(CheckCXXCompilerFlag)

    if (ARM_TARGET EQUAL 8)
        set(XMRIG_ARMv8 ON)
        add_definitions(/DXMRIG_ARMv8)

        CHECK_CXX_COMPILER_FLAG(-march=armv8-a+crypto XMRIG_ARM_CRYPTO)

        if (XMRIG_ARM_CRYPTO)
            add_definitions(/DXMRIG_ARM_CRYPTO)
            set(ARM8_CXX_FLAGS "-march=armv8-a+crypto")
        else()
            set(ARM8_CXX_FLAGS "-march=armv8-a")
        endif()
    elseif (ARM_TARGET EQUAL 7)
        set(XMRIG_ARMv7 ON)
        add_definitions(/DXMRIG_ARMv7)
    endif()
endif()
########################################################
################# FLAGS ################################
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD 11)

if ("${CMAKE_BUILD_TYPE}" STREQUAL "")
    set(CMAKE_BUILD_TYPE Release)
endif()

if (CMAKE_BUILD_TYPE STREQUAL "Release")
    add_definitions(/DNDEBUG)
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES GNU)

    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wno-strict-aliasing")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -Ofast")

    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
    #-Wno-class-memaccess => Throws an error in Linux
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Ofast -s")

    if (XMRIG_ARMv8)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${ARM8_CXX_FLAGS}")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${ARM8_CXX_FLAGS} -flax-vector-conversions")
    elseif (XMRIG_ARMv7)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpu=neon")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfpu=neon -flax-vector-conversions")
    else()
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -maes")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maes")
    endif()

    if (WIN32)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static")
    else()
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++")
    endif()

    add_definitions(/D_GNU_SOURCE)

    if (${CMAKE_VERSION} VERSION_LESS "3.1.0")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
    endif()

    #set(CMAKE_C_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -gdwarf-2")

elseif (CMAKE_CXX_COMPILER_ID MATCHES MSVC)

    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /Ox /Ot /Oi /MT /GL")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Ox /Ot /Oi /MT /GL")
    add_definitions(/D_CRT_SECURE_NO_WARNINGS)
    add_definitions(/D_CRT_NONSTDC_NO_WARNINGS)
    add_definitions(/DNOMINMAX)

elseif (CMAKE_CXX_COMPILER_ID MATCHES Clang)

    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -Ofast -funroll-loops -fmerge-all-constants")

    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wno-missing-braces")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Ofast -funroll-loops -fmerge-all-constants")

    if (XMRIG_ARMv8)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${ARM8_CXX_FLAGS}")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${ARM8_CXX_FLAGS}")
    elseif (XMRIG_ARMv7)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpu=neon -march=${CMAKE_SYSTEM_PROCESSOR}")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfpu=neon -march=${CMAKE_SYSTEM_PROCESSOR}")
    else()
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -maes")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maes")
    endif()

endif()
########################################################
################### ASM ####################################
if (WITH_ASM AND NOT XMRIG_ARM AND CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(XMRIG_ASM_LIBRARY "xmrig-asm")

    if (CMAKE_C_COMPILER_ID MATCHES MSVC)
        enable_language(ASM_MASM)

        if (MSVC_TOOLSET_VERSION GREATER_EQUAL 141)
            set(XMRIG_ASM_FILES
                "cryptonote/src/MultiHashing/xmrig/crypto/asm/cn_main_loop.asm"
                "cryptonote/src/MultiHashing/xmrig/crypto/asm/CryptonightR_template.asm"
            )
        else()
            set(XMRIG_ASM_FILES
                "cryptonote/src/MultiHashing/xmrig/crypto/asm/win64/cn_main_loop.asm"
                "cryptonote/src/MultiHashing/xmrig/crypto/asm/win64/CryptonightR_template.asm"
            )
        endif()

        set_property(SOURCE ${XMRIG_ASM_FILES} PROPERTY ASM_MASM)
    else()
        enable_language(ASM)

        if (WIN32 AND CMAKE_C_COMPILER_ID MATCHES GNU)
            set(XMRIG_ASM_FILES
                "cryptonote/src/MultiHashing/xmrig/crypto/asm/win64/cn_main_loop.S"
                "cryptonote/src/MultiHashing/xmrig/crypto/asm/CryptonightR_template.S"
            )
        else()
            set(XMRIG_ASM_FILES
                "cryptonote/src/MultiHashing/xmrig/crypto/asm/cn_main_loop.S"
                "cryptonote/src/MultiHashing/xmrig/crypto/asm/CryptonightR_template.S"
            )
        endif()

        set_property(SOURCE ${XMRIG_ASM_FILES} PROPERTY C)
    endif()

    add_library(${XMRIG_ASM_LIBRARY} STATIC ${XMRIG_ASM_FILES})
    set(XMRIG_ASM_SOURCES cryptonote/src/MultiHashing/xmrig/crypto/Asm.h cryptonote/src/MultiHashing/xmrig/crypto/Asm.cpp cryptonote/src/MultiHashing/xmrig/crypto/CryptonightR_gen.cpp)
    set_property(TARGET ${XMRIG_ASM_LIBRARY} PROPERTY LINKER_LANGUAGE C)
else()
    set(XMRIG_ASM_SOURCES "")
    set(XMRIG_ASM_LIBRARY "")
    add_definitions(/DXMRIG_NO_ASM)
endif()
###################################################################
############################### CN GPU ############################
if (WITH_CN_GPU AND CMAKE_SIZEOF_VOID_P EQUAL 8)

    if (XMRIG_ARM)
        set(CN_GPU_SOURCES cryptonote/src/MultiHashing/xmrig/crypto/cn_gpu_arm.cpp)

        if (CMAKE_CXX_COMPILER_ID MATCHES GNU OR CMAKE_CXX_COMPILER_ID MATCHES Clang)
            set_source_files_properties(cryptonote/src/MultiHashing/xmrig/crypto/cn_gpu_arm.cpp PROPERTIES COMPILE_FLAGS "-O3")
        endif()
    else()
        set(CN_GPU_SOURCES cryptonote/src/MultiHashing/xmrig/crypto/cn_gpu_avx.cpp cryptonote/src/MultiHashing/xmrig/crypto/cn_gpu_ssse3.cpp)

        if (CMAKE_CXX_COMPILER_ID MATCHES GNU OR CMAKE_CXX_COMPILER_ID MATCHES Clang)
            set_source_files_properties(cryptonote/src/MultiHashing/xmrig/crypto/cn_gpu_avx.cpp PROPERTIES COMPILE_FLAGS "-O3 -mavx2")
            set_source_files_properties(cryptonote/src/MultiHashing/xmrig/crypto/cn_gpu_ssse3.cpp PROPERTIES COMPILE_FLAGS "-O3")
        elseif (CMAKE_CXX_COMPILER_ID MATCHES MSVC)
            set_source_files_properties(cryptonote/src/MultiHashing/xmrig/crypto/cn_gpu_avx.cpp PROPERTIES COMPILE_FLAGS "/arch:AVX")
        endif()
    endif()
else()
    set(CN_GPU_SOURCES "")

    add_definitions(/DXMRIG_NO_CN_GPU)
endif()
#####################################################################






file(GLOB_RECURSE CRYPTONOTE_SOURCES
    cryptonote/external/miniupnpc/connecthostport.c
    cryptonote/external/miniupnpc/igd_desc_parse.c
    cryptonote/external/miniupnpc/minisoap.c
    cryptonote/external/miniupnpc/miniupnpc.c
    cryptonote/external/miniupnpc/miniwget.c
    cryptonote/external/miniupnpc/minixml.c
    cryptonote/external/miniupnpc/portlistingparse.c
    cryptonote/external/miniupnpc/receivedata.c
    cryptonote/external/miniupnpc/upnpcommands.c
    cryptonote/external/miniupnpc/upnpreplyparse.c
    cryptonote/src/BlockchainExplorer/BlockchainExplorer.cpp
    cryptonote/src/BlockchainExplorer/BlockchainExplorerErrors.cpp
    cryptonote/src/Common/Base58.cpp
    cryptonote/src/Common/CommandLine.cpp
    cryptonote/src/Common/Util.cpp
    cryptonote/src/Common/StringTools.cpp
    cryptonote/src/Common/JsonValue.cpp
    cryptonote/src/Common/ConsoleTools.cpp
    cryptonote/src/Common/MemoryInputStream.cpp
    cryptonote/src/Common/PathTools.cpp
    cryptonote/src/Common/StdInputStream.cpp
    cryptonote/src/Common/StdOutputStream.cpp
    cryptonote/src/Common/StreamTools.cpp
    cryptonote/src/Common/StringOutputStream.cpp
    cryptonote/src/Common/StringView.cpp
    cryptonote/src/Common/VectorOutputStream.cpp
    cryptonote/src/Common/ScopeExit.cpp
    cryptonote/src/ConnectivityTool/ConnectivityTool.cpp
    cryptonote/src/crypto/blake256.c
    cryptonote/src/crypto/chacha8.c
    cryptonote/src/crypto/crypto-ops-data.c
    cryptonote/src/crypto/crypto-ops.c
    cryptonote/src/crypto/crypto.cpp
    cryptonote/src/crypto/groestl.c
    cryptonote/src/crypto/hash-extra-blake.c
    cryptonote/src/crypto/hash-extra-groestl.c
    cryptonote/src/crypto/hash-extra-jh.c
    cryptonote/src/crypto/hash-extra-skein.c
    cryptonote/src/crypto/hash.c
    cryptonote/src/crypto/jh.c
    cryptonote/src/crypto/keccak.c
    cryptonote/src/crypto/oaes_lib.c
    cryptonote/src/crypto/random.c
#    cryptonote/src/crypto/skein.c #already included in c_skein.c xmrig crypto library
    cryptonote/src/crypto/slow-hash.c
    cryptonote/src/crypto/slow-hash.cpp
    cryptonote/src/crypto/tree-hash.c
    cryptonote/src/CryptoNoteCore/*.cpp
    cryptonote/src/CryptoNoteProtocol/*.cpp
    cryptonote/src/Daemon/*.cpp
    cryptonote/src/HTTP/*.cpp
    cryptonote/src/InProcessNode/InProcessNode.cpp
    cryptonote/src/InProcessNode/InProcessNodeErrors.cpp
    cryptonote/src/Logging/*.cpp
    cryptonote/src/NodeRpcProxy/NodeErrors.cpp
    cryptonote/src/NodeRpcProxy/NodeRpcProxy.cpp
    cryptonote/src/P2p/*.cpp
    cryptonote/src/Rpc/*.cpp
    cryptonote/src/Serialization/*.cpp
    cryptonote/src/Transfers/BlockchainSynchronizer.cpp
    cryptonote/src/Transfers/SynchronizationState.cpp
    cryptonote/src/Transfers/TransfersConsumer.cpp
    cryptonote/src/Transfers/TransfersContainer.cpp
    cryptonote/src/Transfers/TransfersSubscription.cpp
    cryptonote/src/Transfers/TransfersSynchronizer.cpp
    cryptonote/src/Wallet/*.cpp
    cryptonote/src/WalletLegacy/KeysStorage.cpp
    cryptonote/src/WalletLegacy/WalletLegacy.cpp
    cryptonote/src/WalletLegacy/WalletHelper.cpp
    cryptonote/src/WalletLegacy/WalletLegacySerializer.cpp
    cryptonote/src/WalletLegacy/WalletLegacySerialization.cpp
    cryptonote/src/WalletLegacy/WalletTransactionSender.cpp
    cryptonote/src/WalletLegacy/WalletUnconfirmedTransactions.cpp
    cryptonote/src/WalletLegacy/WalletUserTransactionsCache.cpp
    cryptonote/src/System/ContextGroup.cpp
    cryptonote/src/System/Event.cpp
    cryptonote/src/System/EventLock.cpp
    cryptonote/src/System/InterruptedException.cpp
    cryptonote/src/System/Ipv4Address.cpp
    cryptonote/src/System/TcpStream.cpp
    cryptonote/src/JsonRpcServer/*.cpp
    cryptonote/src/PaymentGate/*.cpp
    cryptonote/src/PaymentGateService/*.cpp
    cryptonote/src/Miner/*.cpp
    )

file(GLOB_RECURSE WALLET_SOURCES src/*.cpp)
file(GLOB_RECURSE WALLET_HEADERS src/*.h)
file(GLOB_RECURSE WALLET_FORMS src/Gui/*/*.ui)

if(MSVC)
    include_directories(Platform/Windows)
    set(WALLET_SOURCES ${WALLET_SOURCES} ${CMAKE_SOURCE_DIR}/Platform/Windows/ApplicationEventHandler.cpp)
elseif(APPLE)
    include_directories(Platform/OSX)
    include_directories(Platform/Posix)
    set(WALLET_SOURCES ${WALLET_SOURCES} ${CMAKE_SOURCE_DIR}/Platform/OSX/ApplicationEventHandler.h ${CMAKE_SOURCE_DIR}/Platform/OSX/ApplicationEventHandler.mm ${CMAKE_SOURCE_DIR}/Platform/OSX/INotificationMonitorObserver.h ${CMAKE_SOURCE_DIR}/Platform/OSX/NotificationMonitor.h ${CMAKE_SOURCE_DIR}/Platform/OSX/NotificationMonitor.mm ${CMAKE_SOURCE_DIR}/Platform/OSX/WalletApplication.mm )
    set(WALLET_SOURCES ${WALLET_SOURCES} ${CMAKE_SOURCE_DIR}/Platform/Posix/System/MemoryMappedFile.cpp)
else()
    include_directories(Platform/Linux)
    include_directories(Platform/Posix)
    set(WALLET_SOURCES ${WALLET_SOURCES} ${CMAKE_SOURCE_DIR}/Platform/Linux/ApplicationEventHandler.cpp)
    set(WALLET_SOURCES ${WALLET_SOURCES} ${CMAKE_SOURCE_DIR}/Platform/Posix/System/MemoryMappedFile.cpp)
endif()


set(WALLET_RESOURCES src/resources.qrc)

set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)

Qt5_wrap_cpp(WALLET_HEADERS_MOC ${WALLET_HEADERS})
qt5_wrap_ui(WALLET_FORMS_HEADERS ${WALLET_FORMS})
qt5_add_resources(WALLET_RESOURCES_RCC ${WALLET_RESOURCES})

if (WIN32)
    if (NOT MSVC)
        message(FATAL_ERROR "Only MSVC is supported on this platform")
    endif ()

    add_definitions(/D_CRT_SECURE_NO_WARNINGS /D_WIN32_WINNT=0x0600 /DSTATICLIB)
    include_directories(cryptonote/src/Platform/msc)

    set(PLATFORM_DIR Windows)
    set(BUILD_PLATFORM WIN32)
    set(BUILD_RESOURCES src/monetaverdewallet.rc)
    set(QTMAIN Qt5::WinMain)

elseif (UNIX)

    set(CRYPTONOTE_SOURCES ${CRYPTONOTE_SOURCES} cryptonote/external/miniupnpc/minissdpc.c)

    if (APPLE)
        enable_language(ASM)
        file(GLOB_RECURSE OBJC_SOURCES src/*.mm)
        set(WALLET_SOURCES ${WALLET_SOURCES} ${OBJC_SOURCES})
        set(PLATFORM_DIR OSX)
        set(MACOSX_BUNDLE_INFO_STRING "Monetaverde GUI wallet v${VERSION} ${GIT_REVISION}")
        set(MACOSX_BUNDLE_LONG_VERSION_STRING "${VERSION} ${GIT_REVISION}")
        set(MACOSX_BUNDLE_BUNDLE_NAME MonetaverdeWallet)
        set(MACOSX_BUNDLE_SHORT_VERSION_STRING "${VERSION}")
        set(MACOSX_BUNDLE_BUNDLE_VERSION "${VERSION}")

        find_package(Qt5PrintSupport REQUIRED)
        include_directories(/usr/include/malloc)

        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maes -std=c++11 -stdlib=libc++")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11 -maes -D_DARWIN_C_SOURCE")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -framework Cocoa -framework OpenGL -framework CoreFoundation -framework Carbon -framework IOKit -L/usr/lib")

        set(MACOSX_BUNDLE_ICON_FILE monetaverde.icns)
        set(APPLICATION_ICON src/images/monetaverde.icns)
        set_source_files_properties(${APPLICATION_ICON} PROPERTIES MACOSX_PACKAGE_LOCATION "Resources")

        set(BUILD_PLATFORM MACOSX_BUNDLE)
        set(BUILD_RESOURCES ${APPLICATION_ICON})

        GET_TARGET_PROPERTY(QT_LIB_DIR "${Qt5Widgets_LIBRARIES}" LOCATION)
        GET_FILENAME_COMPONENT(QT_LIB_DIR "${QT_LIB_DIR}" PATH)

        # remove ranlib complaining for no symbols
        set(CMAKE_C_ARCHIVE_CREATE   "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
        set(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
        set(CMAKE_C_ARCHIVE_FINISH   "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
        set(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
        set(RELEASE_FLAGS "-Wno-uninitialized -Wno-deprecated-declarations")
    else(APPLE)
        set(PLATFORM_DIR Linux)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maes -std=c++11")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -maes -std=c11")
    endif (APPLE)
endif ()

set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${DEBUG_FLAGS}")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${DEBUG_FLAGS}")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${RELEASE_FLAGS}")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${RELEASE_FLAGS}")

include_directories(cryptonote/src/Platform/${PLATFORM_DIR})
file(GLOB PLATFORM_SOURCES cryptonote/src/Platform/${PLATFORM_DIR}/System/*)
set(CRYPTONOTE_SOURCES ${CRYPTONOTE_SOURCES} ${PLATFORM_SOURCES} ${CRYPTONOTE_PLATFORM})

add_library(${CRYPTONOTE_LIB} STATIC ${CRYPTONOTE_SOURCES} "${MultiHashing}" "${MultiHashing_OS}" "${XMRIG_ASM_SOURCES}" "${CN_GPU_SOURCES}")
set_target_properties(${CRYPTONOTE_LIB} PROPERTIES COMPILE_DEFINITIONS _GNU_SOURCE)
target_link_libraries(${CRYPTONOTE_LIB} rocksdb ${Boost_LIBRARIES} ${XMRIG_ASM_LIBRARY} ${Qt5DBus_LIBRARIES})

add_executable(${PROJECT_NAME} ${BUILD_PLATFORM} ${BUILD_RESOURCES} ${WALLET_SOURCES} ${WALLET_HEADERS} ${WALLET_FORMS_HEADERS} ${WALLET_RESOURCES_RCC})

set_target_properties(${PROJECT_NAME} PROPERTIES COMPILE_DEFINITIONS _GNU_SOURCE)
target_link_libraries(${PROJECT_NAME} rocksdb ${Boost_LIBRARIES} ${XMRIG_ASM_LIBRARY} ${QTMAIN} ${CRYPTONOTE_LIB} ${Qt5DBus_LIBRARIES})

if (APPLE)
    set_target_properties(${PROJECT_NAME} PROPERTIES MACOSX_BUNDLE_INFO_PLIST ${CMAKE_SOURCE_DIR}/Platform/OSX/MacOSXBundleInfo.plist.in)
    find_package(Qt5PrintSupport REQUIRED)
    target_link_libraries(${PROJECT_NAME} Qt5::PrintSupport)
elseif (UNIX)
    target_link_libraries(${PROJECT_NAME} -lpthread)
elseif (WIN32)
    target_link_libraries(${PROJECT_NAME} rpcrt4.lib wtsapi32.lib  Imm32 Iphlpapi Winmm)
endif (APPLE)

target_link_libraries(${PROJECT_NAME} Qt5::Widgets Qt5::Gui Qt5::Network)

add_subdirectory(cryptonote/external)
message(STATUS "--------------------- Monetaverde Wallet building details ----------------------------")
message(STATUS "Build : ${CMAKE_BUILD_TYPE}")
message(STATUS "Compiler : ${CMAKE_C_COMPILER_ID} version ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "Branch : ${GIT_BRANCH}")
message(STATUS "Version : ${VERSION}")
message(STATUS "Revision : ${GIT_REVISION}")
message(STATUS "Portable : ${PORTABLE}" )
message(STATUS "----------------------------------------------------------")
configure_file("cryptonote/src/version.h.in" "version.h")
configure_file("cryptonote/src/version.h.in" ${CMAKE_SOURCE_DIR}/src/version.h)
configure_file("src/CryptoNoteWalletConfig.h.in" "CryptoNoteWalletConfig.h")
