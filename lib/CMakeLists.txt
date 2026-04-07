#[[ [ADAPT_IMGUI_BUNDLE]
    Lift the previously undocumented `_BUILD_STATIC` /
    `_HAVE_AUTO_FONT_FIND` / `GLYPH_RENDER_TYPE` cache variables to proper
    MICROTEX_* options with documented defaults that match the original
    upstream behavior. This makes the build configuration discoverable
    via `cmake -L` and avoids the `set(... CACHE BOOL "" FORCE)` smell
    on the consumer side.

    `_DISABLE_ALIAS` is dropped entirely: VERSION / SOVERSION are no-ops
    on static libraries anyway (they only produce the `libfoo.so.1`
    symlinks for shared libs), so the combined condition
    `if (NOT MICROTEX_BUILD_STATIC)` is already the complete and correct
    gate. The original upstream condition
    `NOT DEFINED _DISABLE_ALIAS AND NOT _DISABLE_ALIAS` was buggy anyway
    — it only fired when the variable was completely undefined.
]]
option(MICROTEX_BUILD_STATIC "Build microtex as a static library (default: SHARED)" OFF)

if (MICROTEX_BUILD_STATIC)
    add_library(microtex STATIC "")
else ()
    add_library(microtex SHARED "")
endif ()

if (MSVC)
    add_compile_options("/utf-8")
    target_compile_features(microtex PUBLIC cxx_std_17)
    if (MICROTEX_BUILD_STATIC)
        # [ADAPT_IMGUI_BUNDLE]
        # For static builds, expose MICROTEX_STATIC to BOTH the microtex
        # target (via PRIVATE/PUBLIC) and any consumer linking against it
        # (via INTERFACE/PUBLIC). This makes microtexexport.h expand
        # MICROTEX_EXPORT to nothing on both sides, so the static lib has
        # plain symbols and consumers reference them directly (no
        # __imp_* DLL import stubs).
        target_compile_definitions(microtex PUBLIC -DMICROTEX_STATIC)
    else ()
        target_compile_definitions(microtex PRIVATE -DMICROTEX_LIBRARY)
    endif ()
    target_compile_definitions(microtex PUBLIC -D_HAS_STD_BYTE=1)
else ()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fexceptions")
endif ()

set_target_properties(
    microtex PROPERTIES
    CXX_VISIBILITY_PRESET hidden
)

if (NOT MICROTEX_BUILD_STATIC)
    set_target_properties(
        microtex PROPERTIES
        VERSION ${PROJECT_VERSION}
        SOVERSION ${MICROTEX_API_VERSION}
    )
endif ()

set(
    _SRCS
    # atom folder
    atom/atom.cpp
    atom/atom_basic.cpp
    atom/atom_char.cpp
    atom/atom_misc.cpp
    atom/atom_matrix.cpp
    atom/atom_row.cpp
    atom/atom_space.cpp
    atom/atom_stack.cpp
    atom/atom_accent.cpp
    atom/atom_scripts.cpp
    atom/atom_vrow.cpp
    atom/atom_operator.cpp
    atom/atom_zstack.cpp
    atom/atom_sideset.cpp
    atom/atom_font.cpp
    atom/atom_delim.cpp
    atom/atom_root.cpp
    atom/atom_frac.cpp
    atom/atom_fence.cpp
    atom/atom_box.cpp
    atom/atom_text.cpp
    atom/colors_def.cpp
    # box folder
    box/box.cpp
    box/box_factory.cpp
    box/box_group.cpp
    box/box_single.cpp
    # core folder
    core/split.cpp
    core/formula.cpp
    core/formula_def.cpp
    core/glue.cpp
    core/localized_num.cpp
    core/parser.cpp
    core/debug_config.cpp
    # macro folder
    macro/macro.cpp
    macro/macro_def.cpp
    macro/macro_misc.cpp
    macro/macro_scripts.cpp
    macro/macro_accent.cpp
    macro/macro_colors.cpp
    macro/macro_space.cpp
    macro/macro_delims.cpp
    macro/macro_frac.cpp
    macro/macro_styles.cpp
    macro/macro_fonts.cpp
    # env folder
    env/env.cpp
    env/units.cpp
    # utils folder
    utils/string_utils.cpp
    utils/utf.cpp
    utils/utils.cpp
    # otf folder
    otf/clm.cpp
    otf/fontsense.cpp
    otf/glyph.cpp
    otf/otf.cpp
    otf/path.cpp
    # unimath folder
    unimath/font_src.cpp
    unimath/math_type.cpp
    unimath/uni_char.cpp
    unimath/uni_font.cpp
    unimath/uni_symbol.cpp
    # graphic folder
    graphic/font_style.cpp
    graphic/graphic_basic.cpp
    graphic/graphic.cpp
    # render folder
    render/render.cpp
    render/builder.cpp

    microtex.cpp
)

message(STATUS "Have cwrapper: ${HAVE_CWRAPPER}")
if (DEFINED HAVE_CWRAPPER AND HAVE_CWRAPPER)
    set(
        _SRCS
        ${_SRCS}
        wrapper/callback.cpp
        wrapper/graphic_wrapper.cpp
        wrapper/cwrapper.cpp
        wrapper/byte_seq.cpp
    )
    target_compile_definitions(microtex PUBLIC -DHAVE_CWRAPPER)
endif ()

