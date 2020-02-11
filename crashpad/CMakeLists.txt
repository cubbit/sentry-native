cmake_minimum_required(VERSION 3.0)
project(crashpad LANGUAGES ASM C CXX)

if(WIN32)
	enable_language(ASM_MASM)
endif()

add_subdirectory(zlib)

set(
    mc_sources
    mini_chromium/base/debug/alias.cc
    mini_chromium/base/files/file_path.cc
    mini_chromium/base/files/scoped_file.cc
    mini_chromium/base/logging.cc
    mini_chromium/base/process/memory.cc
    mini_chromium/base/rand_util.cc
    mini_chromium/base/strings/string16.cc
    mini_chromium/base/strings/string_number_conversions.cc
    mini_chromium/base/strings/stringprintf.cc
    mini_chromium/base/strings/utf_string_conversion_utils.cc
    mini_chromium/base/strings/utf_string_conversions.cc
    mini_chromium/base/synchronization/lock.cc
    mini_chromium/base/third_party/icu/icu_utf.cc
    mini_chromium/base/threading/thread_local_storage.cc
)

if(WIN32)
  list(
      APPEND
      mc_sources
      mini_chromium/base/strings/string_util_win.cc
      mini_chromium/base/synchronization/lock_impl_win.cc
      mini_chromium/base/threading/thread_local_storage_win.cc
      mini_chromium/base/process/process_metrics_win.cc
  )
else()
  list(
      APPEND
      mc_sources
      mini_chromium/base/files/file_util_posix.cc
      mini_chromium/base/posix/safe_strerror.cc
      mini_chromium/base/synchronization/condition_variable_posix.cc
      mini_chromium/base/synchronization/lock_impl_posix.cc
      mini_chromium/base/threading/thread_local_storage_posix.cc
      mini_chromium/base/process/process_metrics_posix.cc
  )
endif()

if(APPLE)
  list(
      APPEND
      mc_sources
      mini_chromium/base/mac/close_nocancel.cc
      mini_chromium/base/mac/foundation_util.mm
      mini_chromium/base/mac/mach_logging.cc
      mini_chromium/base/mac/scoped_mach_port.cc
      mini_chromium/base/mac/scoped_mach_vm.cc
      mini_chromium/base/mac/scoped_nsautorelease_pool.mm
      mini_chromium/base/strings/sys_string_conversions_mac.mm
  )
endif()

add_library(mc ${mc_sources})
target_include_directories(
    mc PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/mini_chromium>"
)

if(MSVC)
  target_compile_definitions(
      mc
      PUBLIC
      "_HAS_EXCEPTIONS=0"
      "_CRT_SECURE_NO_WARNINGS"
      "NOMINMAX"
      "WIN32_LEAN_AND_MEAN"
      "_UNICODE"
      "UNICODE"
  )
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
endif()

string(COMPARE EQUAL "${CMAKE_SYSTEM_NAME}" "Linux" is_linux)
if(is_linux)
  target_compile_definitions(mc PUBLIC "_FILE_OFFSET_BITS=64")
endif()

if(APPLE)
  target_link_libraries(mc PUBLIC "-framework ApplicationServices")
  target_link_libraries(mc PUBLIC "-framework CoreFoundation")
  target_link_libraries(mc PUBLIC "-framework Foundation")
  target_link_libraries(mc PUBLIC "-framework IOKit")
  target_link_libraries(mc PUBLIC "-framework Security")
endif()


set(
	client_sources
	crashpad/client/annotation.cc
	crashpad/client/annotation_list.cc
	crashpad/client/crash_report_database.cc
	crashpad/client/crashpad_info.cc
	crashpad/client/prune_crash_reports.cc
	crashpad/client/settings.cc
)

if(APPLE)
	list(
		APPEND
		client_sources
		crashpad/client/crash_report_database_mac.mm
		crashpad/client/crashpad_client_mac.cc
		crashpad/client/simulate_crash_mac.cc
	)
endif()

if (UNIX AND NOT APPLE)
	list(
		APPEND
		client_sources
		crashpad/client/crashpad_client_linux.cc
		crashpad/client/client_argv_handling.cc
		crashpad/client/crashpad_info_note.S
	)
endif()

if(WIN32)
	list(
		APPEND
		client_sources
		crashpad/client/crash_report_database_win.cc
		crashpad/client/crashpad_client_win.cc
	)
endif()

