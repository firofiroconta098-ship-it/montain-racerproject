# Montanha Racer (estilo Hill Climb) - C++03

> **Caminho para `default.xex` (RGH/JTAG, Aurora): veja `README_XEX.md`** (backend `src/platform_xsdk`, `Makefile.xsdk`).
> O XEX **ainda nao foi gerado** - o toolchain nao pode ser construido onde isto foi preparado; o README_XEX.md diz o que foi e o que nao foi verificado.

## Estado real do projeto
| Parte | Status |
|---|---|
| Física, terreno, moedas, combustível, garagem, menu, pausa, fim de jogo, save (núcleo) | feito e testado (`tests/headless_test.cpp`, `-std=c++03`) |
| Backend PC (SDL2 >= 2.0.18) | feito, **não executado** (sem SDL/janela no ambiente de desenvolvimento) |
| Backend Xbox 360 (`src/platform_xdk/main_xdk.cpp`) | **completo, sem TODO** (D3D9 do 360, XInput, XContent, loop). **Não compilado contra o XDK** (proprietário, sem acesso). Lógica validada no PC contra um stub: `tests/xdk_backend_check.cpp` |
| Arquivo `.XEX` | **não gerado** - exige o toolchain do XDK (abaixo) |

## Testes no PC
    cmake -S . -B build && cmake --build build
    cd build
    ./headless_test          # física/garagem sem vídeo
    ./xdk_backend_check      # backend Xbox 360 contra o stub (grava stub_*/ e shots/*.ppm aqui)
    ./montanha_racer         # jogo com SDL2 (se instalado)

`xdk_backend_check` executa o `main()` real do backend (renomeado) com roteiros de controle e um
rasterizador de software no lugar da GPU. Cobre: batching e ordem de desenho, estado D3D exigido,
letterbox 4:3 (buffer 1280x960 cabe na EDRAM), save com round-trip pelo `Game`, save corrompido,
sem perfil, seletor cancelado, pausa por desconexão do controle e saída para o dashboard.
Roda com ASan/UBSan limpos. **O stub reflete o que o backend assume sobre o XDK; ele não prova
que as assinaturas existem no XDK real.** Isso só o compilador do XDK confirma.

## O que o backend Xbox 360 faz
- **Render**: `Direct3DCreate9` -> `CreateDevice` 1280x720 (A8R8G8B8, front `D3DFMT_LE_X8R8G8B8`,
  sem MSAA, vsync). Sem pipeline fixo no 360: 1 vertex shader + 1 pixel shader HLSL "cor por vertice"
  (`vs_2_0`/`ps_2_0`) compilados na inicialização com `D3DXCompileShader`. `fillQuad`/`fillCircle`
  acumulam triângulos num buffer e descarregam com `DrawPrimitiveUP` (<= 3000 vértices por chamada;
  ~4800 vértices e ~2 chamadas por quadro medidos no jogo). Quads/círculos fora da tela são descartados.
  TV 4:3: buffer 1280x960 com viewport 16:9 centralizado (letterbox).
- **Entrada**: `XInputGetState` nas 4 portas; usa o primeiro controle conectado e o mantém. Zona morta
  radial 0,24 com reescala; gatilhos com limiar 30/255.
- **Save**: `XContent` (SAVEDGAME). No boot escolhe o primeiro perfil logado e abre o seletor de
  dispositivo do sistema (enquanto isso apresenta quadros pretos para a UI aparecer). Arquivo de 68 bytes,
  big-endian explícito, assinatura+versão+CRC32, e validação de faixas (um save corrompido volta ao
  padrão; o núcleo indexa `level[selected]` sem checar limites). Sem perfil ou seletor cancelado => o
  jogo roda normalmente, **sem salvar nessa sessão** (um aviso ao jogador seria exigência de certificação).
- **Loop**: `QueryPerformanceCounter`, dt limitado a 0,1 s. Controle desconectado durante a corrida =>
  pausa. "SAIR" do menu chama `XLaunchNewImage` para voltar ao dashboard.

## Caminho até o .XEX (ambiente com XDK licenciado)
**Necessário**
1. **Xbox 360 XDK** (Microsoft; só para desenvolvedores licenciados, sob NDA): headers/libs
   (`xtl.h`, D3D9, D3DX9, XInput, XContent), compilador C++ para PowerPC e `imagexex`.
2. **Visual Studio** na versão que o seu XDK integra (consulte as notas de versão do XDK; os XDKs finais
   usam o VS2010) e Windows compatível com esse XDK.
