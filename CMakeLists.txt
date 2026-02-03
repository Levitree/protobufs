set(PROTO_BASE_DIR "${CMAKE_CURRENT_LIST_DIR}/proto")
set(NANOPB_OUT_PATH "${CMAKE_CURRENT_LIST_DIR}/generated/nanopb")
set(BUF_TEMPLATE "${CMAKE_CURRENT_LIST_DIR}/buf.gen.yaml")

# ---------------------------------------------------------------------------
# Configure-time codegen (runs buf generate if inputs changed)
# ---------------------------------------------------------------------------

find_program(BUF_EXECUTABLE buf)
if(BUF_EXECUTABLE AND NOT CMAKE_BUILD_EARLY_EXPANSION)
    file(GLOB_RECURSE PROTO_SRCS CONFIGURE_DEPENDS "${PROTO_BASE_DIR}/*.proto")
    file(GLOB_RECURSE NANOPB_OPTIONS CONFIGURE_DEPENDS "${PROTO_BASE_DIR}/*.options")

    set(BUF_CODEGEN_STAMP "${CMAKE_CURRENT_BINARY_DIR}/buf_codegen.stamp")
    set(BUF_CODEGEN_INPUTS
        ${PROTO_SRCS}
        ${NANOPB_OPTIONS}
        "${CMAKE_CURRENT_LIST_DIR}/buf.yaml"
        "${CMAKE_CURRENT_LIST_DIR}/buf.gen.yaml"
    )

    set(_buf_hash_material "")
    foreach(_f IN LISTS BUF_CODEGEN_INPUTS)
        if(EXISTS "${_f}")
            file(SHA256 "${_f}" _f_sha)
            string(APPEND _buf_hash_material "${_f}|${_f_sha};")
        endif()
    endforeach()
    string(SHA256 BUF_CODEGEN_HASH "${_buf_hash_material}")

    set(_run_buf_codegen TRUE)
    if(EXISTS "${BUF_CODEGEN_STAMP}")
        file(READ "${BUF_CODEGEN_STAMP}" _prev_hash)
        string(STRIP "${_prev_hash}" _prev_hash)
        if(_prev_hash STREQUAL BUF_CODEGEN_HASH)
            # Also verify output directory and files exist before skipping
            if(EXISTS "${NANOPB_OUT_PATH}" AND IS_DIRECTORY "${NANOPB_OUT_PATH}")
                file(GLOB _nanopb_check "${NANOPB_OUT_PATH}/*.pb.c")
                if(_nanopb_check)
                    set(_run_buf_codegen FALSE)
                else()
                    message(STATUS "Nanopb output files missing, forcing regeneration")
                endif()
            else()
                message(STATUS "Nanopb output directory missing, forcing regeneration")
            endif()
        endif()
    endif()

    if(_run_buf_codegen)
        message(STATUS "Generating nanopb sources via buf (configure-time)")
        execute_process(
            COMMAND "${BUF_EXECUTABLE}" generate --template "${BUF_TEMPLATE}"
            WORKING_DIRECTORY "${CMAKE_CURRENT_LIST_DIR}"
            RESULT_VARIABLE BUF_GEN_RESULT
            OUTPUT_VARIABLE BUF_GEN_STDOUT
            ERROR_VARIABLE BUF_GEN_STDERR
        )

        if(NOT BUF_GEN_RESULT EQUAL 0)
            message(WARNING
                "buf generate failed with exit code ${BUF_GEN_RESULT}.\n"
                "stderr:\n${BUF_GEN_STDERR}\n"
            )
        else()
            file(WRITE "${BUF_CODEGEN_STAMP}" "${BUF_CODEGEN_HASH}\n")
        endif()
    else()
        message(STATUS "Nanopb sources up-to-date (buf codegen skipped)")
    endif()
endif()

# ---------------------------------------------------------------------------
# Collect sources
# ---------------------------------------------------------------------------

# Nanopb runtime sources
set(NANOPB_CORE_SRCS
    "${CMAKE_CURRENT_LIST_DIR}/nanopb/pb_common.c"
    "${CMAKE_CURRENT_LIST_DIR}/nanopb/pb_encode.c"
    "${CMAKE_CURRENT_LIST_DIR}/nanopb/pb_decode.c"
)

# Generated nanopb sources (CONFIGURE_DEPENDS triggers re-configure if files change)
# Note: CONFIGURE_DEPENDS is invalid during IDF early expansion (script mode)
if(CMAKE_BUILD_EARLY_EXPANSION)
    file(GLOB_RECURSE NANOPB_GEN_SRCS "${NANOPB_OUT_PATH}/*.pb.c")
else()
    file(GLOB_RECURSE NANOPB_GEN_SRCS CONFIGURE_DEPENDS "${NANOPB_OUT_PATH}/*.pb.c")
endif()

# Combine all sources
set(PROTOBUFS_SRCS
    ${NANOPB_CORE_SRCS}
    ${NANOPB_GEN_SRCS}
)

# Include directories
set(PROTOBUFS_INCLUDE_DIRS
    "${CMAKE_CURRENT_LIST_DIR}/nanopb"
    "${NANOPB_OUT_PATH}"
)

# ---------------------------------------------------------------------------
# ESP-IDF component registration (only when building as IDF component)
# ---------------------------------------------------------------------------
if(ESP_PLATFORM)
    idf_component_register(
        SRCS ${PROTOBUFS_SRCS}
        INCLUDE_DIRS ${PROTOBUFS_INCLUDE_DIRS}
    )

    target_compile_definitions(${COMPONENT_LIB} PUBLIC PB_BUFFER_ONLY)
    target_compile_definitions(${COMPONENT_LIB} PUBLIC PB_LITTLE_ENDIAN_8BIT)
    target_compile_definitions(${COMPONENT_LIB} PUBLIC PB_NO_ERRMSG)
else()
    # Standard CMake library for non-IDF builds
    add_library(protobufs STATIC ${PROTOBUFS_SRCS})
    target_include_directories(protobufs PUBLIC ${PROTOBUFS_INCLUDE_DIRS})
    target_compile_definitions(protobufs PUBLIC PB_BUFFER_ONLY)
    target_compile_definitions(protobufs PUBLIC PB_LITTLE_ENDIAN_8BIT)
    target_compile_definitions(protobufs PUBLIC PB_NO_ERRMSG)
endif()