add_definitions(
    -DCRASHPAD_ZLIB_SOURCE_EXTERNAL
)
if(WIN32)
	add_definitions(
		-D_HAS_EXCEPTIONS=0
		-D_CRT_SECURE_NO_WARNINGS
		-DNOMINMAX
		-DWIN32_LEAN_AND_MEAN
		-D_UNICODE
		-DUNICODE
		-DNTDDI_VERSION=NTDDI_VISTA
		-D_WIN32_WINNT=_WIN32_WINNT_VISTA
		-DWINVER=_WIN32_WINNT_VISTA
		)
endif()


include_directories(
    crashpad
    crashpad/client
    mini_chromium
    zlib
)

if(WIN32)
    include_directories(crashpad/compat/win)
else()
    include_directories(crashpad/compat/non_win)
endif()

set(
	handler_sources
	crashpad/handler/crash_report_upload_thread.cc
	crashpad/handler/main.cc
	crashpad/handler/handler_main.cc
	crashpad/handler/minidump_to_upload_parameters.cc
	crashpad/handler/prune_crash_reports_thread.cc
	crashpad/handler/user_stream_data_source.cc
)

if(APPLE)
	list(
		APPEND
		handler_sources
		crashpad/handler/mac/crash_report_exception_handler.cc
		crashpad/handler/mac/exception_handler_server.cc
		crashpad/handler/mac/file_limit_annotation.cc
		)
endif()

if(UNIX AND NOT APPLE)
	list(
		APPEND
		handler_sources
		crashpad/handler/linux/crash_report_exception_handler.cc
		crashpad/handler/linux/exception_handler_server.cc
		)
endif()

if(WIN32)
	list(
		APPEND
		handler_sources
		crashpad/handler/win/crash_report_exception_handler.cc
  )
endif()

set(
	minidump_sources
	crashpad/minidump/minidump_annotation_writer.cc
	crashpad/minidump/minidump_byte_array_writer.cc
	crashpad/minidump/minidump_context_writer.cc
	crashpad/minidump/minidump_crashpad_info_writer.cc
	crashpad/minidump/minidump_exception_writer.cc
	crashpad/minidump/minidump_extensions.cc
	crashpad/minidump/minidump_file_writer.cc
	crashpad/minidump/minidump_handle_writer.cc
	crashpad/minidump/minidump_memory_info_writer.cc
	crashpad/minidump/minidump_memory_writer.cc
	crashpad/minidump/minidump_misc_info_writer.cc
	crashpad/minidump/minidump_module_crashpad_info_writer.cc
	crashpad/minidump/minidump_module_writer.cc
	crashpad/minidump/minidump_rva_list_writer.cc
	crashpad/minidump/minidump_simple_string_dictionary_writer.cc
	crashpad/minidump/minidump_stream_writer.cc
	crashpad/minidump/minidump_string_writer.cc
	crashpad/minidump/minidump_system_info_writer.cc
	crashpad/minidump/minidump_thread_id_map.cc
	crashpad/minidump/minidump_thread_writer.cc
	crashpad/minidump/minidump_unloaded_module_writer.cc
	crashpad/minidump/minidump_user_extension_stream_data_source.cc
	crashpad/minidump/minidump_user_stream_writer.cc
	crashpad/minidump/minidump_writable.cc
	crashpad/minidump/minidump_writer_util.cc
)

set(
	util_sources
	crashpad/util/file/delimited_file_reader.cc
	crashpad/util/file/file_io.cc
	crashpad/util/file/file_reader.cc
	crashpad/util/file/file_seeker.cc
	crashpad/util/file/file_writer.cc
	crashpad/util/file/scoped_remove_file.cc
	crashpad/util/file/string_file.cc
	crashpad/util/misc/initialization_state_dcheck.cc
	crashpad/util/misc/lexing.cc
	crashpad/util/misc/metrics.cc
	crashpad/util/misc/pdb_structures.cc
	crashpad/util/misc/random_string.cc
	crashpad/util/misc/range_set.cc
	crashpad/util/misc/reinterpret_bytes.cc
	crashpad/util/misc/scoped_forbid_return.cc
	crashpad/util/misc/time.cc
	crashpad/util/misc/uuid.cc
	crashpad/util/misc/zlib.cc
	crashpad/util/net/http_body.cc
	crashpad/util/net/http_body_gzip.cc
	crashpad/util/net/http_multipart_builder.cc
	crashpad/util/net/http_transport.cc
	crashpad/util/net/url.cc
	crashpad/util/numeric/checked_address_range.cc
	crashpad/util/stdlib/aligned_allocator.cc
	crashpad/util/stdlib/string_number_conversion.cc
	crashpad/util/stdlib/strlcpy.cc
	crashpad/util/stdlib/strnlen.cc
	crashpad/util/string/split_string.cc
	crashpad/util/thread/thread.cc
	crashpad/util/thread/thread_log_messages.cc
    crashpad/util/thread/worker_thread.cc
    crashpad/util/process/process_memory.cc
    crashpad/util/process/process_memory_range.cc
)