# source files
target_sources(microtex PRIVATE ${_SRCS})
target_include_directories(
    microtex PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:include/microtex>
)

###################
# compile options #
###################

# compile option: if debug graphics
option(GRAPHICS_DEBUG "If enable graphics debug" ON)
if (GRAPHICS_DEBUG)
    target_compile_definitions(microtex PRIVATE -DGRAPHICS_DEBUG)
endif ()

# [ADAPT_IMGUI_BUNDLE]
# Renamed from `_HAVE_AUTO_FONT_FIND` to a proper option, and fixed the
# typo on the variable expansion (was `${_HAVE_AUTO_FONT_FIND_}` with a
# trailing underscore, which silently expanded to empty).
# When OFF, microtex skips the std::filesystem / boost::filesystem
# dependency check, useful for consumers that supply font paths
# explicitly (e.g. ImGui Bundle's imgui_microtex wrapper).
# This option is exported to users via 'microtexconfig.h'.
option(MICROTEX_HAVE_AUTO_FONT_FIND "Enable auto font find via std::filesystem (default ON)" ON)
# Keep upstream macro name for consumption inside the lib + the generated config header.
set(HAVE_AUTO_FONT_FIND ${MICROTEX_HAVE_AUTO_FONT_FIND})

macro(CHECK_FS LIB_NAME)
    if (EXISTS "${CMAKE_SOURCE_DIR}/test/has_filesystem.cpp")
        if (${LIB_NAME} STREQUAL "none")
            try_compile(
                _HAS_FS
                "${CMAKE_BINARY_DIR}/temp"
                "${CMAKE_SOURCE_DIR}/test/has_filesystem.cpp"
                CMAKE_FLAGS -DCMAKE_CXX_STANDARD=17 -DCMAKE_CXX_STANDARD_REQUIRED=ON
            )
        else ()
            try_compile(
                _HAS_FS
                "${CMAKE_BINARY_DIR}/temp"
                "${CMAKE_SOURCE_DIR}/test/has_filesystem.cpp"
                CMAKE_FLAGS -DCMAKE_CXX_STANDARD=17 -DCMAKE_CXX_STANDARD_REQUIRED=ON
                LINK_LIBRARIES ${LIB_NAME}
            )
        endif ()
        set(_HAS_STD_FS ${_HAS_FS})
    endif ()
endmacro()

if (HAVE_AUTO_FONT_FIND)
    # iterate over the preset std c++ filesystem libraries
    # `none` means there is no filesystem library to link
    foreach (LIB IN ITEMS none;stdc++fs;c++fs;c++experimental)
        check_fs(${LIB})
        message(STATUS "Test for: ${LIB}, has std fs: ${_HAS_STD_FS}")
        if (_HAS_STD_FS)
            # special case: use the experimental filesystem
            if (${LIB} STREQUAL "c++experimental")
                target_compile_definitions(microtex PRIVATE -DUSE_EXPERIMENTAL_FILESYSTEM)
            endif ()
            if (NOT ${LIB} STREQUAL "none")
                target_link_libraries(microtex ${LIB})
            endif ()
            break()
        endif ()
    endforeach ()
    # oops... no std filesystem was found, use boost instead
    if (NOT _HAS_STD_FS)
        message(STATUS "No std c++ filesystem was found, use boost filesystem instead")
        find_package(Boost REQUIRED COMPONENTS filesystem)
        target_link_libraries(microtex PRIVATE Boost::filesystem)
        target_compile_definitions(microtex PRIVATE USE_BOOST_FILESYSTEM)
    endif ()
endif ()

# [ADAPT_IMGUI_BUNDLE]
# Renamed from `_GLYPH_RENDER_TYPE` / `GLYPH_RENDER_TYPE` to a proper
# documented cache variable. Values:
#   0: use path AND typeface (default, full feature set)
#   1: use path only
#   2: use typeface only (smallest, no vector path data needed)
set(MICROTEX_GLYPH_RENDER_TYPE 0 CACHE STRING
    "MicroTeX glyph render type: 0=path+typeface, 1=path only, 2=typeface only")
set_property(CACHE MICROTEX_GLYPH_RENDER_TYPE PROPERTY STRINGS 0 1 2)
target_compile_definitions(microtex PRIVATE -DGLYPH_RENDER_TYPE=${MICROTEX_GLYPH_RENDER_TYPE})

configure_file(microtexconfig.h.in microtexconfig.h @ONLY)
target_include_directories(microtex PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)

################
# installation #
################
if (_HAS_MICROTEX_INSTALL)
    microtex_install_target(microtex)
    microtex_install_headers(
        microtex
        HEADERS ${CMAKE_CURRENT_BINARY_DIR}/microtexconfig.h microtexexport.h microtex.h microtexapi.h
    )
    microtex_install_headers(microtex PREFIX utils HEADERS utils/types.h)
    microtex_install_headers(microtex PREFIX render HEADERS render/render.h)
    microtex_install_headers(microtex PREFIX unimath HEADERS unimath/font_src.h unimath/font_meta.h)
    microtex_install_headers(
        microtex
        PREFIX graphic
        HEADERS graphic/graphic.h graphic/font_style.h graphic/graphic_basic.h
    )
    if (DEFINED HAVE_CWRAPPER AND HAVE_CWRAPPER)
        microtex_install_headers(
            microtex
            PREFIX wrapper
            HEADERS wrapper/callback.h wrapper/cwrapper.h
        )
    endif ()
endif ()
