# Montanha Racer -> `default.xex` (Xbox 360 RGH/JTAG, executavel pelo Aurora)

## Resumo honesto
| Item | Estado |
|---|---|
| Nucleo (`src/core`), backends XDK e libxenon | **byte-identicos** ao que voce enviou. Nada removido, nada reescrito |
| Backend novo `src/platform_xsdk/main_xsdk.cpp` | **escrito, completo** (video, entrada, save, loop, saida). Usa so funcoes declaradas nos headers do x360-sdk |
| Compila contra os **headers reais** do x360-sdk (commit `53653ac`) | **SIM** - `g++ -std=c++98/03/11/17 -Wall -Wextra`, sem avisos (PC, so checagem de tipos/assinaturas) |
| Logica validada no PC (`tests/xsdk_backend_check.cpp`) | **SIM**, ASan/UBSan limpos: ordem de chamadas, layout de vertice, lotes <=3000, blend, controle de fence (hazard), save, entrada, pausa por desconexao, SAIR. Capturas de tela renderizadas por rasterizador de software conferidas (menu, corrida, pausa) |
| Compilado com o toolchain PowerPC (`powerpc-xsdk-elf-g++`) | **NAO.** O toolchain exige compilar GCC 15 + binutils + newlib a partir de ftp.gnu.org / sourceware.org, bloqueados no ambiente onde preparei isto |
| `default.xex` gerado | **NAO.** Nenhum .xex foi produzido. Nao existe aqui um binario que eu tenha compilado |
| Rodou no console / Aurora | **NAO testado.** So voce pode confirmar |

