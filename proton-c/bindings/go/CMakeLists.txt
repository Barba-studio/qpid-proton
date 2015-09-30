#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

set(GO_BUILD_FLAGS "" CACHE STRING "Flags for 'go build'")
set(GO_TEST_FLAGS "-v -race" CACHE STRING "Flags for 'go test'")

separate_arguments(GO_BUILD_FLAGS)
separate_arguments(GO_TEST_FLAGS)

if (BUILD_GO)
  # Following are CACHE INTERNAL so examples/CMakeLists.txt can see them.
  set(GO_ENV ${env_py} --
    "GOPATH=${CMAKE_CURRENT_SOURCE_DIR}"
    "CGO_CFLAGS=-I${CMAKE_SOURCE_DIR}/proton-c/include"
    "CGO_LDFLAGS=-L${CMAKE_BINARY_DIR}/proton-c"
    ${GO_EXE} CACHE INTERNAL "Run go with environment set")

  # Set rpath so test and example executables will use the proton library from this build.
  execute_process(COMMAND ${GO_EXE} version OUTPUT_VARIABLE go_out)
  if (go_out MATCHES "gccgo")
    set(GO_RPATH_FLAGS -gccgoflags "-Wl,-rpath=${CMAKE_BINARY_DIR}/proton-c")
  else()
    set(GO_RPATH_FLAGS -ldflags "-r ${CMAKE_BINARY_DIR}/proton-c")
  endif()

  set(GO_BUILD ${GO_ENV} build ${GO_BUILD_FLAGS} ${GO_RPATH_FLAGS} CACHE INTERNAL "Run go build")
  set(GO_INSTALL ${GO_ENV} install ${GO_BUILD_FLAGS} CACHE INTERNAL "Run go install")
  set(GO_TEST ${GO_ENV} test ${GO_BUILD_FLAGS} ${GO_RPATH_FLAGS} ${GO_TEST_FLAGS} CACHE INTERNAL "Run go test")

  # Install packages in the source tree, go tools aren't friendly otherwise.
  # All build output goes in git-ignored pkg or bin subdirectories.
  set(qgo "qpid.apache.org/proton")
  set(packages ${qgo} ${qgo}/amqp ${qgo}/concurrent ${qgo}/internal)
  add_custom_target(go-packages ALL
    COMMAND ${GO_INSTALL} ${packages}
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    DEPENDS qpid-proton)

  add_test(
    NAME go_test_packages
    COMMAND ${GO_TEST} ${packages}
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}")

  list(APPEND ADDITIONAL_MAKE_CLEAN_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/pkg
    ${CMAKE_CURRENT_SOURCE_DIR}/bin)

  set (GO_INSTALL_DIR ${SHARE_INSTALL_DIR}/gocode/src CACHE PATH "Installation directory for Go code")
  mark_as_advanced (GO_INSTALL_DIR)
  install(DIRECTORY src/qpid.apache.org DESTINATION ${GO_INSTALL_DIR} COMPONENT Go)

endif(BUILD_GO)