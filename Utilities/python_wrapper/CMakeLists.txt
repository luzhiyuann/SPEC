list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
find_package(PythonExtensions REQUIRED)
find_package(NumPy REQUIRED)
find_package(F2PY REQUIRED)
find_package(F90Wrap REQUIRED)

# Fortran preprocessing compiler 
if(CMAKE_Fortran_COMPILER_ID STREQUAL Intel)
	set(FPP_COMPILER fpp)
    set(FPP_COMPILE_FLAGS "")
else()
	set(FPP_COMPILER ${CMAKE_Fortran_COMPILER})
    set(FPP_COMPILE_FLAGS -E -cpp)
endif()

get_directory_property(COMP_DEFS COMPILE_DEFINITIONS)
message(STATUS "Compile definitions for preprocessor are ${COMP_DEFS}")
string(REPLACE ";" " " COMP_DEF_STR "${COMPILE_DEFINITIONS}")

	
function(preprocess_fortran outvar)
    message(STATUS "preprocess_fortran arguments: ${outvar}, followed by ${ARGN}")
    set(srcs)
    foreach(f ${ARGN})
        # is it a Fortran file?
        if(f MATCHES "\\.[Ff](9[05])?")
          	message(STATUS "Got fortran file: ${f}")
          	# construct output filename
          	if(NOT IS_ABSOLUTE "${f}")
            	get_filename_component(f "${f}" ABSOLUTE)
       		endif()
        	file(RELATIVE_PATH r "${CMAKE_CURRENT_SOURCE_DIR}" "${f}")
        	get_filename_component(e "${r}" EXT)
        	get_filename_component(n "${r}" NAME_WE)
        	get_filename_component(p "${r}" PATH)
        	set(of "${CMAKE_CURRENT_BINARY_DIR}/${n}_fpp${e}")
        	message(STATUS "Output name: ${of}")
        	# preprocess the thing
        	if (CMAKE_Fortran_COMPILER_ID STREQUAL Intel)
				add_custom_command(OUTPUT "${of}"
					COMMAND ${FPP_COMPILER} ${FPP_COMPILE_FLAGS} ${COMP_DEF_STR} "${f}" "${of}"
					IMPLICIT_DEPENDS Fortran "${f}"
					COMMENT "Preprocessing ${f}"
					VERBATIM
				)
        	else()
				add_custom_command(OUTPUT "${of}"
					COMMAND ${FPP_COMPILER} ${FPP_COMPILE_FLAGS} ${COMP_DEF_STR} "${f}" -o "${of}"
					IMPLICIT_DEPENDS Fortran "${f}"
					COMMENT "Preprocessing ${f}"
					VERBATIM
				)
        	endif()
        	list(APPEND srcs "${of}")
      		#else()
       		#  list(APPEND srcs "${f}")
     	endif()
    endforeach()
    # return the (preprocessed) sources
    set(${outvar} "${srcs}" PARENT_SCOPE)
endfunction()

#message(STATUS "fortran_src_files is ${fortran_src_files}")
preprocess_fortran(fpp_files ${fortran_src_files})
#message(STATUS "fpp_files is ${fpp_files}")

# ----------------------------------------------------------------------------
# NOTE: There is no way to identify the f90wrap---.f90 files ahead of running f90wrap
# NOTE: The files produced have no one->one relation with the source files.
# NOTE: So giving the names of f90wrap_---.f90 files manually 
#-----------------------------------------------------------------------------
set(f90wrap_output_files ${CMAKE_CURRENT_BINARY_DIR}/f90wrap_global_m_fpp.f90
                         ${CMAKE_CURRENT_BINARY_DIR}/f90wrap_inputlist_m_fpp.f90
                         ${CMAKE_CURRENT_BINARY_DIR}/f90wrap_intghs_m_fpp.f90
                         ${CMAKE_CURRENT_BINARY_DIR}/f90wrap_msphdf5_m_fpp.f90
                         ${CMAKE_CURRENT_BINARY_DIR}/f90wrap_newton_m_fpp.f90
                         ${CMAKE_CURRENT_BINARY_DIR}/f90wrap_toplevel.f90
)

