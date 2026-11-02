# Montanha Racer - porte do backend XDK para libxenon (Xbox 360 RGH/JTAG)

## Resumo honesto
| Item | Estado |
|---|---|
| Nucleo (`src/core`) e backend XDK (`src/platform_xdk`) | **byte-identicos** ao original (nada removido, nada alterado) |
| Backend libxenon (`src/platform_xenon/main_xenon.cpp`) | **escrito, completo** (video, entrada, save, loop, saida). Logica validada no PC contra um stub (`tests/xenon_backend_check.cpp`, `-std=c++03 -Werror`, ASan/UBSan limpos) |
| Compilado com `xenon-g++` | **NAO** - o ambiente onde preparei isto nao tem o toolchain nem rede. Quem decide e `make -f Makefile.xenon` na sua maquina |
| `default.xex` | **O libxenon nao gera .xex** (ver abaixo). O que ele gera e `.elf32` |
| Shaders | **Faltam 2 blobs binarios** (`shaders/vs.bin`, `shaders/ps.bin`): o libxenon nao compila HLSL. Fontes HLSL em `shaders/*.hlsl` |

## Por que nao existe `default.xex` com libxenon
libxenon e uma biblioteca *bare-metal*: o programa assume o console inteiro e e carregado pelo **XeLL Reloaded**
como `.elf32` (ELF32), nao pelo kernel da Microsoft. Um `.xex` roda *dentro* do kernel/hipervisor e e feito com
o toolchain do XDK (`imagexex`). Sao dois mundos; eu nao conheco (e nao vou inventar) um conversor ELF->XEX.
Dois caminhos reais:

1. **libxenon (este projeto)** -> `montanha_racer.elf32` -> copie como `xenon.elf` na raiz de um pendrive FAT32 e
   boote o XeLL. Para iniciar a partir do dashboard (Aurora/FSD), o pacote do DashLaunch traz uma pasta
   `XellLaunch` com um `default.xex` **pronto** que apenas inicia o XeLL (esse xex nao e produzido aqui).
2. **XDK (README.md original)** -> `default.xex` de verdade, via `imagexex`, com `src/platform_xdk/main_xdk.cpp`.

## Dependencias
- Linux (ou WSL). Toolchain libxenon: `build-xenon-toolchain toolchain` e depois `libxenon` e `libs` (dependencias
  Debian listadas pelo projeto: flex bison gcc-multilib libgmp3-dev libmpfr-dev libmpc-dev texinfo git-core
  build-essential wget file). Ou a imagem Docker `free60/libxenon:latest` (monte o projeto e rode `make`).
- `export DEVKITXENON=/usr/local/xenon` (ou o seu `PREFIX`).
- Python 3 (so para `tools/bin2h.py`).
- No console: RGH/JTAG com **XeLL Reloaded**; pendrive FAT32. Para save: libfat instalado (`build-xenon-toolchain libs`).

## Processo de build
    export DEVKITXENON=/usr/local/xenon
    bash tools/verify_symbols.sh          # confere headers/libs/campos REAIS (ver "Rastreabilidade")
    # gere shaders/vs.bin e shaders/ps.bin (secao Shaders), depois:
    make -f Makefile.xenon                # -> montanha_racer.elf32
    cp montanha_racer.elf32 /caminho/do/pendrive/xenon.elf

`make` roda `verify_symbols.sh` primeiro e **aborta** se algum simbolo/campo nao existir no seu libxenon.

## Shaders
O backend usa 1 vertex shader + 1 pixel shader "cor por vertice" (equivalentes aos do backend XDK).
Vertice = `float2 POSITION` + `float4 COLOR` (24 bytes); o backend ja entrega a posicao em NDC.
Formas de obter os blobs (nao testei nenhuma aqui):
- Compilador de shaders citado na documentacao do libxenon (`shadercompiler.exe <hlsl> <saida> /vs` e `/ps`) a partir de
  `shaders/vs.hlsl` e `ps.hlsl`; salve como `shaders/vs.bin` / `shaders/ps.bin`.
- Toolchain aberto `xenosworks/xenos-toolchain` (Python): o README dele diz emitir containers aceitos por
  `Xe_LoadShaderFromMemory` e traz um exemplo "passthrough" (`examples/author_passthrough.py`). **Confira que o layout
  de vertice dele e pos float2 + cor float4**; senao ajuste `vbf_` em `XenonRenderer::init()` (e `Vtx`).