if(UNIX)
	list(
		APPEND
		util_sources
		crashpad/util/file/directory_reader_posix.cc
		crashpad/util/file/file_io_posix.cc
		crashpad/util/file/filesystem_posix.cc
		crashpad/util/misc/clock_posix.cc
		crashpad/util/posix/close_stdio.cc
		crashpad/util/posix/close_multiple.cc
		crashpad/util/posix/double_fork_and_exec.cc
		crashpad/util/posix/drop_privileges.cc
		crashpad/util/posix/scoped_dir.cc
		crashpad/util/posix/scoped_mmap.cc
		crashpad/util/posix/signals.cc
		crashpad/util/posix/symbolic_constants_posix.cc
		crashpad/util/synchronization/semaphore_posix.cc
		crashpad/util/thread/thread_posix.cc
  )
endif()

if(APPLE)
	list(
		APPEND
		util_sources
		crashpad/util/mac/launchd.mm
		crashpad/util/mac/mac_util.cc
		crashpad/util/mac/service_management.cc
		crashpad/util/mac/xattr.cc
		crashpad/util/mach/child_port_handshake.cc
		crashpad/util/mach/child_port_server.cc
		crashpad/util/mach/composite_mach_message_server.cc
		crashpad/util/mach/exc_client_variants.cc
		crashpad/util/mach/exc_server_variants.cc
		crashpad/util/mach/exception_behaviors.cc
		crashpad/util/mach/exception_ports.cc
		crashpad/util/mach/exception_types.cc
		crashpad/util/mach/mach_extensions.cc
		crashpad/util/mach/mach_message.cc
		crashpad/util/mach/mach_message_server.cc
		crashpad/util/mach/notify_server.cc
		crashpad/util/mach/scoped_task_suspend.cc
		crashpad/util/mach/symbolic_constants_mach.cc
		crashpad/util/mach/task_for_pid.cc
		crashpad/util/misc/capture_context_mac.S
		crashpad/util/misc/clock_mac.cc
		crashpad/util/misc/paths_mac.cc
		crashpad/util/net/http_transport_mac.mm
		crashpad/util/posix/process_info_mac.cc
		crashpad/util/process/process_memory_mac.cc
		crashpad/util/synchronization/semaphore_mac.cc
	)
endif()

if(UNIX AND NOT APPLE)
	list(
		APPEND
		util_sources
		crashpad/util/net/http_transport_socket.cc
		crashpad/util/linux/auxiliary_vector.cc
		crashpad/util/linux/direct_ptrace_connection.cc
		crashpad/util/linux/exception_handler_client.cc
		crashpad/util/linux/exception_handler_protocol.cc
		crashpad/util/linux/memory_map.cc
		crashpad/util/linux/proc_stat_reader.cc
		crashpad/util/linux/ptrace_broker.cc
		crashpad/util/linux/ptrace_client.cc
		crashpad/util/linux/ptracer.cc
		crashpad/util/linux/scoped_pr_set_ptracer.cc
		crashpad/util/linux/scoped_ptrace_attach.cc
		crashpad/util/linux/thread_info.cc
		crashpad/util/misc/capture_context_linux.S
		crashpad/util/misc/paths_linux.cc
		crashpad/util/posix/process_info_linux.cc
		crashpad/util/process/process_memory_linux.cc
	)
endif()

if(WIN32)
	list(
		APPEND
		util_sources
		crashpad/util/file/directory_reader_win.cc
		crashpad/util/file/file_io_win.cc
		crashpad/util/file/filesystem_win.cc
		crashpad/util/misc/clock_win.cc
		crashpad/util/misc/paths_win.cc
		crashpad/util/misc/time_win.cc
		crashpad/util/net/http_transport_win.cc
		crashpad/util/synchronization/semaphore_win.cc
		crashpad/util/thread/thread_win.cc
		crashpad/util/win/command_line.cc
		crashpad/util/win/critical_section_with_debug_info.cc
		crashpad/util/win/exception_handler_server.cc
		crashpad/util/win/get_function.cc
		crashpad/util/win/get_module_information.cc
		crashpad/util/win/handle.cc
		crashpad/util/win/initial_client_data.cc
		crashpad/util/win/module_version.cc
		crashpad/util/win/nt_internals.cc
		crashpad/util/win/ntstatus_logging.cc
		crashpad/util/win/process_info.cc
		crashpad/util/win/registration_protocol_win.cc
		crashpad/util/win/scoped_handle.cc
		crashpad/util/win/scoped_local_alloc.cc
		crashpad/util/win/scoped_process_suspend.cc
		crashpad/util/win/scoped_set_event.cc
		crashpad/util/win/session_end_watcher.cc
		crashpad/util/misc/capture_context_win.asm
		crashpad/util/win/safe_terminate_process.asm
		crashpad/util/process/process_memory_win.cc
	)
