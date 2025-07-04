set(EMULATOR_COMMON_SRCS
    riscv_callbacks.cpp
    riscv_callbacks.h
    riscv_config.h
    riscv_platform.cpp
    riscv_platform.h
    riscv_platform_impl.cpp
    riscv_platform_impl.h
    riscv_prelude.cpp
    riscv_prelude.h
    riscv_sail.h
    rvfi_dii.h
    rvfi_dii.cpp
    riscv_sim.cpp
    riscv_softfloat.c
    riscv_softfloat.h
    config_utils.h
    config_utils.cpp
)

foreach (xlen IN ITEMS 32 64)
    foreach (flen IN ITEMS 32 64)
        set(arch "rv${xlen}")
        if (flen EQUAL 32)
            string(APPEND arch "f")
        else()
            string(APPEND arch "d")
        endif()

        add_executable(riscv_sim_${arch}
            "${CMAKE_BINARY_DIR}/riscv_model_${arch}.c"
            "${CMAKE_BINARY_DIR}/riscv_model_${arch}.h"
            ${EMULATOR_COMMON_SRCS}
        )
        # The generated model is not warnings-clean, silence them.
        set(_generated_c_warning_opt_out
            -Wno-extra
            -Wno-unused
            -Wno-uninitialized
        )
        set_source_files_properties("${CMAKE_BINARY_DIR}/riscv_model_${arch}.c"
                                    PROPERTIES COMPILE_OPTIONS "${_generated_c_warning_opt_out}")

        if (NOT arch IN_LIST DEFAULT_ARCHITECTURES)
            set_target_properties(riscv_sim_${arch} PROPERTIES EXCLUDE_FROM_ALL TRUE)
        endif()

        add_dependencies(riscv_sim_${arch} generated_model_${arch})

        target_link_libraries(riscv_sim_${arch}
            PRIVATE softfloat sail_runtime default_config
        )

        target_include_directories(riscv_sim_${arch}
            # So the generated C can find riscv_platform/prelude.h"
            PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}"
            # So we can find riscv_model_${arch}.h
                    "${CMAKE_BINARY_DIR}"
        )

        # This is necessary so it can pick the right riscv_model_${arch}.h
        # to #include. We can remove it when there is a single model with
        # runtime xlen and flen.
        target_compile_definitions(riscv_sim_${arch}
            PRIVATE "RISCV_MODEL_HEADER=<riscv_model_${arch}.h>"
        )

        # Enable Sail coverage collection.
        if (COVERAGE)
            target_compile_definitions(riscv_sim_${arch}
                PRIVATE "SAILCOV"
            )
        endif()

        # TODO: Enable warnings when we use the #include trick
        # to include the generated Sail code. Currently it
        # generates too many warnings to turn these on globally.

        # target_compile_options(riscv_sim_${arch} PRIVATE
        #     -Wall -Wextra
        #     # Too annoying at the moment.
        #     -Wno-unused-parameter
        # )

        install(TARGETS riscv_sim_${arch}
            OPTIONAL
            RUNTIME DESTINATION "bin"
        )
    endforeach()
endforeach()
