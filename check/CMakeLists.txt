include(CTest)

if (FORTRAN AND NOT BUILD_EXTRA_UNIT_ONLY)
    set(CMAKE_Fortran_MODULE_DIRECTORY ${HIGHS_BINARY_DIR}/modules)
    add_executable(fortrantest TestFortranAPI.f90)
    if (NOT FAST_BUILD)
        target_link_libraries(fortrantest libhighs FortranHighs)
    else()
        target_link_libraries(fortrantest highs FortranHighs)
    endif()
    target_include_directories(fortrantest PUBLIC 
        ${HIGHS_SOURCE_DIR}/src/interfaces
        ${HIGHS_SOURCE_DIR}/check)
endif()

# if (CUPLDP_GPU AND BUILD_CXX)
#     # add a test
#     add_executable(testcudalin ${HIGHS_SOURCE_DIR}/src/pdlp/cupdlp/cuda/test_cuda_linalg.c)
#     add_executable(testcublas ${HIGHS_SOURCE_DIR}/src/pdlp/cupdlp/cuda/test_cublas.c)

#     set_target_properties(testcudalin PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
#     #target_include_directories(cudalinalg PRIVATE ${CUPDLP_INCLUDE_DIR}/cuda)
#     target_link_libraries(testcudalin PRIVATE highs ${CUDA_LIBRARY})

#     set_target_properties(testcublas PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
#     target_link_libraries(testcublas PRIVATE highs ${CUDA_LIBRARY})
# endif()

