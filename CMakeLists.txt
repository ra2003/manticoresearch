cmake_minimum_required ( VERSION 2.8 )

FOREACH ( policy CMP0012 CMP0013 CMP0014 CMP0048 CMP0074 )
	IF ( POLICY ${policy} )
		CMAKE_POLICY ( SET ${policy} NEW )
	ENDIF ()
ENDFOREACH ()

STRING(TIMESTAMP BUILD_DATE "%s")
set(ENV{SOURCE_DATE_EPOCH} ${BUILD_DATE})

# override default options which affects windows builds
# that is necessary to change /MD to /MT, and also to set libraries on win to implicitly defined
set ( CMAKE_USER_MAKE_RULES_OVERRIDE ${CMAKE_CURRENT_SOURCE_DIR}/cmake/c_flag_overrides.cmake )
set ( CMAKE_USER_MAKE_RULES_OVERRIDE_CXX ${CMAKE_CURRENT_SOURCE_DIR}/cmake/cxx_flag_overrides.cmake )

# set default build type to RelWithDebInfo (if nothing else provided from outside yet)
IF ( NOT MSVC AND NOT XCODE AND NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES )
	SET ( CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Choose the type of build." FORCE )
	set_property ( CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
			"MinSizeRel" "RelWithDebInfo" )
ENDIF ()

set ( CMAKE_DISABLE_FIND_PACKAGE_PkgConfig ON )
# runaround to configure error caused by absence of pkg_check_modules
macro( pkg_check_modules )
endmacro()

if (MSVC)
	MESSAGE(STATUS "MSVC is ${MSVC}")
endif()

PROJECT ( MANTICORE )
SET ( MANTICORE_CMAKE_DIR "${MANTICORE_SOURCE_DIR}/cmake" )
SET ( CMAKE_MODULE_PATH "${MANTICORE_SOURCE_DIR}/cmake" )
SET ( MANTICORE_MODULE_PATH "${CMAKE_MODULE_PATH}")
SET ( EXTRA_LIBRARIES )
SET ( EXTRA_SEARCHD_LIBRARIES )

option ( DISABLE_TESTING "Disable all testing (if you just build a distr)" OFF )

if (DEFINED ENV{DIAGNOSTIC})
	set(DIAGNOSTIC "$ENV{DIAGNOSTIC}")
endif()

if (DEFINED ENV{LIBS_BUNDLE})
	set(LIBS_BUNDLE "$ENV{LIBS_BUNDLE}")
endif()

if ( NOT LIBS_BUNDLE )
	get_filename_component(LIBS_BUNDLE "${MANTICORE_SOURCE_DIR}/../bundle" ABSOLUTE)
endif ()

if (NOT IS_ABSOLUTE ${LIBS_BUNDLE})
	set(LIBS_BUNDLE "${MANTICORE_BINARY_DIR}/${LIBS_BUNDLE}")
endif ()

if (DEFINED ENV{CACHEB})
	set(CACHEB "$ENV{CACHEB}")
endif()

if (NOT DEFINED CACHEB)
	get_filename_component(CACHEB "${LIBS_BUNDLE}/../cache" ABSOLUTE)
endif()

if (NOT IS_ABSOLUTE ${CACHEB})
	set(CACHEB "${MANTICORE_BINARY_DIR}/${CACHEB}")
endif ()

if (DEFINED CACHEB)
	SET ( CACHEB "${CACHEB}" CACHE PATH "Cache dir where unpacked sources and builds found." )
endif()

# simpler packages: provide -DPACK=1, and DISTR_BUILD will be set from env $DISTR, easier in dockers
if (PACK)
	set(DISTR_BUILD "$ENV{DISTR}")
endif ()


# Make release build for the pointed distr
# That will override defaults and give possibility to build
# the distribution with minimal command line
if ( DISTR_BUILD )
	# set default options to be included into build
	set ( DISTR "${DISTR_BUILD}" CACHE STRING "Choose the distr." )
	set ( CMAKE_BUILD_TYPE "RelWithDebInfo" )
	set ( WITH_MYSQL 1 CACHE BOOL "Forced Mysql" )
	set ( WITH_EXPAT 1 CACHE BOOL "Forced Expat" )
	set ( WITH_PGSQL 1 CACHE BOOL "Forced Pgsql" )
	set ( WITH_RE2 1 CACHE BOOL "Forced RE2" )
	set ( WITH_STEMMER 1 CACHE BOOL "Forced Stemmer" )
	SET ( DISABLE_TESTING 1 CACHE BOOL "Forced no testing" FORCE )
	if ( NOT "${DISTR_BUILD}" STREQUAL "macosbrew" )
	 SET ( CMAKE_INSTALL_PREFIX "/usr" CACHE FILEPATH "Forced install to /usr" FORCE )
	endif ()
	message ( STATUS "DISTR_BUILD applied.
Package will be set to ${DISTR_BUILD},
also option forced to ON values: WITH_MYSQL, WITH_EXPAT, WITH_PGSQL, WITH_RE2, WITH_STEMMER.
Also LIBS_BUNDLE is automatically set to folder 'bundle' placed one level above the sources" )
endif ()

# Collect build info which will be embedded into binaries
SET ( BANNER )
# append any text to build info
macro( cfginfo msgtext )
	list ( APPEND BANNER "${msgtext}" )
endmacro()

# append values of given var to build info
macro( memcfgvalues )
	FOREACH ( var ${ARGN} )
		if ( ${var} )
			list ( APPEND BANNER "${var}=${${var}}" )
		endif ( ${var} )
	endforeach ()
endmacro()


memcfgvalues ( CMAKE_BUILD_TYPE )
memcfgvalues ( DISTR_BUILD )

if (NOT WIN32)
	option(USE_GALERA "Enable building Galera replication library and using it" ON)
endif()

# boost for boost::context, also used in galera build
include(GetBoostContext)

if (USE_GALERA)
	include(GetGALERA)
	if (NOT HAVE_GALERA)
		set(USE_GALERA OFF CACHE FORCE)
	endif ()
endif ()

find_package(OpenSSL COMPONENTS SSL )
if (OpenSSL_FOUND)
	option(USE_SSL "Enabled building with OpenSSL" 1)
	memcfgvalues(USE_SSL)

	if (USE_SSL)
		include ( helpers )
		ac_check_func_ex ( TLS_server_method "openssl/ssl.h" OpenSSL::SSL )
		ac_check_func_ex ( TLSv1_2_method "openssl/ssl.h" OpenSSL::SSL )
		ac_check_func_ex ( TLSv1_1_server_method "openssl/ssl.h" OpenSSL::SSL )
	endif (USE_SSL)
endif (OpenSSL_FOUND)

# ctest variables
set ( DEFAULT_CTEST_CONFIGURATION_TYPE "Debug" )

if ( NOT DISABLE_TESTING )
	include ( CTest )
endif ()

set ( SPHINX_TAG "" CACHE STRING "The tag which will be added to build" )
set ( PKGSUFFIXES ) # list of suffixes which will be append to package name, will be '-'-separated
set ( PACKAGE_NAME "manticore" )
set ( COMPANY_NAME "Manticore Software LTD" )

message ( STATUS "Checking for endianess" )
include ( SphEndianess )
include ( helpers )

if ( WIN32 )
	set ( USE_WINDOWS 1 )
	set ( USE_ODBC 1 )

	# on Win we skip all the tests and just use pre-defined folder
	include ( select_library_dir )
	include ( CMakeDependentOption )
	#	expat iconv mysql pq
	if ( HAVE_expat )
		option ( WITH_EXPAT "compile with libexpat support" ON )
		if ( WITH_EXPAT )
			set ( USE_LIBEXPAT 1 )
			include_directories ( ${expat_INCLUDE} )
			CMAKE_DEPENDENT_OPTION ( DL_EXPAT "load expat library dynamically" OFF "USE_LIBEXPAT" ON )
			if ( DL_EXPAT )
				set ( DL_EXPAT 1 )
				set ( EXPAT_LIB libexpat.dll )
			else ( DL_EXPAT )
				list ( APPEND EXTRA_LIBRARIES ${expat_LIB} )
			endif ()
			install ( FILES ${expat_ROOT}/libs/libexpat.dll DESTINATION bin COMPONENT applications )
		endif ()
	endif ()

	if ( HAVE_iconv )
		option ( WITH_ICONV "compile with iconv support" ON )
		if ( WITH_ICONV )
			set ( USE_LIBICONV 1 )
			include_directories ( ${iconv_INCLUDE} )
			list ( APPEND EXTRA_LIBRARIES ${iconv_LIB} )
			install ( FILES ${iconv_ROOT}/bin/iconv.dll DESTINATION bin COMPONENT applications )
		endif ()
	endif ()

	if ( HAVE_mysql )
		option ( WITH_MYSQL "compile with mysql support" ON )
		if ( WITH_MYSQL )
			set ( USE_MYSQL 1 )
			include_directories ( ${mysql_INCLUDE} )

			CMAKE_DEPENDENT_OPTION ( DL_MYSQL "load mysql library dynamically" OFF "USE_MYSQL" ON )
			if ( DL_MYSQL )
				set ( DL_MYSQL 1 )
				set ( MYSQL_LIB libmysql.dll )
			else ( DL_MYSQL )
				list ( APPEND EXTRA_LIBRARIES ${mysql_LIB} )
			endif ()
			install ( FILES ${mysql_ROOT}/bin/libmysql.dll DESTINATION bin COMPONENT applications )
		endif ()
	endif ()

	if ( HAVE_pq )
		option ( WITH_PGSQL "compile with pq support" ON )
		if ( WITH_PGSQL )
			set ( USE_PGSQL 1 )
			include_directories ( ${pq_INCLUDE} )

			CMAKE_DEPENDENT_OPTION ( DL_PGSQL "load pgsql library dynamically" OFF "USE_PGSQL" ON )
			if ( DL_PGSQL )
				set ( DL_PGSQL 1 )
				set ( PGSQL_LIB libpq.dll )
			else ( DL_PGSQL )
				list ( APPEND EXTRA_LIBRARIES ${pq_LIB} )
			endif ()
#			LIST ( APPEND PKGSUFFIXES "pgsql" )
			IF ( CMAKE_EXE_LINKER_FLAGS MATCHES "x64" )
				install ( FILES
						${pq_ROOT}/bin/libeay32.dll
						${pq_ROOT}/bin/libiconv-2.dll
						${pq_ROOT}/bin/libintl-8.dll
						${pq_ROOT}/bin/libpq.dll
						${pq_ROOT}/bin/ssleay32.dll
						DESTINATION bin COMPONENT applications )
			else ()
				install ( FILES
						${pq_ROOT}/bin/libeay32.dll
						${pq_ROOT}/bin/libiconv.dll
						${pq_ROOT}/bin/libintl.dll
						${pq_ROOT}/bin/libpq.dll
						${pq_ROOT}/bin/ssleay32.dll
						DESTINATION bin COMPONENT applications )
			endif ()
		endif ()
	endif ()
	
	if ( USE_SSL )
		IF ( CMAKE_EXE_LINKER_FLAGS MATCHES "x64" )
			FIND_FILE ( CRYPTO_DLL
			NAMES "libeay32.dll"
			PATHS "${OPENSSL_INCLUDE_DIR}/.."
			NO_DEFAULT_PATH
			)
			FIND_FILE(OPENSSL_DLL
			NAMES "ssleay32.dll"
			PATHS "${OPENSSL_INCLUDE_DIR}/.."
			NO_DEFAULT_PATH
			)	
		else ()
			FIND_FILE ( CRYPTO_DLL
			NAMES "libcrypto-1_1.dll"
			PATHS "${OPENSSL_INCLUDE_DIR}/.."
			NO_DEFAULT_PATH
			)
			FIND_FILE ( OPENSSL_DLL
			NAMES "libssl-1_1.dll"
			PATHS "${OPENSSL_INCLUDE_DIR}/.."
			NO_DEFAULT_PATH
			)	
		endif ()
		install ( FILES "${CRYPTO_DLL}" DESTINATION bin COMPONENT applications )
		install ( FILES "${OPENSSL_DLL}" DESTINATION bin COMPONENT applications )
	endif ( USE_SSL )
	mark_as_advanced(SSL_EAY_DEBUG SSL_EAY_RELEASE LIB_EAY_DEBUG LIB_EAY_RELEASE)
	
	mark_as_advanced ( CMAKE_CONFIGURATION_TYPES )
	add_definitions ("-D_CRT_SECURE_NO_WARNINGS")
	add_definitions ( "-wd4996" "-wd4706" "-wd4244" "-wd4702" )
else ( WIN32 )

	message ( STATUS "Checking for gcc version" )
	include ( check_veryold_gcc )

	if ( "${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
		set ( HAVE_GCC_LIKE ON )
	endif ()
	mark_as_advanced ( CMAKE_CXX_COMPILER CMAKE_C_COMPILER )

	if ( HAVE_GCC_LIKE )
		set ( CFLAGS "-fno-strict-aliasing -Wall -g" )
		add_definitions ( "-D_FILE_OFFSET_BITS=64" )
		if ( CMAKE_BUILD_TYPE STREQUAL RelWithDebInfo )
			set ( CFLAGS "${CFLAGS} -O2" )
			add_definitions ( "-DNDEBUG" )
		elseif ( CMAKE_BUILD_TYPE STREQUAL Release )
			set ( CFLAGS "-fno-strict-aliasing -Wall -O3" )
			add_definitions ( "-DNDEBUG" )
		elseif ( CMAKE_BUILD_TYPE STREQUAL Debug )
			#set ( CLANGFLAGS "-Wthread-safety" ) # made a different option, off by default
			add_definitions ( "-D_DEBUG" )
		else ()
			set ( CFLAGS "${CFLAGS} -O3" )
			add_definitions ( "-DNDEBUG" )
		endif ()

		option ( STATIC_BINARY "Produce statically linked ELF" OFF)
		if ( STATIC_BINARY )
			set ( CFLAGS "${CFLAGS} -static")
			set ( HAVE_DLOPEN 0 )
			set ( WITH_ODBC 0 )
			set ( WITH_MYSQL 0 )
			set ( WITH_PGSQL 0 )
		endif()

		set ( MY_C_FLAGS "${CFLAGS}" )
		set ( CMAKE_CXX_STANDARD 14 )
		set ( MY_CXX_FLAGS "${CFLAGS} -fno-rtti" )

		# eliminate also some too noisy warnings from CLang
		if ( CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
			set ( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-deprecated-register -Wno-missing-exception-spec -Wno-implicit-exception-spec-mismatch -Wno-invalid-offsetof ${CLANGFLAGS}" )
			set ( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${CLANGFLAGS}" )
		endif ()
	endif ( HAVE_GCC_LIKE )

	find_package (Jemalloc)
	if ( JEMALLOC_FOUND )
		option ( USE_JEMALLOC "Use jemalloc allocator" OFF )
		mark_as_advanced (USE_JEMALLOC)
	endif()

	if ( USE_JEMALLOC )
		if ( JEMALLOC_FOUND )
			list ( APPEND EXTRA_LIBRARIES ${JEMALLOC_LIBRARIES} )
			memcfgvalues ( USE_JEMALLOC )
		else()
			message (FATAL_ERROR "USE_JEMALLOC requested, but no jemalloc found -- Ignore")
		endif()
	endif ()

	message ( STATUS "Storing compiler version" )
	if ( HAVE_GCC_LIKE )
		execute_process ( COMMAND "${CMAKE_CXX_COMPILER}" "-dumpversion" OUTPUT_VARIABLE gcc_ver )
		remove_crlf ( COMPILER ${gcc_ver} )
		message ( STATUS "Compiler is ${COMPILER}" )
	endif ()

	message ( STATUS "Storing system name" )
	if ( UNIX AND NOT CYGWIN )
		execute_process ( COMMAND "uname" "-s" "-m" OUTPUT_VARIABLE OS_UNAME )
		remove_crlf ( OS_UNAME ${OS_UNAME} )
	endif ( UNIX AND NOT CYGWIN )

	message ( STATUS "Looking for threads library" )
	find_package ( Threads REQUIRED )
	list ( APPEND EXTRA_LIBRARIES ${CMAKE_THREAD_LIBS_INIT} )
	ac_check_func_ex ( "pthread_mutex_timedlock" "" "${CMAKE_THREAD_LIBS_INIT}" )
	ac_check_func_ex ( "pthread_cond_timedwait" "" "${CMAKE_THREAD_LIBS_INIT}" )

	message ( STATUS "Checking for standard header files" )
	include ( ac_header_stdc )

	include ( mincore_test )

	message ( STATUS "Checking for specific headers" )
	ac_check_headers ( "execinfo.h;syslog.h;sys/eventfd.h;malloc.h" )
	ac_check_headers ( "sys/prctl.h" )

	# mb use something better. The code below is copy-pasted from automake script
	message ( STATUS "Checking for library functions" )
	ac_search_libs ( "socket" "setsockopt" EXTRA_LIBRARIES )
	ac_search_libs ( "nsl;socket;resolv" "getaddrinfo" EXTRA_LIBRARIES )
	ac_search_libs ( "m" "logf" EXTRA_LIBRARIES )
	ac_search_libs ( "dl;dld" "dlopen" EXTRA_LIBRARIES )
	ac_search_libs ( "anl" "getaddrinfo_a" EXTRA_LIBRARIES )
	if ( HAVE_DLOPEN )
		set ( HAVE_DLERROR 1 )
	endif ( HAVE_DLOPEN )

	include ( CheckFunctionExists )
	ac_check_funcs ( "pread;poll" )
	ac_check_funcs ( "backtrace;backtrace_symbols" )
	ac_check_funcs ( "mremap" )
	ac_check_funcs ( "nanosleep" )
	ac_check_funcs ( "eventfd" )
	ac_check_funcs ( "kqueue" )
	ac_check_funcs ( "pthread_getname_np")
	ac_check_funcs ( "getrlimit;setrlimit" )
	check_function_exists ( epoll_ctl HAVE_EPOLL )
	ac_search_libs ( "rt" "clock_gettime" EXTRA_LIBRARIES )

	sphinx_check_define ( "F_SETLKW" "fcntl.h" )
	message ( STATUS "Checking for atomic function" )
	include ( check_atomic_intrinsics )

	message ( STATUS "Checking for SO_REUSEPORT flag" )
	include ( check_so_reuseport_availability )

	message ( STATUS "Checking for malloc_trim availability" )
	include ( check_malloc_trim_availability )

	message ( STATUS "Checking for malloc_stats availability" )
	include ( check_malloc_stats_availability )

	message ( STATUS "Checking for PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP flag" )
	include ( check_rwlock_prefer_writer )
	include ( check_vitable_pthread_setname_np )

	message ( STATUS "Configuring the options..." )
	message ( STATUS "Option DISABLE_MEMROUTINES ${DISABLE_MEMROUTINES}" )
	option ( DISABLE_MEMROUTINES "If the build is broken because new/delete override, this option will disable the overriding in sphinx" )
	if ( DISABLE_MEMROUTINES )
		SET ( SPH_DONT_OVERRIDE_MEMROUTINES 1 )
		cfginfo ( "DISABLE_MEMROUTINES=ON" )
	endif ( DISABLE_MEMROUTINES )
	mark_as_advanced ( DISABLE_MEMROUTINES )

	message ( STATUS "Option USE_SYSLOG ${USE_SYSLOG}" )
	CMAKE_DEPENDENT_OPTION ( USE_SYSLOG "compile with possibility to use syslog for logging" ON "HAVE_SYSLOG_H" OFF )
	IF ( USE_SYSLOG )
		set ( USE_SYSLOG 1 )
	else ( USE_SYSLOG )
		unset ( USE_SYSLOG )
	ENDIF ( USE_SYSLOG )

	message ( STATUS "Option WITH_ZLIB ${WITH_ZLIB}" )
	option_menu ( ZLIB "compile with zlib support" USE_ZLIB EXTRA_LIBRARIES )
	mark_as_advanced ( WITH_ZLIB )

	message ( STATUS "Option WITH_ODBC ${WITH_ODBC}" )
	unset ( _DUMMY_LIB )
	option_menu ( ODBC "compile with UnixODBC support" USE_ODBC _DUMMY_LIB )

	message ( STATUS "ODBC found is ${ODBC_FOUND}" )

	CMAKE_DEPENDENT_OPTION ( DL_UNIXODBC "load UnixODBC library dynamically" ON "USE_ODBC;HAVE_DLOPEN" OFF )
	if ( DL_UNIXODBC )
		set ( DL_UNIXODBC 1 )
		GET_SONAME ( ${_DUMMY_LIB} UNIXODBC_LIB )
		message ( STATUS "ODBC will be loaded dynamically in runtime as ${UNIXODBC_LIB}" )
		memcfgvalues ( DL_UNIXODBC UNIXODBC_LIB )
	else ( DL_UNIXODBC )
		list ( APPEND EXTRA_LIBRARIES ${_DUMMY_LIB} )
		message ( STATUS "ODBC will be linked as ${_DUMMY_LIB}" )
	endif ( DL_UNIXODBC )

	# test for EXPAT
	message ( STATUS "Option WITH_EXPAT ${WITH_EXPAT}" )
	unset ( _DUMMY_LIB )
	option_menu ( EXPAT "compile with libexpat support" USE_LIBEXPAT _DUMMY_LIB )
	CMAKE_DEPENDENT_OPTION ( DL_EXPAT "load expat library dynamically" ON "USE_LIBEXPAT;HAVE_DLOPEN" OFF )
	if ( DL_EXPAT )
		set ( DL_EXPAT 1 )
		GET_SONAME ( ${_DUMMY_LIB} EXPAT_LIB )
		message ( STATUS "EXPAT will be loaded dynamically in runtime as ${EXPAT_LIB}" )
		memcfgvalues ( DL_EXPAT EXPAT_LIB )
	else ( DL_EXPAT )
		list ( APPEND EXTRA_LIBRARIES ${_DUMMY_LIB} )
		message ( STATUS "EXPAT will be linked as ${_DUMMY_LIB}" )
	endif ()

	# test for ICONV
	message ( STATUS "Option WITH_ICONV ${WITH_ICONV}" )
	if ( WITH_EXPAT )
		option_menu ( ICONV "compile with iconv support" USE_LIBICONV EXTRA_LIBRARIES )
		mark_as_advanced ( WITH_ICONV )
		memcfgvalues ( USE_LIBICONV )
	endif ( WITH_EXPAT )

	# test for MYSQL
	message ( STATUS "Option WITH_MYSQL ${WITH_MYSQL}" )
	option ( WITH_MYSQL "compile with MySQL support" ON )
	set ( WITH_MYSQL_INCLUDES "" CACHE PATH "path to MySQL header files" )
	set ( WITH_MYSQL_LIBS "" CACHE PATH "path to MySQL library" )
	set ( WITH_MYSQL_ROOT "" CACHE PATH "path to the MySQL bundle (where both header and library lives)" )
	mark_as_advanced ( WITH_MYSQL_INCLUDES WITH_MYSQL_LIBS WITH_MYSQL_ROOT )


	if ( WITH_MYSQL )
		include ( GetMYSQL )
	endif ( WITH_MYSQL )

	# test for POSTGRESQL
	message ( STATUS "Option WITH_PGSQL ${WITH_PGSQL}" )
	option ( WITH_PGSQL "compile with PostgreSQL support" OFF )
	set ( WITH_PGSQL_INCLUDES "" CACHE PATH "path to PostgreSQL header files" )
	set ( WITH_PGSQL_LIBS "" CACHE PATH "path to PostgreSQL library" )
	mark_as_advanced ( WITH_PGSQL_LIBS WITH_PGSQL_INCLUDES )

	if ( WITH_PGSQL )
		include ( GetPGSQL )
	endif ( WITH_PGSQL )
endif ( WIN32 )

message ( STATUS "Checking for unaligned RAM access" )
include ( check_unaligned_ram_access )

ac_check_funcs ( "strnlen" )

option ( USE_SMALLALLOC "Use 'small allocator' derived from loki lib. Bench! Glibc allocator now is fast enough!" )
if ( USE_SMALLALLOC )
	SET ( USE_SMALLALLOC 1 )
	cfginfo ( "USE_SMALLALLOC=ON" )
endif ( USE_SMALLALLOC )
mark_as_advanced ( USE_SMALLALLOC )

INCLUDE ( GNUInstallDirs )

if ( ${CMAKE_INSTALL_FULL_LOCALSTATEDIR} STREQUAL "/usr/var" )
	set ( CMAKE_INSTALL_FULL_LOCALSTATEDIR "/var")
endif()

if ( ${CMAKE_INSTALL_FULL_SYSCONFDIR} STREQUAL "/usr/etc" )
	set ( CMAKE_INSTALL_FULL_SYSCONFDIR "/etc" )
endif ()

SET ( LIBDIR ${CMAKE_INSTALL_LIBDIR} )
SET ( LOCALSTATEDIR ${CMAKE_INSTALL_FULL_LOCALSTATEDIR} )
SET ( LOCALDATADIR "${LOCALSTATEDIR}/data" )
SET ( SHAREDIR "${CMAKE_INSTALL_DATADIR}" )
if (WIN32)
	# on win name of product is already in full path
	SET ( FULL_SHARE_DIR "${CMAKE_INSTALL_FULL_DATADIR}" )
else()
	SET ( FULL_SHARE_DIR "${CMAKE_INSTALL_FULL_DATADIR}/${PACKAGE_NAME}" )
endif()
SET ( BINDIR ${CMAKE_INSTALL_BINDIR} )
SET ( BINDIRFULL ${CMAKE_INSTALL_FULL_BINDIR} )
SET ( MANDIR ${CMAKE_INSTALL_MANDIR} )
SET ( SYSCONFDIR ${CMAKE_INSTALL_FULL_SYSCONFDIR} )
SET ( DOCDIR ${CMAKE_INSTALL_DOCDIR} )
SET ( DOCDIRFULL ${CMAKE_INSTALL_FULL_DOCDIR} )

add_definitions ( "-DLOCALDATADIR=\"${LOCALDATADIR}\"" )
memcfgvalues ( LOCALDATADIR FULL_SHARE_DIR )

find_package ( BISON )
if ( BISON_FOUND )
	option ( USE_BISON "Will use bison generated grammars" ON )
endif ( BISON_FOUND )

find_package ( FLEX )
if ( FLEX_FOUND )
	option ( USE_FLEX "Use fresh flex generated lexers" ON )
endif ( FLEX_FOUND )

if ( USE_BISON )
	set ( CMAKE_GENERATED_GRAMMAR ON )
else ()
	unset ( CMAKE_GENERATED_GRAMMAR )
	include_directories ( "${CMAKE_CURRENT_SOURCE_DIR}/src" )
endif ()

if ( USE_FLEX )
	set ( CMAKE_GENERATED_LEXER ON )
else ()
	unset ( CMAKE_GENERATED_LEXER )
	include_directories ( "${CMAKE_CURRENT_SOURCE_DIR}/src" )
endif ()

# Check for RE2 build
message ( STATUS "Option WITH_RE2 ${WITH_RE2}" )
option ( WITH_RE2 "compile with re2 library support" OFF )
if ( WITH_RE2 )
	OPTION ( WITH_RE2_FORCE_STATIC "force to compile re2 from sources" OFF )
	include ( GetRE2 )
endif ( WITH_RE2 )

# Check for Libstemmer build
message ( STATUS "Option WITH_STEMMER ${WITH_STEMMER}" )
OPTION ( WITH_STEMMER "compile with stemmer support" ON )
IF ( WITH_STEMMER )
	OPTION ( WITH_STEMMER_FORCE_STATIC "force to compile stemmer from sources" OFF )
	include ( GetStemmer )
#	LIST ( APPEND PKGSUFFIXES "stemmer" )
ENDIF ( WITH_STEMMER )

# Check for ICU build
message ( STATUS "Option WITH_ICU ${WITH_ICU}" )
option ( WITH_ICU "compile with ICU library support" ON )
if ( WITH_ICU )
	OPTION ( WITH_ICU_FORCE_STATIC "force to compile ICU from sources" ON )
	include ( GetICU )
endif ( WITH_ICU )


find_package(Valgrind)

# Move this flags here to avoid configuration problems
if ( HAVE_GCC_LIKE AND CMAKE_BUILD_TYPE STREQUAL Debug )
	OPTION ( COVERAGE_TEST "Test coverage" OFF )
	if ( COVERAGE_TEST )
		set ( POSTCFLAGS "-fprofile-arcs -ftest-coverage" )
	endif ( COVERAGE_TEST )

	if ( CMAKE_CXX_COMPILER_ID MATCHES "Clang" )
		OPTION ( THREAD_SAFETY "Clang thread-safety warnings" ON )
		if ( THREAD_SAFETY )
			set ( POSTFLAGS "-Wthread-safety" )
		endif ()
	endif ()

	SET ( SANITIZER None CACHE STRING "Choose a sanitizer checker" )
	set_property ( CACHE SANITIZER PROPERTY STRINGS "None" "Thread"
			"Address" "Memory" )

	if ( SANITIZER STREQUAL "Thread" )
		set ( POSTCFLAGS "${POSTCFLAGS} -O1 -fsanitize=thread -fno-omit-frame-pointer -fPIC" )
	elseif ( SANITIZER STREQUAL "Address" )
		set ( POSTCFLAGS "${POSTCFLAGS} -O1 -fsanitize=address -fno-omit-frame-pointer" )
	elseif ( SANITIZER STREQUAL "Memory" )
		set ( POSTCFLAGS "${POSTCFLAGS} -O1 -fsanitize=memory -fno-omit-frame-pointer" )
	endif () # "None" means no action

	set ( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${POSTCFLAGS}" )
	set ( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${POSTCFLAGS}" )
endif ()

memcfgvalues ( USE_BISON USE_FLEX USE_SYSLOG WITH_EXPAT WITH_ICONV WITH_MYSQL
		WITH_MYSQL_INCLUDES WITH_MYSQL_LIBS WITH_MYSQL_ROOT WITH_ODBC WITH_PGSQL
		WITH_PGSQL_INCLUDES WITH_PGSQL_LIBS WITH_RE2
		WITH_RE2_ROOT WITH_STEMMER WITH_ZLIB GALERA_SONAME )

if ( WIN32 )
	IF ( CMAKE_EXE_LINKER_FLAGS MATCHES "x64" )
		LIST ( APPEND PKGSUFFIXES "x64" )
	else ()
		LIST ( APPEND PKGSUFFIXES "win32" )
	endif ()
endif ()

# a bit of installation stuff
if ( WIN32 )
	install ( DIRECTORY api doc contrib DESTINATION ${SHAREDIR}/doc COMPONENT doc )
	install ( FILES example.sql DESTINATION ${SHAREDIR}/doc COMPONENT doc )
	install ( DIRECTORY misc/stopwords DESTINATION ${SHAREDIR} COMPONENT doc )
	if (USE_ICU)
		install ( FILES ${ICU_DATA} DESTINATION  ${SHAREDIR}/icu COMPONENT doc)
	endif()

	install ( FILES COPYING INSTALL manticore.conf.in
			DESTINATION . COMPONENT config )
	install ( DIRECTORY DESTINATION log COMPONENT config )
	install ( DIRECTORY DESTINATION data COMPONENT config )
endif ()

IF (NOT DISTR_BUILD AND NOT WIN32 )
	file ( READ "manticore.conf.in" _MINCONF )
	string ( REPLACE "@CONFDIR@/log/searchd.pid" "@RUNDIR@/searchd.pid" _MINCONF "${_MINCONF}" )
	string ( REPLACE "@CONFDIR@/log" "@LOGDIR@" _MINCONF "${_MINCONF}" )
	file ( WRITE "${MANTICORE_BINARY_DIR}/manticore.conf.in" "${_MINCONF}" )
	unset ( _MINCONF )
	set ( CONFDIR "${CMAKE_INSTALL_FULL_LOCALSTATEDIR}/lib/manticore" )
	set ( RUNDIR "${CMAKE_INSTALL_FULL_LOCALSTATEDIR}/run/manticore" )
	set ( LOGDIR "${CMAKE_INSTALL_FULL_LOCALSTATEDIR}/log/manticore" )
	configure_file ( "manticore.conf.in" "${MANTICORE_BINARY_DIR}/manticore.conf" @ONLY )

	configure_file ( "${MANTICORE_BINARY_DIR}/manticore.conf.in" "${MANTICORE_BINARY_DIR}/manticore.conf.dist" @ONLY )
	INSTALL ( FILES ${MANTICORE_BINARY_DIR}/manticore.conf.dist
                DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/manticoresearch COMPONENT doc RENAME manticore.conf )
	install ( DIRECTORY misc/stopwords DESTINATION ${CMAKE_INSTALL_DATADIR}/${PACKAGE_NAME} COMPONENT doc)
	if (USE_ICU)
		install(FILES ${ICU_DATA} DESTINATION ${CMAKE_INSTALL_DATADIR}/${PACKAGE_NAME}/icu COMPONENT doc)
	endif ()
	install ( DIRECTORY DESTINATION ${CMAKE_INSTALL_FULL_LOCALSTATEDIR}/lib/manticore/data COMPONENT applications)
	install ( DIRECTORY DESTINATION ${CMAKE_INSTALL_FULL_LOCALSTATEDIR}/run/manticore COMPONENT applications )
	install ( DIRECTORY DESTINATION ${CMAKE_INSTALL_FULL_LOCALSTATEDIR}/log/manticore COMPONENT applications )
endif ()

configure_file ( "COPYING" "${MANTICORE_BINARY_DIR}/COPYING.txt" COPYONLY )

set ( CPACK_SOURCE_IGNORE_FILES "/\\\\.idea/" "/\\\\.git/"
		"/\\\\.svn/" "/autom4te\\\\.cache/" "/bin/" "/junk/" "/win/"
		".swp$" ".orig$" ".log$" ".deps/" )
set ( CPACK_SOURCE_GENERATOR TGZ )

set ( CPACK_PACKAGE_CONTACT "${PACKAGE_NAME} package builds <build@manticoresearch.com>" )
set ( CMAKE_PROJECT_HOMEPAGE_URL "http://manticoresearch.com" )
set ( CPACK_PACKAGE_NAME "${PACKAGE_NAME}" )
set ( CPACK_PACKAGE_VENDOR "${COMPANY_NAME}" )
set ( CPACK_RESOURCE_FILE_LICENSE "${MANTICORE_BINARY_DIR}/COPYING.txt" )
set ( CPACK_PACKAGE_DESCRIPTION_SUMMARY "Manticore full-text search server")

set ( CPACK_COMPONENT_GROUP_BIN_DISPLAY_NAME "Runtime" )

# applications (indexes,indextool etc.)
if ( SPLIT_APPS )
	set ( CPACK_COMPONENT_TOOLS_GROUP "tools" )
	set ( CPACK_COMPONENT_GROUP_TOOLS_DISPLAY_NAME "Runtime" )
	set ( CPACK_COMPONENT_TOOLS_DISPLAY_NAME "${PACKAGE_NAME} applications" )
else()
	set ( CPACK_COMPONENT_TOOLS_GROUP "bin" )
	set ( CPACK_COMPONENT_TOOLS_DISPLAY_NAME "${PACKAGE_NAME} applications" )
endif()


# searchd daemon
set ( CPACK_COMPONENT_APPLICATIONS_GROUP "bin" )
set ( CPACK_COMPONENT_GROUP_APPLICATIONS_DISPLAY_NAME "Runtime" )
set ( CPACK_COMPONENT_APPLICATIONS_DISPLAY_NAME "${PACKAGE_NAME} service" )

# configurations
set ( CPACK_COMPONENT_CONFIG_GROUP "bin" )
set ( CPACK_COMPONENT_CONFIG_DISPLAY_NAME "Configuration examples" )

# documentation files
set ( CPACK_COMPONENT_DOC_GROUP "bin" )
set ( CPACK_COMPONENT_DOC_DISPLAY_NAME "Documentation" )

# converter tool
set ( CPACK_COMPONENT_CONVERTER_GROUP "converter" )
set ( CPACK_COMPONENT_GROUP_CONVERTER_DISPLAY_NAME "Runtime" )
set ( CPACK_COMPONENT_CONVERTER_DISPLAY_NAME "CONVERTER application" )

# development files
set ( CPACK_COMPONENT_DEVEL_GROUP "devel" )
set ( CPACK_COMPONENT_DEVEL_DISPLAY_NAME "Development" )
set ( CPACK_COMPONENT_GROUP_DEVEL_DISPLAY_NAME "Development files" )



set ( CPACK_RPM_COMPONENT_INSTALL 1 )
set ( CPACK_DEB_COMPONENT_INSTALL 1 )
set ( CPACK_ARCHIVE_COMPONENT_INSTALL 1 )

include ( SetBuildType )

if ( CONFFILEDIR )
	add_definitions ( "-DSYSCONFDIR=\"${CONFFILEDIR}\"" )
	cfginfo ( "SYSCONFDIR=${CONFFILEDIR}" )
endif()

set ( CONFIGURE_FLAGS "Configured by CMake with these definitions:" )
FOREACH ( OPTION ${BANNER} )
	set ( CONFIGURE_FLAGS "${CONFIGURE_FLAGS} -D${OPTION}" )
endforeach ()

message ( STATUS "Generating config.h file" )
configure_file ( "${MANTICORE_SOURCE_DIR}/config/config_cmake.h.in"
		"${MANTICORE_BINARY_DIR}/config/config.h" ESCAPE_QUOTES )
add_definitions ( -DHAVE_CONFIG_H )
include_directories ( "${MANTICORE_BINARY_DIR}/config" )
message ( STATUS "${CONFIGURE_FLAGS}" )

set ( SPHINX_PACKAGE_SUFFIX "" )
FOREACH ( SUFFIX ${PKGSUFFIXES} )
	set ( SPHINX_PACKAGE_SUFFIX "${SPHINX_PACKAGE_SUFFIX}-${SUFFIX}" )
endforeach ()

set ( CPACK_PROJECT_CONFIG_FILE "${MANTICORE_BINARY_DIR}/config/CPackOptions.cmake" )

set ( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${MY_C_FLAGS}" )
set ( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${MY_CXX_FLAGS}" )

add_subdirectory ( src )

if ( NOT DISABLE_TESTING )
	add_subdirectory ( test )
	add_subdirectory ( api/libsphinxclient )
endif ()

include ( CPack )