set(F2PY_BUILD_DIR ${CMAKE_CURRENT_BINARY_DIR}/f2py_build)
file(MAKE_DIRECTORY ${F2PY_BUILD_DIR})

set(kind_map_file ${CMAKE_CURRENT_SOURCE_DIR}/kind_map)
set(python_mod_name spec_f90wrapped)
set(python_mod_file ${CMAKE_CURRENT_BINARY_DIR}/${python_mod_name}.py)

add_custom_target(preprocessing ALL
  DEPENDS ${fpp_files}
)

add_custom_command(OUTPUT ${python_mod_file} ${f90wrap_output_files}
    COMMAND "${F90Wrap_EXECUTABLE}" -m "${python_mod_name}" ${fpp_files} -k "${kind_map_file}"
    #IMPLICIT_DEPENDS Fortran ${fpp_files}
    DEPENDS ${fpp_files} ${kind_map_file}
    COMMENT "Executing F90Wrap for" ${fpp_files}
    VERBATIM
)

#add_custom_target("${python_mod_name}_pymod"
#  DEPENDS ${python_mod_file} ${f90wrap_output_files} 
#)

set(f2py_module_name "_${python_mod_name}")
set(generated_module_file ${CMAKE_CURRENT_BINARY_DIR}/${f2py_module_name}${PYTHON_EXTENSION_MODULE_SUFFIX})
message(STATUS "Python exten suffix expansion: ${PYTHON_EXTENSION_MODULE_SUFFIX}")
message(STATUS "f90_wrap_output_files: "  ${f90wrap_output_files})
message(STATUS "f2py_module_name: ${f2py_module_name}")
message(STATUS "generated_module_name: ${generated_module_file}")

include_directories("${NumPy_INCLUDE_DIRS}" "${F2PY_INCLUDE_DIRS}" "${CMAKE_CURRENT_BINARY_DIR}")

# NumPy's Meson f2py backend links path-like libraries before wrapper objects.
# Convert them to -L/-l arguments so static libraries are resolved in link-arg order.
function(append_f2py_link_arg outvar link_arg)
    set(current_args "${${outvar}}")

    if("${link_arg}" MATCHES "^-L.+")
        list(APPEND current_args "${link_arg}")
    elseif("${link_arg}" MATCHES "^-l(.+)")
        set(lib_name "${CMAKE_MATCH_1}")
        # f2py uses library names as Meson variable names; hyphens are invalid there.
        if("${lib_name}" MATCHES "^[A-Za-z_][A-Za-z0-9_.]*$")
            list(APPEND current_args "${link_arg}")
        endif()
    elseif(IS_ABSOLUTE "${link_arg}" AND EXISTS "${link_arg}")
        get_filename_component(lib_dir "${link_arg}" DIRECTORY)
        get_filename_component(lib_name "${link_arg}" NAME)

        if("${lib_name}" MATCHES "^lib(.+)\\.(a|so|dylib)(\\..*)?$")
            list(APPEND current_args "-L${lib_dir}" "-l${CMAKE_MATCH_1}")
        else()
            list(APPEND current_args "${link_arg}")
        endif()
    endif()

    set(${outvar} "${current_args}" PARENT_SCOPE)
endfunction()

set(F2PY_LINK_ARGS)
append_f2py_link_arg(F2PY_LINK_ARGS "${CMAKE_BINARY_DIR}/build/lib/libspec.a")
foreach(link_lib ${SPEC_LINK_LIB})
    append_f2py_link_arg(F2PY_LINK_ARGS "${link_lib}")
endforeach()

