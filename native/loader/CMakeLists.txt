# Stage the freshly-built OpenZen.dll into the loader's binary dir so rc.exe
# can pick it up as an RCDATA resource (embedded in the .exe).
set(EMBED_DIR "${CMAKE_CURRENT_BINARY_DIR}/embedded_dll")
add_custom_command(
    OUTPUT  "${EMBED_DIR}/OpenZen.dll"
    COMMAND ${CMAKE_COMMAND} -E make_directory "${EMBED_DIR}"
    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:OpenZen> "${EMBED_DIR}/OpenZen.dll"
    DEPENDS OpenZen
    COMMENT "Staging OpenZen.dll into loader resources"
)
add_custom_target(stage_dll_for_loader DEPENDS "${EMBED_DIR}/OpenZen.dll")

add_executable(OpenZenLoader WIN32
    src/main.cpp
    src/process_list.cpp
    src/injector.cpp
    src/embedded_dll.cpp
    src/window_title.cpp
    src/ui.cpp
    res/loader.rc
)

add_dependencies(OpenZenLoader stage_dll_for_loader)

target_include_directories(OpenZenLoader PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_SOURCE_DIR}/res
)

# Tell rc.exe to look in EMBED_DIR (where OpenZen.dll was staged) when
# resolving the RCDATA "OpenZen.dll" reference in loader.rc.
set_source_files_properties(res/loader.rc PROPERTIES
    COMPILE_FLAGS "/I\"${EMBED_DIR}\""
)

target_link_libraries(OpenZenLoader PRIVATE
    comctl32
    psapi
    shlwapi
    user32
)

set_target_properties(OpenZenLoader PROPERTIES
    OUTPUT_NAME "OpenZenLoader"
)

if(MSVC)
    # requireAdministrator: ensures the loader can OpenProcess /
    # VirtualAllocEx / CreateRemoteThread against javaw.exe even when the
    # Minecraft launcher (HMCL, MultiMC) was itself started elevated, and
    # avoids the silent injection failure that happens on locked-down user
    # accounts. The user will see one UAC consent prompt when launching
    # OpenZenLoader.exe.
    set_target_properties(OpenZenLoader PROPERTIES
        LINK_FLAGS "/SUBSYSTEM:WINDOWS /MANIFESTUAC:\"level='requireAdministrator' uiAccess='false'\""
    )
endif()