3. **Console de desenvolvimento Xbox 360 (devkit)** ligado ao PC para rodar/depurar. Um `.XEX` gerado
   assim é de desenvolvimento; rodar em console retail exige o processo de assinatura/certificação da
   Microsoft e do publisher.
4. Title ID e arquivo de configuração do XEX (fornecidos pelo seu programa de desenvolvimento).

**Passos**
1. Novo projeto Xbox 360 (executável/vazio) no VS do XDK. Adicione `src/core/game.cpp`,
   `src/core/game.h`, `src/core/platform.h` e `src/platform_xdk/main_xdk.cpp`.
2. `_XBOX` já é definido pela plataforma Xbox 360 (o backend inteiro está sob `#ifdef _XBOX`).
   Recomendado: exceções C++ desligadas (o núcleo foi escrito sem elas).
3. Bibliotecas: use as do template do projeto e garanta `d3d9.lib`, `d3dx9.lib`, `xgraphics.lib`. Se o
   linker reclamar de símbolos de XInput/XContent/XUser, acrescente a lib que a página de cada função
   no help do XDK indica.
4. Compile na plataforma "Xbox 360" -> `.exe` PowerPC.
5. Conversão para `.xex` com `imagexex`. O VS do XDK normalmente roda essa etapa como pós-link
   (propriedades de "Image Conversion"); manualmente, confira opções com `imagexex /?`.
6. Deploy no devkit pelo próprio VS (F5) ou pelas ferramentas de deploy do XDK.

Para build de produção, pré-compile os shaders offline em vez de usar `D3DXCompileShader` em runtime.
Alternativa homebrew (libxenon/Free60) tem outra API/gráfico e **não** compila este backend.

## Símbolos do XDK a conferir (usados de memória, sem os headers)
Se algum divergir, o erro aparece na compilação; os pontos estão concentrados em `main_xdk.cpp`.

*Alta confiança (D3D9/XInput/Win32 como nos samples do XDK):* `void __cdecl main()`, `Direct3DCreate9`,
`D3D_SDK_VERSION`, `D3DPRESENT_PARAMETERS` (campos usados), `D3DFMT_LE_X8R8G8B8`, `CreateDevice(0, D3DDEVTYPE_HAL,
NULL, D3DCREATE_HARDWARE_VERTEXPROCESSING, ...)`, `CreateVertexDeclaration`/`D3DVERTEXELEMENT9`, `D3DXCompileShader`
(`"vs_2_0"`/`"ps_2_0"`), `CreateVertexShader`/`CreatePixelShader`, `SetViewport`, `Clear`, `SetRenderState`,
`DrawPrimitiveUP`, `Present(NULL,NULL,NULL,NULL)`, `XInputGetState`/`XINPUT_*`, `QueryPerformanceCounter/Frequency`,
`CreateFile/ReadFile/WriteFile/CloseHandle`, `OutputDebugStringA`.

*Menor confiança (APIs específicas do 360, confira nome e assinatura):* `XGetVideoMode` + `XVIDEO_MODE.fIsWideScreen`;
`XUserGetSigninState` + `eXUserSigninState_NotSignedIn`; `XShowDeviceSelectorUI` (6 parâmetros, flags = 0);
`XOVERLAPPED`, `XHasOverlappedIoCompleted`, `XGetOverlappedResult`, `XGetOverlappedExtendedError`;
`XCONTENT_DATA` (`DeviceID`, `dwContentType`, `szDisplayName`, `szFileName`), `XCONTENT_MAX_DISPLAYNAME_LENGTH`,
`XCONTENT_MAX_FILENAME_LENGTH`, `XCONTENTTYPE_SAVEDGAME`, `XCONTENTFLAG_OPENEXISTING`, `XCONTENTFLAG_OPENALWAYS`;
`XContentCreate` (8 parâmetros: usuário, raiz, dados, flags, disposição, licença, cache, overlapped);
`XContentClose`; `XLaunchNewImage` + `XLAUNCH_KEYWORD_DEFAULT_APP`.

## Observações sobre o núcleo (não alterado)
- `Game::render` mostra "MOEDAS" como `save.coins + runCoins`, mas na tela de fim `endRun()` já somou
  `runCoins` em `save.coins`: o HUD exibe o dobro (visto: 240 na tela, 120 no save). Só visual.
- O HUD usa margem de 40 px; a área segura de título em 16:9 é ~64 px (lados) x 36 px (topo/base).
- O construtor de `Game` só confere `magic` do save; o backend Xbox 360 valida tudo, mas o SDL (`fwrite`
  da struct crua) não.