if ((NOT FAST_BUILD OR ALL_TESTS) AND NOT (BUILD_EXTRA_UNIT_ONLY))
  # prepare Catch library
  set(CATCH_INCLUDE_DIR ${HIGHS_SOURCE_DIR}/extern)
  add_library(Catch INTERFACE)
  target_include_directories(Catch INTERFACE ${CATCH_INCLUDE_DIR})

  configure_file(${HIGHS_SOURCE_DIR}/check/HCheckConfig.h.in ${HIGHS_BINARY_DIR}/HCheckConfig.h)

  FILE(WRITE ${CMAKE_BINARY_DIR}/testoptions.txt
  "mip_rel_gap=0.0
  mip_abs_gap=0.0")

  # Make test executable
  set(TEST_SOURCES
      TestHighsVersion.cpp
      TestAlienBasis.cpp
      TestDualize.cpp
      TestCallbacks.cpp
      TestCheckSolution.cpp
      TestEkk.cpp
      TestFactor.cpp
      TestMain.cpp
      TestNames.cpp
      TestOptions.cpp
      TestIO.cpp
      TestSort.cpp
      TestSetup.cpp
      TestFilereader.cpp
      TestHighsGFkSolve.cpp
      TestInfo.cpp
      TestBasis.cpp
      TestBasisSolves.cpp
      TestCrossover.cpp
      TestHighsCDouble.cpp
      TestHighsHash.cpp
      TestHighsIntegers.cpp
      TestHighsParallel.cpp
      TestHighsRbTree.cpp
      TestHighsHessian.cpp
      TestHighsModel.cpp
      TestHighsSparseMatrix.cpp
      TestHSet.cpp
      TestICrash.cpp
      TestIis.cpp
      TestIpm.cpp
      TestIpx.cpp
      TestLogging.cpp
      TestLPFileFormat.cpp
      TestLpValidation.cpp
      TestLpModification.cpp
      TestLpOrientation.cpp
      TestModelProperties.cpp
      TestMultiObjective.cpp
      TestPdlp.cpp
      TestPresolve.cpp
      TestQpSolver.cpp
      TestRays.cpp
      TestRanging.cpp
      TestSemiVariables.cpp
      TestThrow.cpp
      TestTspSolver.cpp
      TestUserScale.cpp
      Avgas.cpp
      TestSpecialLps.cpp
      TestLpSolvers.cpp
      TestMipSolver.cpp)

    if (BUILD_EXTRA_UNIT_TESTS)
        list(APPEND CMAKE_MODULE_PATH "${HIGHS_SOURCE_DIR}/check/highs-unit-tests")
        message(STATUS "${HIGHS_SOURCE_DIR}/check/highs-unit-tests")
        include(highs-unit-tests)

        set(TEST_SOURCES ${TEST_SOURCES} ${HIGHS_EXTRA_UNIT_TESTS})
        message(STATUS ${TEST_SOURCES})
    endif()

  add_executable(unit_tests ${TEST_SOURCES})

  if (UNIX)
      target_compile_options(unit_tests PRIVATE "-Wno-unused-variable")
      target_compile_options(unit_tests PRIVATE "-Wno-unused-const-variable")
  endif()

  if (FAST_BUILD)
    target_link_libraries(unit_tests highs Catch)

    if (CUPDLP_GPU)
        if (WIN32)
            target_link_libraries(unit_tests cudalin ${CUDA_LIBRARY})
        else()
            target_link_libraries(unit_tests cudalin ${CUDA_LIBRARY} m)
        endif()
        target_include_directories(unit_tests PRIVATE ${HIGHS_SOURCE_DIR}/src/pdlp/cupdlp/cuda)
        set_target_properties(unit_tests PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
    endif()
  else()
    target_link_libraries(unit_tests libhighs Catch)
  endif()

  include(GNUInstallDirs)
  if(APPLE)
      set_target_properties(unit_tests PROPERTIES INSTALL_RPATH
      "@loader_path/../${CMAKE_INSTALL_LIBDIR};@loader_path")
  elseif(UNIX)
      cmake_path(RELATIVE_PATH CMAKE_INSTALL_FULL_LIBDIR
          BASE_DIRECTORY ${CMAKE_INSTALL_FULL_BINDIR}
          OUTPUT_VARIABLE libdir_relative_path)
      set_target_properties(unit_tests PROPERTIES 
      INSTALL_RPATH "$ORIGIN/${libdir_relative_path}")
  endif()


  # check the C API
  add_executable(capi_unit_tests TestCAPI.c)

  if (FAST_BUILD)
    target_link_libraries(capi_unit_tests highs)
  else()
    target_link_libraries(capi_unit_tests libhighs)
  endif()

  add_test(NAME capi_unit_tests COMMAND capi_unit_tests)

  # Check whether test executable builds OK.
  if (NOT HIGHS_COVERAGE)
    add_test(NAME unit-test-build
            COMMAND ${CMAKE_COMMAND}
                    --build ${HIGHS_BINARY_DIR}
                    --target unit_tests
                    #  --config ${CMAKE_BUILD_TYPE}
            )

    # Avoid that several build jobs try to concurretly build.
    set_tests_properties(unit-test-build
                        PROPERTIES
                        RESOURCE_LOCK unittestbin)

    # create a binary running all the tests in the executable
    add_test(NAME unit_tests_all COMMAND unit_tests --success)
    set_tests_properties(unit_tests_all
                        PROPERTIES
                        DEPENDS unit-test-build)
    set_tests_properties(unit_tests_all PROPERTIES TIMEOUT 10000) 
  else()
    add_test(NAME unit_tests_all COMMAND unit_tests --success)
    set_tests_properties(unit_tests_all PROPERTIES TIMEOUT 10000) 
  endif()

  # An individual test can be added with the command below but the approach
  # above with a single add_test for all the unit tests automatically detects all
  # TEST_CASEs in the source files specified in TEST_SOURCES. Do not define any
  # tests in TestMain.cpp and do not define CATCH_CONFIG_MAIN anywhere else.
  # add_test(NAME correct-print-test COMMAND unit_tests correct-print)

  # --------------------------------------
  # Another way of adding the tests. Needs a script from github repo and a
  # Catch2 installation. So add tests manually if there is no build issues.
  # catch_discover_tests(unit_test)

  # --------------------------------------
  # Run instance tests.
  #
  # define the set of feasible instances
  set(successInstances
      "25fv47\;3149\; 5.5018458883\;"
      "80bau3b\;3686\; 9.8722419241\;"
      "adlittle\;74\; 2.2549496316\;"
      "afiro\;22\;-4.6475314286\;"
      "etamacro\;532\;-7.5571523330\;"
      "greenbea\;5109\;-7.2555248130\;"
      "shell\;623\; 1.2088253460\;"
      "stair\;529\;-2.5126695119\;"
      "standata\;72\; 1.2576995000\;"
      "standgub\;68\; 1.2576995000\;"
      "standmps\;218\; 1.4060175000\;"
      )
  
   set(successMacArmInstances
      "25fv47\;3103\; 5.5018458883\;"
      "80bau3b\;3705\; 9.8722419241\;"
      "adlittle\;74\; 2.2549496316\;"
      "afiro\;22\;-4.6475314286\;"
      "etamacro\;531\;-7.5571523330\;"
      "greenbea\;5156\;-7.2555248130\;"
      "shell\;623\; 1.2088253460\;"
      "stair\;531\;-2.5126695119\;"
      "standata\;72\; 1.2576995000\;"
      "standgub\;68\; 1.2576995000\;"
      "standmps\;218\; 1.4060175000\;"
   )

  set(infeasibleInstances
      "bgetam\;        infeasible"
      "box1\;          infeasible"
      "ex72a\;         infeasible"
      "forest6\;       infeasible"
      "galenet\;       infeasible"
      "gams10am\;      infeasible"
  #    "klein1\;        infeasible"
      "refinery\;      infeasible"
      "woodinfe\;      infeasible"
      )

  set(unboundedInstances
      "gas11\;         unbounded"
      )

  set(failInstances
      )
  
  set(mipInstances 
      "small_mip\;3.2368421\;"
      "flugpl\;1201500\;"
      "lseu\;1120|1119.9999999\;"
      "egout\;(568.1007|568.1006999)\;"
      "gt2\;21166\;"
      "rgn\;82.1999992\;"
      "bell5\;(8966406.49152|8966406.491519|8966406.49151)\;"
      "sp150x300d\;(69|68.9999999)\;"
      "p0548\;(8691|8690.9999999)\;"
      "dcmulti\;188182\;"
      )

  # define settings
  set(settings
      "--presolve=off"
      "--presolve=on"
      "--random_seed=1"
      "--random_seed=2"
      "--random_seed=3"
  #   "--random_seed=4"
  #   "--random_seed=5"
  #   "--parallel=on"
      )

  if (UNIX AND NOT APPLE)
    if (CUPDLP_CPU AND NOT CUPDLP_FORCE_NATIVE)
        set(pdlpInstances
            "25fv47\; 5.50184588\;"
            "adlittle\; 2.254949631\;"
            "afiro\;-4.64753142\;"
            "avgas\;-7.749999999\;"
            "blending\;-3.19999999\;"
            "chip\;-9.000000001\;"
            "e226\;-1.16389293\;"
            "scrs8\; 9.042969511\;"
            "sctest\; 5.749999936\;"
            "shell\; 1.2088253460\;"
            "stair\;-2.51266951\;"
            "standata\; 1.257699499\;"
            "standgub\; 1.25769949\;"
            )
    else()
        if (CUPDLP_GPU)
            set(pdlpInstances
                "25fv47\; 5.5018549\;"
                "adlittle\; 2.254950\;"
                "afiro\;-4.6475325\;"
                "avgas\;-7.75000038\;"
                "blending\;-3.19999999\;"
                "chip\;-9.0000000\;"
                "e226\;-1.16389258\;"
                "scrs8\; 9.0429623\;"
                "sctest\; 5.750000001\;"
                "shell\; 1.20882535\;"
                "stair\;-2.5126695\;"
                "standata\; 1.25769951\;"
                "standgub\; 1.2576993\;"
                )
        else()
        # CUPDLP_GPU or CPU with native termination.
            if (CUPDLP_FORCE_NATIVE)
                set(pdlpInstances
                    "25fv47\; 5.5018360\;"
                    "adlittle\; 2.254953\;"
                    "afiro\;-4.64753126\;"
                    "avgas\;-7.75000038\;"
                    "blending\;-3.19999999\;"
                    "chip\;-9.000000020\;"
                    "e226\;-1.163892070\;"
                    "scrs8\; 9.042970154\;"
                    "sctest\; 5.750000001\;"
                    "shell\; 1.2088253471\;"
                    "stair\;-2.512669020\;"
                    "standata\; 1.257699155\;"
                    "standgub\; 1.257700132\;"
                    )
            endif()
        endif()
    endif()
  elseif(WIN32)
    if (CUPDLP_CPU AND NOT CUPDLP_FORCE_NATIVE)
        # on windows e226 model status is unknown
        # on windows 25fv47 model status can be unknown, with objective 5.5018458957e+03
        set(pdlpInstances
            "25fv47\; 5.5018458\;"
            "adlittle\; 2.25494963\;"
            "afiro\;-4.64753142\;"
            "avgas\;-7.749999999\;"
            "blending\;-3.19999999\;"
            "chip\;-9.000000001\;"
            "scrs8\; 9.0429695\;"
            "sctest\; 5.749999936\;"
            "shell\; 1.2088253460\;"
            "stair\;-2.51266951\;"
            "standata\; 1.2576995\;"
            "standgub\; 1.2576995\;"
            )
    else()
        if (CUPDLP_GPU)
            set(pdlpInstances
                "25fv47\; 5.50185\;" # 549 release 500 debug
                "adlittle\; 2.2549505\;"
                "afiro\;-4.6475325\;"
                "avgas\;-7.750000\;"
                "blending\;-3.19999999\;"
                "chip\;-9.0000000\;"
                "scrs8\; 9.0429693\;"
                "sctest\; 5.7500000\;"
                "shell\; 1.20882535\;"
                "stair\;-2.512669\;" # 96 release 95 debug
                "standata\; 1.2576995\;"
                "standgub\; 1.2576993\;"
                )
        endif()
    endif()
  elseif(APPLE)
    set(pdlpInstances
        "25fv47\; 5.5018458\;"
        "adlittle\; 2.25494963\;"
        "afiro\;-4.64753142\;"
        "avgas\;-7.749999999\;"
        "blending\;-3.19999999\;"
        "chip\;-8.9999999\;"
        "e226\;-1.163892\;"
        "scrs8\; 9.0429695\;"
        "sctest\; 5.749999936\;"
        "shell\; 1.2088253460\;"
        "stair\;-2.5126695\;"
        "standata\; 1.25769950\;"
        "standgub\; 1.25769950\;"
    )
  endif()


  # define a macro to add tests
  #
  # add_instancetests takes an instance group and a status
  # that the solver should report as arguments
  macro(add_instancetests instances solutionstatus)
  # loop over the instances
  foreach(instance ${${instances}})
      # add default tests
      # treat the instance as a tuple (list) of two values
      list(GET instance 0 name)
      list(GET instance 1 iter)

      if(${solutionstatus} STREQUAL "Optimal")
          list(GET instance 2 optval)
      endif()

      # specify the instance and the settings load command
      if(ZLIB AND ZLIB_FOUND AND EXISTS "${HIGHS_SOURCE_DIR}/check/instances/${name}.mps.gz")
        set(inst "${HIGHS_SOURCE_DIR}/check/instances/${name}.mps.gz")
      else()
        set(inst "${HIGHS_SOURCE_DIR}/check/instances/${name}.mps")
      endif()

      # loop over all settings
      foreach(setting ${settings})
          if (FAST_BUILD)
            add_test(NAME ${name}${setting} COMMAND $<TARGET_FILE:highs-bin> ${setting}
                ${inst})
          else()
            add_test(NAME ${name}${setting} COMMAND $<TARGET_FILE:highs> ${setting}
                ${inst})
          endif()

          set_tests_properties (${name}${setting} PROPERTIES
                  DEPENDS unit_tests_all)
          set_tests_properties (${name}${setting} PROPERTIES
                  PASS_REGULAR_EXPRESSION
                  "Model status        : ${solutionstatus}")

          if(${solutionstatus} STREQUAL "Optimal")
              if(${setting} STREQUAL "--presolve=off")
                  set_tests_properties (${name}${setting} PROPERTIES
                          PASS_REGULAR_EXPRESSION
                          "Simplex   iterations: ${iter}\nObjective value     : ${optval}")
              else()
                  set_tests_properties (${name}${setting} PROPERTIES
                          PASS_REGULAR_EXPRESSION
                          "Objective value     : ${optval}")
              endif()
          endif()
      endforeach(setting)
  endforeach(instance)
  endmacro(add_instancetests)

  # add tests for success and fail instances
  if (APPLE AND (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "arm64"))
    add_instancetests(successMacArmInstances "Optimal")
  else()
    add_instancetests(successInstances "Optimal")
  endif()

  add_instancetests(failInstances "Fail")
  add_instancetests(infeasibleInstances "Infeasible")
  add_instancetests(unboundedInstances "Unbounded")


  foreach(instance ${mipInstances})
      list(GET instance 0 name)
      list(GET instance 1 optval)
      # specify the instance and the settings load command
      set(inst "${HIGHS_SOURCE_DIR}/check/instances/${name}.mps")

      foreach(setting ${settings})
          if (FAST_BUILD)
            add_test(NAME ${name}${setting} COMMAND $<TARGET_FILE:highs-bin> ${setting} 
                --options_file ${CMAKE_BINARY_DIR}/testoptions.txt ${inst})
          else()
            add_test(NAME ${name}${setting} COMMAND $<TARGET_FILE:highs> ${setting}
                --options_file ${CMAKE_BINARY_DIR}/testoptions.txt ${inst})
          endif()

          set_tests_properties (${name}${setting} PROPERTIES
                  DEPENDS unit_tests_all)

          set_tests_properties (${name}${setting} PROPERTIES
                  PASS_REGULAR_EXPRESSION
                  "Status            Optimal\n  Primal bound      ${optval}.*\n  Dual bound        ${optval}.*\n  Solution status   feasible\n                    ${optval}.* \\(objective\\)"
                  FAIL_REGULAR_EXPRESSION
                  "Solution status   infeasible")

      endforeach(setting)
  endforeach(instance)

  if(FAST_BUILD)
    foreach(instance_pdlp ${pdlpInstances})
        # add default tests
        # treat the instance as a tuple (list) of two values
        list(GET instance_pdlp 0 name_pdlp)
        list(GET instance_pdlp 1 optval)

        set(inst_pdlp "${HIGHS_SOURCE_DIR}/check/instances/${name_pdlp}.mps")

        add_test(NAME ${name_pdlp}-pdlp-no-presolve COMMAND $<TARGET_FILE:highs-bin> "--solver=pdlp"
            "--presolve=off" ${inst_pdlp})

        set_tests_properties (${name_pdlp}-pdlp-no-presolve PROPERTIES
            PASS_REGULAR_EXPRESSION
            "Model status        : Optimal")

        set_tests_properties (${name_pdlp}-pdlp-no-presolve PROPERTIES
            PASS_REGULAR_EXPRESSION
            "Objective value     : ${optval}")
    endforeach(instance_pdlp)
  endif()

endif()

if (BUILD_EXTRA_PROBLEM_SET)
    list(APPEND CMAKE_MODULE_PATH "${HIGHS_SOURCE_DIR}/check/highs-problem-set")
    message(STATUS "${HIGHS_SOURCE_DIR}/check/highs-problem-set")
    include(highs-problem-set)
endif()

if (BUILD_EXTRA_UNIT_TESTS AND BUILD_EXTRA_UNIT_ONLY)
    # prepare Catch library
    set(CATCH_INCLUDE_DIR ${HIGHS_SOURCE_DIR}/extern)
    add_library(Catch INTERFACE)
    target_include_directories(Catch INTERFACE ${CATCH_INCLUDE_DIR})


    list(APPEND CMAKE_MODULE_PATH "${HIGHS_SOURCE_DIR}/check/highs-unit-tests")
    message(STATUS "${HIGHS_SOURCE_DIR}/check/highs-unit-tests")
    include(highs-unit-tests)

    set(TEST_SOURCES TestMain.cpp ${HIGHS_EXTRA_UNIT_TESTS})
    message(STATUS ${TEST_SOURCES})

    add_executable(unit_tests_extra ${TEST_SOURCES})
    target_link_libraries(unit_tests_extra Catch)

    if (BUILD_CXX)
        configure_file(${HIGHS_SOURCE_DIR}/check/HCheckConfig.h.in ${HIGHS_BINARY_DIR}/HCheckConfig.h)
        target_link_libraries(unit_tests_extra highs)
    endif()

    add_test(NAME unit-test-extra-build
          COMMAND ${CMAKE_COMMAND}
                  --build ${HIGHS_BINARY_DIR}
                  --target unit_tests_extra
                  #  --config ${CMAKE_BUILD_TYPE}
          )

    # Avoid that several build jobs try to concurretly build.
    set_tests_properties(unit-test-extra-build
                        PROPERTIES
                        RESOURCE_LOCK unittestbin)

    # create a binary running all the tests in the executable
    add_test(NAME unit_tests_extra COMMAND unit_tests_extra --success)
    set_tests_properties(unit_tests_extra
                        PROPERTIES
                        DEPENDS unit-test-extra-build)
    set_tests_properties(unit_tests_extra PROPERTIES TIMEOUT 10000) 

    if (CUPDLP_GPU)
        set_target_properties(unit_tests_extra PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
        target_link_libraries(unit_tests_extra ${CUDA_LIBRARY})

        add_executable(cublas_example cublas_example.cpp)
        set_target_properties(cublas_example PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
        target_link_libraries(cublas_example ${CUDA_LIBRARY})

        add_executable(cublas_gpu_start cublas_gpu_start.cpp)
        set_target_properties(cublas_gpu_start PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
        target_link_libraries(cublas_gpu_start ${CUDA_LIBRARY})
    endif()

endif()