if(MPI_Fortran_FOUND)
    separate_arguments(MPI_Fortran_LINK_FLAGS_LIST UNIX_COMMAND "${MPI_Fortran_LINK_FLAGS}")
    foreach(link_arg ${MPI_Fortran_LINK_FLAGS_LIST} ${MPI_Fortran_LIBRARIES})
        append_f2py_link_arg(F2PY_LINK_ARGS "${link_arg}")
    endforeach()
elseif(CMAKE_Fortran_COMPILER MATCHES "(^|/)mpi")
    foreach(show_arg --showme:link -show --show)
        if(NOT MPI_Fortran_WRAPPER_LINK_FLAGS)
            execute_process(
                COMMAND ${CMAKE_Fortran_COMPILER} ${show_arg}
                RESULT_VARIABLE MPI_Fortran_WRAPPER_RESULT
                OUTPUT_VARIABLE MPI_Fortran_WRAPPER_LINK_FLAGS
                ERROR_QUIET
                OUTPUT_STRIP_TRAILING_WHITESPACE
            )
            if(NOT MPI_Fortran_WRAPPER_RESULT EQUAL 0)
                unset(MPI_Fortran_WRAPPER_LINK_FLAGS)
            endif()
        endif()
    endforeach()
    separate_arguments(MPI_Fortran_WRAPPER_LINK_FLAGS_LIST UNIX_COMMAND "${MPI_Fortran_WRAPPER_LINK_FLAGS}")
    foreach(link_arg ${MPI_Fortran_WRAPPER_LINK_FLAGS_LIST})
        append_f2py_link_arg(F2PY_LINK_ARGS "${link_arg}")
    endforeach()
endif()

list(REMOVE_DUPLICATES F2PY_LINK_ARGS)
message(STATUS "f2py link arguments: ${F2PY_LINK_ARGS}")

add_custom_target(${f2py_module_name} ALL
  DEPENDS ${generated_module_file} spec ${f90wrap_output_files}
)

add_custom_command(
    OUTPUT ${generated_module_file}
    COMMAND ${CMAKE_COMMAND} -E env
        FC=${CMAKE_Fortran_COMPILER}
        CC=${CMAKE_C_COMPILER}
        ${F2PY_EXECUTABLE}
        -m ${f2py_module_name} 
        --build-dir ${F2PY_BUILD_DIR}
        --f90exec=${CMAKE_Fortran_COMPILER}
        --f77exec=${CMAKE_Fortran_COMPILER}
        --f90flags="-fopenmp"
        -lgomp
        -c 
        #${SCALAPACK_LIB} ${NETCDF_F} ${NETCDF_C}
        ${f90wrap_output_files} 
        -I${CMAKE_BINARY_DIR}/build/modules/spec_modules
        -I${HDF5_Fortran_INCLUDE_DIRS}
	--verbose
        ${F2PY_LINK_ARGS}
    #IMPLICIT_DEPENDS Fortran ${f90wrap_output_files}
    DEPENDS spec ${f90wrap_output_files}
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    #VERBATIM
    COMMAND_EXPAND_LISTS
)

python_extension_module(${generated_module_file})
install(FILES ${python_mod_file} ${generated_module_file} #${CMAKE_CURRENT_SOURCE_DIR}/__init__.py
        DESTINATION Utilities/python_wrapper/spec
)

#set(PYINIT_STR "import sys\nimport os.path\nsys.path.append(os.path.dirname(__file__))\n\nfrom .spec import *\n")
#set(PYINIT_FILE ${CMAKE_CURRENT_BINARY_DIR}/__init__.py)
#FILE(WRITE ${PYINIT_FILE} ${PYINIT_STR})
#install(FILES ${python_mod_file} ${generated_module_file} ${PYINIT_FILE}
#        DESTINATION spec  # Here spec is directory location
#)
#install(TARGETS xspec spec)
#        LIBRARY DESTINATION ${CMAKE_INSTALL_DIR}/.
#        RUNTIME DESTINATION bin
#)