endif()

set(
	snapshot_sources
	crashpad/snapshot/annotation_snapshot.cc
	crashpad/snapshot/capture_memory.cc
	crashpad/snapshot/cpu_context.cc
	crashpad/snapshot/crashpad_info_client_options.cc
	crashpad/snapshot/handle_snapshot.cc
	crashpad/snapshot/memory_snapshot.cc
	crashpad/snapshot/minidump/exception_snapshot_minidump.cc
	crashpad/snapshot/minidump/memory_snapshot_minidump.cc
	crashpad/snapshot/minidump/minidump_annotation_reader.cc
	crashpad/snapshot/minidump/minidump_context_converter.cc
	crashpad/snapshot/minidump/minidump_simple_string_dictionary_reader.cc
	crashpad/snapshot/minidump/minidump_string_list_reader.cc
	crashpad/snapshot/minidump/minidump_string_reader.cc
	crashpad/snapshot/minidump/module_snapshot_minidump.cc
	crashpad/snapshot/minidump/process_snapshot_minidump.cc
	crashpad/snapshot/minidump/system_snapshot_minidump.cc
	crashpad/snapshot/minidump/thread_snapshot_minidump.cc
	crashpad/snapshot/unloaded_module_snapshot.cc
	crashpad/snapshot/x86/cpuid_reader.cc
	crashpad/snapshot/crashpad_types/crashpad_info_reader.cc
	crashpad/snapshot/crashpad_types/image_annotation_reader.cc
)


if(UNIX)
	list(
		APPEND
		snapshot_sources
		crashpad/snapshot/posix/timezone.cc
	)
endif()

if(APPLE)
	list(
		APPEND
		snapshot_sources
		crashpad/snapshot/mac/cpu_context_mac.cc
		crashpad/snapshot/mac/exception_snapshot_mac.cc
		crashpad/snapshot/mac/mach_o_image_annotations_reader.cc
		crashpad/snapshot/mac/mach_o_image_reader.cc
		crashpad/snapshot/mac/mach_o_image_segment_reader.cc
		crashpad/snapshot/mac/mach_o_image_symbol_table_reader.cc
		crashpad/snapshot/mac/module_snapshot_mac.cc
		crashpad/snapshot/mac/process_reader_mac.cc
		crashpad/snapshot/mac/process_snapshot_mac.cc
		crashpad/snapshot/mac/process_types.cc
		crashpad/snapshot/mac/process_types/all.proctype
		crashpad/snapshot/mac/process_types/annotation.proctype
		crashpad/snapshot/mac/process_types/crashpad_info.proctype
		crashpad/snapshot/mac/process_types/crashreporterclient.proctype
		crashpad/snapshot/mac/process_types/custom.cc
		crashpad/snapshot/mac/process_types/dyld_images.proctype
		crashpad/snapshot/mac/process_types/loader.proctype
		crashpad/snapshot/mac/process_types/nlist.proctype
		crashpad/snapshot/mac/system_snapshot_mac.cc
		crashpad/snapshot/mac/thread_snapshot_mac.cc
	)
endif()

if(UNIX AND NOT APPLE)
	list(
		APPEND
		snapshot_sources
		crashpad/snapshot/linux/cpu_context_linux.cc
		crashpad/snapshot/linux/debug_rendezvous.cc
		crashpad/snapshot/linux/exception_snapshot_linux.cc
		crashpad/snapshot/linux/process_reader_linux.cc
		crashpad/snapshot/linux/process_snapshot_linux.cc
		crashpad/snapshot/linux/system_snapshot_linux.cc
		crashpad/snapshot/linux/thread_snapshot_linux.cc
		crashpad/snapshot/sanitized/memory_snapshot_sanitized.cc
		crashpad/snapshot/sanitized/module_snapshot_sanitized.cc
		crashpad/snapshot/sanitized/process_snapshot_sanitized.cc
		crashpad/snapshot/sanitized/sanitization_information.cc
		crashpad/snapshot/sanitized/thread_snapshot_sanitized.cc
		crashpad/snapshot/elf/elf_dynamic_array_reader.cc
		crashpad/snapshot/elf/elf_image_reader.cc
		crashpad/snapshot/elf/elf_symbol_table_reader.cc
		crashpad/snapshot/elf/module_snapshot_elf.cc
	)