`Xe_ShaderApplyVFetchPatches` reescreve os vfetch do shader para o formato `vbf_`, mas os registradores/semanticas
(POSITION/COLOR) precisam existir no shader.

## O que foi portado (mapeamento)
| XDK | libxenon |
|---|---|
| `Direct3DCreate9/CreateDevice` | `xenos_init()` + `Xe_Init(&dev)`; tamanho do video lido de `Xe_GetFramebufferSurface()->width/height` |
| `D3DXCompileShader` em runtime | blobs pre-compilados + `Xe_LoadShaderFromMemory/InstantiateShader/ShaderApplyVFetchPatches` |
| `DrawPrimitiveUP` (lote 3000 vert.) | `Xe_VBBegin/VBPut/VBEnd` + `Xe_SetStreamSource` + `Xe_DrawPrimitive` (mesmo lote) |
| viewport 16:9 + letterbox 4:3 | letterbox por escala uniforme na CPU + `Xe_SetScissor` (nao ha `SetViewport` no xe.h) |
| `Present` | `Xe_Resolve` + espera de VBlank + `Xe_Sync` + `Xe_VBReclaim` |
| `XInputGetState` (4 portas, zona morta radial 0,24, gatilho 30/255) | `get_controller_data(&c, porta)` + **mesma** zona morta/limiar |
| `XContent` SAVEDGAME + seletor de dispositivo | arquivo comum `HCR_SAVE_PATH` (padrao `uda:/montanha_racer.sav`) via stdio; **mesmo formato** de 68 bytes (BE + CRC32 + validacao); sem dispositivo => joga sem salvar |
| `QueryPerformanceCounter` | `gettimeofday` |
| pausa ao desconectar controle | mantida |
| `XLaunchNewImage` (SAIR) | `xenon_smc_power_reboot()` (reinicia o console; nao existe "voltar ao dashboard" em bare-metal) |
Diferenca inevitavel: sem perfil/Gamerscore/overlay do sistema; o "seletor de dispositivo" some (so USB/FAT por caminho).

## Rastreabilidade (nada inventado - cada API esta marcada no codigo)
- `[DOC]` confirmado na documentacao oficial: toda a API `Xe_*` usada (assinaturas e constantes de `xe.h`),
  `xenos_init`, `console_init`, `kmem_init`, `usb_init`, `usb_do_poll`, `get_controller_data(&c, n)`, campo `.a`,
  `xenon_smc_power_reboot`, flags de compilacao/link (`-DXENON -m32 -mno-altivec -mhard-float -T app.lds ...`),
  conversao `xenon-objcopy -O elf32-powerpc --adjust-vma 0x80000000` + `xenon-strip`.
- `[VERIF]` **nao confirmado por mim** - o script confere no seu `$DEVKITXENON`: caminhos `xenos/xe.h`,
  `xenon_smc/xenon_smc.h`, `fat/fat.h`; `fat_init`; `gettimeofday`; campos do pad `s1_x s1_y lt rt b start up down left right`
  (so em `xenon_pad_map.h`, e a faixa dos valores); indices de porta 0..3.
- `[HW]` so o console confirma: ordem `Clear -> draw -> Resolve -> VBlank -> Sync -> VBReclaim`, se o blend precisa de
  mais que `Xe_SetBlendControl`, eixo Y (compile com `-DHCR_FLIP_Y` se a imagem sair invertida), cor de limpeza.

## Testes
    g++ -std=c++03 -Wall -Wextra -Werror -D'HCR_SAVE_PATH="/tmp/h.sav"' -Itests/xenon_stub \
        tests/xenon_backend_check.cpp src/core/game.cpp -o xenon_check && ./xenon_check
Cobre: assinaturas (contra stub transcrito da doc), formato/pitch do vertice, lote <= 3000, ordem de chamadas por quadro,
letterbox 16:9 / 4:3 / 640x480, save (round-trip, CRC, truncado, fora de faixa), entrada (zona morta, sem controle).
Tambem vale em CMake (`xenon_backend_check`). **O stub reflete a documentacao, nao prova o hardware.**

## Pendencias reais
1. Gerar `shaders/vs.bin` e `ps.bin`.
2. Rodar `verify_symbols.sh` e corrigir o que ele apontar.
3. Primeira execucao no console: validar os itens `[HW]`.
4. (Visual, do README original) HUD "MOEDAS" dobrado na tela de fim; margem de area segura - nao tocados.
