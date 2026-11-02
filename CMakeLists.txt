cmake_minimum_required(VERSION 3.10)
project(MontanhaRacer CXX)
set(CMAKE_CXX_STANDARD 98)   # codigo do nucleo e C++03 de proposito (compiladores antigos do XDK)
add_library(core STATIC src/core/game.cpp)
add_executable(headless_test tests/headless_test.cpp)
target_link_libraries(headless_test core)

# Valida a LOGICA do backend Xbox 360 (src/platform_xdk/main_xdk.cpp) no PC contra um stub do XDK
# (tests/xdk_stub). NAO substitui compilar no XDK. Rode de dentro do diretorio de build:
#   ./xdk_backend_check      (grava stub_*/ e shots/*.ppm no diretorio atual)
add_executable(xdk_backend_check tests/xdk_backend_check.cpp)
target_include_directories(xdk_backend_check PRIVATE tests/xdk_stub)
target_link_libraries(xdk_backend_check core)

# Valida a LOGICA do backend libxenon (src/platform_xenon) no PC contra um stub com as assinaturas da doc do libxenon.
add_executable(xenon_backend_check tests/xenon_backend_check.cpp)
target_include_directories(xenon_backend_check PRIVATE tests/xenon_stub)
target_compile_definitions(xenon_backend_check PRIVATE HCR_SAVE_PATH="hcr_test.sav")
target_link_libraries(xenon_backend_check core)

find_package(SDL2 QUIET)
if(SDL2_FOUND)
  add_executable(montanha_racer src/platform_sdl/main_sdl.cpp)
  target_include_directories(montanha_racer PRIVATE ${SDL2_INCLUDE_DIRS})
  target_link_libraries(montanha_racer core ${SDL2_LIBRARIES})
endif()
# Backends Xbox 360 NAO sao compilados aqui: XDK (README.md) ou libxenon (README_XENON.md + Makefile.xenon).

# Valida o backend xsdk (src/platform_xsdk/main_xsdk.cpp, o que gera default.xex) no PC, compilando contra os
# headers REAIS do x360-sdk. Ative com:  cmake -S . -B build -DXSDK_ROOT=/caminho/para/x360-sdk
# (so precisa dos headers em $XSDK_ROOT/include; nao precisa do toolchain nem do console).
set(XSDK_ROOT "" CACHE PATH "Checkout do openpak/x360-sdk (so os headers sao usados aqui)")
if(XSDK_ROOT AND EXISTS "${XSDK_ROOT}/include/xsdk/gpu.h")
  add_executable(xsdk_backend_check tests/xsdk_backend_check.cpp)
  target_include_directories(xsdk_backend_check PRIVATE "${XSDK_ROOT}/include")
  target_compile_definitions(xsdk_backend_check PRIVATE
    HCR_SAVE_PATH="hcr_xsdk_test.sav" HCR_SAVE_PATH_FALLBACK="hcr_xsdk_test_fb.sav")
  target_link_libraries(xsdk_backend_check core)
  # Modo "primario nao gravavel" -> exercita o save em cache:\ (fallback)
  add_executable(xsdk_backend_check_fallback tests/xsdk_backend_check.cpp)
  target_include_directories(xsdk_backend_check_fallback PRIVATE "${XSDK_ROOT}/include")
  target_compile_definitions(xsdk_backend_check_fallback PRIVATE HCR_TEST_FALLBACK
    HCR_SAVE_PATH="diretorio_que_nao_existe/hcr.sav" HCR_SAVE_PATH_FALLBACK="hcr_xsdk_test_fb.sav")
  target_link_libraries(xsdk_backend_check_fallback core)
endif()
# O .xex em si: Makefile.xsdk (README_XEX.md).