endif()

if(WIN32)
	list(
		APPEND
		snapshot_sources
		crashpad/snapshot/win/capture_memory_delegate_win.cc
		crashpad/snapshot/win/cpu_context_win.cc
		crashpad/snapshot/win/exception_snapshot_win.cc
		crashpad/snapshot/win/memory_map_region_snapshot_win.cc
		crashpad/snapshot/win/module_snapshot_win.cc
		crashpad/snapshot/win/pe_image_annotations_reader.cc
		crashpad/snapshot/win/pe_image_reader.cc
		crashpad/snapshot/win/pe_image_resource_reader.cc
		crashpad/snapshot/win/process_reader_win.cc
		crashpad/snapshot/win/process_snapshot_win.cc
		crashpad/snapshot/win/process_subrange_reader.cc
		crashpad/snapshot/win/system_snapshot_win.cc
		crashpad/snapshot/win/thread_snapshot_win.cc
	)
endif()

set(
	tools_sources
	crashpad/tools/tool_support.cc
)

if(WIN32)
	list(
		APPEND
		tools_sources
		crashpad/third_party/getopt/getopt.cc
	)
endif()

if(APPLE)
  set(def_relative_files "exc.defs" "mach_exc.defs" "notify.defs")
  set(input_files "")
  foreach(x ${def_relative_files})
    # CMAKE_OSX_SYSROOT may be empty (e.g. for Makefile generators),
    # in this case files will be taken from root.
    set(full_path "${CMAKE_OSX_SYSROOT}/usr/include/mach/${x}")
    if(NOT EXISTS "${full_path}")
      message(FATAL_ERROR "File not found: ${full_path}")
    endif()
    list(APPEND input_files "${full_path}")
  endforeach()
  list(APPEND input_files "${CMAKE_CURRENT_LIST_DIR}/crashpad/util/mach/child_port.defs")

  find_package(PythonInterp 2.7 REQUIRED)

  set(output_dir "${CMAKE_CURRENT_BINARY_DIR}/crashpad/util/mach")
  file(MAKE_DIRECTORY "${output_dir}")

  # Create generate rule for each input file. Add each generated output
  # as a source to the target.
  foreach(input ${input_files})
    get_filename_component(name_we "${input}" NAME_WE)
    set(output_files "")
    foreach(suffix "User.c" "Server.c" ".h" "Server.h")
      list(APPEND output_files "${output_dir}/${name_we}${suffix}")
    endforeach()
    add_custom_command(
        OUTPUT
        ${output_files}
        COMMAND
        "${PYTHON_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/crashpad/util/mach/mig.py" "${input}" ${output_files}
        DEPENDS
        "${CMAKE_CURRENT_SOURCE_DIR}/crashpad/util/mach/mig.py" "${input}"
    )
    list(APPEND util_sources ${output_files})
  endforeach()

  include_directories("${CMAKE_CURRENT_BINARY_DIR}/crashpad")
endif()


add_library(crashpad_client ${client_sources})
add_library(crashpad_minidump ${minidump_sources})
add_library(crashpad_util ${util_sources})
add_library(crashpad_snapshot ${snapshot_sources})
add_library(crashpad_tools ${tools_sources})
add_executable(crashpad_handler ${handler_sources})

target_link_libraries(crashpad_handler
	crashpad_client
	crashpad_minidump
	crashpad_util
	crashpad_snapshot
    crashpad_tools
    bsm
	zlibstatic
	mc
)

if(WIN32)
	target_link_libraries(crashpad_handler
		winhttp
		Version
		PowrProf
	)
endif()

install(
	TARGETS crashpad_handler crashpad_client crashpad_util
	LIBRARY DESTINATION "lib"
	ARCHIVE DESTINATION "lib"
	RUNTIME DESTINATION "bin"
)

INSTALL(FILES
		crashpad/util/file/file_io.h
		crashpad/util/file/file_reader.h
		crashpad/util/file/file_seeker.h
		crashpad/util/file/file_writer.h
		crashpad/util/file/scoped_remove_file.h
		DESTINATION "include/util/file"
)

INSTALL(FILES
		crashpad/util/misc/initialization_state.h
		crashpad/util/misc/capture_context.h
		crashpad/util/misc/metrics.h
		crashpad/util/misc/uuid.h
		DESTINATION "include/util/misc"
)

INSTALL(FILES
		crashpad/util/win/scoped_handle.h
		DESTINATION "include/util/win"
)

INSTALL(FILES
		crashpad/client/crash_report_database.h
		crashpad/client/crashpad_client.h
		crashpad/client/settings.h
		DESTINATION "include/client"
)