## Por que este caminho
O libxenon gera ELF32 para o XeLL, nao XEX, e nao existe conversor ELF->XEX para ele. Para obter um XEX **sem o XDK da
Microsoft** achei um SDK aberto que faz exatamente isso: **x360-sdk / xsdk** (https://github.com/openpak/x360-sdk, MIT),
que traz um toolchain GCC para o Xenon, uma camada de GPU/entrada/arquivos e `tools/elf2xex.py`, que empacota o ELF num
XEX2 aceito pelo loader retail de console RGH. O README e o `docs/HARDWARE.md` do proprio SDK afirmam que um XEX
nao assinado feito pelo `elf2xex` foi **lancado pelo Aurora** num console RGH e passou os testes dele (CPU, threads, arquivos, controle, audio, GPU).
**Isso sao alegacoes do projeto deles; eu nao reproduzi.** Ele e jovem (marcos de 2026) e de terceiros: avalie antes de depender.

## O que mudou (somente a camada de plataforma)
Nada do jogo foi tocado. Arquivos novos: `src/platform_xsdk/main_xsdk.cpp`, `shaders/hcr_xsdk_shaders.h`,
`Makefile.xsdk`, `tests/xsdk_backend_check.cpp`, este README. Alterados: `CMakeLists.txt` (alvos de teste opcionais) e uma
linha de aviso no topo de `README.md`.

Mapeamento (o que foi **adaptado**, com o porque; nada foi removido):
| libxenon / XDK | xsdk |
|---|---|
| `Xe_*` / `D3D9` | `xsdk_gpu_*`, `xsdk_present_*`, `xsdk_shader_*`; `set_target` -> draws -> `resolve` -> `present_end` por quadro |
| Shaders HLSL / blobs `vs.bin`,`ps.bin` (faltavam no libxenon) | **resolvido**: `shaders/hcr_xsdk_shaders.h` copia, sem alteracao, os dois containers de microcodigo do SDK (passam posicao e cor), os mesmos que o proprio SDK usa nos testes de console. Vertice passou de `float2+float4` (24 B) para `float4+float4` (32 B) para casar com eles. Os `.hlsl` ficam como documentacao |
| `DrawPrimitiveUP`/`VBPut` | vertices do quadro acumulados e copiados para memoria write-combined da GPU; 3 buffers em rodizio, com **fence** antes de reutilizar cada um; draws de ate 3000 vertices (mesmo lote de antes) |
| Letterbox 4:3 por scissor | **removido da camada de plataforma**: o `present` do SDK fixa 1280x720 e o scaler do console adapta a saida. O jogo ainda desenha em 1280x720 |
| `XInputGetState` / `get_controller_data` | `XamInputGetState(porta,0,&estado)`; **mesma** zona morta radial 0,24, limiar de gatilho 30/255, varredura das 4 portas, pausa ao desconectar |
| Save: XContent (XDK) / `uda:/` (libxenon) | arquivo `game:\montanha_racer.sav` (pasta do XEX), se falhar `cache:\...`; **mesmo formato** de 68 bytes (BE + CRC32 + validacao), mesmo `save_codec.h`. Sem local gravavel => joga sem salvar. Sem seletor de dispositivo/perfil (inexistente fora do XDK) |
| Relogio | `clock_gettime(CLOCK_MONOTONIC)` |
| SAIR | retorna de `main` -> o SDK devolve ao dashboard (Aurora). (No libxenon era reboot) |
| Diagnostico | `xsdk_log_write` -> o SDK grava `game:\default.log` ao lado do XEX |

Observacoes antigas do nucleo (HUD "MOEDAS" dobrado na tela final, margem de area segura) **nao foram tocadas**.

## Como gerar o `default.xex` (na sua maquina; Linux/WSL Debian)
O toolchain demora (compila GCC) e baixa codigo-fonte do GNU; `build.sh` confere SHA-256 dos arquivos.
```bash
git clone https://github.com/openpak/x360-sdk && cd x360-sdk && git checkout 53653ac   # versao contra a qual validei os headers
bash tools/toolchain/build.sh            # instala em ~/xsdk-gcc  (powerpc-xsdk-elf-gcc/g++)
make build/libxsdk.a                     # biblioteca do SDK (o `make` sem alvo constroi tambem os testes do SDK)
cd /caminho/montanha_racer_libxenon/mr_libxenon
make -f Makefile.xsdk XSDK=/caminho/x360-sdk      # -> montanha_racer.elf e default.xex
make -f Makefile.xsdk XSDK=/caminho/x360-sdk dist # -> dist/MontanhaRacer/ (inclui o aviso de licenca BSD que o SDK exige)
```
Se algo falhar, a mensagem do compilador aponta o ponto: o backend usa ~30 funcoes do SDK, todas conferidas nos
headers acima, mas **nunca foi linkado**. Se o commit do SDK mudar, rode os testes de PC com `-DXSDK_ROOT=...`
(abaixo) para ver se alguma assinatura divergiu.

## No console
1. Copie `dist/MontanhaRacer/` (com `default.xex`) para o HD ou USB, numa pasta que o Aurora varre (Aurora -> Configuracoes ->
   Conteudo/Caminhos de busca; depois atualize a biblioteca) ou abra o `default.xex` pelo gerenciador de arquivos do Aurora.
2. O XEX gerado pelo `elf2xex` **nao tem nome/icone nem assinatura**: o Aurora pode listar pelo nome da pasta, sem capa; funciona so em RGH/JTAG.
   O Title ID padrao e o do SDK (`0x5853444B`); mude com `TITLE_ID=...` no make, se quiser.
3. Se travar ou voltar ao dashboard, leia `default.log` (na mesma pasta) e me envie.

## Testes no PC
```bash
cmake -S . -B build -DXSDK_ROOT=/caminho/x360-sdk && cmake --build build
cd build && ./headless_test && ./xsdk_backend_check && ./xsdk_backend_check_fallback
```
(`XSDK_ROOT` so precisa dos headers.) Grava `shots/xsdk_*.ppm` (menu, corrida, pausa, garagem, desconexao).
O detector de hazard foi verificado por mutacao: removendo a espera pelo fence e usando 1 slot, 3 checagens falham.

## O que continua sem prova (so o console responde)
- Se o toolchain do SDK compila/linka este codigo sem ajustes (nunca executado).
- Eixo Y, cor e blend no hardware (os shaders sao os do SDK e o Y segue o `bar_scene` dele, mas nao vi isto num console).
- Primeiro quadro: o eDRAM pode conter lixo antes do primeiro resolve; o jogo pinta a tela inteira, entao nao deve aparecer.
- Se a versao do kernel/dashboard do seu console aceita o XEX (o SDK relata testes em um console RGH especifico).

## Caminhos anteriores (intactos)
`Makefile.xenon` (ELF32 para XeLL; `README_XENON.md`) e `src/platform_xdk` (XDK da Microsoft; `README.md`).
