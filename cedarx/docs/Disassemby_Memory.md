# 14. Atualização da investigação — SoC A733 e engenharia reversa do demo

**Data da atualização:** 22/11/2026  
**Escopo:** incorporar o manual específico do A733 e os resultados mais recentes da análise de `vdecoderdemo`.

Esta seção complementa o relatório anterior. Ela não substitui as conclusões já registradas nem altera os binários do sistema.

## 14.1. Manual oficial do SoC A733

Foi consultado o documento **A733 Octa-core Processor — User Manual, Revision 1.00**, fornecido durante a investigação. Para as capacidades do mecanismo de vídeo deste SoC, esse manual específico deve prevalecer sobre tabelas genéricas de outros SoCs Allwinner.

A seção 14.2 descreve uma arquitetura formada por:

- **Video Control Firmware (VCF):** executado no ARM; recebe e analisa o bitstream, controla o processo e solicita operações ao mecanismo de vídeo.
- **Video Engine (VE):** bloco de hardware responsável pelas operações de decodificação.

O manual lista as seguintes capacidades máximas para o A733:

| Codec/formato | Capacidade indicada no manual A733 |
|---|---|
| H.265 Main/Main10, Level 5.2 | 8192 × 4352, até 24 fps, 10-bit |
| VP9 Profile 0/2, Level 6.1 | 8192 × 8192, até 24 fps, 10-bit |
| AVS2 | 8192 × 4608, até 24 fps, 10-bit |
| H.264 Baseline/Main/High, Level 5.1 | 4096 × 2304, até 30 fps, 8-bit |
| H.263 Baseline | 1080p60 |
| VP8 | 1080p60 |
| MPEG-4 Simple/Advanced Simple, Level 5 | 1080p60 |
| MPEG-2 Main/High | 1080p60 |
| MPEG-1 Main/High | 1080p60 |
| XVID | 1080p60 |
| Sorenson Spark | 1080p60 |
| AVS/AVS+ Jizhun | 1080p60 |
| MJPEG | resolução máxima listada de 16384 × 16384; decodificação até 1080p60 |
| WMV9/VC-1 | 1080p30 |

Esses valores são os limites declarados pelo manual do A733. Não devem ser interpretados como garantia de que qualquer perfil, nível, combinação de ferramentas ou formato de saída será aceito por toda versão da biblioteca CedarX instalada. A validação prática ainda depende do plugin e do firmware disponíveis no sistema.

O manual também descreve registradores do bloco `VE_DEC`, incluindo a região `0x01C0E000–0x01C0FFFF`, além de registradores de clock e de configuração do bloco. Isso confirma a presença de um mecanismo dedicado de decodificação no SoC, mas não documenta por si só a ABI da biblioteca CedarX.

## 14.2. Resultado confirmado do teste H.264 CedarX

O teste documentado na seção 4 continua sendo a evidência prática principal: `vdecoderdemo` decodificou 31 quadros do fluxo H.264 Annex B, com resolução 1920 × 816, e gravou um quadro NV12 de 2.350.080 bytes. O processo terminou com código de saída zero.

A mensagem `hevc parser receive other thread error. exit flag` também apareceu no encerramento desse teste H.264. Portanto, ela deve permanecer registrada como mensagem de encerramento ainda não explicada; não é correto apagá-la da análise, mas tampouco ela invalida o resultado observado (31 quadros decodificados/apresentados e arquivo NV12 produzido).

## 14.3. API e dependências CedarX confirmadas no sistema

A instalação examinada contém, entre outros, os seguintes pacotes e componentes:

```text
libcedarc-dev-2.0.0-arm64  1.0.4
gst-omx-listcomponents     1.18.3-1
```

Bibliotecas relevantes encontradas em `/usr/lib/aarch64-linux-gnu/`:

```text
libMemAdapter.so
libVE.so
libvdecoder.so
libvideoengine.so
libcdc_base.so
libfbm.so
libsbm.so
libawh264.so
libawh265.so
libawvp9HwAL.so
```

`ldd /usr/lib/aarch64-linux-gnu/libvdecoder.so` mostrou dependências resolvidas para `libcdc_base.so`, `libMemAdapter.so`, `libfbm.so`, `libsbm.so`, `libvideoengine.so` e `libVE.so`, além das bibliotecas padrão.

Os símbolos exportados relevantes identificados foram:

- `MemAdapterGetOpsS` em `libMemAdapter.so`;
- `GetVeOpsS`, `VeInitialize` e `VeRelease` em `libVE.so`.

O utilitário `gst-omx-listcomponents` enumerou componentes OMX para H.264, HEVC e VP9, entre outros. Isso comprova que o Core OMX anuncia esses componentes, mas **não** comprova que exista um plugin GStreamer OMX instalado ou que o caminho OMX esteja funcional. Não foi encontrado plugin `*omx*` ou `*cedar*` no diretório de plugins GStreamer examinado.

## 14.4. Sequência de inicialização reconstruída em `initDecoder()`

A desmontagem de `/usr/bin/vdecoderdemo` identificou a função `initDecoder()` no endereço `0x4108`. A sequência observada até agora é:

1. Chama `AddVDPlugin()`.
2. Zera uma estrutura `VConfig` de 224 bytes e uma estrutura `VideoStreamInfo` de 72 bytes.
3. Obtém a tabela de operações de memória por `MemAdapterGetOpsS()`.
4. Chama `CdcMemOpen()` com essa tabela.
5. Inicializa um mutex.
6. Cria o decoder com `CreateVideoDecoder()`.
7. Seleciona o formato de codec a partir do parâmetro interno `nCodecFormat`: os caminhos identificados para os tipos 1 e 2 selecionam, respectivamente, `VIDEO_CODEC_FORMAT_H264` (`0x115`) e `VIDEO_CODEC_FORMAT_H265` (`0x116`). Um terceiro caminho observado também seleciona H.264; não se deve inferir suporte AV1 a partir dele.
8. Lê quatro parâmetros de configuração por `CdcGetConfigParamterInt()`:
   - `pic_4di_num`, padrão 2;
   - `pic_4list_num`, padrão 3;
   - `pic_4rotate_num`, padrão 0;
   - `pic_4smooth_num`, padrão 3.
9. Copia o ponteiro de operações de memória para `VConfig.memops` (offset `0x80` dentro da estrutura usada nessa compilação).
10. Copia um inteiro de configuração para `VConfig.nSetProcInfoFreq` (offset `0xAC` na estrutura observada).
11. Chama `InitializeVideoDecoder(decoder, &VideoStreamInfo, &VConfig)`.
12. Se a inicialização falha, chama `DestroyVideoDecoder()` e zera o handle.

**Ressalva importante:** `initDecoder()` pode retornar zero mesmo quando `InitializeVideoDecoder()` falha. Portanto, o retorno de `initDecoder()` isoladamente não comprova que o decoder foi inicializado. O código que reproduzir esse fluxo deve verificar o handle e o resultado efetivo de `InitializeVideoDecoder()`.

A desmontagem ainda não permitiu fechar todos os campos preenchidos em `VideoStreamInfo` e `VConfig`. Não atribuir valores presumidos aos campos restantes.

## 14.5. Parâmetros de buffers e estrutura `VConfig`

A associação dos quatro parâmetros `pic_4*` aos campos de `VConfig` foi identificada na análise anterior:

| Parâmetro de configuração | Campo de `VConfig` | Valor padrão observado |
|---|---|---:|
| `pic_4di_num` | `nDeInterlaceHoldingFrameBufferNum` | 2 |
| `pic_4list_num` | `nDisplayHoldingFrameBufferNum` | 3 |
| `pic_4rotate_num` | `nRotateHoldingFrameBufferNum` | 0 |
| `pic_4smooth_num` | `nDecodeSmoothFrameBufferNum` | 3 |

Os valores são padrões lidos pelo demo via `CdcGetConfigParamterInt()`; a configuração efetiva pode ser sobrescrita por `/etc/cedarc.conf` ou pelo mecanismo de configuração utilizado pela biblioteca.

O manual de desenvolvimento do Decoder descreve `nFrameBufferNum` normalmente na faixa de 5–8 e `nAlignStride` como 32 em sua orientação geral. Isso não autoriza assumir que o demo usa esses valores: os valores efetivos ainda precisam ser extraídos da desmontagem ou de uma execução instrumentada.

## 14.6. O que a análise de `initParams()` esclareceu

A função `initParams()` foi identificada no endereço `0x6040`. A desmontagem mostra que ela prepara estruturas e buffers auxiliares por canal, incluindo quatro alocações de 2048 bytes para cada um de até cinco canais, além de inicializar contadores e campos auxiliares.

Não foi identificada nessa função a configuração de `nCodecFormat`, `VideoStreamInfo` ou `VConfig`. Esses dados são preparados em outro caminho, especialmente em `initDecoder()`. Portanto, continuar examinando `initParams()` não é a forma mais direta de descobrir os campos da configuração do decoder.

## 14.7. Detalhes da API que devem orientar o protótipo

Os headers instalados definem `VideoDecoder` como um handle opaco (`void *`). A API recebe `VideoDecoder *`, de modo que a variável do programa deve ser tratada como handle opaco e passada conforme a declaração exata do header.

As operações de memória e VE não devem ser omitidas por simplificação. O demo usa `MemAdapterGetOpsS()`, `CdcMemOpen()` e passa o ponteiro de operações para `VConfig.memops`. A interface VE é disponibilizada por `GetVeOpsS()` e estruturas relacionadas, mas ainda é necessário determinar se e como o demo configura explicitamente `veOpsS` e `pVeOpsSelf` no trecho completo da função.

O caminho de dados que o protótipo deverá implementar permanece:

```text
RequestVideoStreamBuffer
        ↓
preencher a região de entrada retornada
        ↓
SubmitVideoStreamData
        ↓
DecodeVideoStream
        ↓
NextPictureInfo / RequestPicture
        ↓
consumir ou copiar o quadro
        ↓
ReturnPicture
```

A assinatura e a semântica exatas de cada chamada devem ser respeitadas conforme `/usr/include/vdecoder.h` e o guia oficial do Decoder. Em particular, o tratamento de buffers circulares, fragmentação de pacotes, PTS, fim de fluxo e retorno de quadros precisa ser implementado; não basta enviar cada pacote FFmpeg diretamente como se fosse necessariamente um frame completo.

## 14.8. Estratégia de integração FFmpeg revisada

O FFmpeg de sistema identificado é `7.1.5-0+deb13u1`, com `libavcodec 61.19.101`. A compilação atual não possui decoder CedarX. A opção `--enable-omx` não seria substituta para a integração direta com a API proprietária `libvdecoder`.

A estratégia continua sendo criar um decoder próprio para `libavcodec` (primeiro H.264), compilar uma cópia isolada do FFmpeg e, em seguida, construir o `libmpv` contra essa cópia. O FFmpeg do sistema e a instalação original do Stremio devem permanecer intactos durante o desenvolvimento.

Para a saída, a primeira implementação pode copiar os planos NV12 do `VideoPicture` para um `AVFrame` com alinhamento e dimensões corretos. A possibilidade de exportar/importar DMA-BUF e evitar cópias deve ser investigada separadamente; não há ainda evidência suficiente para afirmar que os buffers retornados pelo decoder possam ser compartilhados diretamente com o mpv.

## 14.9. Próxima ação de engenharia

Antes de escrever o harness C, completar a leitura da desmontagem de `initDecoder()` para identificar os valores concretos dos campos de `VideoStreamInfo` e `VConfig`. O trecho prioritário é `0x4108–0x4480` de `/usr/bin/vdecoderdemo`.

O harness deverá então reproduzir a inicialização observada, validar o retorno de `InitializeVideoDecoder()`, decodificar o mesmo fluxo Annex B já testado e exportar quadros NV12. Só depois dessa validação deve começar a implementação do decoder em FFmpeg.

---

## 14.10. Nova análise da desmontagem de `initDecoder()` (0x4108–0x4480)

A desmontagem adicional confirma a sequência de inicialização e esclarece alguns offsets. A função começa chamando `AddVDPlugin()`, zera 224 bytes para `VConfig` em `sp+0x78` e 72 bytes para `VideoStreamInfo` em `sp+0x30`. Em seguida:

1. Chama `MemAdapterGetOpsS()` e guarda o ponteiro no contexto em `params+232`.
2. Chama `CdcMemOpen()` com esse ponteiro.
3. Inicializa um mutex no contexto.
4. Chama `CreateVideoDecoder()` e guarda o handle no início do contexto.
5. Lê `params+104` para selecionar o codec. O valor 1 grava `0x115` (H.264) em `VideoStreamInfo.eCodecFormat`; o valor 2 grava `0x116` (H.265); o valor 3 também grava `0x115` (H.264). O valor 3 não constitui evidência de AV1.
6. Lê os parâmetros `pic_4di_num`, `pic_4list_num`, `pic_4rotate_num` e `pic_4smooth_num`, com padrões 2, 3, 0 e 3, respectivamente.
7. Copia o ponteiro de operações de memória para `sp+0xF8`, que corresponde a `VConfig+0x80`, confirmando `VConfig.memops`.
8. Copia o inteiro de `params+240` para `sp+0x124`, correspondente a `VConfig+0xAC`, identificado no header como `nSetProcInfoFreq`.

Há também leituras de `params+256` e `params+260`, armazenadas temporariamente em `sp+0xA8` e usadas em mensagens de log. Neste trecho elas **não** são copiadas para `VideoStreamInfo`; portanto, não se deve concluir ainda que esses sejam os campos `nWidth` e `nHeight` do decoder. A estrutura `VideoStreamInfo` foi zerada e, até o endereço `0x4480`, o único campo cuja escrita foi identificada é `eCodecFormat`, no offset `sp+0x30`.

O trecho termina antes da chamada a `InitializeVideoDecoder()`. Ainda é necessário examinar a continuação da função para descobrir se os campos de resolução, taxa de quadros, `VConfig` e operações VE são preenchidos depois de `0x4480`. Os quatro parâmetros `pic_4*` são consultados nessa região, mas sua associação definitiva aos campos da estrutura deve ser confirmada pelas instruções que os copiam para `VConfig`.

**Conclusão parcial:** a inicialização da memória CedarX e a seleção de codec estão confirmadas por desmontagem. A configuração completa de `VideoStreamInfo` e `VConfig` ainda não está fechada; não escrever o harness com valores presumidos.

## 14.11. Conclusão da desmontagem de `initDecoder()` (0x4480–0x4590)

A continuação da desmontagem resolve a associação dos quatro parâmetros de buffers com os campos de `VConfig` e confirma a chamada de inicialização:

| Offset na pilha | Offset relativo a `VConfig` (base `sp+0x78`) | Campo | Valor padrão passado pelo demo |
|---:|---:|---|---:|
| `sp+224` | `+104` (`0x68`) | `nDeInterlaceHoldingFrameBufferNum` | 2 |
| `sp+228` | `+108` (`0x6C`) | `nDisplayHoldingFrameBufferNum` | 3 |
| `sp+232` | `+112` (`0x70`) | `nRotateHoldingFrameBufferNum` | 0 |
| `sp+236` | `+116` (`0x74`) | `nDecodeSmoothFrameBufferNum` | 3 |

A associação é consistente com a ordem dos campos em `/usr/include/vdecoder.h`. Os valores são padrões fornecidos a `CdcGetConfigParamterInt()` e podem ser substituídos pela configuração CedarX.

A chamada em `0x4494` recebe os argumentos conforme a convenção AArch64:

```c
InitializeVideoDecoder(
    decoder,       // x0: handle guardado no contexto
    &videoInfo,    // x1: sp + 0x30
    &vconfig       // x2: sp + 0x78
);
```

O retorno é guardado em `sp+44` (`0x2C`). Se o retorno for diferente de zero, o demo chama `DestroyVideoDecoder()` e zera o handle no contexto. Se for zero, preserva o handle. A função também inicializa um `pthread_rwlock_t` no contexto e, no caminho final mostrado, retorna zero. Portanto, o retorno de `initDecoder()` não é um indicador confiável do sucesso da inicialização CedarX: o código chamador deve verificar o resultado direto de `InitializeVideoDecoder()` e o handle resultante.

### Implicações para o harness

1. Reproduzir a sequência `AddVDPlugin()` → `MemAdapterGetOpsS()` → `CdcMemOpen()` → `CreateVideoDecoder()`.
2. Preencher `VideoStreamInfo` e `VConfig` com os campos realmente observados, incluindo os quatro contadores de buffers e `VConfig.memops`.
3. Verificar diretamente o retorno de `InitializeVideoDecoder()`; em caso de falha, destruir o decoder e encerrar a tentativa.
4. Não copiar cegamente os offsets do executável para o código C. Usar os nomes e tipos declarados nos headers, e validar o layout com `sizeof`/`offsetof` no compilador ARM64.

**Estado da investigação:** os quatro campos de holding buffers estão agora identificados. A chamada a `InitializeVideoDecoder()` e seu tratamento de falha também estão confirmados. Ainda falta identificar, a partir da desmontagem completa e dos parâmetros de execução, os campos concretos de resolução/taxa de quadros de `VideoStreamInfo` e os demais campos de `VConfig`; não assumir que os campos não vistos sejam inicializados automaticamente pela biblioteca.

## 14.12. Análise de `parserThreadFunc()` — solicitação de buffer (0x55D8–0x59A0)

A desmontagem da thread de parser mostra o mecanismo usado pelo demo para obter espaço no buffer de entrada CedarX. A função aloca inicialmente um buffer auxiliar de `0x100000` bytes (1 MiB), recupera o handle do decoder e o objeto/parser associado ao contexto, e zera uma área local de 48 bytes usada para os resultados e metadados da leitura.

Antes de solicitar espaço, o demo consulta:

```c
VideoStreamBufferSize(decoder, 0);
VideoStreamDataSize(decoder, 0);
```

Subtrai o segundo valor do primeiro para estimar o espaço livre no buffer de bitstream. A thread também lê, sob `pthread_rwlock`, uma flag no offset `+68` da estrutura de contexto. Se os quatro bits inferiores dessa flag não forem zero, o parser não prossegue naquele ciclo. Há ainda um contador de espera limitado por `0x7CF` (1999), após o qual o demo marca o bit 0 da flag e segue para o caminho de saída/erro.

Quando há espaço suficiente, a função prepara quatro endereços locais e chama `RequestVideoStreamBuffer()` com sete argumentos AArch64:

```c
RequestVideoStreamBuffer(
    decoder,             // x0
    required_size,       // x1: espaço livre calculado
    &buffer,             // x2: sp + 0x68
    &buffer_size,        // x3: sp + 0x78
    &ring_buffer,        // x4: sp + 0x70
    &ring_buffer_size,   // x5: sp + 0x7C
    0                    // x6: índice do stream
);
```

A associação dos quatro ponteiros decorre da preparação dos registradores em `0x5824–0x5854`. Se a chamada retorna valor diferente de zero, o demo espera 50 ms e tenta novamente. Em caso de retorno zero, compara `required_size` com a soma dos tamanhos das duas regiões retornadas (`buffer_size + ring_buffer_size`). Se a soma for insuficiente, registra a condição, marca a flag de erro no contexto e encerra esse caminho.

Depois da solicitação bem-sucedida, consulta `VideoStreamFrameNum(decoder, 0)`. Se o número de frames no buffer exceder `0x400` (1024), espera 50 ms antes de continuar. Em seguida chama `RawParserRead(parser, &local_read_info)`, indicando que a leitura do arquivo/parser ocorre depois de reservar espaço no buffer CedarX.

**O que este trecho comprova:**

- O demo não entrega diretamente um pacote FFmpeg à API CedarX. Primeiro consulta a ocupação do buffer e solicita regiões de escrita com `RequestVideoStreamBuffer()`.
- A API retorna duas regiões (`buffer` e `ring_buffer`), coerentes com a possibilidade de a área solicitada atravessar o fim de um buffer circular.
- O índice de stream usado nesta chamada é zero.
- `RawParserRead()` é chamado depois da reserva, mas a cópia dos bytes para as regiões retornadas e a chamada a `SubmitVideoStreamData()` ainda não aparecem no trecho analisado.
- O controle de back-pressure considera espaço livre, capacidade total das regiões devolvidas e quantidade de frames já enfileirados.

**Implicação para o harness:** implementar a solicitação de buffer respeitando as duas regiões retornadas. Não presumir que `RequestVideoStreamBuffer()` sempre devolve uma única região contígua com todo o tamanho solicitado. A próxima parte da desmontagem deve esclarecer como o demo copia os dados lidos pelo parser, preenche `VideoStreamDataInfo` e chama `SubmitVideoStreamData()`.

**Limite da conclusão:** esta região não mostra como `RawParserRead()` preenche os metadados locais nem como a estrutura `VideoStreamDataInfo` é montada. Também não identifica a origem de largura, altura ou SPS/PPS de `VideoStreamInfo`; esses pontos permanecem em aberto.

## 14.13. Continuação de `parserThreadFunc()` — leitura e submissão do bitstream (0x59A0–0x5AE0)

A desmontagem seguinte completa o caminho de alimentação do decoder. Após `RequestVideoStreamBuffer()` retornar sucesso e `RawParserRead()` preencher a estrutura local de leitura, o demo monta uma estrutura `VideoStreamDataInfo` de 64 bytes e chama `SubmitVideoStreamData()`.

### Sequência observada

1. Se `RawParserRead()` retornar erro, o demo registra a falha, marca o bit 0 da flag de contexto em `context+68` sob `pthread_rwlock` e sai do caminho de submissão.
2. Em caso de sucesso, zera 64 bytes em `sp+0x98`, usados como `VideoStreamDataInfo`.
3. Copia valores da estrutura local de leitura para os campos da estrutura CedarX:

| Campo de `VideoStreamDataInfo` | Offset relativo à estrutura (`sp+0x98`) | Origem observada |
|---|---:|---|
| `pData` | `+0` | ponteiro lido de `[sp+104]` |
| `nLength` | `+8` | valor de `[sp+136]` (32 bits) |
| `nPts` | `+16` | valor de `[sp+128]` (64 bits) |
| `nPcr` | `+24` | valor de `[sp+144]` (64 bits) |
| `bIsFirstPart` | `+32` | definido como `1` |
| `bIsLastPart` | `+36` | definido como `1` |
| `nID` | `+40` | permanece zero após `memset` |
| `nStreamIndex` | `+44` | permanece zero após `memset` |
| `bValid` | `+48` | definido como `1` |
| `bVideoInfoFlag` | `+52` | permanece zero após `memset` |
| `pVideoInfo` | `+56` | permanece NULL após `memset` |

4. Chama `SubmitVideoStreamData(decoder, &stream_data_info, 0)`, isto é, usa o handle do decoder, o endereço `sp+0x98` e índice de stream zero.
5. Se `SubmitVideoStreamData()` retornar valor diferente de zero, o demo marca novamente o bit 0 da flag de contexto e segue para o caminho de erro/saída. Se retornar zero, continua o loop de parser.

### Relação com `RequestVideoStreamBuffer()` e `RawParserRead()`

O trecho anterior mostra que os endereços locais passados como argumentos de saída de `RequestVideoStreamBuffer()` são `sp+104`, `sp+120`, `sp+112` e `sp+124`. Em seguida, `RawParserRead(parser, sp+104)` recebe a mesma área local como estrutura de leitura. O código posterior obtém dessa área o ponteiro de dados (`[sp+104]`), o tamanho (`[sp+136]`), o PTS (`[sp+128]`) e o PCR (`[sp+144]`). Isso estabelece o encadeamento entre a reserva de espaço no buffer CedarX, a leitura do parser e a montagem de `VideoStreamDataInfo`.

A definição exata da estrutura privada usada por `RawParserRead()` ainda não foi identificada; portanto, não atribuir nomes semânticos definitivos aos demais offsets dessa estrutura local. Também não se deve concluir apenas por esse trecho se `RawParserRead()` copia bytes, lê diretamente nas regiões reservadas ou administra a divisão entre buffer linear e ring buffer. A chamada recebe a estrutura já inicializada com os resultados de `RequestVideoStreamBuffer()`, mas o corpo de `RawParserRead()` precisa ser examinado para fechar esse detalhe.

### Implicações para o harness

- A submissão CedarX usa `VideoStreamDataInfo`, não apenas um ponteiro e tamanho isolados.
- Para o fluxo observado, `bIsFirstPart=1`, `bIsLastPart=1`, `bValid=1`, `nStreamIndex=0`, `nID=0`, `bVideoInfoFlag=0` e `pVideoInfo=NULL`.
- `pData`, `nLength`, `nPts` e `nPcr` devem ser preenchidos com os valores efetivamente produzidos pela etapa de leitura/reserva; não assumir que os dados estejam sempre em uma região contígua sem inspecionar `RawParserRead()`.
- O retorno de `SubmitVideoStreamData()` deve ser verificado. O demo considera qualquer retorno diferente de zero uma falha e sinaliza a flag de erro no contexto.

**Estado atualizado:** o fluxo de entrada agora está rastreado até `SubmitVideoStreamData()`. Permanecem pendentes a análise do corpo de `RawParserRead()`, a identificação de como `VideoStreamInfo` recebe dimensões e parâmetros do codec, e a desmontagem da thread de decode para documentar a chamada/ritmo de `DecodeVideoStream()`.

## 14.14. Desmontagem de `RawParserRead()` — cópia para os buffers CedarX (0x2CFC–0x2EC8)

A desmontagem de `RawParserRead(parser, local_read_info)` fecha a dúvida da seção 14.13: a função **copia explicitamente** os bytes da origem mantida pelo parser para as duas regiões previamente devolvidas por `RequestVideoStreamBuffer()`. Ela também implementa a cópia em duas partes quando o pacote ultrapassa a capacidade da primeira região.

### Layout da estrutura local confirmado pelos acessos

Considerando que `local_read_info` aponta para `sp+104` em `parserThreadFunc()`, os acessos de `RawParserRead()` permitem mapear:

| Offset relativo a `local_read_info` | Offset absoluto na pilha de `parserThreadFunc()` | Uso confirmado |
|---:|---:|---|
| `+0` | `sp+104` | ponteiro do primeiro segmento (`buffer`) devolvido por `RequestVideoStreamBuffer()`; destino da primeira cópia |
| `+8` | `sp+112` | ponteiro do segundo segmento (`ring_buffer`); destino da segunda cópia quando necessária |
| `+16` | `sp+120` | capacidade do primeiro segmento (`buffer_size`), lida como inteiro de 32 bits com extensão de sinal |
| `+24` | `sp+128` | PTS; preservado por `RawParserRead()` e posteriormente copiado para `VideoStreamDataInfo.nPts` |
| `+32` | `sp+136` | quantidade de bytes solicitada para esta leitura; usada como tamanho da cópia e depois como `VideoStreamDataInfo.nLength` |
| `+40` | `sp+144` | PCR; preservado por `RawParserRead()` e posteriormente copiado para `VideoStreamDataInfo.nPcr` |

Os campos em `+24` e `+40` não são lidos nem escritos no corpo desmontado de `RawParserRead()`; a associação com PTS/PCR decorre do uso posterior em `parserThreadFunc()` (seção 14.13). O segundo tamanho retornado por `RequestVideoStreamBuffer()` fica em outro local da área local (sp+124), mas não é consultado por `RawParserRead()` neste trecho.

### Origem dos bytes e controle de consumo

A origem é calculada a partir do objeto parser:

```c
source = parser->data + parser->offset;
```

No binário, `parser+16` fornece o ponteiro-base e `parser+24` fornece o deslocamento atual. O campo `parser+32` é o total de bytes ainda disponíveis: se for menor ou igual a zero, a função retorna `-1` sem copiar.

A cópia segue dois caminhos:

- **Leitura cabe no primeiro segmento:** se `local_read_info->length <= local_read_info->buffer_size`, faz um único `memcpy(buffer, source, length)`.
- **Leitura atravessa o limite do primeiro segmento:** copia `buffer_size` bytes para `buffer` e, em seguida, copia `length - buffer_size` bytes para `ring_buffer`, começando na origem em `source + buffer_size`.

Depois da cópia, atualiza o estado do parser:

```c
parser->offset += length;
parser->remaining -= length;
```

A função retorna zero. A desmontagem não mostra uma validação explícita de que a capacidade do segundo segmento seja suficiente para `length - buffer_size`; a thread anterior verifica a soma dos tamanhos retornados contra o espaço solicitado, mas o tamanho individual do segundo segmento não é testado dentro desta função.

### Relação com `VideoStreamDataInfo`

A análise combinada das seções 14.13 e 14.14 confirma que `RawParserRead()` escreve os bytes diretamente nos endereços retornados por `RequestVideoStreamBuffer()`. Em seguida, `parserThreadFunc()` submete ao CedarX o primeiro ponteiro (`buffer`) e o tamanho total solicitado (`length`), com `bIsFirstPart=1` e `bIsLastPart=1`. Quando houve divisão em dois segmentos, o segundo trecho está no `ring_buffer`, contíguo logicamente ao primeiro no buffer circular gerenciado pelo decoder.

Portanto, não se deve interpretar `VideoStreamDataInfo.pData` como necessariamente apontando para uma alocação linear independente que contém todos os bytes. É o ponteiro do primeiro segmento da reserva CedarX; o decoder conhece a organização circular associada ao stream buffer.

**Estado atualizado:** o caminho de alimentação está agora rastreado desde `RequestVideoStreamBuffer()` até a cópia dos bytes e `SubmitVideoStreamData()`, incluindo o caso de wrap-around. A próxima análise de maior valor é `DecodeThread()`, para identificar como o demo chama `DecodeVideoStream()` e como trata o ritmo/retorno da decodificação.

## 14.15. Início da desmontagem de `DecodeThread()` — espera por dados e flags de controle (0x4898–0x4A80)

O trecho inicial de `DecodeThread()` mostra a preparação da thread e a política de espera antes do processamento do stream. **A chamada a `DecodeVideoStream()` ainda não aparece no intervalo recebido**; a desmontagem termina em `0x4A80`, no meio de uma sequência de logging/controle.

### Preparação e espera pelo stream

- O primeiro argumento (`x0`) é guardado como ponteiro para o contexto da thread.
- O handle do decoder é obtido do primeiro ponteiro armazenado no contexto (`context[0]`).
- A thread consulta `VideoStreamFrameNum(decoder, 0)` e guarda o resultado.
- Enquanto a contagem for menor ou igual a `0xC7` (199), executa `usleep(2000)` e consulta novamente.
- O contador de tentativas é incrementado; quando ultrapassa 100, a espera por esse critério é encerrada mesmo que a contagem não tenha alcançado 200.
- Em seguida, imprime valores de diagnóstico e aguarda mais `usleep(50000)` (50 ms).

Assim, o demo procura acumular pelo menos 200 frames enfileirados antes de avançar, mas essa espera é limitada a aproximadamente 100 ciclos de 2 ms (além da espera fixa posterior). Não é uma garantia de que sempre existam 200 frames: o limite de tentativas permite prosseguir mesmo sem atingir o limiar.

### Leitura sincronizada das flags

Depois da espera, a thread adquire `pthread_rwlock` no endereço `context+8`, lê:

- `context+64`, guardado na variável local `sp+60`;
- `context+68`, guardado na variável local `sp+56`;

e libera o lock.

O fluxo seguinte testa a flag de `context+68`:

- `flags & 0xF`: verifica se algum dos quatro bits inferiores está ativo;
- se algum estiver ativo, testa `flags & 0x7` e, conforme o valor e o nível global de log, pode emitir uma mensagem de diagnóstico;
- depois testa separadamente `flags & 0x8`.

O significado semântico de cada bit ainda não deve ser ampliado além do que já foi observado nas outras rotinas. `parserThreadFunc()` marca o bit 0 em caminhos de erro de leitura/submissão; este trecho confirma que `DecodeThread()` consulta os mesmos bits sob sincronização.

### Limite desta evidência e continuação necessária

O intervalo recebido termina em `0x4A80`, no meio da função. Portanto, ainda não é possível afirmar quais argumentos são passados a `DecodeVideoStream()`, qual valor é usado para `bEndOfStream`, `bDecodeKeyFrameOnly`, `bDropBFrameIfDelay` ou `nCurrentTimeUs`, nem como o retorno é tratado.

**Próximo trecho a examinar:** continuar a desmontagem a partir de `0x4A70` até aproximadamente `0x4C40`, para cobrir a lógica de flags e localizar a chamada efetiva a `DecodeVideoStream()`.

## 14.16. Continuação de `DecodeThread()` — chamada a `DecodeVideoStream()` e tratamento dos retornos (0x4A70–0x4C40)

A continuação da desmontagem localiza a chamada efetiva à API CedarX `DecodeVideoStream()` e permite reconstruir seus argumentos e o controle do loop.

### Condição para chamar o decoder

A thread usa a cópia de `context+68` obtida sob `pthread_rwlock`:

- Se `flags & 0xF` for diferente de zero, não chama `DecodeVideoStream()` nesse ciclo. Segue para o caminho de encerramento da thread, que marca `context+68 |= 0x40`.
- Se os quatro bits inferiores forem zero, prossegue para a chamada ao decoder.

Portanto, qualquer uma das quatro flags inferiores ativas impede a chamada nessa passagem. Isso é coerente com o uso anterior do bit 0 por `parserThreadFunc()` para sinalizar erro, mas não atribui significado definitivo aos demais bits.

### Argumentos reais de `DecodeVideoStream()`

Em `0x4AE0–0x4AF4`, os registradores AArch64 são preparados assim:

```c
ret = DecodeVideoStream(
    decoder,             // x0: handle obtido do contexto
    context_plus_64,     // w1: bEndOfStream
    0,                   // w2: bDecodeKeyFrameOnly
    0,                   // w3: bDropBFrameIfDelay
    0                    // x4: nCurrentTimeUs
);
```

O valor de `context+64` havia sido lido sob o mesmo rwlock e guardado em `sp+60`. O binário usa esse valor como `bEndOfStream`. Os outros três parâmetros são explicitamente zerados. Assim, o demo não limita a decodificação a keyframes, não pede descarte de B-frames por atraso e não fornece tempo corrente à API.

### Tratamento do retorno

O retorno de `DecodeVideoStream()` é guardado em `sp+52` e tratado desta forma:

- **`0`**: não incrementa o contador de eventos especiais; volta ao caminho de sincronização e continua o loop.
- **`1` ou `3`**: incrementa o contador em `context+100` e continua o loop.
- **Valor negativo**: emite diagnóstico (dependendo do nível de log) e, sob `pthread_rwlock`, ativa `context+68 |= 0x02`.
- **`5`**: não retorna ao loop. Se `context+64 == 1`, emite a mensagem específica de fim de stream; independentemente disso, segue para o encerramento da thread.
- **Outros valores não negativos**: também seguem para o encerramento, pois não satisfazem os casos `0`, `1` ou `3` que retornam ao loop.

No caminho de encerramento, a thread adquire o rwlock e ativa `context+68 |= 0x40`. O código, portanto, usa o bit `0x40` como sinalização de que a `DecodeThread` terminou; essa interpretação é sustentada pelo local e pela posição da escrita no fluxo de saída, embora o nome simbólico do campo não esteja disponível no binário.

### Consequências para o harness

A chamada mínima que reproduz os parâmetros observados no demo é:

```c
int ret = DecodeVideoStream(decoder, end_of_stream, 0, 0, 0);
```

O harness deve executar essa chamada em loop, respeitando a sincronização e a condição de erro/encerramento. Não deve tratar todo retorno diferente de zero como falha: `1` e `3` são estados transitórios no fluxo deste demo; valores negativos são tratados como erro; `5` encerra o ciclo. A semântica completa dos códigos deve ser conferida no header/documentação da versão CedarX instalada antes de atribuir nomes a eles.

### Estado da reconstrução

Com as seções 14.12–14.16, já temos o caminho principal de entrada e decodificação: o parser solicita espaço ao SBM, copia o bitstream para o buffer (inclusive wrap-around), submete `VideoStreamDataInfo` e a thread de decode chama `DecodeVideoStream()` com os parâmetros acima. Falta reconstruir o consumo das imagens decodificadas (`NextPictureInfo`/`RequestPicture`, `ReturnPicture`) e o tratamento de encerramento/liberação dos buffers.

**Próximo trecho a examinar:** a continuação de `DecodeThread()` após `0x4C40`, para fechar a rotina e identificar se há operações adicionais de sincronização ou espera; depois, a análise de `displayPictureThreadFunc()` para reconstruir a saída de `VideoPicture`.

## 14.17. Fim de `DecodeThread()` e início de `savePicture()` (0x4C38–0x4D20)

A nova desmontagem fecha a rotina de decodificação e começa a revelar a rotina usada para salvar uma `VideoPicture`.

### Encerramento de `DecodeThread()`

Em `0x4C10–0x4C34`, a thread:

1. Adquire o `pthread_rwlock` localizado em `context+8`.
2. Lê a palavra de flags em `context+68`.
3. Executa `flags |= 0x40` e grava o resultado.
4. Libera o rwlock.

Em seguida, `0x4C38–0x4C78` pode imprimir uma mensagem de diagnóstico, dependendo do nível de log. Finalmente, `0x4C78–0x4C7C` chama `pthread_exit(NULL)`.

Isso confirma que o bit `0x40` em `context+68` é ativado no caminho de saída normal da `DecodeThread`, além do caminho de saída observado anteriormente quando a thread detecta flags inferiores ativas. A interpretação mais segura é tratá-lo como sinalizador de término dessa thread; não atribuir nomes simbólicos adicionais sem encontrar a estrutura-fonte ou outros usos que confirmem a semântica.

### Início de `savePicture()`

A função começa em `0x4C80`. A convenção AArch64 indica os argumentos iniciais em `x0` e `x1`; o código os salva na pilha como:

- argumento 1 (`x0`) em `sp+56`;
- argumento 2 (`x1`) em `sp+48`.

Depois, em `0x4C94–0x4C9C`, lê um ponteiro de `argumento_2 + 128` e o guarda em `sp+88`. Esse ponteiro é passado a `fopen()` em `0x4CFC–0x4D04`, com o modo armazenado em uma string constante do binário. Portanto, o campo no deslocamento 128 do objeto apontado pelo segundo argumento fornece o caminho/nome do arquivo que será aberto.

O primeiro argumento é testado contra NULL em `0x4CA8–0x4CB0`:

- se for NULL, a função emite um diagnóstico (quando o nível de log permite) e salta para o bloco de saída em `0x4FA4`;
- se não for NULL, tenta abrir o arquivo com `fopen()`; se a abertura falhar, segue para o tratamento de erro iniciado após `0x4D14`.

**Limite da conclusão:** este intervalo não mostra ainda como os planos de pixels da `VideoPicture` são gravados, nem permite identificar com segurança o tipo semântico do primeiro argumento. Também não basta, isoladamente, para afirmar que `argumento_2` é diretamente uma `VideoPicture`; isso será confirmado seguindo os call sites e a continuação da função.

### Próximo trecho a examinar

Continuar `savePicture()` de `0x4D18` até `0x4FA8` para identificar o formato de escrita, os ponteiros/strides usados e a liberação do arquivo. Em seguida, examinar `displayPictureThreadFunc()` para localizar o produtor dos argumentos de `savePicture()` e a sequência de aquisição/retorno das imagens CedarX.

## 14.18. `savePicture()` — sincronização de cache e gravação dos planos (0x4D18–0x4FA8)

A desmontagem recebida permite reconstruir a forma como o demo exporta os dados de imagem de uma picture para arquivo. O primeiro argumento da função é tratado como uma estrutura de picture: o código lê largura em `picture+12`, altura em `picture+16` e o endereço-base dos pixels em `picture+80`. A identificação formal de todos os campos deve ser conferida com `VideoPicture` do header instalado, mas esses offsets são consistentes com a estrutura usada pela API.

### Tamanho do plano Y

Em `0x4D5C–0x4D80`, calcula:

```c
y_size = width * height;
quarter_size = y_size / 4;
```

O valor `y_size` é usado como tamanho do primeiro plano escrito no arquivo. O código não usa o campo de stride para calcular esse tamanho: grava `width*height` bytes a partir do endereço-base.

### Invalidação/sincronização do cache

Em `0x4E10–0x4E5C`, verifica o campo `contexto+232`:

- se for diferente de zero, usa `contexto+240` como tamanho para a operação de cache;
- se for zero, calcula `width*height*3/2`.

Em `0x4E60–0x4E7C`, chama:

```c
CdcMemFlushCache(
    contexto->memops,   // campo em contexto+232
    picture->data,      // campo em picture+80
    flush_size
);
```

A interpretação do primeiro campo como ponteiro de operações de memória é compatível com a API CedarX (`ScMemOpsS *`), mas deve ser confirmada no layout do objeto de contexto antes de nomear definitivamente o campo. A operação ocorre antes de ler os pixels para o arquivo, garantindo que a CPU veja os dados atualizados pelo hardware.

### Escrita dos planos

Depois da sincronização, o código executa `fwrite()` do primeiro plano:

```c
fwrite(picture->data, 1, width * height, file);
```

Em seguida consulta o formato de saída guardado em `contexto+256`:

- Para os valores **1** e **4**, grava dois planos adicionais, cada um com `width*height/4` bytes. O endereço do primeiro plano de crominância é `data + width*height`; o segundo é `data + width*height + width*height/4`.
- Para os valores **5** e **6**, grava um único plano adicional com `width*height/2` bytes, começando em `data + width*height`.
- Outros valores não provocam escrita de plano de crominância nesse trecho.

Isso distingue dois layouts de saída: um layout planar 4:2:0 (Y mais dois planos de crominância) para os códigos 1/4 e um layout semiplanar 4:2:0 (Y mais um plano UV/VU intercalado) para os códigos 5/6. **A desmontagem isolada não permite decidir qual dos dois códigos semiplanares corresponde a NV12 e qual a NV21, nem qual a ordem U/V dos códigos planares.** Essa associação precisa ser verificada nos enums do header e, se necessário, por inspeção dos bytes gerados.

Finalmente, `fclose()` é chamado em `0x4F98–0x4F9C`.

### Consequências para o harness e FFmpeg

1. O caminho demonstrado para exportar uma imagem usa o endereço-base da picture, sincroniza o cache com `CdcMemFlushCache()` e depois lê Y e crominância.
2. Para o formato semiplanar, o demo considera a crominância contígua após `width*height`, com tamanho `width*height/2`.
3. Para o formato planar, considera dois planos de `width*height/4` cada.
4. O código de demonstração grava dimensões compactas (`width*height`) e não evidencia tratamento de padding por stride na escrita. Isso não prova que os buffers CedarX sejam sempre compactos; o harness deve respeitar `nLineStride` e os endereços/offsets reais da `VideoPicture`, em vez de copiar cegamente a lógica simplificada de `savePicture()`.
5. Os códigos numéricos de `ePixelFormat` ainda precisam ser mapeados aos nomes do header instalado. Não fixar `NV12` apenas porque o teste anterior foi salvo com extensão `.nv12`; confirmar o enum `6` antes de codificar o formato no decoder FFmpeg.

### Próxima análise

Examinar `displayPictureThreadFunc()` para identificar como a thread obtém a picture, que estrutura passa a `savePicture()`, como trata `NextPictureInfo()`/`RequestPicture()` e quando chama `ReturnPicture()`. Em paralelo, conferir no header a enumeração de `ePixelFormat` para mapear com precisão os códigos 1, 4, 5 e 6.

## 14.19. `displayPictureThreadFunc()` — inicialização, controle de parada e aquisição de picture (0x4FBC–0x51C0)

A desmontagem cobre o começo da thread que consome os frames decodificados.

### Inicialização da thread

- O argumento recebido em `x0` é guardado em `sp+24` e depois em `sp+88`; ele é o contexto da thread.
- Em `0x4FFC–0x5000`, a função lê o ponteiro armazenado no primeiro campo desse contexto e o guarda em `sp+96`. Esse ponteiro é usado mais adiante como primeiro argumento de `RequestPicture()`; pelo uso e pela API, trata-se do handle `VideoDecoder`.
- Em `0x5004–0x500C`, chama `calloc(5, 24)`, reservando espaço zerado para cinco elementos de 24 bytes. A finalidade exata dessa tabela ainda não aparece neste trecho; não atribuir-lhe um tipo sem examinar os usos posteriores.
- Se `calloc()` falhar, a thread adquire o rwlock em `context+8`, ativa `context+68 |= 0x04`, libera o lock, registra diagnóstico e termina pelo bloco comum de saída (`0x55AC`). O bit `0x04` é, portanto, um sinal de falha observado no caminho de alocação desta thread; o nome semântico ainda não está confirmado.

### Preparação e verificação das flags

Quando a alocação tem sucesso, a função:

1. zera o campo de 32 bits em `context+88` (`0x50D0–0x50D4`);
2. chama `GetNowUs()` e guarda o timestamp em `sp+72`;
3. aguarda 100 microssegundos (`usleep(100)`);
4. lê `context+68` sob o rwlock;
5. testa `flags & 0x0F`.

Se algum dos quatro bits inferiores estiver ativo, não solicita uma picture nesse ciclo e segue para o caminho de diagnóstico/saída. Se nenhum estiver ativo, prossegue para adquirir uma imagem.

### Aquisição da picture

Em `0x5160–0x5168`, prepara o segundo argumento como zero e chama:

```c
picture = RequestPicture(decoder, 0);
```

O handle do decoder vem do ponteiro salvo em `sp+96`. O resultado é guardado em `sp+112` e testado contra NULL em `0x5170–0x5178`. A assinatura do header declara retorno `VideoPicture *`; portanto, o ponteiro retornado é tratado como uma picture. Se retornar NULL, a thread desvia para o tratamento posterior em `0x5468`. O nome da API sugere uma solicitação de picture, mas a distinção exata entre picture recém-solicitada e picture pronta para exibição deve ser validada pela documentação e pelo restante do fluxo; não equiparar automaticamente essa chamada a `NextPictureInfo()`.

A partir de `0x517C`, o código começa a registrar informações da picture quando o nível de log permite. O trecho fornecido termina em `0x51C0`, antes de mostrar a chamada subsequente a `savePicture()` e a devolução da picture ao decoder.

### O que ainda não está demonstrado

- O uso dos cinco elementos de 24 bytes alocados por `calloc(5, 24)`.
- Se `NextPictureInfo()` também é chamado nesta thread, e em que ordem em relação a `RequestPicture()`.
- Os argumentos exatos passados a `savePicture()`.
- O ponto em que `ReturnPicture(decoder, picture)` é chamado. A picture deve ser devolvida conforme o contrato de ownership da API, mas a posição exata no fluxo ainda precisa ser vista no restante da desmontagem.

### Próximo trecho

Continuar a desmontagem de `displayPictureThreadFunc()` a partir de `0x51C0` para acompanhar o processamento da picture adquirida, identificar a chamada de `savePicture()` e localizar `ReturnPicture()`.

## 14.20. `displayPictureThreadFunc()` — metadados da picture e contagem de frames (0x51C0–0x53A0)

Este trecho continua o processamento da `VideoPicture *` retornada por `RequestPicture()`.

### Campos consultados na `VideoPicture`

Os acessos ao ponteiro guardado em `sp+112` mostram:

- offset `+8`: campo de 32 bits usado nos logs como formato de pixel (`ePixelFormat`, conforme o layout de `VideoPicture` nos headers);
- offset `+12`: largura (`nWidth`);
- offset `+16`: altura (`nHeight`).

Em `0x5234–0x5274`, a função prepara e emite um log com largura e altura. Em `0x528C–0x52C0`, emite outro log com o valor do campo em `+8`, que corresponde ao formato de pixel. O trecho não altera esses campos: apenas os lê para diagnóstico.

### Contador e medição de tempo

- `sp+44` é inicializado com zero no início da thread (`0x4FEC`) e é incrementado em `0x5328–0x5330` após o processamento de cada iteração com picture não nula.
- `GetNowUs()` é chamado em `0x5304`; a diferença em relação ao timestamp guardado em `sp+72` é calculada em `0x5308–0x5314`.
- Essa duração é acumulada no campo local de 64 bits em `sp+80` (`0x5318–0x5324`).
- O contador é comparado com dois valores de 32 bits do contexto: `context+224` e `context+228` (`0x5334–0x5364`). Esses valores correspondem aos parâmetros de quantidade de buffers de desentrelaçamento e de exibição já observados em `initDecoder()` e copiados para `VConfig` nos offsets respectivos. Aqui são usados como limites de contagem; não confundir esses limites com a quantidade total de frames do stream.

Quando o contador está no intervalo entre `context+224` e a soma de `context+224` com `context+228`, o código pode emitir um log adicional (`0x5368–0x539C`). O trecho termina antes do destino do desvio em `0x53B8`, portanto ainda não permite concluir o que acontece com a picture nesse ponto.

### Limites da conclusão

Este intervalo confirma que a thread consulta largura, altura e formato de pixel da picture e mantém uma contagem/medição temporal. Ele **não** mostra ainda a chamada a `savePicture()`, `NextPictureInfo()` ou `ReturnPicture()`. Também não revela se os cinco registros alocados no início da thread guardam metadados de frames ou outro estado; isso depende dos acessos posteriores.

### Próximo trecho

Continuar a desmontagem a partir de `0x53A0` para acompanhar o destino do contador, a utilização da tabela alocada e as chamadas de processamento/devolução da picture.

## 14.21. `displayPictureThreadFunc()` — exportação seletiva, lista de pictures e encerramento (0x53A0–0x55AC)

Este trecho completa a maior parte do ciclo de consumo das pictures.

### Chamada a `savePicture()` e seleção por faixa

Em `0x53AC–0x53B4`, a função chama:

```c
savePicture(picture, context);
```

A ordem dos argumentos é confirmada pelo AArch64 ABI e pelo prólogo de `savePicture()`: `x0` (picture) é salvo em `sp+56`, e `x1` (contexto) em `sp+48`. Isso identifica definitivamente o primeiro argumento como `VideoPicture *` e o segundo como o contexto do demo. O campo `context+128` lido no início de `savePicture()` é, portanto, um caminho/nome de arquivo guardado no contexto.

A chamada não ocorre para todas as pictures. O contador local `sp+44` é comparado com `context+224` e com `context+224 + context+228`. Quando está no intervalo inclusivo inferior e exclusivo superior — isto é, `counter >= context+224` e `counter < context+224 + context+228` — o código passa pelo bloco de log em `0x5368–0x53A8` e chama `savePicture()`. Fora dessa faixa, desvia diretamente para `addPictureToList()`.

Após `savePicture()`, o contador é comparado com `context+92`. Se `counter >= context+92`, o código ativa `context+68 |= 0x08` sob rwlock e inicia o encerramento. Assim, `context+92` funciona como limite de quantidade de pictures processadas nesta thread; a semântica nominal do campo ainda deve ser confirmada em `initParams()`/headers.

### Inserção na lista de pictures

Se o limite não foi atingido, a função chama `addPictureToList()` em `0x5444–0x5460` com cinco argumentos:

```c
addPictureToList(
    decoder,             // x0: handle VideoDecoder
    picture_list,        // x1: memória alocada por calloc(5, 24)
    &local_arg_a,        // x2: endereço em sp+56
    &local_arg_b,        // x3: endereço em sp+64
    picture              // x4: VideoPicture *
);
```

Os dois argumentos intermediários são endereços de variáveis locais; seus significados exatos dependem da implementação de `addPictureToList()`. A chamada demonstra que a tabela alocada com `calloc(5, 24)` é passada para essa função e participa da gestão das pictures. Não concluir ainda que ela seja uma fila FIFO ou que controle diretamente o retorno ao decoder: isso exige desmontar `addPictureToList()`.

Depois da chamada, o fluxo volta ao ciclo de espera em `0x50E0`.

### Tratamento de `RequestPicture() == NULL` e drenagem no encerramento

Quando `RequestPicture(decoder, 0)` retorna NULL, a thread lê as flags `context+68` sob rwlock e testa o bit `0x40`:

- Se o bit `0x40` ainda não estiver ativo, volta a aguardar e tenta novamente.
- Se estiver ativo, chama `ValidPictureNum(decoder, 0)` e guarda o número retornado em `sp+48`.
- Se `ValidPictureNum()` retornar valor positivo, volta ao ciclo para continuar buscando pictures.
- Se retornar zero ou negativo, segue para a saída.

Isso demonstra um mecanismo de drenagem: após o sinal de parada (`0x40`), a thread continua solicitando pictures enquanto o decoder reportar pictures válidas pendentes. Não sair imediatamente ao receber NULL evita abandonar frames que ainda estejam disponíveis.

### Contabilização e devolução final

No caminho comum de saída:

1. Soma o contador local `sp+44` ao acumulador `context+88` (`0x5508–0x551C`).
2. Ativa `context+68 |= 0x80` sob rwlock (`0x5520–0x5548`).
3. Chama `ReturnAllPicture(decoder, picture_list)` (`0x554C–0x5558`).

Portanto, a thread sinaliza em `context+68` o bit `0x80` ao concluir e entrega a tabela de pictures à rotina `ReturnAllPicture()`. **Este trecho não mostra chamadas diretas a `ReturnPicture()`**; a devolução individual pode estar dentro de `addPictureToList()` ou de `ReturnAllPicture()`. O próximo passo é desmontar essas duas rotinas para estabelecer exatamente quando cada `VideoPicture` é devolvida e como a tabela de cinco entradas é organizada.

### Fluxo consolidado até aqui

```text
RequestPicture(decoder, 0)
    ├── NULL
    │    ├── parada ainda não sinalizada (bit 0x40): aguarda e tenta novamente
    │    └── parada sinalizada: consulta ValidPictureNum()
    │         ├── > 0: continua drenando pictures
    │         └── <= 0: encerra o loop
    └── picture válida
         ├── em faixa selecionada: savePicture(picture, context)
         ├── se limite context+92 atingido: ativa bit 0x08 e encerra
         └── caso contrário: addPictureToList(decoder, list, ..., picture)

saída:
    atualiza context+88
    ativa bit 0x80
    ReturnAllPicture(decoder, picture_list)
```

### Próxima análise

Desmontar `addPictureToList()` (`0x45C0`) e `ReturnAllPicture()` (`0x4834`). Isso deve esclarecer o ownership dos buffers, a chamada efetiva a `ReturnPicture()` e se as cinco entradas representam pictures retidas para posterior devolução.

## 14.22. `addPictureToList()` — fila encadeada e devolução com atraso (0x45C0–0x4830)

A desmontagem confirma que `addPictureToList()` administra uma lista encadeada de registros de 24 bytes e devolve ao decoder a picture mais antiga quando a lista passa a conter mais de um elemento.

### Argumentos e estrutura de cada entrada

Pelo prólogo e pelos usos dos registradores AArch64, a assinatura efetiva é compatível com:

```c
addPictureToList(
    VideoDecoder *decoder,  // x0
    Entry *entries,         // x1: vetor de cinco entradas
    Entry **head,           // x2: ponteiro para a cabeça da lista
    Entry **tail,           // x3: ponteiro para a cauda da lista
    VideoPicture *picture   // x4
);
```

A função lê inicialmente `*head` e `*tail` (`0x45E0–0x45F4`). Para cada entrada, calcula `entries + i * 24` (`0x465C–0x4678`) e testa o inteiro em `+8`. Se esse campo estiver zerado, reserva a entrada, grava `1` em `+8` e guarda o ponteiro da picture em `+0` (`0x4680–0x46C4`). O campo `+16` é usado como ponteiro para a próxima entrada.

Assim, o layout observado de cada slot é:

| Offset | Conteúdo | Evidência de uso |
|---|---|---|
| `+0` | `VideoPicture *` | gravado ao reservar; lido antes de `ReturnPicture()` |
| `+8` | flag de slot ocupado | testado para procurar slot livre; definido como 1 e zerado ao liberar |
| `+16` | próximo `Entry *` | encadeamento da lista |

O vetor tem cinco slots, coerente com `calloc(5, 24)` observado na thread de exibição.

### Inserção na lista

Se cabeça e cauda estiverem vazias, o slot recém-reservado passa a ser ambos (`0x46E0–0x4708`). Caso já exista uma lista, o slot é anexado ao final: `tail->next = new_entry`, `new_entry->next = NULL`, e a cauda passa a apontar para a nova entrada (`0x470C–0x4724`).

Em seguida, a função percorre os links a partir da cabeça e conta os elementos (`0x4728–0x4770`). Se houver mais de um elemento, remove a cabeça da lista: guarda o antigo `head->next` como nova cabeça, zera `old_head->next` e limpa a flag de slot ocupado (`0x477C–0x479C`). Depois espera 2 ms (`usleep(2000)`) e chama:

```c
ReturnPicture(decoder, old_head->picture);
```

A chamada está em `0x47F8–0x4808`. Finalmente, grava a cabeça e a cauda atualizadas nas variáveis apontadas pelos argumentos `x2` e `x3` (`0x480C–0x4820`).

### Consequência para o ciclo de vida dos buffers

O comportamento observado é de uma fila FIFO com retenção de uma picture: ao inserir uma nova picture, se a fila ficar com mais de uma entrada, a mais antiga é retirada e devolvida por `ReturnPicture()`. Portanto, em operação normal, a função não acumula indefinidamente as pictures; mantém a mais recente retida e devolve a anterior após um atraso fixo de 2 ms.

A existência de cinco slots fornece capacidade para armazenar até cinco registros, mas **não significa que cinco pictures sejam normalmente retidas simultaneamente**. A lógica de remoção da cabeça quando a contagem excede um mantém a fila em um elemento após cada inserção bem-sucedida, desde que a função seja chamada sequencialmente e encontre um slot livre.

Se a busca não encontrar slot livre, `sp+72` permanece NULL; o comportamento desse caminho deve ser interpretado com cautela e pode gerar uma inserção inválida. Não é necessário reproduzi-lo no decoder próprio: a integração deve implementar gestão de referências/buffers robusta, guiada pela API e pelos resultados do harness, não copiar bugs potenciais do demo.

### Limite desta análise

A função confirma a chamada individual a `ReturnPicture()` e a política FIFO com retenção de uma picture. Ainda falta examinar `ReturnAllPicture()` (`0x4834`) para saber como a entrada que permanece na lista é devolvida no encerramento e se há tratamento adicional de lista vazia ou de slots.

## 14.23. `ReturnAllPicture()` — devolução de todas as pictures retidas (0x4834–0x4894)

A desmontagem confirma que `ReturnAllPicture(decoder, head)` percorre a lista encadeada de entradas criada por `addPictureToList()` e chama `ReturnPicture()` para cada `VideoPicture` ainda retida.

### Fluxo observado

1. Guarda `decoder` (x0) e o ponteiro inicial da lista (x1).
2. Inicializa um contador local em zero (`0x4844`).
3. Enquanto o ponteiro da entrada atual não for NULL (`0x4880–0x4888`):
   - lê `entry->picture` no offset `+0`;
   - chama `ReturnPicture(decoder, entry->picture)` (`0x484C–0x485C`);
   - zera a flag `entry->occupied` no offset `+8` (`0x4860–0x4868`);
   - avança para `entry->next`, no offset `+16` (`0x4868–0x4870`);
   - incrementa o contador (`0x4874–0x487C`).
4. Retorna o número de entradas percorridas (`0x488C–0x4894`).

Pseudocódigo equivalente ao comportamento observado:

```c
int ReturnAllPicture(VideoDecoder *decoder, Entry *entry)
{
    int count = 0;

    while (entry != NULL) {
        ReturnPicture(decoder, entry->picture);
        entry->occupied = 0;
        entry = entry->next;
        count++;
    }

    return count;
}
```

### Conclusão sobre ownership e encerramento

Combinando esta rotina com `addPictureToList()`:

- cada picture inserida na lista passa a ser responsabilidade da fila temporária do demo;
- quando a fila contém mais de uma entrada, `addPictureToList()` remove a mais antiga e chama `ReturnPicture()` nela;
- no encerramento, `ReturnAllPicture()` devolve a(s) picture(s) que ainda estiver(em) na lista;
- a rotina de limpeza zera a flag de ocupação, mas não zera explicitamente os campos `picture` ou `next`; isso não altera o fato de que o percurso termina pelo ponteiro `next` e a cabeça/cauda deixam de ser usadas após a thread terminar.

A desmontagem não mostra tratamento do valor de retorno de `ReturnPicture()` nem sincronização própria dentro de `ReturnAllPicture()`. Não se deve presumir que essa rotina possa ser chamada simultaneamente com inserções: o demo chama-a no caminho de encerramento da thread de exibição, depois de sair do loop de aquisição.

### Ciclo de vida consolidado

```text
RequestPicture(decoder, 0)
    -> picture válida
    -> savePicture(picture, context), quando selecionada
    -> addPictureToList(..., picture)
         -> insere no fim da lista
         -> se tamanho > 1:
              remove a cabeça
              ReturnPicture(decoder, picture_antiga)

encerramento da displayPictureThreadFunc:
    -> ReturnAllPicture(decoder, head)
         -> ReturnPicture() para cada entrada remanescente
         -> limpa flag occupied
```

**Implicação para o decoder FFmpeg/CedarX:** toda `VideoPicture` obtida deve ter um caminho de devolução definido. Não basta manter ponteiros para `pData0/pData1` e descartá-los quando o frame for consumido. A integração deve manter a referência CedarX válida enquanto o frame estiver em uso pelo consumidor e chamar `ReturnPicture()` somente quando essa referência puder ser liberada. A política FIFO de uma picture do demo é um exemplo de retenção, não uma regra obrigatória para o decoder FFmpeg; a duração real deve ser governada pelo ciclo de referências do FFmpeg e pela capacidade do FBM.

## 14.24. `displayPictureThreadFunc()` — ligação entre `savePicture()`, a lista de pictures e a devolução dos buffers (0x53A0–0x5558)

Os novos trechos de `displayPictureThreadFunc()` permitem identificar os argumentos usados em `savePicture()` e fechar o ciclo de ownership das pictures.

### Identificação dos argumentos de `savePicture()`

Em `0x53AC–0x53B4`, o caller executa:

```c
savePicture(local_picture, context);
```

Pela convenção AArch64, `x0` recebe o primeiro argumento e `x1` o segundo. Os valores carregados são:

- `x0 = [sp + 112]`;
- `x1 = [sp + 88]`.

O segundo argumento é o contexto da thread: o mesmo ponteiro é usado logo depois para acessar o campo `+68` e o rwlock em `+8` (`0x5410–0x5434`), e também o campo `+92` (`0x53B8–0x53C8`). Isso coincide com o contexto já observado em outras partes do demo. O primeiro argumento é a picture: o mesmo valor `[sp+112]` é passado como quinto argumento de `addPictureToList()` (`0x5444–0x5460`); dentro dessa função, ele é armazenado no campo `entry+0` e posteriormente entregue a `ReturnPicture(decoder, picture)` (`0x47F8–0x4808`). Como `ReturnPicture()` recebe `VideoPicture *`, fica estabelecido que o primeiro argumento de `savePicture()` também é `VideoPicture *`.

Portanto, a assinatura efetiva observada é compatível com:

```c
savePicture(VideoPicture *picture, DemoContext *context);
```

Os nomes `DemoContext` e `picture` são nomes descritivos usados nesta documentação, não nomes de tipos C recuperados do binário.

### Fluxo após salvar a picture

Depois de `savePicture()`, o código compara um contador local (`[sp+44]`) com o inteiro em `context+92` (`0x53B8–0x53C8`). Se o limite não foi atingido, obtém o tempo atual e chama `addPictureToList()` com o decoder, a estrutura de slots, os ponteiros para cabeça e cauda e a picture (`0x543C–0x5460`). Assim, a picture é salva antes de ser inserida na fila de retenção.

Se o limite configurado já foi atingido, o código ativa `context->flags |= 0x08` sob o rwlock (`0x540C–0x5438`), sinalizando o caminho de encerramento/limite da thread. O significado nominal desse bit ainda não foi localizado em header ou documentação; não atribuir um nome sem confirmação.

### Encerramento e devolução das pictures

No caminho de saída, a thread:

1. lê `context+88` e soma o contador local `[sp+44]` (`0x5508–0x551C`);
2. ativa o bit `0x80` em `context+68`, sob o rwlock (`0x5520–0x5548`);
3. chama `ReturnAllPicture(decoder, head)` (`0x554C–0x5558`).

O bit `0x80` é registrado como flag observada, sem nome semântico confirmado. A chamada final a `ReturnAllPicture()` confirma que as pictures ainda retidas na lista são devolvidas ao decoder antes de a thread concluir.

### Ciclo de vida comprovado

```text
VideoPicture *picture
    -> savePicture(picture, context)
    -> addPictureToList(..., picture)
         -> se a fila exceder uma entrada:
              ReturnPicture(decoder, picture_antiga)
    -> encerramento:
         ReturnAllPicture(decoder, head)
              -> ReturnPicture() para cada picture restante
```

Isso confirma que `savePicture()` recebe um `VideoPicture *`, mas ainda não revela como os planos de imagem são copiados para o arquivo. Para determinar formato, strides, offsets e tamanho efetivamente gravado, é necessário continuar a desmontagem de `savePicture()` e correlacioná-la aos campos de `VideoPicture` do header.

### Observação sobre `initDecoder()` (0x4108–0x4480)

O trecho reenviado confirma a sequência já documentada: `AddVDPlugin()`, zeragem local de `VConfig` (224 bytes) e `VideoStreamInfo` (72 bytes), obtenção de `ScMemOpsS` por `MemAdapterGetOpsS()`, `CdcMemOpen()`, criação do decoder e seleção de `eCodecFormat` conforme o parâmetro de codec. Também confirma a leitura dos parâmetros `pic_4di_num` (default 2), `pic_4list_num` (default 3), `pic_4rotate_num` (default 0) e `pic_4smooth_num` (default 3). Não há informação nova nesse trecho sobre strides ou formato de saída; os campos restantes de `VConfig` e `VideoStreamInfo` precisam ser acompanhados na continuação da função.

## 14.25. `savePicture()` — layout de escrita dos planos e invalidação de cache (0x4D18–0x4FA0)

A desmontagem de `savePicture()` fecha a análise da gravação dos frames. O primeiro argumento é `VideoPicture *picture` e o segundo é o contexto do demo, conforme identificado pelo caller em `displayPictureThreadFunc()` (seção 14.24).

### Dimensões e tamanhos calculados

O código lê:

- `picture+12` como a largura usada no cálculo;
- `picture+16` como a altura usada no cálculo;
- `picture+80` como o endereço do primeiro plano, passado a `CdcMemFlushCache()` e usado como origem dos `fwrite()`.

Calcula `Y_size = width * height` e `quarter_size = Y_size / 4`. Portanto, a rotina grava os planos usando dimensões visíveis width × height, sem aplicar explicitamente `nLineStride` no cálculo desses comprimentos. Isso descreve o comportamento do demo; não prova que o stride retornado pelo decoder seja sempre igual à largura. A compatibilidade observada no teste 1920×816 deve ser validada também para resoluções/alinhamentos diferentes.

### Invalidação de cache

Antes de gravar, chama:

```c
CdcMemFlushCache(context->memops, picture->pData0, flush_size);
```

O ponteiro `context->memops` corresponde ao campo `+232` do contexto, já identificado em `initDecoder()`. O endereço de dados é carregado de `picture+80`. O tamanho passado é:

- `picture+240`, se o campo de 32 bits em `picture+232` for diferente de zero;
- caso contrário, `width * height * 3 / 2`.

Os campos `picture+232` e `picture+240` não foram semanticamente identificados no header consultado; ficam registrados apenas como offsets observados. Não inferir que sejam dimensões ou strides.

### Layout efetivamente escrito

O primeiro `fwrite()` sempre grava `Y_size` bytes a partir de `picture->pData0`.

Em seguida, o formato de saída configurado no contexto (`context+256`) seleciona a escrita da crominância:

| `context+256` | Operação observada após o plano Y | Layout compatível |
|---:|---|---|
| 1 ou 4 | grava `quarter_size` bytes em `pData0 + Y_size`, depois mais `quarter_size` bytes em `pData0 + Y_size + quarter_size` | dois planos de crominância separados (4:2:0 planar) |
| 5 ou 6 | grava `2 * quarter_size` bytes em `pData0 + Y_size` | crominância intercalada em um único plano (4:2:0 semiplanar) |
| outros | não há gravação adicional de crominância neste trecho | formato não caracterizado por esta rotina |

A rotina fecha o arquivo com `fclose()` em todos os caminhos de escrita mostrados.

### Relação com o teste NV12

O teste anterior executou `vdecoderdemo` com `-outFmat 6`, recebeu `Picture Size: 1920x816` e produziu um arquivo de 2.350.080 bytes, exatamente `1920 * 816 * 3 / 2`. A desmontagem mostra que o ID 6 segue o caminho semiplanar: plano Y seguido por um bloco de crominância com metade do tamanho de Y. Isso é consistente com NV12, mas a distinção entre NV12 e NV21 (ordem UV versus VU) ainda precisa ser confirmada pelo enum de formato nos headers ou por inspeção dos valores dos pixels. Portanto, o que está comprovado pelo binário é **4:2:0 semiplanar**, não ainda a ordem UV/VU.

### Implicações para a integração FFmpeg

1. A picture entregue pelo CedarX contém um bloco de dados acessível por `pData0`; neste caminho de saída, o demo trata Y e crominância como regiões contíguas a partir desse endereço.
2. Para formatos 1/4, a crominância é escrita como dois planos contíguos; para 5/6, como um plano intercalado.
3. A rotina chama `CdcMemFlushCache()` antes de ler os bytes pela CPU. A integração não deve omitir a operação de coerência de cache quando usar memória CedarX não coerente.
4. O demo não usa `nLineStride` nos tamanhos nem nos offsets apresentados neste trecho. Não copiar essa simplificação para o decoder FFmpeg sem confirmar o stride real e o alinhamento de cada `VideoPicture`.
5. Para o decoder FFmpeg, a implementação deverá expor linhas/planos com `linesize[]` corretos. Se o buffer CedarX tiver padding por linha, será necessário copiar linha a linha ou configurar adequadamente o AVFrame; não assumir `linesize[0] == width` apenas porque o arquivo de teste 1920×816 teve o tamanho compacto esperado.

**Conclusão:** o caminho de saída do demo fornece um modelo funcional para exportar dados 4:2:0 a partir de `VideoPicture`, incluindo flush de cache e os dois layouts de crominância. A identificação exata de NV12 versus NV21, os valores dos enums e a relação entre largura e `nLineStride` continuam pendentes de confirmação nos headers.

## 14.26 — Confirmação dos enums de pixel format em `vbasetype.h`

A saída do comando `grep` em `/usr/include/vbasetype.h` identifica formalmente os valores de `enum EPIXELFORMAT`:

| Valor | Enum | Formato |
|---:|---|---|
| 0 | `PIXEL_FORMAT_DEFAULT` | padrão |
| 1 | `PIXEL_FORMAT_YUV_PLANER_420` | YUV 4:2:0 planar |
| 2 | `PIXEL_FORMAT_YUV_PLANER_422` | YUV 4:2:2 planar |
| 3 | `PIXEL_FORMAT_YUV_PLANER_444` | YUV 4:4:4 planar |
| 4 | `PIXEL_FORMAT_YV12` | YV12 (4:2:0 planar, ordem Y/V/U) |
| 5 | `PIXEL_FORMAT_NV21` | NV21 (4:2:0 semiplanar, ordem VU) |
| 6 | `PIXEL_FORMAT_NV12` | NV12 (4:2:0 semiplanar, ordem UV) |
| 7–9 | `PIXEL_FORMAT_YUV_MB32_*` | formatos tiled MB32 |
| 10–13 | `PIXEL_FORMAT_RGBA`, `ARGB`, `ABGR`, `BGRA` | formatos RGB de 32 bits |
| 14–17 | `YUYV`, `YVYU`, `UYVY`, `VYUY` | formatos packed 4:2:2 |
| 18–21 | `PLANARUV/VU_422/444` | formatos planares 4:2:2 e 4:4:4 |
| 22–23 | `P010_UV`, `P010_VU` | P010 |
| 24–31 | formatos Allwinner 10-bit e AFBC | formatos específicos |

**Confirmação importante:** o argumento `-outFmat 6` usado no teste do `vdecoderdemo` seleciona `PIXEL_FORMAT_NV12`, não apenas um formato semiplanar genérico. O formato 5 seleciona NV21.

Isso resolve a ambiguidade da seção 14.25: o caminho de `savePicture()` para `context+256 == 5 || context+256 == 6` grava um plano de crominância intercalado; combinado com o enum, o caso 6 é UV (NV12) e o caso 5 é VU (NV21). O binário grava o bloco de crominância sem conversão visível nesse trecho.

### Relação com o teste já executado

O teste produziu `cedar-frame.nv12` usando `-outFmat 6`, com resolução 1920×816 e tamanho de arquivo de 2.350.080 bytes (`1920 * 816 * 3 / 2`). Agora há confirmação dupla: o tamanho corresponde a YUV 4:2:0 semiplanar e o enum do header identifica o formato selecionado como NV12.

### Limite da conclusão: stride e estrutura completa

O trecho recebido do header também mostra o início de `VIDEOPICTURE`:

- `nID` offset 0;
- `nStreamIndex` offset 4;
- `ePixelFormat` offset 8;
- `nWidth` offset 12;
- `nHeight` offset 16;
- `nLineStride` offset 20;
- `nTopOffset` offset 24;
- `nLeftOffset` offset 28;
- `nBottomOffset` offset 32;
- `nRightOffset` offset 36.

O output do comando recebido foi truncado antes do restante da estrutura. Assim, os campos posteriores, incluindo o offset formal de `pData0`, devem continuar sendo conferidos no header completo. A análise binária anterior observa acesso a `picture+80`, mas ainda é necessário cruzar esse offset com a declaração completa e com `offsetof()` em um pequeno programa C.

A confirmação do enum não altera a ressalva sobre stride: `savePicture()` calcula os comprimentos por `width * height`, sem usar explicitamente `nLineStride`. Para FFmpeg, ainda precisamos examinar o stride real e o layout da memória retornada pelo CedarX.

## 14.26 — Estrutura completa `VideoPicture` e offsets confirmados

O usuário forneceu a declaração completa de `VideoPicture` (`/usr/include/vbasetype.h`, linhas 198–290). Considerando ABI AArch64 (ponteiros e `size_addr` com 8 bytes; alinhamento de `int64_t` em 8 bytes), os campos relevantes ficam nos seguintes offsets:

| Offset | Campo | Tipo |
|---:|---|---|
| 0 | `nID` | `int` |
| 4 | `nStreamIndex` | `int` |
| 8 | `ePixelFormat` | `int` |
| 12 | `nWidth` | `int` |
| 16 | `nHeight` | `int` |
| 20 | `nLineStride` | `int` |
| 24 | `nTopOffset` | `int` |
| 28 | `nLeftOffset` | `int` |
| 32 | `nBottomOffset` | `int` |
| 36 | `nRightOffset` | `int` |
| 40 | `nFrameRate` | `int` |
| 44 | `nAspectRatio` | `int` |
| 48 | `bIsProgressive` | `int` |
| 52 | `bTopFieldFirst` | `int` |
| 56 | `bRepeatTopField` | `int` |
| 64 | `nPts` | `int64_t` |
| 72 | `nPcr` | `int64_t` |
| 80 | `pData0` | `char *` |
| 88 | `pData1` | `char *` |
| 96 | `pData2` | `char *` |
| 104 | `pData3` | `char *` |
| 112 | `bMafValid` | `int` |
| 120 | `pMafData` | `char *` |
| 128 | `nMafFlagStride` | `int` |
| 132 | `bPreFrmValid` | `int` |
| 136 | `nBufId` | `int` |
| 144 | `phyYBufAddr` | `size_addr` |
| 152 | `phyCBufAddr` | `size_addr` |
| 160 | `pPrivate` | `void *` |
| 168 | `nBufFd` | `int` |
| 172 | `nBufStatus` | `int` |

Os offsets foram deduzidos da declaração e do alinhamento AArch64; ainda é recomendável validá-los mecanicamente com `offsetof()` no host antes de escrever código de integração. Campos posteriores da estrutura incluem flags de erro, metadados HDR, parâmetros de cor, dados de lower bits, tamanhos e informações de frame.

### Cruzamento com `savePicture()`

A desmontagem de `savePicture()` lê:
- `picture+12`: `nWidth`;
- `picture+16`: `nHeight`;
- `picture+80`: `pData0`.

A declaração completa agora confirma esses três offsets. O campo `nLineStride` está em `picture+20`, mas o caminho de dump examinado não o consulta: calcula o tamanho Y como `nWidth * nHeight` e os tamanhos de crominância como `nWidth * nHeight / 4` por plano (planar) ou `/2` para o plano intercalado (semiplanar). Para NV12, escreve `Y_size` bytes a partir de `pData0` e, em seguida, `Y_size/2` bytes a partir de `pData0 + Y_size`.

**Interpretação e ressalva:** o código do demo trata a memória iniciada em `pData0` como um bloco contíguo no formato de dump compacto. Isso explica o arquivo de 1920×816 com tamanho `width*height*3/2`. Não prova que todos os buffers CedarX tenham stride igual à largura nem que `pData1` seja sempre `pData0 + width*height`; para o decoder FFmpeg, é necessário verificar `nLineStride`, ponteiros de cada plano e alinhamento em frames reais. O dump simplificado do demo não deve ser tomado como contrato geral de layout.

### Próxima verificação recomendada

Compilar um pequeno programa C contra os headers instalados e imprimir `sizeof(VideoPicture)` e `offsetof(VideoPicture, ...)` para validar ABI/offsets sem depender apenas do cálculo manual. Depois, registrar em runtime `nWidth`, `nHeight`, `nLineStride`, `pData0`, `pData1`, `pData2`, `nBufFd` e `nBufId` para um frame CedarX NV12 real.

## 14.27 — Validação compilada da ABI AArch64 de `VideoPicture`

O usuário compilou e executou `/tmp/check_videopicture.c` usando `gcc` no próprio Orange Pi, incluindo `<vbasetype.h>` e consultando `sizeof`/`offsetof` diretamente. A saída foi:

```text
sizeof(VideoPicture) = 320

nWidth                   offset= 12 size=4
nHeight                  offset= 16 size=4
nLineStride              offset= 20 size=4
nPts                     offset= 64 size=8
nPcr                     offset= 72 size=8
pData0                   offset= 80 size=8
pData1                   offset= 88 size=8
pData2                   offset= 96 size=8
pData3                   offset=104 size=8
nBufId                   offset=136 size=4
phyYBufAddr              offset=144 size=8
phyCBufAddr              offset=152 size=8
pPrivate                 offset=160 size=8
nBufFd                   offset=168 size=4
nBufStatus               offset=172 size=4
```

Esta é validação empírica da ABI usada pelo compilador AArch64 no equipamento, e não apenas uma estimativa baseada no alinhamento da estrutura. Confirma também o tamanho total de 320 bytes.

### Consequências para a integração

- A leitura de `picture+80` na desmontagem de `savePicture()` corresponde definitivamente a `VideoPicture.pData0`.
- `nLineStride` está em `+20`, e não foi consultado pelo caminho de dump analisado.
- `pData0` a `pData3`, `nBufId`, endereços físicos, `pPrivate` e `nBufFd` têm offsets agora validados para a ABI local.
- O futuro wrapper FFmpeg deve incluir o header oficial e acessar os membros por nome, evitando hardcode de offsets. A validação é útil para auditoria e para diagnosticar incompatibilidades de ABI.
- `sizeof(VideoPicture) == 320` descreve esta compilação/configuração dos headers. A estrutura possui campos condicionais, como os campos sob `VE_SUPPORT_DECODER_LBC_MODE`; builds com macros diferentes podem ter tamanho final diferente. Os offsets dos campos validados acima são os observados nesta build.

### Estado da investigação

A estrutura e os offsets necessários para localizar dimensões, stride, PTS e ponteiros de planos estão confirmados. Ainda não sabemos os valores efetivos de `nLineStride`, `pData1` e `pData2` em frames decodificados, nem se o buffer NV12 é sempre linear/contíguo. A próxima etapa útil é instrumentar o harness CedarX para registrar esses campos em runtime, mantendo o `VideoPicture` adquirido até terminar a leitura e chamando `ReturnPicture()` somente quando o consumidor liberar o frame.

## 14.28 — GDB disponível e funções CedarX importadas dinamicamente

O usuário verificou a disponibilidade do depurador e a tabela de símbolos dinâmicos de `/usr/bin/vdecoderdemo` com:

```bash
command -v gdb
readelf -Ws /usr/bin/vdecoderdemo \
  | grep -E 'RequestPicture|ReturnPicture|InitializeVideoDecoder|DecodeVideoStream'
```

Resultado relevante:

```text
/usr/bin/gdb
    21: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND RequestPicture
    33: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND ReturnPicture
    47: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND DecodeVideoStream
    48: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND InitializeVideoDecoder
   159: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND RequestPicture
   174: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND ReturnPicture
   201: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND DecodeVideoStream
   202: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND InitializeVideoDecoder
```

### Interpretação

- O executável tem GDB disponível em `/usr/bin/gdb`; não é necessário instalar o pacote.
- `RequestPicture`, `ReturnPicture`, `DecodeVideoStream` e `InitializeVideoDecoder` aparecem como símbolos `UND` (undefined) com binding `GLOBAL` na tabela dinâmica. Isso indica que o executável importa essas funções de uma biblioteca compartilhada, em vez de fornecer suas implementações dentro do próprio executável.
- Como são referências dinâmicas globais, há uma boa perspectiva de interceptar as chamadas por breakpoints do GDB e, em princípio, também por interposição dinâmica. Isso ainda precisa ser confirmado durante a execução: resolução do símbolo, nome da biblioteca carregada e comportamento do breakpoint dependem do carregamento/versão das bibliotecas.
- As duas ocorrências de cada símbolo refletem tabelas/representações de símbolos do ELF; não significam duas implementações independentes.

### Próxima etapa

Usar o GDB para executar o `vdecoderdemo` com o mesmo arquivo H.264 de teste e observar primeiro `InitializeVideoDecoder()`, `RequestPicture()` e `ReturnPicture()`. A inspeção deverá registrar os argumentos e, após o retorno de `RequestPicture()`, os campos de `VideoPicture` já validados na seção 14.27: dimensões, `nLineStride`, `ePixelFormat`, ponteiros dos planos, `nBufId` e `nBufFd`. Ainda não há dados runtime desses campos; não se deve inferir stride ou layout dos planos apenas a partir do dump compacto produzido pelo demo.

## 14.29 — Dependências compartilhadas de `vdecoderdemo`

O usuário executou `ldd /usr/bin/vdecoderdemo`. As dependências resolvidas foram:

```text
libcdc_base.so       => /lib/aarch64-linux-gnu/libcdc_base.so
libMemAdapter.so     => /lib/aarch64-linux-gnu/libMemAdapter.so
libvdecoder.so       => /lib/aarch64-linux-gnu/libvdecoder.so
libfbm.so            => /lib/aarch64-linux-gnu/libfbm.so
libsbm.so            => /lib/aarch64-linux-gnu/libsbm.so
libpthread.so.0      => /lib/aarch64-linux-gnu/libpthread.so.0
libm.so.6            => /lib/aarch64-linux-gnu/libm.so.6
librt.so.1           => /lib/aarch64-linux-gnu/librt.so.1
libdl.so.2           => /lib/aarch64-linux-gnu/libdl.so.2
libVE.so             => /lib/aarch64-linux-gnu/libVE.so
libvideoengine.so    => /lib/aarch64-linux-gnu/libvideoengine.so
libc.so.6            => /lib/aarch64-linux-gnu/libc.so.6
/lib/ld-linux-aarch64.so.1
```

### Interpretação

- O executável carrega explicitamente `libvdecoder.so`, além das bibliotecas de suporte para memória (`libMemAdapter.so`), gerenciamento de buffers (`libfbm.so`, `libsbm.so`), base CDC (`libcdc_base.so`) e engine (`libVE.so`, `libvideoengine.so`).
- A saída de `ldd` confirma a resolução dessas dependências no sistema. Por si só, ela não identifica qual biblioteca exporta cada função importada nem prova que uma função específica será resolvida a partir de `libvdecoder.so`.
- A identificação precisa deve ser feita consultando a tabela dinâmica de símbolos definidos (`nm -D --defined-only`) nas bibliotecas candidatas. Isso permitirá definir breakpoints qualificados e compreender a resolução dos símbolos antes de executar a sessão GDB.

### Próxima verificação

Consultar as exportações dinâmicas de `libvdecoder.so`, `libVE.so` e `libvideoengine.so` para localizar as implementações de `InitializeVideoDecoder`, `DecodeVideoStream`, `RequestPicture` e `ReturnPicture`. Ainda não foi atribuída definitivamente cada função a uma biblioteca com base apenas no `ldd`.

## 14.30 — Localização das implementações da API CedarX

O usuário consultou os símbolos dinâmicos definidos nas cinco bibliotecas candidatas:

```bash
for lib in \
  libvdecoder.so \
  libVE.so \
  libvideoengine.so \
  libfbm.so \
  libsbm.so
do
    echo
    echo "========== $lib =========="
    nm -D --defined-only "/lib/aarch64-linux-gnu/$lib" 2>/dev/null \
      | grep -E ' (InitializeVideoDecoder|DecodeVideoStream|RequestPicture|ReturnPicture)$'
done
```

Resultado:

```text
========== libvdecoder.so ==========
0000000000007d10 T DecodeVideoStream
0000000000005984 T InitializeVideoDecoder
0000000000009800 T RequestPicture
0000000000009b3c T ReturnPicture

========== libVE.so ==========

========== libvideoengine.so ==========

========== libfbm.so ==========

========== libsbm.so ==========
```

### Conclusões confirmadas

- As quatro funções públicas consultadas estão exportadas por `libvdecoder.so`.
- `T` indica símbolo definido na seção de código da biblioteca (símbolo global de texto).
- Os valores mostrados são offsets/valores de símbolo no ELF da biblioteca, não endereços absolutos de memória em runtime:
  - `InitializeVideoDecoder`: `0x5984`
  - `DecodeVideoStream`: `0x7d10`
  - `RequestPicture`: `0x9800`
  - `ReturnPicture`: `0x9b3c`
- As outras quatro bibliotecas consultadas não exportam esses quatro nomes. Isso não significa que não participem internamente da decodificação; apenas que não fornecem esses símbolos públicos específicos.
- Combinando esta saída com a seção 14.28, temos a cadeia de resolução confirmada: `vdecoderdemo` importa as funções dinamicamente e `libvdecoder.so` as exporta.

### Próxima etapa de depuração

Os breakpoints podem ser definidos diretamente pelos nomes das funções no GDB, por exemplo `break InitializeVideoDecoder`, `break RequestPicture` e `break ReturnPicture`. No AArch64, os primeiros argumentos de função são passados em `x0`, `x1`, etc. Assim:

- em `InitializeVideoDecoder(decoder, videoInfo, vconfig)`, `x0` é o handle do decoder, `x1` aponta para `VideoStreamInfo` e `x2` aponta para `VConfig`;
- em `RequestPicture(decoder, streamIndex)`, `x0` é o decoder e `w1` é o índice do stream;
- em `ReturnPicture(decoder, picture)`, `x0` é o decoder e `x1` aponta para `VideoPicture`.

A próxima execução GDB deve usar o mesmo arquivo `/tmp/cedar-test.h264` e os mesmos parâmetros do teste que já decodificou 31 frames. No breakpoint de `RequestPicture`, será necessário avançar até o retorno da função para inspecionar o ponteiro retornado em `x0`; os metadados da picture devem ser lidos apenas quando esse ponteiro for não nulo. Não assumir que o primeiro breakpoint já contém uma picture válida.

## 14.29 — Inspeção GDB de `InitializeVideoDecoder()` e configuração real do demo

### Execução e breakpoint

O `vdecoderdemo` foi iniciado sob GDB como root, com o mesmo teste H.264 previamente validado:

```bash
sudo gdb -q --args /usr/bin/vdecoderdemo \
  -i /tmp/cedar-test.h264 \
  -codFmat 1 \
  -n 31 \
  -ss 1 \
  -sn 1 \
  -o /tmp/cedar-gdb.nv12 \
  -outFmat 6
```

Breakpoint configurado em `InitializeVideoDecoder`. O GDB parou na entrada da função, na thread 2, e identificou os parâmetros:

```text
InitializeVideoDecoder (
  pDecoder=0xfffff0016a30,
  pVideoInfo=0xfffff7b9e720,
  pVconfig=0xfffff7b9e768
)
```

O fonte `vdecoder.c` não está disponível no sistema, mas o GDB conseguiu recuperar os tipos a partir das informações de depuração e inspecionar os objetos em memória.

### `VideoStreamInfo` observado em runtime

`ptype VideoStreamInfo` confirmou a estrutura declarada no header. A inspeção de `*pVideoInfo` retornou:

```text
 eCodecFormat = 277
 nWidth = 0
 nHeight = 0
 nFrameRate = 0
 nFrameDuration = 0
 nAspectRatio = 0
 bIs3DStream = 0
 nCodecSpecificDataLen = 0
 pCodecSpecificData = 0x0
 bSecureStreamFlag = 0
 bSecureStreamFlagLevel1 = 0
 bIsFramePackage = 0
 h265ReferencePictureNum = 0
 bReOpenEngine = 0
 bIsFrameCtsTestFlag = 0
 bCheckInitFlag = 0
```

Conclusões:

- `eCodecFormat = 277` decimal, equivalente a `0x115`, confirma em runtime a seleção do formato H.264 já identificada na desmontagem de `initDecoder`.
- Largura, altura, taxa de quadros, duração, aspect ratio e dados específicos do codec estão zerados na chamada inicial. Portanto, neste caminho do demo, esses parâmetros não são fornecidos previamente em `VideoStreamInfo`; o decoder recebe inicialmente uma estrutura zerada nesses campos e precisa obter as informações do bitstream (por exemplo, SPS do H.264) durante o processamento.
- O ponteiro de dados específicos do codec é nulo e seu comprimento é zero.

### `VConfig` observado em runtime

`ptype VConfig` confirmou a estrutura, incluindo os campos finais `DecVcuConfig sVcuConfig`. A inspeção de `*pVconfig` mostrou:

| Campo | Valor observado |
|---|---:|
| `eOutputPixelFormat` | 6 (`PIXEL_FORMAT_NV12`) |
| `nDeInterlaceHoldingFrameBufferNum` | 2 |
| `nDisplayHoldingFrameBufferNum` | 3 |
| `nRotateHoldingFrameBufferNum` | 0 |
| `nDecodeSmoothFrameBufferNum` | 3 |
| `memops` | `0xfffff7f75008` (`_ionMemOpsS`) |
| `eCtlAfbcMode` | `DISABLE_AFBC_ALL_SIZE` |
| `nAfbcSecMode` | `DISABLE_AFBC_ALL_SIZE` |
| `eCtlIptvMode` | `DISABLE_IPTV_ALL_SIZE` |

Os demais campos exibidos estavam zerados, incluindo `bScaleDownEn`, `bRotationEn`, `bSecOutputEn`, `bSupportMaf`, `bDispErrorFrame`, `nVbvBufferSize`, `nFrameBufferNum`, `bGpuBufValid`, `nAlignStride`, `bIsSoftDecoderFlag`, `bVirMallocSbm`, `bSupportPallocBufBeforeDecode`, `veOpsS`, `pVeOpsSelf`, `nVeFreq`, `nChannelNum`, `nSupportMaxWidth`, `nSupportMaxHeight`, `commonConfigFlag`, `bATMFlag` e todos os membros de `sVcuConfig`.

### Conclusões técnicas

1. O formato de saída NV12 não é apenas inferido da opção `-outFmat 6`: está efetivamente presente em `VConfig.eOutputPixelFormat` na chamada real a `InitializeVideoDecoder()`.
2. Os quatro contadores de buffers holding observados em runtime confirmam os valores extraídos da desmontagem: 2/3/0/3.
3. `VConfig.memops` aponta para `_ionMemOpsS`, confirmando que o demo fornece ao decoder o adaptador de memória CedarX/ION.
4. `veOpsS` e `pVeOpsSelf` estão nulos neste ponto. Não concluir que o VE não foi inicializado: os logs anteriores à parada mostram `VeInitialize`, abertura de `/dev/cedar_dev` e criação do contexto VE. A inicialização pode ocorrer por outro caminho interno da biblioteca.
5. `nFrameBufferNum` e `nAlignStride` estão zerados na configuração fornecida pelo demo; não assumir os valores típicos descritos em documentação como valores efetivamente usados por este executável.
6. A observação confirma que a resolução ainda não foi preenchida em `VideoStreamInfo` na entrada de `InitializeVideoDecoder`; é necessário observar o decoder depois de processar o bitstream para descobrir quando e como os parâmetros são determinados.

### Próxima etapa

Manter a execução parada ou configurar breakpoints adicionais em `RequestPicture` e `ReturnPicture`. O objetivo é inspecionar o `VideoPicture` retornado, especialmente `ePixelFormat`, `nWidth`, `nHeight`, `nLineStride`, `pData0`, `pData1`, `pData2`, `nBufId` e `nBufFd`. Em `RequestPicture`, inspecionar o valor de retorno em `x0` após a execução da função e somente dereferenciá-lo se for não nulo. Não avançar para a integração FFmpeg antes de confirmar layout, stride e propriedade/vida útil dos buffers.

## 14.29 — Primeira chamada de `RequestPicture()` retorna NULL

Durante a execução de `/usr/bin/vdecoderdemo` sob GDB, com o teste H.264 de 1920×816 e `-n 31`, o breakpoint em `RequestPicture()` foi atingido na thread 6 (`LWP 32744`). O GDB identificou os argumentos:

- `pDecoder = 0xfffff0016a30`
- `nStreamIndex = 0`

A inspeção dos registradores AArch64 na entrada confirmou `x0 = 0xfffff0016a30` e `w1 = 0`.

Ao executar `finish`, o GDB informou:

```text
Value returned is $5 = (VideoPicture *) 0x0
```

Portanto, **essa primeira chamada específica de `RequestPicture()` retornou NULL**. A tentativa de dereferenciar `$pic` falhou corretamente porque o ponteiro é nulo; não há uma estrutura `VideoPicture` válida para inspecionar nesse retorno.

### Interpretação

Um retorno nulo nesta chamada não demonstra falha de decodificação. `RequestPicture()` pode ser consultada antes de haver um frame disponível; o chamador `displayPictureThreadFunc()` precisa tratar esse caso e tentar novamente ou aguardar. O teste H.264 anterior já havia decodificado e apresentado 31 frames com sucesso fora do GDB, mas ainda precisamos observar uma chamada posterior que retorne um ponteiro não nulo.

### Próxima etapa de depuração

Com o breakpoint `RequestPicture` ainda ativo, executar `continue` para alcançar a próxima chamada. Repetir `finish` e verificar o valor de retorno em `x0`. Só atribuir/dereferenciar `VideoPicture *` quando o retorno for diferente de zero. Quando houver um frame válido, registrar pelo menos `ePixelFormat`, `nWidth`, `nHeight`, `nLineStride`, `pData0`–`pData3`, `nBufId`, `nBufFd` e `nBufStatus`. Não remover o breakpoint de `ReturnPicture`, pois ele será necessário para investigar o ciclo de vida do buffer.

### 14.29 — `RequestPicture()` sob GDB: retornos nulos repetidos

Na sessão GDB do teste H.264 (`/tmp/cedar-test.h264`, 31 frames), o breakpoint em `RequestPicture()` foi atingido na thread 6, dentro de `displayPictureThreadFunc()`, com `pDecoder=0xfffff0016a30` e `nStreamIndex=0`. Em duas chamadas observadas, `finish` mostrou retorno `VideoPicture * = 0x0`.

Isso comprova apenas que não havia um `VideoPicture` disponível para essas chamadas específicas. Não comprova falha de inicialização nem falha de decodificação: `RequestPicture()` pode ser consultada antes de o decoder disponibilizar um frame, e a thread de exibição pode repetir a consulta. O breakpoint/`finish` também altera o escalonamento das threads e pode afetar o momento em que o decoder produz frames.

Próxima investigação: desabilitar temporariamente o breakpoint de `RequestPicture()` e parar em `DecodeVideoStream()` para verificar se o thread de decode está efetivamente chamando a API, quais argumentos usa e quais códigos de retorno recebe. Depois retomar a inspeção de `RequestPicture()` sem pausar repetidamente todas as chamadas.

## 14.30 — Primeira chamada de `DecodeVideoStream()` observada no GDB

Após desabilitar o breakpoint 2 de `RequestPicture()`, foi configurado um breakpoint em `DecodeVideoStream()`. A execução parou na thread 5 (`LWP 32743`) e o GDB reconheceu os argumentos:

```text
pDecoder = 0xfffff0016a30
bEndOfStream = 0
bDecodeKeyFrameOnly = 0
bDropBFrameIfDelay = 0
nCurrentTimeUs = 0
```

A inspeção dos registradores mostrou:

```text
$w1 = 0
$w2 = 0
$w3 = 0
$w4 = 0
```

Na ABI AArch64, `nCurrentTimeUs` é o quinto argumento e deve ser inspecionado pelo registrador de 64 bits `$x4` (não `$w4`); o GDB, contudo, também exibiu o parâmetro nomeado como zero na assinatura da função.

Ao executar `finish`, o GDB informou:

```text
Value returned is $12 = 0
```

Conforme a desmontagem anterior de `DecodeThread()` (seção 14.16), retorno `0` é o caminho normal de continuação do loop de decodificação. Portanto, esta chamada não retornou erro, mas isoladamente também não prova que um frame foi produzido ou disponibilizado em `RequestPicture()`.

### Próxima etapa

Observar chamadas subsequentes de `DecodeVideoStream()` e correlacionar seus retornos com `RequestPicture()`. Evitar pausar repetidamente a thread de exibição: breakpoints em funções chamadas em loop podem alterar o escalonamento e atrasar a disponibilidade dos frames. O objetivo continua sendo capturar uma chamada de `RequestPicture()` que retorne um ponteiro não nulo; somente então inspecionar `VideoPicture` e seu stride/layout.

## 14.31 — DMA-BUF do frame NV12: mapeamento, leitura dos planos e encerramento da sincronização

Durante a inspeção de `ReturnPicture()` no processo `/usr/bin/vdecoderdemo` (PID 32460), foi identificado um frame com os seguintes parâmetros:

- `ePixelFormat = 6` (`PIXEL_FORMAT_NV12`);
- resolução de 1920 × 816;
- `nLineStride = 1920`;
- `pData0 = 0xfffff414f000` (plano Y);
- `pData1 = 0xfffff42cd800` (plano UV);
- `nBufFd = 9`;
- `nBufSize = 2.350.080` bytes.

O descritor FD 9 referencia um DMA-BUF `/dmabuf` do heap `system`, com tamanho de 2.351.104 bytes. A diferença de 1.024 bytes em relação a `nBufSize` permanece registrada como tamanho adicional/alinhamento observado; sua finalidade exata não foi determinada.

Foi criado um mapeamento do FD 9 com `mmap()` usando tamanho de 2.351.104 bytes, `PROT_READ` e `MAP_SHARED`. O endereço retornado foi `0xffffe6994000`. O GDB não conseguiu examinar diretamente essa região com `x/32bx $mapped` (mensagem `Cannot access memory`), embora `/proc/32460/maps` mostrasse o mapeamento como `r--s`.

Foi então iniciada a sincronização DMA-BUF para leitura com `DMA_BUF_IOCTL_SYNC` (`ioctl(9, 0x40086200, $sync)`) e `flags = 1` (`DMA_BUF_SYNC_START | DMA_BUF_SYNC_READ`), com retorno 0. A leitura direta pelo comando `x` do GDB continuou falhando. Entretanto, `memcpy()` executado no próprio processo conseguiu copiar os bytes do mapeamento para memória convencional, que o GDB leu normalmente.

### Validação do plano Y

Foram copiados 256 bytes de `$mapped` e 256 bytes de `pPicture->pData0`. A comparação:

```gdb
p (int)memcmp($copy, $copy_original, 256)
```

retornou `0`. Portanto, os 256 bytes iniciais do mapeamento DMA-BUF são idênticos aos 256 bytes do plano Y exposto em `VideoPicture.pData0`.

### Validação do plano UV

Foram copiados 256 bytes de `pPicture->pData1` e 256 bytes do endereço `$mapped + 1566720`. O deslocamento corresponde a `1920 * 816`, isto é, ao tamanho do plano Y para a geometria/stride observada. A comparação:

```gdb
p (int)memcmp($copy_uv, $copy_uv_map, 256)
```

retornou `0`. Portanto, os 256 bytes iniciais do plano UV também correspondem exatamente à região do mapeamento DMA-BUF no deslocamento esperado para NV12.

### Encerramento da sincronização

Após concluir as leituras, a sincronização foi encerrada com `flags = 2` (`DMA_BUF_SYNC_END | DMA_BUF_SYNC_READ`). A chamada:

```gdb
p (int)ioctl(9, 0x40086200, $sync)
```

retornou `0`, confirmando que o ioctl de encerramento foi aceito.

### Conclusões e limites

1. O FD 9 mapeado com `mmap()` dá acesso ao conteúdo do mesmo DMA-BUF referenciado pelos ponteiros `pData0` e `pData1` do `VideoPicture` observado.
2. A correspondência foi comprovada por igualdade byte a byte (`memcmp == 0`) em amostras de 256 bytes dos planos Y e UV.
3. A leitura via `memcpy()` dentro do processo funciona, embora o GDB não consiga ler diretamente o endereço mapeado com seus comandos de exame de memória.
4. O teste valida as amostras iniciais dos dois planos e o deslocamento UV para este frame; ainda não valida o conteúdo integral, todos os frames, nem o comportamento de cache ao longo da reprodução.
5. A sincronização DMA-BUF `START | READ` e o respectivo `END | READ` retornaram sucesso (0).

### Próxima etapa

Com o acesso aos planos Y e UV demonstrado, o próximo teste pode copiar o frame completo para memória convencional ou para um arquivo NV12, respeitando o tamanho/stride efetivos e o ciclo de vida do `VideoPicture`. Antes de automatizar a exportação, deve-se confirmar quando o buffer deixa de pertencer ao decoder e em que ponto `ReturnPicture()` pode ser chamado com segurança. Não manter ponteiros para o DMA-BUF depois da devolução do `VideoPicture`.

## 14.38 — Ciclo de vida do VideoPicture: análise de addPictureToList()

### Pilha de chamadas observada

No breakpoint de `ReturnPicture()`, `bt` mostrou:

```text
#0 ReturnPicture(pDecoder=..., pPicture=...) at vdecoder.c:1843
#1 addPictureToList()
#2 displayPictureThreadFunc()
#3 libc thread start
```

Isso confirma que, nesta ocorrência, a thread de exibição chamou `addPictureToList()`, que por sua vez chamou `ReturnPicture()`.

### Estruturas e comportamento reconstruídos da desmontagem AArch64

A desmontagem de `addPictureToList()` em `0xaaaaaaaa45c0–0xaaaaaaaa4830` mostra:

- Os argumentos são preservados na pilha: `x0` em `sp+56`, `x1` em `sp+48`, `x2` em `sp+40`, `x3` em `sp+32` e `x4` em `sp+24`.
- O endereço em `x4` é tratado como o objeto/entrada associado ao frame. Se `x4` ou `x1` for nulo, a função segue o caminho de aviso e não executa a inserção normal.
- A função percorre até cinco entradas usando passo de 24 bytes (`i * 24`). Em cada entrada, lê um campo de estado em `+8`; quando encontra estado zero, grava estado 1 e armazena o ponteiro recebido em `+0`.
- Os nós da lista encadeada usam campos nos offsets `+0`, `+8` e `+16`: ponteiro associado, estado e próximo nó, respectivamente, conforme os acessos observados.
- Os endereços de cabeça e cauda são recebidos por referência (argumentos preservados em `sp+40` e `sp+32`) e atualizados durante a inserção.
- Depois de inserir, a função percorre a lista encadeada contando nós.
- Se a contagem for maior que um, remove o nó da cabeça, zera seu campo `next` (`+16`) e seu estado (`+8`), espera 2.000 microssegundos por `usleep(2000)` e chama `ReturnPicture()` com o ponteiro guardado no campo `+0` do nó removido.
- Ao final, grava os novos ponteiros de cabeça e cauda nas referências recebidas.

### Interpretação

O comportamento observado é compatível com uma fila FIFO que retém normalmente um frame: quando um novo frame é inserido e a fila passa a ter mais de um elemento, o mais antigo é removido e devolvido ao decoder após uma espera de 2 ms. Assim, a devolução não precisa ocorrer imediatamente após a chegada do frame à thread de exibição.

A pilha confirma que `ReturnPicture()` é alcançado a partir de `displayPictureThreadFunc()` via `addPictureToList()`. A desmontagem, isoladamente, não prova todos os caminhos de limpeza da fila (por exemplo, drenagem no encerramento), nem a semântica completa dos tipos C dos argumentos; esses pontos ainda exigem inspeção do código-fonte ou de outros callsites.

### Implicações para a integração FFmpeg/libmpv

- Não reter `pData0`, `pData1` ou o FD do frame como se fossem propriedade permanente do consumidor.
- Qualquer cópia para um buffer de propriedade do FFmpeg deve ocorrer antes de `ReturnPicture()` devolver o `VideoPicture` ao decoder.
- Se a integração quiser zero-copy, precisará de um mecanismo explícito de retenção/release que impeça a reutilização do DMA-BUF enquanto o consumidor ainda o utiliza. A API observada não demonstra, até aqui, tal mecanismo.
- O atraso de 2 ms não deve ser tratado como garantia de validade do frame para consumidores externos; a fronteira segura é o ciclo de propriedade controlado pelo decoder.

### Próxima investigação

Examinar a função `displayPictureThreadFunc()` e os callsites de `addPictureToList()` para determinar se `savePicture()`/cópia ocorre antes da inserção e para identificar o tratamento da fila no encerramento.

## 14.39 — Ordem de gravação e retenção do frame em displayPictureThreadFunc()

### Evidência na desmontagem

A desmontagem de `displayPictureThreadFunc()` (`0xaaaaaaaa4fbc–0xaaaaaaaa55d4`) mostra a seguinte sequência no caminho normal em que `RequestPicture()` retorna um `VideoPicture` não nulo:

1. `RequestPicture()` é chamado em `0xaaaaaaaa5168`, e o ponteiro retornado é guardado em `sp+112`.
2. O contador local de frames (`sp+44`) é incrementado em `0xaaaaaaaa5328–0xaaaaaaaa5330`.
3. A função compara esse contador com valores do contexto (`context+224` e `context+228`). Quando o contador está dentro da faixa de captura configurada, chama `savePicture(pPicture, context)` em `0xaaaaaaaa53b4`.
4. Em seguida, verifica a condição de encerramento baseada em `context+92`. Se ainda não for hora de encerrar, prepara os argumentos e chama `addPictureToList()` em `0xaaaaaaaa5460`.

Portanto, no caminho em que a condição de captura é satisfeita e a thread continua normalmente, a chamada de `savePicture()` precede a chamada de `addPictureToList()` para aquele frame.

### Conclusão sobre o ciclo de vida

Combinando esta desmontagem com a análise de `addPictureToList()` (seção 14.38), temos evidência de que:

- O frame é obtido por `RequestPicture()`.
- Para frames selecionados pela faixa de captura, `savePicture()` é executado antes de o frame ser inserido na fila de retenção.
- `addPictureToList()` mantém uma fila encadeada e, quando há mais de um elemento, remove o mais antigo e chama `ReturnPicture()` para devolvê-lo ao decoder.
- Assim, a gravação feita por `savePicture()` acontece antes da possível devolução daquele frame pela lógica normal da fila.

### Limites da conclusão

A chamada a `savePicture()` é condicional: não ocorre necessariamente para todos os frames. A condição depende dos contadores/limites no contexto (`+224`, `+228` e `+92`). A desmontagem também mostra um caminho de encerramento que chama `ReturnAllPicture()`, portanto a limpeza final da fila tem uma rota separada.

A ordem observada é específica do caminho normal analisado. Ela não demonstra, por si só, que um consumidor externo possa guardar os ponteiros do frame após `addPictureToList()` ou após `ReturnPicture()`. Para a integração FFmpeg/libmpv, continua sendo necessário copiar os dados para memória própria antes de devolver o `VideoPicture`, ou implementar retenção explícita do buffer até o último consumidor liberar a referência.

### Próxima etapa

Investigar `ReturnAllPicture()` e os parâmetros de contexto que controlam a faixa de captura e o encerramento. Isso permitirá completar a descrição dos caminhos de liberação, inclusive no shutdown da thread.


## 14.40 — Drenagem da fila no encerramento: ReturnAllPicture()

### Evidência na desmontagem AArch64

A função `ReturnAllPicture()` ocupa `0xaaaaaaaa4834–0xaaaaaaaa4894`. A desmontagem mostra este laço:

1. Recebe o decoder em `x0` e o ponteiro inicial da lista em `x1`; ambos são preservados na pilha (`sp+24` e `sp+16`).
2. Enquanto o ponteiro da entrada atual (`sp+16`) não for nulo, lê o ponteiro armazenado no início da entrada (`[entry+0]`).
3. Chama `ReturnPicture(decoder, picture)`.
4. Após o retorno da chamada, grava zero no campo de estado da entrada (`[entry+8]`).
5. Lê o próximo nó de `[entry+16]` e avança para ele.
6. Incrementa um contador local e repete até o ponteiro da entrada ser nulo.
7. Retorna esse contador em `w0`.

Pseudocódigo equivalente ao fluxo observado:

```c
int ReturnAllPicture(VideoDecoder *decoder, Entry *entry)
{
    int count = 0;
    while (entry != NULL) {
        ReturnPicture(decoder, entry->picture);
        entry->occupied = 0;
        entry = entry->next;
        count++;
    }
    return count;
}
```

Os nomes `picture`, `occupied` e `next` são rótulos explicativos inferidos dos offsets e do comportamento; a desmontagem não fornece os nomes C originais dos campos.

### O que fica comprovado

- A rotina percorre a lista encadeada inteira até `NULL`.
- Cada entrada visitada provoca uma chamada a `ReturnPicture()` com o ponteiro armazenado em `+0`.
- O campo de estado em `+8` é zerado **depois** da chamada a `ReturnPicture()`.
- O próximo nó é obtido de `+16`; a rotina não zera explicitamente esse ponteiro.
- Não há `usleep()` nesse laço de drenagem.
- A função retorna a quantidade de entradas percorridas, não um status de sucesso/erro de `ReturnPicture()` (o valor de retorno deste último não é acumulado nem testado no trecho desmontado).

### Relação com o caminho normal

Na desmontagem de `displayPictureThreadFunc()`, o encerramento chama `ReturnAllPicture()` em `0xaaaaaaaa5558`, depois de sair do ciclo normal de obtenção de frames. Isso complementa o caminho de `addPictureToList()` descrito na seção 14.38:

- Durante a execução normal, `addPictureToList()` mantém a fila e devolve a entrada mais antiga quando a lista passa a ter mais de um elemento.
- No encerramento, `ReturnAllPicture()` percorre e devolve todas as entradas que ainda estiverem na lista.

Assim, os dois caminhos de devolução estão identificados na desmontagem do demo: remoção incremental durante a reprodução e drenagem completa no encerramento.

### Implicações para a integração

A rotina confirma que o demo não deixa intencionalmente os frames restantes na fila ao terminar a thread de exibição. Para uma integração FFmpeg/libmpv, o princípio continua sendo devolver cada `VideoPicture` exatamente uma vez e somente depois de terminar qualquer leitura/cópia necessária. A desmontagem não demonstra um mecanismo de referência compartilhada que permita manter o frame em uso depois de `ReturnPicture()`.

Também é importante não copiar literalmente o pseudocódigo sem reproduzir a sincronização e a propriedade da lista do demo: a desmontagem isolada não revela todos os bloqueios/locks em torno da chamada de encerramento nem os tipos C exatos dos nós.

### Próxima etapa

A próxima investigação útil é examinar o caminho de criação e inicialização da fila e os pontos em que `displayPictureThreadFunc()` recebe seus argumentos. Isso permitirá identificar a estrutura C real dos nós, o proprietário da lista e a sincronização usada entre a thread de exibição e o restante do demo.

## 14.41 — Inicialização do decoder CedarX em `initDecoder()`

A desmontagem completa de `initDecoder()` (`0xaaaaaaaa4108`–`0xaaaaaaaa45bc`) permite reconstruir a ordem de inicialização do demo e confirmar os tamanhos das duas estruturas passadas a `InitializeVideoDecoder()`.

### Estruturas locais preparadas

No prólogo, a função reserva 352 bytes de stack. Em seguida:

- `memset(sp + 0x78, 0, 224)` zera 224 bytes. Essa área é passada como terceiro argumento de `InitializeVideoDecoder()` e corresponde à estrutura `VConfig` usada pelos headers CedarX.
- `memset(sp + 0x30, 0, 72)` zera 72 bytes. Essa área é passada como segundo argumento e corresponde a `VideoStreamInfo`.

Na chamada em `0xaaaaaaaa4494`, os registradores AArch64 são preparados assim:

- `x0`: `VideoDecoder` armazenado em `context + 0`;
- `x1`: endereço `sp + 0x30` (`VideoStreamInfo`, 72 bytes);
- `x2`: endereço `sp + 0x78` (`VConfig`, 224 bytes).

A chamada efetiva ocorre em `0xaaaaaaaa4494` para `InitializeVideoDecoder@plt`; o status retornado em `w0` é guardado em `sp + 44`.

### Sequência de inicialização observada

1. A função chama `AddVDPlugin()` em `0xaaaaaaaa4128`.
2. Zera as áreas locais de `VideoStreamInfo` e `VConfig`.
3. Chama `MemAdapterGetOpsS()` em `0xaaaaaaaa414c` e grava o ponteiro retornado em `context + 232`.
4. Se o ponteiro de `memops` for nulo, segue para o caminho de erro e retorna `-1`.
5. Com `memops` válido, chama `CdcMemOpen(memops)` em `0xaaaaaaaa41bc`.
6. Inicializa um mutex em `context + 176` com `pthread_mutex_init()`.
7. Chama `CreateVideoDecoder()` em `0xaaaaaaaa4210` e grava o resultado em `context + 0`.
8. Se o decoder criado for nulo, segue para o caminho de erro e retorna `-1`.
9. Lê `context + 104` para selecionar valores de codec/stream: quando o valor é 1, grava `0x115` (277) na variável local em `sp + 48` e escreve 1 em `*(context + 248) + 8`; quando é 2, grava `0x116` (278) e escreve 2 nesse campo; quando é 3, grava `0x115` (277). Esses valores são observados na desmontagem, mas os nomes semânticos dos campos e a relação completa com `VideoStreamInfo` ainda precisam ser confirmados.
10. Lê `context + 256` e `context + 260` para variáveis locais usadas no caminho de diagnóstico/log. A desmontagem não basta para atribuir nomes C definitivos a esses offsets.
11. Consulta quatro parâmetros inteiros de configuração por `CdcGetConfigParamterInt()`, com valores padrão passados em `w1` de 2, 3, 0 e 3, respectivamente. Os resultados são guardados em `sp + 224`, `sp + 228`, `sp + 232` e `sp + 236`.
12. Copia `context + 232` para `sp + 248` e `context + 240` para `sp + 292`; esses valores são usados no preparo/log da configuração.
13. Chama `InitializeVideoDecoder(decoder, &VideoStreamInfo, &VConfig)`.
14. Se o status retornado for diferente de zero, chama `DestroyVideoDecoder()` e zera `context + 0`.
15. Inicializa um `pthread_rwlock_t` em `context + 8` e retorna zero no caminho final mostrado.

### Interpretação e limites

- Os tamanhos observados, 72 bytes para `VideoStreamInfo` e 224 bytes para `VConfig`, são coerentes com as estruturas declaradas nos headers CedarX utilizados nesta investigação.
- `MemAdapterGetOpsS()` e `CdcMemOpen()` fazem parte da preparação do adaptador de memória antes da criação/inicialização efetiva do decoder.
- A ordem observada é `MemAdapterGetOpsS()` → `CdcMemOpen()` → inicialização do mutex → `CreateVideoDecoder()` → preparação dos parâmetros → `InitializeVideoDecoder()`.
- A função inicializa tanto um mutex em `context + 176` quanto um rwlock em `context + 8`. A desmontagem desta função, sozinha, não mostra em quais caminhos e threads esses locks são adquiridos; não se deve presumir que protegem especificamente a fila de frames sem examinar os respectivos callsites.
- Embora os endereços locais e tamanhos das estruturas estejam claros, o trecho não mostra todos os campos de `VideoStreamInfo`/`VConfig` sendo preenchidos individualmente. As áreas começam zeradas; valores adicionais são passados por meio de parâmetros de configuração e devem ser correlacionados com a ABI/headers e com um breakpoint imediatamente antes de `InitializeVideoDecoder()`.
- O `context + 104` seleciona os valores 277/278 e modifica um campo em `context + 248`; não atribuir ainda nomes como codec ID, pixel format ou stream index sem confirmar a estrutura que contém `context` e o destino `context + 248`.

### Próximo passo de análise

O próximo passo é parar imediatamente antes de `InitializeVideoDecoder()` e inspecionar os argumentos reais (`x0`, `x1`, `x2`), além dos campos das estruturas. Isso permitirá comparar os valores efetivos com os headers e esclarecer quais dos parâmetros de configuração consultados são copiados para `VConfig` por outra rotina ou por código não evidente neste trecho.


## 14.42 — Valores efetivos de VideoStreamInfo e VConfig antes de InitializeVideoDecoder()

Breakpoint colocado em `0xaaaaaaaa4494`, imediatamente antes da chamada a `InitializeVideoDecoder()`, no processo `/usr/bin/vdecoderdemo` executando H.264 (`-codFmat 1`) com saída NV12 (`-outFmat 6`). Registradores AArch64 observados:

- `x0 = 0xfffff0016a30`: instância `VideoDecoder*` (decoder).
- `x1 = 0xfffff7b9e720`: endereço da estrutura `VideoStreamInfo`.
- `x2 = 0xfffff7b9e768`: endereço da estrutura `VConfig`.

O GDB conseguiu interpretar diretamente as estruturas pelos tipos dos headers.

### VideoStreamInfo

Valores imediatamente antes da chamada:

- `eCodecFormat = 277` (`0x115`).
- `nWidth = 0`, `nHeight = 0`.
- `nFrameRate = 0`, `nFrameDuration = 0`, `nAspectRatio = 0`.
- `nCodecSpecificDataLen = 0`, `pCodecSpecificData = NULL`.
- `bIs3DStream`, `bSecureStreamFlag`, `bSecureStreamFlagLevel1`, `bIsFramePackage`, `bReOpenEngine`, `bIsFrameCtsTestFlag` e `bCheckInitFlag` = 0.
- `h265ReferencePictureNum = 0`.

**Conclusão limitada ao estado pré-inicialização:** o demo passa o codec 277 sem dimensões, frame rate ou codec-specific data pré-carregados. Isso é compatível com o decoder descobrir os parâmetros ao analisar o bitstream, mas a confirmação exige inspecionar a estrutura depois de `InitializeVideoDecoder()` e/ou após o parser processar o fluxo. Não preencher largura/altura artificialmente na integração antes dessa verificação.

### VConfig

Valores observados:

- `eOutputPixelFormat = 6` (NV12).
- `memops = 0xfffff7f75008` (`_ionMemOpsS`), retornado por `MemAdapterGetOpsS()`.
- `nDeInterlaceHoldingFrameBufferNum = 2`.
- `nDisplayHoldingFrameBufferNum = 3`.
- `nRotateHoldingFrameBufferNum = 0`.
- `nDecodeSmoothFrameBufferNum = 3`.
- `eCtlAfbcMode = DISABLE_AFBC_ALL_SIZE`; `nAfbcSecMode = DISABLE_AFBC_ALL_SIZE`.
- `eCtlIptvMode = DISABLE_IPTV_ALL_SIZE`.
- `veOpsS = NULL`, `pVeOpsSelf = NULL`.
- Todos os demais campos exibidos pelo GDB estão zerados, incluindo escala, rotação, saída secundária, `nVbvBufferSize`, `nFrameBufferNum`, `bGpuBufValid`, `nAlignStride`, `bIsSoftDecoderFlag`, `bVirMallocSbm` e `bSupportPallocBufBeforeDecode`.

### Implicações para a integração

1. A saída NV12 é solicitada explicitamente na configuração do decoder, não inferida do formato do bitstream.
2. O demo fornece `memops` Ion/CedarX; não é uma configuração de memória genérica. A integração precisa fornecer um `ScMemOpsS` compatível e respeitar seu ciclo de vida.
3. Os parâmetros de retenção 2/3/0/3 são valores reais observados, não apenas defaults presumidos pela desmontagem. Eles influenciam a quantidade de buffers mantidos para deinterlace, display, rotação e suavização de decode.
4. `veOpsS` e `pVeOpsSelf` nulos neste caminho não significam que o hardware VE não foi inicializado: o log anterior mostra `VeInitialize()` e abertura de `/dev/cedar_dev` com FD 4. A relação entre o VE global e esses campos da configuração ainda precisa ser esclarecida.
5. Os valores são um snapshot **antes** da chamada. Não demonstram se `InitializeVideoDecoder()` altera `VideoStreamInfo`, `VConfig` ou os campos de contexto.

### Próximo passo

Colocar um breakpoint na instrução imediatamente após a chamada (`0xaaaaaaaa4498`) e, sem sair da thread, reexaminar `*(VideoStreamInfo*)$x1` e `*(VConfig*)$x2`. Como a função ainda está ativa, os endereços de stack permanecem válidos. Comparar o antes/depois mostrará se a inicialização preenche dimensões ou modifica parâmetros.

## 14.43 — Estado das estruturas imediatamente após InitializeVideoDecoder()

A execução foi retomada a partir do breakpoint na entrada de `InitializeVideoDecoder()` e parou no breakpoint em `0xaaaaaaaa4498`, a instrução imediatamente posterior à chamada em `0xaaaaaaaa4494`. Os ponteiros para as estruturas foram preservados antes da chamada em `$pInfo` e `$pCfg`, evitando depender dos registradores caller-saved `x1` e `x2` após o retorno.

### VideoStreamInfo após a chamada

O conteúdo de `*$pInfo` permaneceu idêntico ao snapshot pré-chamada:

- `eCodecFormat = 277` (`0x115`, H.264).
- `nWidth = 0`, `nHeight = 0`.
- `nFrameRate = 0`, `nFrameDuration = 0`, `nAspectRatio = 0`.
- `nCodecSpecificDataLen = 0`, `pCodecSpecificData = NULL`.
- Os demais flags e campos exibidos continuam zerados.

**Conclusão:** `InitializeVideoDecoder()` não preencheu as dimensões, a taxa de quadros ou os dados específicos do codec nessa estrutura durante esta chamada. A hipótese de que o demo deixe esses parâmetros para serem descobertos posteriormente pelo parser do bitstream continua plausível, mas ainda precisa ser verificada após a submissão/análise do fluxo.

### VConfig após a chamada

Comparando `*$pCfg` antes e depois, foram observadas estas alterações:

| Campo | Antes | Depois |
|---|---:|---:|
| `veOpsS` | `NULL` | `0xfffff7d9dce8` (`_veAwOpsS`) |
| `pVeOpsSelf` | `NULL` | `0xfffff000cb50` |
| `nSetProcInfoFreq` | `0` | `60` |

Os demais valores exibidos permaneceram iguais, incluindo `eOutputPixelFormat = 6` (NV12), `memops = 0xfffff7f75008` (`_ionMemOpsS`) e os contadores de retenção 2/3/0/3.

**Conclusão:** durante `InitializeVideoDecoder()`, a configuração recebeu a tabela de operações VE específica da plataforma (`_veAwOpsS`) e um ponteiro de contexto/instância VE (`pVeOpsSelf`). Isso demonstra que esses campos são inicializados pela chamada, embora estivessem nulos no argumento de entrada. `nSetProcInfoFreq` também foi ajustado para 60. Não atribuir ainda significado operacional mais específico a esse último campo além do nome declarado no header.

### O que ainda falta confirmar

O breakpoint em `0xaaaaaaaa4498` confirma que a chamada retornou e que a execução alcançou a instrução seguinte. O dump das estruturas, por si só, **não informa o código de retorno** de `InitializeVideoDecoder()`. A desmontagem de `initDecoder()` mostra que o status não zero aciona `DestroyVideoDecoder()` e zera o ponteiro do decoder; portanto, é necessário inspecionar o registrador de retorno `x0` neste breakpoint (ou a variável local que recebe `w0`) antes de classificar a inicialização como bem-sucedida.

Próxima verificação recomendada no GDB, ainda parado em `0xaaaaaaaa4498`:

```gdb
p/d (int)$w0
p/x $x0
```

O primeiro comando apresenta o status de retorno de 32 bits com sinal; o segundo preserva a representação hexadecimal do registrador. Um status zero significa que `InitializeVideoDecoder()` retornou sucesso; valor diferente de zero indica erro e deve ser correlacionado com o caminho de limpeza da função chamadora.


## 14.44 — Código de retorno de InitializeVideoDecoder(): sucesso confirmado

No breakpoint em `0xaaaaaaaa4498`, imediatamente após a chamada a `InitializeVideoDecoder()`, foram consultados os registradores de retorno AArch64:

```gdb
p/d (int)$w0
p/x $x0
```

Resultados observados:

- `w0 = 0` (decimal).
- `x0 = 0x0` (hexadecimal).

Pela convenção de chamada AArch64, o valor de retorno inteiro de 32 bits é devolvido em `w0`; a leitura de `x0` também mostra zero neste caso. O status zero confirma que `InitializeVideoDecoder()` retornou sucesso nesta execução.

### Conclusão consolidada desta execução

1. `InitializeVideoDecoder()` recebeu o decoder H.264 (`eCodecFormat = 277`) e a configuração NV12 (`eOutputPixelFormat = 6`).
2. A chamada retornou `0`, portanto o caminho de erro da função chamadora — `DestroyVideoDecoder()` seguido do zeramento do ponteiro do decoder — não foi acionado por esta chamada.
3. Durante a chamada, `VConfig.veOpsS` passou de `NULL` para `_veAwOpsS` (`0xfffff7d9dce8`) e `VConfig.pVeOpsSelf` passou de `NULL` para `0xfffff000cb50`. `nSetProcInfoFreq` passou de `0` para `60`.
4. `VideoStreamInfo` permaneceu inalterado: largura, altura, frame rate e dados específicos do codec continuam zerados neste ponto.

Assim, temos evidência direta de inicialização bem-sucedida do decoder CedarX e de associação da configuração ao conjunto de operações VE da plataforma. Isso ainda não prova, isoladamente, que todos os frames subsequentes serão decodificados nem que a saída estará integrada ao pipeline de vídeo do Stremio; essas etapas precisam ser verificadas separadamente.

### Próxima investigação

A próxima etapa útil é acompanhar a atualização de `VideoStreamInfo` e o fluxo de submissão do bitstream até `DecodeVideoStream()`, além de confirmar quando o decoder passa a reportar dimensões efetivas e disponibilizar `VideoPicture`. Não é necessário repetir a inspeção do retorno de `InitializeVideoDecoder()` nesta execução: o status zero já está confirmado.


## 14.45 — `SubmitVideoStreamData()` e encaminhamento ao SBM (GDB)

A desmontagem completa de `SubmitVideoStreamData()` em `libvdecoder.so` (AArch64, endereço base observado `0xfffff7f38cc8`) mostrou que a função não faz parsing H.264 diretamente. Ela seleciona um descritor por `nStreamBufIndex`, agrega fragmentos conforme `bIsFirstPart`/`bIsLastPart` e encaminha os dados ao Stream Buffer Manager.

### Organização observada

- `pDecoder` é guardado no frame local; `nStreamBufIndex` é o terceiro argumento (`w2`).
- A função obtém o ponteiro do SBM por uma tabela indexada por `nStreamBufIndex + 0x2a` (índice multiplicado por 8).
- O descritor temporário por fluxo é calculado como `pDecoder + 0x160 + nStreamBufIndex * 64`.
- `VideoStreamDataInfo.bIsFirstPart` está no offset `+32`; quando diferente de zero, inicia um fragmento novo e trata eventual conteúdo anterior pendente.
- `VideoStreamDataInfo.bIsLastPart` está no offset `+36`; quando diferente de zero, o fragmento acumulado é finalizado e o descritor é limpo.
- No caminho de fragmento final, a função chama `FlushSbmData()` quando há dados pendentes e chama `SbmAddStream()` para encaminhar o descritor ao SBM. Há também um caminho que chama `SbmAddStream()` diretamente ao finalizar o fragmento.
- A função copia para o descritor os campos de dados/tamanho/timestamps e outros campos de metadados. `bVideoInfoFlag` (offset `+52`) e `pVideoInfo` (offset `+56`) são copiados; nesta função não foi observada interpretação direta do SPS/PPS.

### Valores concretos capturados

Na chamada observada na thread 4 (`parserThreadFunc()`): `pData=0xfffff6d032f6`, `nLength=54173`, `bIsFirstPart=1`, `bIsLastPart=1`, `nStreamBufIndex=0`, `bVideoInfoFlag=0`, `pVideoInfo=NULL`. O ponteiro `pData` pertence ao mapeamento `/dmabuf:` RW-S `0xfffff6510000–0xfffff6d10000`. Leitura direta via GDB falhou, mas cópia para heap convencional com `malloc()` + `memcpy()` funcionou.

Os primeiros bytes copiados foram `00 00 00 01 41 ...`, isto é, Annex B com NAL header `0x41` (NAL type 1, slice não-IDR). O scanner inicial encontrou ocorrências sobrepostas de start codes de 3 e 4 bytes; a inspeção dos offsets `0xcd79`, `0xcdb1`, `0xce8f` e `0xd21f` mostrou que o byte após as sequências `00 00 00 01` era `0x00`, portanto não são cabeçalhos NAL H.264 válidos. A ocorrência em `0xd007` tem byte `0x04` após o start code (tipo 4), mas ainda precisa ser contextualizada antes de classificá-la como NAL real. Não foi identificado SPS (tipo 7) nem PPS (tipo 8) nesse bloco.

### Próximo ponto de investigação

Desmontar `SbmAddStream()` (chamada direta observada em `0xfffff7f35064`) para seguir o encaminhamento ao SBM/parser. Também examinar `FlushSbmData()` se necessário. A hipótese de que SPS/PPS foram enviados em chamada anterior permanece em aberto; a desmontagem de `SubmitVideoStreamData()` não prova isso.


## 14.46 — Despacho de `SbmAddStream()` e identificação de `SbmFrameAddStream()` (GDB)

A desmontagem completa de `SbmAddStream()` em `libvdecoder.so` mostrou que ela é um wrapper de despacho indireto, sem processamento próprio do bitstream:

```asm
ldr x2, [x0, #64]
blr x2
```

O objeto SBM mantém, no offset `+64` (`+0x40`), um ponteiro de função. `SbmAddStream(pSelf, pDataInfo)` carrega esse ponteiro e chama-o com os mesmos argumentos (`x0` = objeto SBM; `x1` = descritor `VideoStreamDataInfo`).

No breakpoint em `SbmAddStream+32`, foram observados:
- `pSelf` (`x0`) = `0xfffff000c5c0`
- `pDataInfo` (`x1`) = `0xfffff0016b90`
- destino do callback (`x2`) = `0xfffff7eeca48`

`info symbol 0xfffff7eeca48` identificou o destino como `SbmFrameAddStream`, na seção `.text` de `/lib/aarch64-linux-gnu/libsbm.so`. Portanto, o callback efetivo selecionado pelo objeto SBM nesta execução é `SbmFrameAddStream()`.

Os primeiros 20 comandos de máquina de `SbmFrameAddStream()` mostram o prólogo, salvamento de argumentos e uma leitura global compatível com a proteção de stack; em seguida, verifica se `pSelf` e `pDataInfo` são nulos. O corpo ainda não foi examinado integralmente. Não se deve concluir ainda como o descritor é copiado, enfileirado ou interpretado.

### Próximo passo

Desmontar integralmente `SbmFrameAddStream()` com `disassemble /r SbmFrameAddStream` para localizar a implementação real de inserção do pacote no SBM e eventuais chamadas auxiliares.

## 14.46 — Implementação de SbmFrameAddStream() (GDB)

A desmontagem integral de `SbmFrameAddStream()` em `/lib/aarch64-linux-gnu/libsbm.so` (`0xfffff7eeca48–0xfffff7eecd8c`) revelou o mecanismo de inserção no SBM.

### Despacho

`SbmAddStream(pSelf, pDataInfo)` lê o ponteiro de função em `pSelf+64` e chama-o indiretamente. Na execução observada, o destino era `0xfffff7eeca48`, identificado por `info symbol` como `SbmFrameAddStream`.

Argumentos observados no breakpoint de `SbmAddStream`:
- `pSelf = 0xfffff000c5c0`
- `pDataInfo = 0xfffff0016b90`
- callback carregado em `x2 = 0xfffff7eeca48`.

### Validações e bloqueio

A função rejeita `pSelf == NULL` ou `pDataInfo == NULL`. Depois adquire o lock do objeto SBM. Sob o lock:
- lê o ponteiro de payload em `pDataInfo+0` e rejeita-o se for nulo;
- compara o contador em `pSelf+236` com o limite em `pSelf+232`; se o contador não for menor, retorna `-1`;
- verifica se `pDataInfo->nLength` (offset +8) somado ao acumulado em `pSelf+216` excede a capacidade em `pSelf+200`; se exceder, retorna `-1`.

Os retornos de erro liberam o lock. Os nomes semânticos dos campos internos ainda são inferidos pelos usos observados; confirmar com a estrutura/debug symbols antes de tratá-los como nomes oficiais.

### Cópia do descritor, não do payload

A função obtém um índice em `pSelf+248`, calcula o destino como:

`*(void **)(pSelf+224) + índice * 64`

E executa `memcpy(destino, pDataInfo, 64)`. Portanto, a cópia explícita observada é de 64 bytes do descritor `VideoStreamDataInfo` (ou estrutura compatível), não do conteúdo apontado por `pDataInfo->pData`.

Depois incrementa o índice de escrita em `pSelf+248`, com retorno a zero quando alcança o limite em `pSelf+232`. Incrementa os contadores em `pSelf+236` e `pSelf+240`.

### Contabilidade do buffer circular

A função soma `pDataInfo->nLength` ao acumulado em `pSelf+216`. Calcula o novo ponteiro a partir do ponteiro em `pSelf+208` mais o comprimento do pacote. Compara-o com o limite em `pSelf+192`; se ultrapassar esse limite, subtrai a capacidade em `pSelf+200`. O resultado é gravado de volta em `pSelf+208`.

Isso é compatível com gerenciamento de um buffer circular e uma fila de descritores. A desmontagem, isoladamente, não prova que o payload já foi escrito no buffer nem identifica quem realizou essa escrita. A hipótese mais provável é que o descritor aponte para bytes previamente disponibilizados/preenchidos no buffer SBM; confirmar inspecionando os ponteiros e os dados em runtime e a rotina que fornece espaço ao produtor.

### Sinalização da thread consumidora

O semáforo em `pSelf+0x130` é consultado por `sem_getvalue()`. Se o valor for zero, a função chama `sem_post()`. Isso indica sinalização de uma thread consumidora quando a fila/buffer recebe dados, embora a identidade e o comportamento da thread devam ser confirmados no código de espera.

Ao final, libera o lock e retorna zero. A função não chama diretamente parser H.264, não procura start codes e não interpreta SPS/PPS. O parser deve consumir os descritores/dados por outro caminho/thread.

### Próxima verificação

Inspecionar em runtime os campos de `pSelf` e `pDataInfo` no breakpoint de `SbmFrameAddStream`, relacionando `pDataInfo->pData` e `pDataInfo->nLength` com os campos internos `pSelf+192`, `+200`, `+208`, `+216`, `+224`, `+232`, `+236`, `+240` e `+248`. Isso permitirá testar se `pDataInfo->pData` aponta para a região circular e qual o significado efetivo dos contadores.

## 14.47 — Estado runtime de SbmFrameAddStream() e confirmação dos campos SBM

No breakpoint imediatamente antes da chamada indireta de `SbmAddStream()`, o callback resolvido foi `SbmFrameAddStream()` em `libsbm.so`, com:

- `pSelf = 0xfffff000c5c0`
- `pDataInfo = 0xfffff0016b90`
- `pDataInfo->pData = 0xfffff6d032f6`
- `pDataInfo->nLength = 0x0000d39d = 54.173` bytes.

Dump do descritor (`x/16wx 0xfffff0016b90`):

- offset `+0`: `0xfffff6d032f6` (ponteiro de dados);
- offset `+8`: `0x0000d39d` (comprimento, little-endian);
- offset `+16..+28`: zero;
- offset `+32`: `1`;
- offset `+36..+44`: zero;
- offset `+48`: `1`;
- offset `+52..+60`: zero.

O descritor atual corresponde ao pacote anteriormente observado em `SubmitVideoStreamData()`: mesmo endereço de dados e mesmo comprimento de 54.173 bytes.

### Campos internos observados em pSelf

O dump `x/8gx pSelf+192` retorna:

| Offset | Valor | Leitura sustentada pela desmontagem |
|---|---:|---|
| `+192` | `0x0000fffff6d0ffff` | Limite superior usado na comparação para wrap do ponteiro circular |
| `+200` | `0x0000000000800000` | Capacidade usada nos cálculos: 8 MiB |
| `+208` | `0x0000fffff6d032f6` | Ponteiro atualizado somando o comprimento e aplicando wrap |
| `+216` | `0x00000000007b007b` | Acumulado de bytes usado na verificação de capacidade; valor decimal 8.061.051 |
| `+224` | `0x0000fffff64ef010` | Ponteiro para a tabela de descritores |
| `+232` | `0x0000000000000800` | Limite de entradas usado no wrap do índice: 2.048 |
| `+236` | `0x0000000000000035` | Contador incrementado após inserir descritor: 53 no instante do dump |
| `+240` | `0x0000000000000034` | Segundo contador incrementado após inserir descritor: 52 no instante do dump |
| `+248` | `0x00000000000000a0` | Índice usado para selecionar o slot da tabela: 160 no instante do dump |
| `+252` | `0x000000000000006b` | Campo adjacente ainda não identificado |

A distinção de offsets é importante: os campos em `+232/+236` e `+248` são acessados como inteiros de 32 bits. Em particular, `+232` é o limite 2048, `+236` é incrementado como contador, e `+248` é o índice usado no cálculo `tabela + índice*64` e depois incrementado com wrap pelo limite em `+232`. Os campos em `+240`, `+244` e `+252` ainda não têm semântica estabelecida.

O valor `pSelf+216 = 0x007b007b` é tratado pela função como acumulado de bytes: a rotina soma `pDataInfo->nLength` e compara com a capacidade `pSelf+200`. Embora seja um valor alto (cerca de 96,1% de 8 MiB), não deve ser interpretado como corrupção sem uma observação temporal adicional.

### Implicação para o caminho dos dados

A função copia 64 bytes do descritor para `*(pSelf+224) + índice*64`; não há `memcpy` do payload nessa rotina. O endereço `pDataInfo->pData` também coincide com o ponteiro interno em `pSelf+208` antes do avanço calculado. A rotina calcula o próximo ponteiro somando `nLength` e, se ultrapassar o limite em `pSelf+192`, subtrai a capacidade de `pSelf+200`.

Isso confirma a gestão de descritores e a aritmética de wrap de um buffer circular. Ainda não prova, isoladamente, quem preencheu os bytes nem se o descritor já está sendo consumido pelo parser no momento da chamada.


## 14.47 — Confirmação da cópia do descritor na fila SBM (GDB)

Breakpoint em `SbmFrameAddStream+524` (`0xfffff7eecc54`), imediatamente após o retorno de `memcpy`, confirmou os argumentos:

- destino: `0xfffff64f1810` (`x0`)
- origem: `0xfffff0016b90` (`x1`)
- tamanho: `0x40` = 64 bytes (`x2`)

Comparação com `x/16wx`:

```text
Origem 0xfffff0016b90:
f6d032f6 0000ffff 0000d39d 00000000
00000000 00000000 00000000 00000000
00000001 00000000 00000000 00000000
00000001 00000000 00000000 00000000

Destino 0xfffff64f1810:
f6d032f6 0000ffff 0000d39d 00000000
00000000 00000000 00000000 00000000
00000001 00000000 00000000 00000000
00000001 00000000 00000000 00000000
```

Os 64 bytes são idênticos. Portanto, nesta etapa `SbmFrameAddStream()` copia integralmente o descritor para o slot da fila (`tabela + índice*64`), preservando o ponteiro do payload `0xfffff6d032f6` e o comprimento `0xd39d` (54.173 bytes). Não copia o payload de vídeo nessa chamada.

O breakpoint confirma o slot observado para esta chamada, mas a identificação de todos os campos do descritor e o mecanismo de consumo pela thread do parser ainda dependem de análise adicional.

### Ressalva sobre o buffer circular

A desmontagem mostra o cálculo do próximo ponteiro de escrita como `pSelf+208 + pDataInfo->nLength`; compara esse endereço com o limite em `pSelf+192` e, se ultrapassar, subtrai a capacidade em `pSelf+200` (8 MiB). Assim, `+192` é usado como limite superior e `+200` como tamanho/capacidade na operação de wrap; não deve ser descrito como endereço-base sem evidência adicional.

O valor observado em `pSelf+216` foi `0x007b007b` (8.061.051 decimal). A função o trata como acumulado de bytes, somando o comprimento recebido. Esse valor é próximo da capacidade de 8 MiB; é necessário observar o estado em outros instantes para determinar se é um estado normal de ocupação ou se há alguma particularidade na estrutura/execução.


## 14.48 — Confirmação da cópia do descritor e identificação do consumidor SBM

### Cópia de 64 bytes confirmada

No breakpoint em `SbmFrameAddStream+524`, imediatamente após `memcpy`, foram examinados os 64 bytes da origem (`pDataInfo`, endereço `0xfffff0016b90`) e do destino (`0xfffff64f1810`). Os dois dumps `x/16wx` são idênticos palavra por palavra. Isso confirma experimentalmente que o slot da tabela recebe uma cópia integral do descritor de 64 bytes. O descritor mantém `pData=0xfffff6d032f6` e `nLength=0xd39d` (54.173 bytes); o payload não é copiado por esse `memcpy`.

### Símbolos do SBM identificados

A saída de `info functions Sbm` revelou, entre outras, estas rotinas de `sbmFrameBase.c`:

- `SbmFrameAddStream()` — linha 839; produtor já analisado.
- `SbmFrameRequestFramePic()` — linha 915; candidata principal para retirar um descritor da fila de quadros.
- `SbmFrameReturnFramePic()` — linha 957; candidata para devolver o descritor após o consumo.
- `SbmFrameFlushFramePic()` — linha 1002; rotina de flush.
- `SbmFrameRequestBuffer()` — linha 778; solicitação de espaço/buffer.

Também aparecem `SbmStreamRequestStream()` e `SbmStreamReturnStream()` em `sbmStream.c`; pertencem ao caminho de SBM de fluxo e não devem ser confundidas com as rotinas `SbmFrame*`. O próximo teste deve verificar se `SbmFrameRequestFramePic()` é realmente chamado durante a execução e em qual thread.

### Cuidado com scheduler-locking no GDB

Como `set scheduler-locking on` estava ativo durante a investigação, um `continue` pode deixar as demais threads sem executar. Antes de esperar que a thread consumidora atinja um breakpoint, desativar esse modo (`set scheduler-locking off`). Em seguida, colocar breakpoint em `SbmFrameRequestFramePic` e continuar.


## 14.49 — Chamada real de SbmFrameRequestFramePic pelo decodificador H.264

O breakpoint em `SbmFrameRequestFramePic` foi atingido após `set scheduler-locking off`. O argumento `pSelf` foi `0xfffff000c5c0`, o mesmo objeto SBM usado por `SbmFrameAddStream` e pela thread `CedarcSbmThread`.

A pilha de chamadas no instante do breakpoint foi:

```text
#0 SbmFrameRequestFramePic(pSelf=0xfffff000c5c0)
#1 H264DetectIsFullFrame()                 libawh264.so
#2 H264DecoderDecodeSbmFrame()             libawh264.so
#3 VideoEngineDecode()                     libvideoengine.so
#4 DecodeVideoStream(...)                  libvdecoder.so
#5 DecodeThread()                          vdecoderdemo
```

O `info threads` mostrou seis threads. As duas relevantes estavam simultaneamente associadas ao mesmo SBM:

- Thread 3 (`CedarcSbmThread`, LWP 37236): executando `searchStartCode(pSbm=0xfffff000c5c0, ...)` em `sbmFrameH264.c:402`.
- Thread 5 (`vdecoderdemo`, LWP 37432): parada em `SbmFrameRequestFramePic`, chamada pelo caminho `H264DetectIsFullFrame` → `H264DecoderDecodeSbmFrame`.
- Thread 4 (`vdecoderdemo`, LWP 37431): executando `SubmitVideoStreamData`, o produtor de dados.

Isso comprova que o decodificador H.264 consulta `SbmFrameRequestFramePic` durante `H264DecoderDecodeSbmFrame`, enquanto uma thread separada do SBM executa `searchStartCode`. A thread 4 também estava no caminho de submissão de dados. Portanto, produtor, thread de parser e consumidor do decodificador são caminhos concorrentes sobre o mesmo objeto SBM.

No breakpoint, `x0=0xfffff000c5c0` corresponde ao argumento `pSelf`. `x1=0xfffff7eecd90` foi apenas registrado; como o breakpoint está na entrada da função, não se deve atribuir significado a `x1` sem examinar o prólogo e a desmontagem.

**Próxima investigação:** desmontar `SbmFrameRequestFramePic` para identificar as condições de fila vazia, a seleção do slot, a cópia/retorno do descritor e a atualização do ponteiro/índice de leitura. Depois, comparar com `SbmFrameReturnFramePic`.


## 14.50 — Desmontagem de SbmFrameRequestFramePic: fila de quadros e índice de leitura

Desmontagem obtida em GDB para `SbmFrameRequestFramePic`, endereço `0xfffff7eecd90`, tamanho 660 bytes. O breakpoint foi atingido na entrada da função, com `pSelf=0xfffff000c5c0`.

### Fluxo confirmado pela desmontagem

1. Verifica `pSelf` contra NULL.
2. Adquire o lock por `lock(pSelf)`. Se o lock falhar, retorna NULL.
3. Lê um inteiro de 32 bits em `pSelf+288`. Se for zero, libera o lock e retorna NULL: esse campo funciona como contador de quadros disponíveis nesta fila.
4. Lê o ponteiro-base da tabela em `pSelf+272`.
5. Lê o índice de leitura em `pSelf+292` e calcula o endereço do slot como:
   `slot = *(void **)(pSelf+272) + index * 80`
   O cálculo ARM64 é `index * 5`, seguido de deslocamento à esquerda por 4 bits; portanto, stride de **80 bytes por slot**.
6. Incrementa o índice em `pSelf+292`. Se o novo índice alcança o limite armazenado em `pSelf+280`, zera o índice para fazer o wrap-around.
7. Decrementa o contador em `pSelf+288`.
8. Libera o lock e retorna o endereço calculado do slot.

A função não executa `memcpy` do payload nem chama `sem_wait`. Ela retorna um ponteiro para o slot da tabela. Se o contador estiver zerado, retorna NULL imediatamente.

### Campos identificados nesta rotina

| Offset em `pSelf` | Operação observada | Interpretação confirmada até aqui |
|---:|---|---|
| `+272` | `ldr x2, [x0,#272]` | Ponteiro-base da tabela de slots |
| `+280` | comparado com o índice após incremento | Limite do índice de leitura / tamanho lógico da tabela |
| `+288` | testado para zero e decrementado | Contador de slots/quadros disponíveis |
| `+292` | usado para calcular slot, incrementado e reiniciado | Índice de leitura |
| `+712` | testado para valor igual a 1 | Flag usada apenas no caminho de logging desta função; sem semântica adicional confirmada |

### Distinção entre as duas estruturas observadas

`SbmFrameAddStream` copia 64 bytes do `VideoStreamDataInfo` para uma tabela de entrada, cujo endereço-base está em `pSelf+224` e cujo índice de escrita está em `pSelf+248`; o slot dessa tabela usa stride de 64 bytes (`index << 6`). Já `SbmFrameRequestFramePic` calcula um slot em outra tabela, baseada em `pSelf+272`, com stride de 80 bytes e índice de leitura em `pSelf+292`. Portanto, não se deve confundir a tabela de descritores de entrada de 64 bytes com a tabela de slots de quadros de 80 bytes.

A desmontagem mostra que `SbmFrameRequestFramePic` consome uma entrada da fila de quadros, mas ainda não revela integralmente o layout dos 80 bytes nem o produtor correspondente dessa tabela. O próximo passo é inspecionar `SbmFrameReturnFramePic` e, se necessário, o código que publica/preenche os slots.


## 14.51 — Desmontagem de SbmFrameReturnFramePic: validação e reposição do slot

Desmontagem obtida em GDB para `SbmFrameReturnFramePic`, endereço `0xfffff7eed024`, tamanho 616 bytes. A assinatura inferida pelos registradores/armazenamentos de argumentos é `SbmFrameReturnFramePic(pSelf, pFrameInfo)`.

### Fluxo confirmado pela desmontagem

1. Verifica se `pSelf` e o segundo argumento são diferentes de NULL; caso contrário retorna `-1`.
2. Há um caminho de logging que testa `pSelf+712`; depois a função adquire `lock(pSelf)`.
3. Lê o inteiro em `pSelf+284`. Se for zero, libera o lock e retorna `-1`. A semântica exata desse contador ainda não foi identificada.
4. Lê o índice em `pSelf+292` e calcula o índice anterior (`index - 1`). Se ficar negativo, usa `pSelf+280 - 1`, fazendo wrap-around.
5. Calcula o endereço do slot anterior na tabela-base `pSelf+272`, com stride de 80 bytes: `slot = base + previous_index * 80`.
6. Compara esse endereço com o segundo argumento (`pFrameInfo`). Se forem diferentes, emite mensagem de erro e chama `abort()`. Portanto, a função exige que o ponteiro devolvido corresponda exatamente ao slot imediatamente anterior ao índice de leitura atual.
7. Se forem iguais, grava o índice anterior em `pSelf+292`, incrementa o contador em `pSelf+288`, libera o lock e retorna 0.

### Consequência para o ciclo Request/Return

A sequência de `SbmFrameRequestFramePic` e `SbmFrameReturnFramePic` é mais específica do que uma fila FIFO convencional: Request calcula o slot no índice atual, avança `pSelf+292` (com wrap-around) e decrementa `pSelf+288`; Return só aceita o slot imediatamente anterior ao índice atual, recua `pSelf+292` para esse slot e incrementa `pSelf+288`. Isso repõe o contador e restaura o índice para o slot devolvido. A comparação estrita seguida de `abort()` demonstra que a devolução fora dessa ordem/slot não é aceita. Ainda não é possível concluir, apenas com essas duas rotinas, se o desenho pressupõe estritamente uma única solicitação pendente ou se a camada superior garante a devolução em ordem LIFO.

### Campos adicionais

| Offset em `pSelf` | Operação observada | Interpretação até aqui |
|---:|---|---|
| `+272` | Base usada para calcular o slot | Ponteiro-base da tabela de slots de 80 bytes |
| `+280` | Usado como limite no wrap-around | Limite/tamanho lógico da tabela |
| `+284` | Testado contra zero antes da devolução | Contador/estado que habilita Return; sem nome semântico confirmado |
| `+288` | Incrementado após Return | Contador de slots/quadros disponíveis, consistente com o decremento em Request |
| `+292` | Índice atual; recuado e gravado | Índice de leitura/slot atual |
| `+712` | Testado para valor igual a 1 em ramo de logging | Flag sem semântica confirmada |

**Ponto ainda em aberto:** a função não copia nem libera o payload. Ela valida o endereço do slot e atualiza apenas o índice/contador, sob lock. O layout interno dos 80 bytes e o significado de `pSelf+284` precisam ser esclarecidos examinando quem preenche a tabela e quem chama Return.


## 14.52 — Breakpoint de Return ainda não atingido; interrupção anterior em SubmitVideoStreamData

O usuário configurou `break SbmFrameReturnFramePic` (breakpoint 12 em `0xfffff7eed034`) e executou `continue`. A execução parou antes no breakpoint 8, já existente, em `SubmitVideoStreamData` (`vdecoder.c:1544`), na thread 4 (`vdecoderdemo`, LWP 37431). A pilha observada foi:

```text
#0 SubmitVideoStreamData(pDecoder=0xfffff0016a30, pDataInfo=0xfffff5cdd8a8, nStreamBufIndex=0)
#1 parserThreadFunc()
#2 libc
#3 libc
```

Registradores no ponto de interrupção: `x0=0xfffff0016a30` (pDecoder) e `x1=0xfffff5cdd8a8` (pDataInfo). A função `SbmFrameReturnFramePic` ainda não foi observada nesta execução; portanto, estes dados não identificam seu chamador nem o endereço do slot devolvido. O fato de o breakpoint 8 ter interrompido primeiro não indica falha no breakpoint 12.

**Próximo procedimento:** desabilitar temporariamente o breakpoint 8 (`disable 8`) e continuar (`continue`) para permitir que a execução alcance `SbmFrameReturnFramePic`. Se outro breakpoint anterior interromper, identificar e desabilitar apenas o breakpoint que está impedindo o avanço, preservando o breakpoint 12. Quando o breakpoint 12 for atingido, coletar `bt` e `info registers x0 x1`.


## 14.52 — Chamada real de SbmFrameReturnFramePic observada em execução

O usuário conseguiu atingir o breakpoint 12 em `SbmFrameReturnFramePic`. Antes disso, o breakpoint 8 em `SubmitVideoStreamData` interrompeu a execução; foi desabilitado com `disable 8`. Durante o avanço, também ocorreram interrupções em `ReturnPicture` (breakpoint 3), `SbmFrameRequestFramePic` (breakpoint 11) e `SbmAddStream` (breakpoint 9). Esses eventos mostram atividade simultânea das threads de submissão, decodificação e retorno de imagens, mas não alteram a interpretação da função de retorno.

### Contexto de chamada confirmado

No breakpoint 12, GDB apresentou:

```text
Thread 5 "vdecoderdemo" hit Breakpoint 12,
SbmFrameReturnFramePic (pSelf=0xfffff000c5c0, pDataInfo=0xfffff00297e0)

#0  SbmFrameReturnFramePic (pSelf=0xfffff000c5c0, pDataInfo=0xfffff00297e0)
#1  H264DetectIsFullFrame () from libawh264.so
#2  H264DecoderDecodeSbmFrame () from libawh264.so
#3  VideoEngineDecode () from libvideoengine.so
#4  DecodeVideoStream (...) at vdecoder.c:1285
#5  DecodeThread ()
```

Registradores na entrada da função:

- `x0 = 0xfffff000c5c0`: objeto SBM (`pSelf`).
- `x1 = 0xfffff00297e0`: descritor recebido (`pDataInfo`).

### Conclusão atualizada

Está confirmado em execução que `SbmFrameReturnFramePic` é chamada diretamente por `H264DetectIsFullFrame`, durante `H264DecoderDecodeSbmFrame`. Portanto, a devolução do slot do SBM faz parte do fluxo interno de decodificação H.264, e não deve ser confundida com `ReturnPicture`, que apareceu separadamente na thread 6. A observação de `ReturnPicture` e `SbmFrameReturnFramePic` em threads distintas reforça que são caminhos diferentes: o primeiro pertence ao gerenciamento de `VideoPicture` no demo; o segundo gerencia slots de quadro no SBM.

O argumento `pDataInfo=0xfffff00297e0` ainda precisa ser inspecionado. A desmontagem de `SbmFrameReturnFramePic` indica que a função calcula o endereço esperado como `base + índice_anterior * 80` e compara esse endereço com o segundo argumento; se forem diferentes, chama `abort()`. Ainda não foi verificado se `0xfffff00297e0` coincide com o slot esperado, nem foram lidos os campos desse descritor.

### Próximo passo de depuração

Com o processo parado na entrada do breakpoint 12, inspecionar os 80 bytes do descritor:

```gdb
x/20gx 0xfffff00297e0
```

Se GDB não conseguir ler como palavras de 64 bits, tentar:

```gdb
x/80bx 0xfffff00297e0
```

Em seguida, correlacionar os bytes com o slot calculado pela função e com os campos do objeto SBM (`pSelf+272`, `pSelf+280` e `pSelf+292`). Não presumir ainda o significado de cada campo do descritor.


## 14.54 — Inspeção da estrutura auxiliar referenciada pelo slot do SBM

Breakpoint em `SbmFrameReturnFramePic` confirmou o chamador `H264DetectIsFullFrame()` (via `H264DecoderDecodeSbmFrame`). O argumento `pDataInfo` foi `0xfffff00297e0`, dentro da tabela de slots de 80 bytes.

Dump `x/20wx 0xfffff00297e0` (80 bytes):

| Offset | Valor |
|---|---:|
| +0x00 | `0xf002b910` (parte baixa de ponteiro) |
| +0x04 | `0x0000ffff` |
| +0x08 | `0x00000020` |
| +0x0C | `0x00000001` |
| +0x10 | `0x0000cdbb` |
| +0x14 | `0` |
| +0x18 | `0xf656c80b` (parte baixa de ponteiro) |
| +0x1C | `0x0000ffff` |
| +0x20 | `0` |
| +0x24 | `0` |
| +0x28 | `0xffffffff` |
| +0x2C | `0xffffffff` |
| +0x30 | `1` |
| +0x34 | `1` |
| +0x38 | `0` |
| +0x3C | `0` |

Ponteiros reconstruídos: `0xfffff002b910` em +0x00 e `0xfffff656c80b` em +0x18.

Inspeção `x/16wx 0xfffff002b910` da estrutura apontada pelo primeiro ponteiro:

- +0x00 = 1
- +0x04 = 1
- +0x08/+0x0C = ponteiro `0xfffff656c80f`
- +0x10 = `0xcdb7`
- demais palavras observadas até +0x3C = 0

Correlação observada: o ponteiro da estrutura auxiliar (`0xfffff656c80f`) e o ponteiro do slot (+0x18, `0xfffff656c80b`) diferem por 4 bytes; os valores associados (`0xcdb7` e `0xcdbb`) também diferem por 4. Ambos parecem referenciar posições próximas na mesma região do bitstream. Hipótese ainda não confirmada: os endereços podem delimitar início de cabeçalho e início de payload, ou duas posições relacionadas ao parser. Não atribuir semântica definitiva aos campos sem rastrear seus usos.

A evidência reforça que os slots SBM carregam referências e metadados de bitstream, em vez de necessariamente conter cópias dos dados comprimidos.


## 14.54 — Leitura do buffer associado ao slot: falha de acesso direto no GDB

Em 2026-11-22, durante a inspeção de `SbmFrameReturnFramePic`, foi tentada a leitura direta de 32 bytes a partir de `0xfffff656c800` com `x/32bx`. O GDB respondeu `Cannot access memory at address 0xfffff656c800`. Isso não demonstra que o ponteiro seja inválido: o endereço deriva dos campos do slot e da estrutura auxiliar, e buffers DMA-BUF previamente observados no processo também não eram legíveis por `x`, embora pudessem ser copiados com `memcpy` executado no processo inferior.

Próximo teste proposto: alocar um buffer temporário no processo inferior, copiar 64 bytes a partir de `0xfffff656c800` usando `memcpy` e inspecionar a cópia. Exemplo no GDB:

```gdb
set $probe = (unsigned char*)malloc(64)
call (void*)memcpy($probe, (void*)0xfffff656c800, 64)
x/64bx $probe
```

A cópia deve ser tratada como teste de leitura, não como prova antecipada de que o conteúdo é um cabeçalho H.264. Se `memcpy` provocar falha, interromper a inspeção e reavaliar a validade/mapeamento do endereço.


## 14.55 — Leitura do buffer apontado pelo slot SBM usando memcpy (2026-11-22)

### Contexto

A leitura direta pelo GDB de `0xfffff656c800` com `x/32bx` falhou (`Cannot access memory`). A tentativa inicial de `malloc()` foi interrompida porque outras threads pararam em breakpoints. Após desabilitar os breakpoints 10, 11 e 12 e ativar temporariamente `set scheduler-locking on`, a chamada inferior foi concluída.

### Comandos executados

```gdb
set scheduler-locking on
set $probe = (unsigned char*)malloc(64)
call (void*)memcpy($probe, (void*)0xfffff656c800, 64)
x/64bx $probe
set scheduler-locking off
```

`memcpy()` retornou o endereço do buffer de destino `0xffffe0002d10` e a leitura foi bem-sucedida.

### Bytes observados

```text
0xffffe0002d10: 0x6f 0x7f 0xee 0x30 0x45 0xe8 0x7f 0x07
0xffffe0002d18: 0xc0 0x1a 0xc4 0x00 0x00 0x00 0x01 0x41
0xffffe0002d20: 0x9a 0xe0 0x4a 0x37 0x1b 0xe0 0x00 0x86
0xffffe0002d28: 0x74 0xc4 0x1d 0x6a 0xcf 0xfe 0xec 0x5f
0xffffe0002d30: 0xfe 0x03 0x02 0x2b 0x86 0xd9 0x99 0xa1
0xffffe0002d38: 0xbc 0xc5 0xc4 0x65 0x6f 0x06 0x05 0xc6
0xffffe0002d40: 0x0c 0x9e 0xb4 0x8a 0x36 0x67 0x35 0x63
0xffffe0002d48: 0x1f 0x0c 0x5c 0xa4 0x3d 0x6a 0xed 0x4f
```

### Interpretação

- A sequência `00 00 00 01 41` começa no deslocamento `+14` em relação à base copiada `0xfffff656c800`.
- `00 00 00 01` é um start code Annex B; `0x41` indica NAL H.264 tipo 1 (slice não-IDR).
- O slot retornado por `SbmFrameReturnFramePic` contém em `+0x18` o ponteiro `0xfffff656c80b`, isto é, deslocamento `+11` em relação à base copiada. Nesse endereço, o start code aparece três bytes adiante.
- A estrutura auxiliar em `0xfffff002b910` contém em `+0x08` o ponteiro `0xfffff656c80f`, quatro bytes depois de `0xfffff656c80b`; o campo em `+0x10` é `0xcdb7`, enquanto o slot principal contém `0xcdbb` em `+0x10`.
- A relação entre esses ponteiros e comprimentos sugere posições relacionadas dentro da mesma região de memória, mas ainda não determina a semântica dos campos nem prova que o endereço seja o início efetivo do payload H.264.
- A leitura via `memcpy()` demonstra que o processo consegue acessar a região, embora a leitura direta do GDB falhe. Isso é compatível com o comportamento observado anteriormente em buffers DMA-BUF.

### Próxima verificação

Copiar 64 bytes começando exatamente em `0xfffff656c80b` e em `0xfffff656c80f`. A comparação permitirá identificar o que cada ponteiro efetivamente referencia, sem assumir previamente a semântica dos campos da estrutura.


## 14.56 — Comparação dos ponteiros do slot SBM e da estrutura auxiliar

Com `set scheduler-locking on`, foram feitas duas cópias de 64 bytes por `memcpy()` no processo inferior, pois a leitura direta com `x` no endereço DMA-BUF não funcionava. Ambas as cópias foram concluídas com sucesso.

- Slot principal, ponteiro em `pDataInfo+0x18`: `0xfffff656c80b`. Os primeiros bytes são `00 00 00 01 41 9a e0 4a ...`.
- Estrutura auxiliar, ponteiro em `aux+0x08`: `0xfffff656c80f`. Os primeiros bytes são `41 9a e0 4a 37 1b e0 00 ...`.
- A diferença entre os endereços é exatamente 4 bytes.

Conclusão observacional: ambos apontam para a mesma sequência H.264; o primeiro aponta para o prefixo Annex B de quatro bytes (`00 00 00 01`), e o segundo aponta para o byte de cabeçalho NAL (`0x41`). `0x41 & 0x1f = 1`, portanto o NAL é do tipo 1 (slice não-IDR). A estrutura auxiliar guarda um ponteiro deslocado quatro bytes em relação ao ponteiro do slot. A semântica completa dos campos ainda depende da análise do chamador `H264DetectIsFullFrame()`.

A cópia iniciada em `0xfffff656c800` também funcionou por `memcpy()`, embora `x/32bx` diretamente nesse endereço retornasse `Cannot access memory`. Isso reforça que a leitura por código executado no processo inferior é necessária para esse mapeamento.
\n\n## 14.59 — Desmontagem de H264DetectIsFullFrame e análise dos slots SBM\n\nA desmontagem de `H264DetectIsFullFrame()` (base `0xfffff6f12484`) confirmou como a rotina consome os slots retornados por `SbmFrameRequestFramePic()`.\n\n### Estrutura de dados confirmada em runtime\n\nA rotina executa, em essência:\n\n```asm\nldr x2, [x0]       // primeiro campo do slot: ponteiro para estrutura auxiliar\nldr x1, [x2, #8]   // auxiliar+8: ponteiro para dados\nldr w0, [x2]       // auxiliar+0: campo de controle\nldr w2, [x2, #16]  // auxiliar+16: valor de tamanho/comprimento\n```\n\nIsso corresponde aos valores observados no slot `0xfffff00297e0` e na estrutura auxiliar `0xfffff002b910`:\n\n- slot `+0x00` = `0xfffff002b910` (ponteiro para estrutura auxiliar);\n- auxiliar `+0x00` = `1`;\n- auxiliar `+0x04` = `1`;\n- auxiliar `+0x08` = `0xfffff656c80f` (ponteiro usado pela rotina como endereço dos dados);\n- auxiliar `+0x10` = `0xcdb7` (valor usado como comprimento/limite associado aos dados).\n\nO slot principal `+0x18` continha `0xfffff656c80b`, exatamente quatro bytes antes do ponteiro auxiliar. A leitura por `memcpy` demonstrou que `0xfffff656c80b` começa com `00 00 00 01 41`, enquanto `0xfffff656c80f` começa com `41`. Portanto, o primeiro endereço aponta para o start code Annex B e o segundo para o cabeçalho NAL, sem o start code. O byte `0x41` indica NAL type 1 (slice não-IDR).\n\n### Percurso dos slots\n\nA função inicializa um vetor local de até 32 ponteiros (slots de 8 bytes na pilha), chama indiretamente o callback em `objeto+72` para solicitar slots e armazena os slots não nulos. O contador `w28` é incrementado no loop e a comparação com `0x20` limita a varredura a 32 iterações.\n\nNo final, percorre os ponteiros armazenados e chama indiretamente o callback em `objeto+80`, que corresponde a `SbmFrameReturnFramePic()`. Isso confirma o ciclo Request/Return observado nos breakpoints.\n\n### Análise dos dados do NAL\n\nPara os slots aceitos pela filtragem, a rotina usa o ponteiro de dados (`auxiliar+8`) e o valor em `auxiliar+16` para calcular um limite. Compara esse limite e o início dos dados com um endereço de referência obtido do contexto do decoder. Em um caminho, copia bytes para um buffer temporário local com `memcpy`; em seguida chama `H264InitGetBitsSw`, `H264GetBitsSw(..., 8)` e `H264GetUeGolombSw`. Isso comprova a leitura de um campo de 8 bits seguida da decodificação de um valor unsigned Exp-Golomb.\n\nA rotina compara o valor Exp-Golomb obtido com um valor de referência mantido em `w21`. A finalidade exata do campo H.264 ainda não está identificada apenas por esta desmontagem; não atribuir ainda um nome semântico como `first_mb_in_slice` ou `slice_type` sem acompanhar a posição exata do bitstream e a implementação dos auxiliares.\n\nA filtragem também consulta `auxiliar+0` e trata especificamente o valor 5; valores 1 e 2 seguem outro caminho de análise, enquanto outros valores podem ser ignorados. O significado definitivo desse campo de controle permanece pendente.\n\n### Conclusões confirmadas\n\n1. `H264DetectIsFullFrame()` solicita até 32 slots SBM por meio do callback Request.\n2. O primeiro campo do slot aponta para uma estrutura auxiliar.\n3. O campo auxiliar `+8` é usado como ponteiro para dados; `+16` é usado como comprimento/limite.\n4. O campo slot `+0x18` observado aponta para o start code Annex B, quatro bytes antes do ponteiro de dados da estrutura auxiliar.\n5. A rotina analisa bits do conteúdo H.264 e compara um valor Exp-Golomb entre entradas.\n6. Os slots solicitados são devolvidos pelo callback Return ao final da rotina.\n\nPermanecem em aberto a semântica dos campos auxiliares `+0` e `+4`, a identificação exata do campo Exp-Golomb lido e a interpretação completa dos limites usados na cópia.\n

## 14.56 — Desmontagem de H264GetBitsSw e leitura do cabeçalho de slice

A desmontagem de `H264GetBitsSw()` confirma os campos da estrutura de leitura usados pela rotina:

- `+416`: ponteiro para o buffer de bits (`ldr x5, [x0, #416]`).
- `+432`: posição atual em bits (`ldr w4, [x0, #432]`).

A função lê bytes do buffer, extrai o número solicitado de bits e atualiza a posição. O caso `n == 1` extrai um único bit; o caminho geral trata leituras de múltiplos bits.

Em `H264DetectIsFullFrame()`, a sequência observada é `H264InitGetBitsSw(x22, x27, 64)`, `H264GetBitsSw(x22, 8)` e `H264GetUeGolombSw(x22)`. O buffer temporário começa no ponteiro auxiliar `0xfffff656c80f`, cujo primeiro byte observado é `0x41`. Assim, os oito bits iniciais consumidos correspondem ao cabeçalho NAL; o campo Exp-Golomb seguinte, para NALs slice comuns dos tipos 1 e 5, corresponde a `first_mb_in_slice`.

A rotina compara o valor decodificado com um valor anterior (`cmp w0, w21`; ramo `b.le` para retorno 1). Isso é compatível com uma heurística de detecção de mudança de quadro baseada na progressão de `first_mb_in_slice`, mas a semântica completa depende da desmontagem de `H264GetUeGolombSw()` e da análise de todos os caminhos do loop.


## 14.57 — Identificação do campo Exp-Golomb em H264DetectIsFullFrame

A desmontagem de `H264GetUeGolombSw()` confirma que a função decodifica um código unsigned Exp-Golomb (`ue(v)`) a partir do cursor de bits mantido em `ctx+432`, usando o buffer apontado por `ctx+416`. Ela lê uma janela de bits, determina o comprimento do código e atualiza o cursor. As tabelas consultadas são tabelas auxiliares de decodificação; a função retorna o valor `ue(v)`.

Em `H264DetectIsFullFrame()`, a sequência efetiva é:

```c
H264InitGetBitsSw(ctx, temporary_buffer, 64);
H264GetBitsSw(ctx, 8);          // consome o cabeçalho NAL
value = H264GetUeGolombSw(ctx); // primeiro ue(v) do RBSP
```

O buffer temporário começa no ponteiro `aux+8`, que no caso observado é `0xfffff656c80f`; seu primeiro byte é `0x41`. Os oito bits consumidos são, portanto, o cabeçalho NAL. Para NALs de slice dos tipos 1 (non-IDR) e 5 (IDR), o primeiro campo `ue(v)` após o cabeçalho NAL é `first_mb_in_slice`, conforme a sintaxe do `slice_header`. O tipo 2 (data partition A), que também passa pela filtragem observada, possui o cabeçalho de slice correspondente. Assim, a identificação do campo como `first_mb_in_slice` é sustentada pela ordem dos campos H.264 e pela sequência de chamadas.

A rotina compara o valor decodificado (`w0`) com o valor de referência em `w21` (`cmp w0, w21`). O ramo `b.le` leva ao caminho que define `w21=1` e encerra a varredura. Isso é compatível com uma detecção baseada na progressão/ordem de `first_mb_in_slice`. Não se deve, contudo, resumir toda a lógica como uma simples comparação de slices consecutivos: o controle de `w21`, `w23`, os NALs ignorados e os caminhos de saída exige considerar o loop completo.

### Detalhes do leitor de bits

- `ctx+416`: ponteiro para o buffer de bits.
- `ctx+432`: posição atual em bits.
- `H264GetBitsSw(ctx, 8)`: avança oito bits, isto é, um byte.
- `H264GetUeGolombSw(ctx)`: decodifica um valor `ue(v)` e avança o cursor pelo comprimento do código.

A implementação usa leitura de bytes e tabelas de lookup para decodificar o código Exp-Golomb. A desmontagem não mostra, dentro dessas duas funções, remoção explícita de bytes de emulação `0x03`; portanto, não inferir que o leitor faça sozinho a conversão EBSP→RBSP. Essa conversão pode ocorrer antes, ou os dados examinados podem simplesmente não exigir remoção nesse trecho.

### Conclusão atualizada

A rotina `H264DetectIsFullFrame()` analisa o primeiro campo `ue(v)` do cabeçalho de slice, identificado como `first_mb_in_slice` para os NALs de slice processados. Isso estabelece que o SBM fornece à rotina dados H.264 utilizáveis para a análise do cabeçalho, e que a detecção não depende apenas dos metadados do slot. A interpretação exata de todos os resultados e dos caminhos de saída permanece sujeita à análise do loop completo.


## 14.58 — Reconstrução do controle de comparação em H264DetectIsFullFrame

A análise conjunta do loop em `H264DetectIsFullFrame()` refina a interpretação da comparação de `first_mb_in_slice`. Não é correto descrevê-la simplesmente como comparação entre slices consecutivos.

### Inicialização e referência

Antes do loop, a função define `w21 = -1` e `w23 = -1`. Para cada slot:

1. Solicita um slot pelo callback Request.
2. Obtém o ponteiro da estrutura auxiliar (`slot+0`) e lê o ponteiro dos dados (`aux+8`) e o campo de controle (`aux+0`).
3. Filtra o campo de controle. Os valores 1, 2 e 5 seguem o caminho de análise do slice; os valores 7 e 8 passam por um caminho especial; outros valores são ignorados pela lógica principal.
4. Para os NALs processados como slices, prepara até 64 bytes, inicializa o leitor de bits, consome 8 bits do cabeçalho NAL e decodifica o primeiro `ue(v)`, identificado como `first_mb_in_slice`.

### Comparação e atualização

No endereço `H264DetectIsFullFrame+356` (`0xfffff6f125e8`), o valor recém-decodificado em `w0` é copiado para `w23` e comparado com `w21`:

```asm
mov w23, w0
cmp w0, w21
b.le <+532>
```

Se `w0 <= w21` em comparação signed, o caminho `+532` define `w21=1` e segue para a rotina de devolução dos slots. Se o valor for maior, a função continua.

No caminho de continuação, `cmn w21, #1` testa se `w21 == -1`. Se ainda estiver no valor sentinela, `csel w21, w21, w23, ne` escolhe `w23` quando a condição é EQ; assim, a primeira leitura de slice válida inicializa `w21` com o primeiro `first_mb_in_slice` observado. Nas iterações seguintes, `w21` permanece como referência inicial, enquanto `w23` recebe o valor recém-lido.

Portanto, a comparação relevante é entre cada novo `first_mb_in_slice` e a referência inicial estabelecida pelo primeiro slice analisado, e não necessariamente entre dois slices consecutivos. Uma regressão ou repetição (`novo <= referência`) aciona o retorno 1.

### Tratamento especial dos valores 7 e 8

O campo em `aux+0` é comparado com 5; valores maiores são subtraídos de 7 e comparados com 1. Isso faz os valores 7 e 8 seguirem um caminho especial, sem executar a leitura normal do cabeçalho de slice naquele ponto. Se a referência ainda não foi inicializada, a varredura pode continuar; se já foi inicializada, o caminho especial pode acionar o retorno 1. Não atribuir nomes semânticos definitivos a esse campo apenas com base nesses valores: a hipótese de que ele represente `nal_unit_type` é plausível, mas precisa ser confirmada na origem da estrutura auxiliar.

### Ponto de observação em runtime

Para confirmar os valores reais, colocar breakpoint imediatamente antes da comparação:

```gdb
break *0xfffff6f125e8
continue
info registers w0 w21 w23 w28
```

Nesse ponto:
- `w0`: `first_mb_in_slice` recém-decodificado;
- `w21`: referência atual (sentinela `0xffffffff` antes do primeiro slice válido, depois o valor do primeiro slice);
- `w23`: valor anterior/temporário até ser sobrescrito por `w0` na instrução seguinte;
- `w28`: contador de iterações do loop.

Depois de registrar uma ocorrência, usar `continue` para observar a próxima comparação. O breakpoint pode ser atingido várias vezes; registrar os valores em sequência é mais informativo que uma única parada.

### Correção de precisão

A identificação de `first_mb_in_slice` é sustentada pela posição do cursor: a rotina consome o byte de cabeçalho NAL e decodifica o primeiro `ue(v)` do RBSP. A interpretação do retorno 1 como detecção de fronteira/condição de quadro é consistente com o nome e a lógica de progressão, mas a semântica completa dos caminhos especiais e do campo `aux+0` permanece pendente de confirmação em runtime ou na desmontagem do produtor desses descritores.


## 14.59 — Preparação do breakpoint na comparação de `first_mb_in_slice`

Foi criado com sucesso o breakpoint GDB nº 13 no endereço `0xfffff6f125e8`, identificado como `H264DetectIsFullFrame+356`. Ele está imediatamente antes das instruções que copiam `w0` para `w23` e comparam `w0` com `w21`.

Saída de `info breakpoints`: o breakpoint 13 está habilitado. Também estavam habilitados os breakpoints antigos nº 1 (`InitializeVideoDecoder`), 3 (`ReturnPicture`), 5 e 6 (em `initDecoder`), 7 (`GetVideoStreamInfo`) e 9 (`SbmAddStream`). Os breakpoints 2, 4, 8, 10, 11 e 12 estavam desabilitados.

Para evitar interrupções não relacionadas durante a captura dos valores do parser, a próxima ação recomendada é desabilitar os breakpoints antigos 1, 3, 5, 6, 7 e 9, mantendo o nº 13 ativo, e executar `continue`. Quando o nº 13 parar, coletar `info registers w0 w21 w23 w28`. Nesse ponto, `w0` deve conter o `first_mb_in_slice` recém-decodificado; `w21`, a referência inicial ou o sentinela `-1`; `w23` ainda não terá sido atualizado pela instrução seguinte; e `w28` é o contador do loop. Não avançar a execução antes de registrar esses valores.


## 14.60 — Breakpoints antigos desabilitados e execução retomada

O usuário executou `disable 1 3 5 6 7 9`. A confirmação por `info breakpoints` mostrou todos os breakpoints de 1 a 12 desabilitados (`Enb n`) e o breakpoint 13 habilitado (`Enb y`), no endereço `0xfffff6f125e8` (`H264DetectIsFullFrame+356`).

Em seguida, `continue` retomou a execução e o GDB informou a troca para a thread 4, LWP 37431 (`[Switching to Thread 0xfffff5cde180 (LWP 37431)]`). A saída fornecida termina nesse aviso, sem mostrar ainda uma parada no breakpoint 13 nem um novo prompt `(gdb)`. Portanto, não há valores de registradores para interpretar neste estágio. Aguardar a próxima parada; quando o breakpoint 13 for atingido, executar `info registers w0 w21 w23 w28` e preservar os valores antes de continuar.


## 14.61 — Primeira captura no breakpoint da comparação de `first_mb_in_slice`

O breakpoint 13 em `0xfffff6f125e8` (`H264DetectIsFullFrame+356`) foi atingido na thread 5 (`LWP 37432`). A coleta `info registers w0 w21 w23 w28` retornou:

- `w0 = 0x0`: primeiro `first_mb_in_slice` decodificado tem valor 0.
- `w21 = 0xffffffff`: sentinela -1, indicando que a referência ainda não foi inicializada.
- `w23 = 0xffffffff`: ainda contém o sentinela, pois a instrução `mov w23, w0` está imediatamente após o endereço do breakpoint e ainda não foi executada.
- `w28 = 0`: primeira iteração do loop.

A captura é consistente com a interpretação da rotina: primeiro slice analisado, valor `first_mb_in_slice` igual a zero e referência ainda não inicializada. A próxima ação é inspecionar as instruções a partir do PC atual com `x/8i $pc`, sem avançar a execução. Com a sequência exata visível, executar passo a passo e observar quando `w23` e `w21` recebem o valor zero.


## 14.62 — Trecho inicial e caminhos de saída de H264DecoderDecodeSbmFrame (+0 a +328; +1604 a +1648)

Foi recebida a desmontagem de `H264DecoderDecodeSbmFrame()` nos intervalos `+0..+328` e `+1604..+1648`.

### Inicialização e estado auxiliar

No prólogo:

- `x19` preserva o argumento original `x0` (contexto/objeto do decoder).
- `x25 = [x0+400]`.
- `x22` recebe inicialmente `[x25+40]`; no caminho em que `[x25+48] == 0`, é substituído por `[x25+32]`.
- `x21 = x22 + 0xC000` e `x20 = x22 + 0x8000`.
- O endereço `x22 + 0xE000` é guardado em `[sp+104]`.
- O argumento original `w2` é guardado em `[sp+124]`.

No caminho `[x25+48] == 0`, são inicializados/gravados campos relativos a `x21`:

- `[x21+9420] = 0`
- `[x21+9528] = w1`
- `[x21+9544] = w3`
- `[x21+9552] = w2` (recuperado da pilha)
- `[x20+25920] = x4`

Esses campos permanecem sem rótulos semânticos; a desmontagem mostra apenas seus offsets e valores gravados.

### Máquina de estados em `[x22+0xE000]+1216`

O byte lido em `+104` é testado em sequência:

- `0`: desvia para `H264DecoderDecodeSbmFrame+1096`.
- `1`: desvia para `+2456`.
- `7`: segue pelo caminho que começa em `+264`.
- Outros valores: seguem para `+1972`, onde `w28` recebe `5` e o fluxo retorna ao laço/saída comum.

Isso confirma que o byte não é uma flag booleana simples. Os valores 0, 1 e 7 têm caminhos específicos; não há base suficiente neste trecho para nomear semanticamente cada estado.

### Contador e tabela de registros com stride 24

No caminho `+264`, o decoder carrega `[x20+25640]` em `x26`. Se nulo, desvia para `+988`. Caso contrário:

- lê o contador `[x21+9168]` em `w2`;
- lê a base da tabela em `[x26]`;
- calcula `endereço = base + contador * 24`;
- lê `w23` no offset `+0` do registro;
- lê um ponteiro em `+8`;
- lê `w24` em `+16`;
- grava o ponteiro de `+8` em `[x20+25520]`.

A evidência permite afirmar que o código percorre registros de 24 bytes e usa o contador em `[x21+9168]`. O significado dos três campos ainda não foi identificado.

Se `w23 == 7`, o código chama indiretamente a callback armazenada em `[objeto+72]` (`+316..+324`) e preserva o retorno em `x4`. A identidade da callback em `+72` já foi confirmada anteriormente como `SbmFrameRequestFramePic()`. Portanto, neste caminho, o decoder solicita uma estrutura de quadro ao SBM. O valor retornado é mantido em `x4` para uso posterior; este fragmento, isoladamente, não mostra ainda todo o processamento desse retorno.

### Caminho de callback em +1604

No intervalo `+1604..+1648`:

1. `x2 = [x0+88]` seleciona a callback no offset `+88` do objeto de callbacks.
2. `x1 = [x20+25640]` é passado como segundo argumento; `x4` é salvo temporariamente em `[sp+104]`.
3. A callback `+88` é chamada indiretamente.
4. O objeto de callbacks é recarregado e `x2 = [x0+80]` seleciona a callback `+80`.
5. O campo `[x20+25640]` é zerado.
6. `x1` recebe o valor preservado de `x4` e a callback `+80` é chamada.
7. O fluxo retorna ao caminho comum em `+600`.

A callback `+80` já foi identificada como `SbmFrameReturnFramePic()`. A callback `+88` ainda não foi identificada. O padrão observado mostra uma chamada `+88` seguida da devolução, via `+80`, da estrutura cujo ponteiro estava em `x4`; não se deve atribuir semântica adicional à callback desconhecida sem examinar sua implementação.

### Relação com a análise anterior

Este fragmento liga diretamente o caminho de `H264DecoderDecodeSbmFrame()` à callback `SbmFrameRequestFramePic()` já estudada e mostra também um caminho de devolução por `SbmFrameReturnFramePic()`. Isso é coerente com o mecanismo de slots do SBM já documentado: a callback de Request fornece uma estrutura e a callback de Return devolve uma estrutura específica. A tabela de 24 bytes e o byte de estado em `+1216` são estruturas distintas e ainda precisam ser rastreadas até seus produtores.

### Próxima investigação

Para identificar a callback em `[objeto+88]`, inspecionar em runtime o objeto apontado por `[x20+25496]` e comparar os três ponteiros de função em offsets `+72`, `+80` e `+88`. O próximo trecho útil da desmontagem continua a partir de `+328`, especialmente para acompanhar o uso de `x4`, `w23` e `w24` e identificar o produtor/consumidor da tabela de registros.


## 14.62 — Continuação de H264DecoderDecodeSbmFrame (+328 a +596)

Desmontagem recebida:
- `+328..+596`: `0xfffff6f10828` até `0xfffff6f10934`.
- Função: `H264DecoderDecodeSbmFrame()`.
- Continuação do trecho que chama `SbmFrameRequestFramePic()` em `+316..+324`.

### 1. Slot solicitado e primeiro campo da estrutura auxiliar

Após a callback em `+324`, `x0` contém o ponteiro retornado. Em `+328`, ele é preservado em `x4`. Se for NULL, o fluxo vai para `+1576`.

Se não for NULL:

```asm
+336  ldr x1, [x0]       // primeiro campo do slot: ponteiro para estrutura auxiliar
+344  ldr w1, [x1]       // aux+0
+348  and w1, w1, #0xfffffffb  // limpa o bit 2
+352  cmp w1, #1
+356  b.eq +1604
```

Portanto, o primeiro campo da estrutura auxiliar é lido como um valor de 32 bits e testado após limpar o bit 2 (`value & ~4`). Se o resultado for 1, o código segue para o caminho especial em `+1604`, que chama a callback de offset `+88` e depois a callback `+80` para devolver o slot preservado em `x4` (ver seção sobre o caminho +1604). Não atribuir significado semântico definitivo ao bit 2 ou ao valor 1 sem identificar quem constrói esse campo.

Se o teste não desviar, o código chama indiretamente a callback em `[objeto_de_callbacks+80]`, passando `x4` como argumento (`+360..+368`). Essa callback já foi identificada como `SbmFrameReturnFramePic()`. Assim, neste ramo, o slot solicitado é devolvido imediatamente antes de continuar usando os dados da estrutura auxiliar/tabela.

### 2. Extração de registro de 24 bytes

O código então executa:

```asm
+372  ldr x0, [x26]
+376  add x0, x0, x28
+380  ldr w0, [x0, #4]
+384  and w28, w23, #0xff
+388  str w0, [x22, #24]
+392  mov w2, w28
+396  mov w3, w24
+400  mov x1, x22
+404  ldrb w4, [x19, #440]
+412  mov x0, x19
+416  bl H264DecodeProcessData
```

O trecho anterior (`+268..+304`) já havia calculado `x28 = contador * 24`, com `x1=[x26]` como base da tabela, lido `w23` do primeiro DWORD do registro, `x1=[registro+8]`, `w24=[registro+16]`, e armazenado o ponteiro de `[registro+8]` em `[x20+25520]`.

Neste trecho:
- `w0` é carregado de `registro+4` e gravado em `[x22+24]`;
- `w28` recebe os 8 bits inferiores de `w23`;
- `w24` é passado como terceiro argumento inteiro;
- `w4` recebe o byte `[x19+440]` e é passado como quarto argumento;
- `H264DecodeProcessData(x19, x22, w28, w24, w4)` é chamado (argumentos AArch64 em `x0..x4`).

O código confirma a existência de registros de 24 bytes e que seus campos em offsets `+0`, `+4`, `+8` e `+16` são consumidos. A semântica dos campos continua parcialmente desconhecida.

### 3. Leitura de 8 bits e processamento da NAL

Após `H264DecodeProcessData`, o decoder carrega a callback em `[x19+448]` e a chama com `w1=8` (`+416..+428`). Isso é uma chamada indireta para uma operação de leitura de 8 bits, compatível com a função de leitura de bits do contexto do decoder; a identidade exata depende da configuração da callback em runtime.

Depois, se o valor global apontado por `x27` for maior que 2, o código chama:

```asm
+444  ldrb w4, [x25, #48]
+452  mov w2, w28
+456  mov x1, x22
+460  lsl w3, w24, #3
+464  mov x0, x19
+468  bl H264ProcessNaluUnit
```

Logo, `H264ProcessNaluUnit()` recebe:
- `x0 = x19`: contexto/objeto do decoder;
- `x1 = x22`: estrutura de trabalho/descritor usado pelo decoder;
- `w2 = w28`: byte inferior do primeiro DWORD do registro (`w23 & 0xff`);
- `w3 = w24 * 8`: comprimento convertido de bytes para bits;
- `w4 = [x25+48]`: flag de estado.

O resultado de `H264ProcessNaluUnit()` é salvo em `w28`. Em seguida, `[x21+9648]` é incrementado por `w24`, indicando que esse contador acumula o comprimento em bytes fornecido pelo registro. O valor de retorno de `H264ProcessNaluUnit()` é então comparado com vários códigos/estados (`0`, `1`, `2`, `4`, `0xfb`, `0xfc`, `0xff` em trechos posteriores); não presumir que todos sejam erros sem identificar a convenção de retorno.

Se `w23 == 6`, há um desvio específico para `+632`. Caso contrário, o retorno é mascarado com `~2` e comparado com 1; outros testes usam o byte assinado em `[x22]` e valores especiais `0xfb`/`0xfc` para escolher o próximo caminho.

### 4. Caminho de estado especial 0xff

Quando o retorno em `w28` é `0xff`, o código:
- grava byte `2` em `[sp_context+1120]`;
- grava byte `4` em `[sp_context+1216]`;
- se `[x20+25640]` não for NULL, chama a callback `+80` (`SbmFrameReturnFramePic`) com esse ponteiro e zera `[x20+25640]`;
- verifica o byte de estado em `[sp_context+1216]` e pode seguir para o caminho de retorno em `+1980`.

Esse ramo demonstra que `+1216` é um byte de estado consultado e atualizado pelo decoder, mas seu enum completo ainda não está identificado.

### 5. Conclusões confirmadas e limites

Confirmado pela desmontagem:
- `SbmFrameRequestFramePic()` retorna o slot que chega em `x4`.
- O primeiro campo do slot aponta para uma estrutura auxiliar; o DWORD em `aux+0` é filtrado por `& ~4`.
- A callback em offset `+80` é `SbmFrameReturnFramePic()` e pode ser chamada imediatamente após a Request neste caminho.
- Há uma tabela de registros com stride de 24 bytes. O decoder lê os campos `+0`, `+4`, `+8` e `+16`.
- `H264DecodeProcessData()` recebe o contexto, a estrutura de trabalho, o byte inferior do primeiro DWORD do registro, o comprimento e uma flag.
- `H264ProcessNaluUnit()` recebe o mesmo byte identificador e o comprimento em bits (`w24 << 3`); o contador `[x21+9648]` acumula `w24` em bytes.
- O resultado de `H264ProcessNaluUnit()` conduz uma máquina de estados com retornos/códigos especiais.

Ainda não confirmado:
- O significado semântico dos DWORDs dos registros de 24 bytes.
- O significado do bit 2 em `aux+0`.
- A identidade exata da callback `[x19+448]` em runtime.
- A enumeração e o significado de todos os retornos de `H264ProcessNaluUnit()`.
- Se os registros de 24 bytes representam NALUs já delimitadas, fragmentos ou outra unidade intermediária. O comprimento e a chamada de processamento são compatíveis com uma unidade de dados/NAL, mas isso precisa ser confirmado pela origem da tabela.

### Próxima investigação

O passo mais informativo é identificar a callback armazenada em `[x19+448]` e observar em runtime os campos do registro de 24 bytes imediatamente antes de `H264DecodeProcessData()` e `H264ProcessNaluUnit()`. Para isso, usar breakpoint em `0xfffff6f1087c` e inspecionar `x19`, `x22`, `w23`, `w24`, `w28`, `x26`, `[x26]` e os 24 bytes do registro selecionado. Não alterar os dados em memória.

## 14.64 — Breakpoint em H264DecodeProcessData: dados reais do slot (runtime)

Breakpoint 14 em `0xfffff6f1087c` (`H264DecoderDecodeSbmFrame+412`), imediatamente antes da chamada a `H264DecodeProcessData()`.

Registradores observados:

```text
x19 = 0xfffff000cc80
x22 = 0xfffff002d1a0
x23 = 1
x24 = 0xcdb7 (52663)
x26 = 0xfffff00297e0
x28 = 1
```

Memória em `x26`:

```text
0xfffff00297e0: 0xf002b910 0x0000ffff 0x00000020 0x00000001
0xfffff00297f0: 0x0000cdbb 0x00000000
```

Interpretação segura:
- `x26` é o slot de 80 bytes retornado pela callback Request, como já estabelecido.
- O primeiro qword do slot é o ponteiro `0xfffff002b910` para a estrutura auxiliar.
- O DWORD em `slot+8` é `0x20`; o DWORD em `slot+12` é `1`.
- O DWORD em `slot+16` é `0xcdbb` (52.667), enquanto `x24` contém `0xcdb7` (52.663). A diferença observada é de quatro bytes.
- `x23=1` e `x28=1`; o código anterior transforma `w23` em `w28` com `and w28,w23,#0xff`. Portanto, o terceiro argumento de `H264DecodeProcessData()` é 1.
- `x24=0xcdb7` é o comprimento passado no quarto argumento; o código subsequente também o converte em bits (`w24 << 3`) para `H264ProcessNaluUnit()`.
- `x22` aponta para uma estrutura de contexto/trabalho, mas os primeiros 24 bytes observados são zero. Isso não demonstra que a estrutura inteira esteja zerada; os campos relevantes estão em offsets mais altos.

A chamada em `+412` é preparada assim:

```asm
mov w2, w28       // argumento 3: 1
mov w3, w24       // argumento 4: 0xcdb7
ldrb w4, [x19,#440] // argumento 5: flag armazenada no contexto
mov x1, x22       // argumento 2: contexto de trabalho
mov x0, x19       // argumento 1: contexto do decoder
bl H264DecodeProcessData
```

O breakpoint confirma os valores dos registradores e do slot, mas ainda não identifica a semântica de `slot+8`, `slot+12`, nem a razão exata da diferença de quatro bytes entre `slot+16` e `x24`. Também não permite concluir o valor de `aux+0`, porque a estrutura auxiliar apontada pelo primeiro qword do slot ainda precisa ser inspecionada nesta mesma execução.

### Próxima verificação

Sem avançar o processo, inspecionar a estrutura auxiliar e os argumentos restantes:

```gdb
x/6wx 0xfffff002b910
x/2gx $x19+440
info registers x0 x1 x2 x3 x4
```

O primeiro comando deve revelar `aux+0`, `aux+4`, `aux+8` e `aux+16`; os outros permitem confirmar a flag do quinto argumento e a preparação completa da chamada. Não modificar memória.

## 14.64 — Confirmação da estrutura auxiliar e dos argumentos de H264DecodeProcessData (+412)

Breakpoint 14 em `0xfffff6f1087c`, imediatamente antes da chamada `H264DecodeProcessData()`:

```gdb
x/6wx 0xfffff002b910
0xfffff002b910: 0x00000001 0x00000001 0xf656c80f 0x0000ffff
0xfffff002b920: 0x0000cdb7 0x00000000

x/2gx $x19+440
0xfffff000ce38: 0x0000000000000001 0x0000fffff6f06e14

info registers x0 x1 x2 x3 x4
x0 = 0xfffff000cc80
x1 = 0xfffff002d1a0
x2 = 1
x3 = 0xcdb7
x4 = 1
```

### Conclusões confirmadas

- A estrutura auxiliar começa em `0xfffff002b910` e contém:
  - `aux+0 = 1`;
  - `aux+4 = 1`;
  - `aux+8 = 0xfffff656c80f` (ponteiro para os dados);
  - `aux+16 = 0xcdb7` (52.663 bytes).
- O ponteiro `aux+8` é quatro bytes posterior ao ponteiro `slot+0x18` observado anteriormente (`0xfffff656c80b`). O segundo começa pelo prefixo Annex B `00 00 00 01`; o primeiro começa diretamente no byte `0x41`, cabeçalho de uma NAL H.264 do tipo 1.
- O comprimento auxiliar `0xcdb7` é quatro bytes menor que o comprimento em `slot+16` (`0xcdbb`). Portanto, os quatro bytes do prefixo Annex B não fazem parte do intervalo apontado por `aux+8` e não são contados no comprimento auxiliar. A diferença de quatro bytes está explicada pela remoção lógica do start code, não por truncamento inexplicado.
- `[x19+440] = 1`. O qword seguinte, em `[x19+448]`, contém o endereço `0xfffff6f06e14`, identificado anteriormente como `H264GetBitsSw`.
- Os cinco argumentos efetivos preparados para `H264DecodeProcessData()` são:
  1. `x0 = 0xfffff000cc80` — contexto do decoder (`x19`);
  2. `x1 = 0xfffff002d1a0` — contexto de trabalho (`x22`);
  3. `w2 = 1` — valor derivado do primeiro DWORD da estrutura auxiliar (`aux+0`);
  4. `w3 = 0xcdb7` — comprimento da região NAL, sem o prefixo Annex B;
  5. `w4 = 1` — byte de configuração em `[x19+440]`.
- `aux+0=1` confirma o valor observado no teste anterior do campo de controle da estrutura auxiliar. Ainda não se deve chamar esse campo de “tipo NAL”: o tipo H.264 é codificado nos cinco bits inferiores do byte apontado por `aux+8`; `0x41 & 0x1f = 1` identifica NAL type 1. O campo `aux+0` é um metadado separado, cujo significado exato ainda precisa ser estabelecido.
- O ponteiro de função em `[x19+448]` e a flag em `[x19+440]` são coerentes com a seleção do caminho de processamento SW/HW observada na desmontagem. Com a flag igual a 1, a rotina posterior seleciona `H264ProcessExtraDataSW`; isso não prova, isoladamente, que toda a decodificação do quadro ocorra em software.

### Próximo passo

Executar `ni` uma vez para passar por cima da chamada a `H264DecodeProcessData()` e inspecionar o retorno em `w0`/`x0`. Em seguida, verificar se a execução chega à chamada indireta em `+428`, cujo alvo foi confirmado como `H264GetBitsSw`:

```gdb
ni
info registers x0 w0 pc
```

Não alterar os argumentos nem a memória nesta etapa.

## 14.65 — Retorno de H264DecodeProcessData (+412 a +416)

No breakpoint 14, a execução estava imediatamente antes da chamada a `H264DecodeProcessData()` em `0xfffff6f1087c`. Foi executado um único `ni`:

```gdb
ni
0x0000fffff6f10880 in H264DecoderDecodeSbmFrame ()
info registers x0 w0 pc
x0  = 0x0
w0  = 0x0
pc  = 0xfffff6f10880 <H264DecoderDecodeSbmFrame+416>
```

### Conclusões confirmadas

- A chamada `H264DecodeProcessData()` terminou e o fluxo retornou ao chamador, em `H264DecoderDecodeSbmFrame+416`.
- O retorno em `w0` é zero (`x0=0`). Isso é compatível com sucesso/ausência de erro, mas a semântica exata do código de retorno ainda depende da convenção da biblioteca ou da análise dos chamadores; não atribuir significado definitivo apenas a este caso.
- A próxima instrução em `+416` carrega o ponteiro de função de `[x19+448]`; em `+424`, prepara `w1=8` e, em `+428`, chama esse ponteiro indiretamente:

```asm
ldr x2, [x19, #448]
mov x0, x19
mov w1, #8
blr x2
```

- O valor de `[x19+448]` foi inspecionado no passo anterior e é `0xfffff6f06e14`, endereço já identificado como `H264GetBitsSw`. Portanto, a próxima chamada será `H264GetBitsSw(x19, 8)`, que lê oito bits do bitstream usando o contexto do decoder.
- Ainda não foi observado o retorno dessa chamada nem o efeito sobre o cursor de bits. O próximo passo é avançar uma instrução sobre o `blr` e inspecionar `w0`, `pc` e o cursor `[x19+432]` (offset usado por `H264GetBitsSw` para a posição em bits).

### Próximo passo

No estado atual, o PC está em `+416`. A chamada indireta está em `+428`; são quatro instruções até ela (`ldr`, `mov x0`, `mov w1`, `blr`). Execute `ni` quatro vezes, confirmando que o PC retorna a `+432`, e então inspecione:

```gdb
ni
ni
ni
ni
info registers x0 w0 pc
x/wd $x19+432
```

Assim veremos o retorno de `H264GetBitsSw` e o cursor de bits do contexto. Não modificar memória.


## 14.68 — Retorno de H264GetBitsSw: leitura do cabeçalho NAL (breakpoint 14)

### Evidência de runtime

Após `H264DecodeProcessData()` retornar zero em `H264DecoderDecodeSbmFrame+416`, foi executado o trecho que carrega `[x19+448]` e chama a função com `w1=8`. O ponteiro já havia sido identificado como `H264GetBitsSw` (`0xfffff6f06e14`).

Saída do GDB após quatro instruções `ni`:

```text
x0  = 0x41
w0  = 0x41
pc  = 0xfffff6f10890 <H264DecoderDecodeSbmFrame+432>
[x19+432] = 8
```

### Interpretação

- `H264GetBitsSw(ctx, 8)` retornou `0x41` (decimal 65).
- O cursor de bits em `ctx+432` ficou em 8, confirmando o consumo de exatamente oito bits.
- O byte `0x41` é o cabeçalho NAL observado anteriormente no endereço `aux+8` (`0xfffff656c80f`), após o prefixo Annex B de quatro bytes.
- Em H.264, `0x41` (`01000001b`) decodifica-se como `forbidden_zero_bit=0`, `nal_ref_idc=2` e `nal_unit_type=1` (slice não-IDR).

Esta observação confirma em runtime a leitura do cabeçalho NAL pelo leitor de bits em software. Não demonstra, isoladamente, que a decodificação completa do quadro seja feita em software: o caminho posterior ainda precisa ser acompanhado.

### Próximo ponto de investigação

O PC está em `H264DecoderDecodeSbmFrame+432` (`0xfffff6f10890`). A instrução seguinte lê `[x27]` e compara o valor com 2 antes de decidir entre o caminho de `H264ProcessNaluUnit()` e outro ramo. Inspecionar `x27` e avançar esse teste permitirá determinar qual ramo é tomado para este NAL type 1.


## 14.69 — GLOBAL_LOG_LEVEL e seleção do caminho de processamento da NAL (+432 a +464)

### Evidência de runtime

No ponto `H264DecoderDecodeSbmFrame+432`, após a chamada a `H264GetBitsSw(ctx, 8)`, foi inspecionado `x27`:

```text
(gdb) x/wd $x27
0xfffff7fa0008 <GLOBAL_LOG_LEVEL>: 5
```

Após executar a carga e a comparação:

```text
w0 = 5
pc = 0xfffff6f10898 <H264DecoderDecodeSbmFrame+440>
```

### Interpretação

A instrução em `+432` carrega `GLOBAL_LOG_LEVEL` em `w0`; em `+436`, compara esse valor com 2; em `+440`, o branch condicional `b.ls` decide se desvia para `+932`. Como `GLOBAL_LOG_LEVEL=5`, a condição `5 <= 2` é falsa. Portanto, quando a instrução de branch for executada, o fluxo seguirá em frente para `+444`, e não para `+932`.

**Correção de interpretação:** `[x27]` não é um campo de controle da NAL nem do decoder; é a variável global `GLOBAL_LOG_LEVEL`. A comparação regula um caminho condicionado ao nível de logging. Não se deve atribuir significado de codec ao valor 5.

O caminho sequencial chega à chamada `H264ProcessNaluUnit()` em `+464` (`0xfffff6f108b0`), desde que nenhuma condição intermediária altere o fluxo. Isso ainda precisa ser confirmado executando o branch.

### Próximo passo

Com o PC ainda em `+440`, execute apenas:

```gdb
ni
info registers pc
```

O PC esperado é `0xfffff6f1089c` (`+444`), confirmando que o branch não foi tomado. Depois podemos colocar um breakpoint diretamente na chamada de `H264ProcessNaluUnit()` (`0xfffff6f108b0`) para examinar os argumentos reais.


## 14.67 — Confirmação do desvio após GLOBAL_LOG_LEVEL e preparação de H264ProcessNaluUnit (+440 a +464)

### Evidência de runtime

Após o usuário executar `ni` no endereço `0xfffff6f10898` (`H264DecoderDecodeSbmFrame+440`), o GDB informou:

```text
(gdb) ni
0x0000fffff6f1089c in H264DecoderDecodeSbmFrame ()
(gdb) info registers pc
pc = 0xfffff6f1089c <H264DecoderDecodeSbmFrame+444>
```

Isso confirma que o branch `b.ls` em `+440` não foi tomado com `GLOBAL_LOG_LEVEL=5`; a execução prosseguiu para `+444`. O desvio para `+932` só ocorre quando o nível global de log é menor ou igual a 2.

### Próxima preparação de argumentos

A partir de `+444`, a sequência é:

```asm
+444  ldrb w4, [x25, #48]
+448  mov  w2, w28
+452  mov  x1, x22
+456  lsl  w3, w24, #3
+460  mov  x0, x19
+464  bl   H264ProcessNaluUnit
```

Assim, imediatamente antes da chamada:
- `x0 = x19`: contexto principal do decoder;
- `x1 = x22`: contexto de trabalho;
- `w2 = w28`: resultado/estado retornado por `H264ProcessNaluUnit` ainda será obtido;
- `w3 = w24 << 3`: comprimento convertido de bytes para bits, se `w24` representar o comprimento em bytes;
- `w4 = *(uint8_t *)(x25+48)`: estado/flag de configuração ainda sem semântica determinada.

O comprimento em `w3` é passado em bits porque o código faz explicitamente `lsl #3`. O valor concreto de `w24` no breakpoint anterior era `0xcdb7`, logo o comprimento esperado em bits é `0x66db8` (421.304), caso não tenha sido alterado entre os pontos.

### Próximo passo de depuração

Colocar breakpoint na própria instrução `bl` permite observar todos os argumentos já preparados, sem avançar para dentro da função:

```gdb
break *0xfffff6f108b0
continue
info registers x0 x1 x2 x3 x4
x/wd $x25+48
x/wd $x21+9648
```

O campo `[x21+9648]` é atualizado logo após o retorno da chamada, somando `w24`; portanto, inspecioná-lo antes da chamada fornece o contador acumulado anterior a esta NAL. Ainda não se atribui semântica definitiva a esse contador ou à flag `[x25+48]`.


## 14.67 — Argumentos reais na chamada de H264ProcessNaluUnit (+464)

Breakpoint 15 em `0xfffff6f108b0`, imediatamente antes da chamada `H264ProcessNaluUnit(x19, x22, w28, w24 << 3, flag)`:

Valores observados:
- `x0 = 0xfffff000cc80` — contexto do decoder (`x19`).
- `x1 = 0xfffff002d1a0` — contexto de trabalho (`x22`).
- `w2 = 1` — valor derivado de `w23 & 0xff`, associado ao campo `aux+0` observado como 1.
- `w3 = 0x66db8 = 421.304` — comprimento em bits; corresponde exatamente a `0xcdb7 * 8`, isto é, 52.663 bytes.
- `w4 = 0` — byte `[x25+48]`.
- `[x25+48] = 0` — estado/flag do objeto externo; sem semântica confirmada.
- `[x21+9648] = 0` antes da chamada — contador acumulado, cujo incremento ocorre após o retorno.

Conclusões:
1. O comprimento passado a `H264ProcessNaluUnit` é o comprimento da NAL sem o start code Annex B, convertido de bytes para bits.
2. O valor de `w2` permanece 1 para esta NAL, coerente com `aux+0 = 1`; o significado exato desse metadado continua não identificado.
3. A flag passada em `w4` é zero nesta execução. Não atribuir significado funcional sem rastrear os outros valores possíveis.
4. O contador em `[x21+9648]` ainda está zerado antes da chamada. No código, ele é incrementado por `w24` após o retorno de `H264ProcessNaluUnit`.

Próximo passo sugerido: entrar na implementação efetiva de `H264ProcessNaluUnit` para identificar se ela faz parsing de slice, configura estado do hardware VE ou encaminha dados ao pipeline de decodificação. Primeiro, verificar a resolução do símbolo e o stub PLT; não presumir que o endereço PLT seja a implementação.



## 14.65 — H264ProcessNaluUnit: classificação de NAL e preparação do caminho VE

### Referência cruzada com a documentação do decoder

Esta análise deve ser lida em conjunto com as seções anteriores sobre `VideoDecoder`, `VConfig`, `VideoStreamInfo`, `VeOpsS`, `VeConfig` e o teste funcional do `vdecoderdemo`. A documentação da API explica a arquitetura pública e a inicialização; a desmontagem abaixo detalha o fluxo interno do plugin H.264, que não é descrito integralmente pelos headers públicos.

A distinção é importante: a documentação não é irrelevante. Ela fornece os tipos, contratos da API, estruturas e a arquitetura VE. A engenharia reversa está preenchendo os detalhes internos que os headers não expõem.

### Entrada observada

Implementação real identificada em `0xfffff6f154c0` (`H264ProcessNaluUnit`), não o stub PLT em `0xfffff6f03870`.

Argumentos observados no breakpoint de entrada:

```text
x0 = 0xfffff000cc80   contexto do decoder
x1 = 0xfffff002d1a0   contexto de trabalho
w2 = 1                valor de controle/tipo recebido
w3 = 0x66db8           comprimento em bits (421304)
x4 = 0                 valor observado no registrador; ainda não validado como argumento semântico
```

O prólogo preserva `x0` em `x20`, `x1` em `x19`, `w3` em `w21`, carrega `[x0+400]` em `x24` e preserva `w4 & 0xff` em `w25`. Também grava `w2` em `[x1+28]`.

### Estados e classificação

O trecho `+56..+128`:

- lê byte `[x24+48]`; se não zero, desvia para `+380`;
- calcula uma base a partir de `x1+0xC000` e consulta DWORDs nos offsets `+9300` e `+9304`; se qualquer um for 1, desvia para caminhos especiais `+884` e `+932`;
- calcula `x23 = x19 + 0xE000` e zera o byte `[x23+1224]`;
- classifica `w2` por comparações com 7, 5 e 6.

Para o valor observado `w2=1`, não são tomados os desvios para NAL type 7, 5 ou 6. Portanto, a execução segue para o caminho genérico que começa em `+688` (fora do intervalo desmontado nesta amostra).

Os ramos identificáveis no trecho:

- `w2 == 7`: ramo em `+948`;
- `w2 == 5`: ramo em `+1316`;
- `w2 == 6`: chama `H264DecodeSei(x20, x19, w21)` em `+136`.

A classificação é compatível com tipos NAL H.264 (SPS=7, IDR=5, SEI=6), mas a interpretação do argumento `w2` deve continuar vinculada ao fluxo de dados observado, pois o chamador o deriva de um campo de registro interno, não extrai diretamente o cabeçalho NAL nesta função.

### Caminho SEI e controle de estado

No ramo `w2==6`, a função:

1. chama `H264DecodeSei`;
2. guarda o retorno em `w22`;
3. consulta `[x19+0x25a4]` e, se não zero, zera um DWORD em `[x19+0xC000+9296]`;
4. consulta o byte `[x19+28]` e valida-o contra `0x14` por meio de uma máscara de bits;
5. carrega pares de ponteiros em `[x20+256]`, consulta `[x26+9992]` e pode chamar indiretamente a callback em `[x2+40]`.

O significado dos campos de estado e da callback ainda não foi identificado. Não atribuir nomes semânticos aos offsets apenas com base na forma do código.

### Consultas ao mecanismo VE no ramo SEI

Importante: estas instruções estão dentro do ramo `w2 == 6` (SEI), depois de `H264DecodeSei()` e das verificações de estado. Elas **não** são executadas pelo caso atual `w2 == 1`. Nesse ramo SEI, em `+240..+260`, aparecem chamadas nesta ordem:

```asm
H264CheckBsDmaBusy(x20)
H264VeIsr(x20)
H264GetDecodeMbNum(x20)
```

Depois, a função lê campos do contexto de trabalho em `[x19+200]`, `[x19+192]`, `[x19+30]`, consulta um global e executa cálculos/armazenamentos, incluindo `[x19+152]`.

Essas chamadas demonstram que o ramo SEI consulta rotinas internas associadas ao estado do bitstream/DMA e ao mecanismo VE. Ainda não demonstram, isoladamente, que o hardware executou a decodificação de um slice: para isso, é necessário analisar os corpos de `H264CheckBsDmaBusy`, `H264VeIsr`, `H264GetDecodeMbNum` e o caminho genérico a partir de `+688`, identificando escrita de registradores, espera por interrupção e submissão do trabalho.

A relação com a documentação pública é a seguinte:

- `VConfig.veOpsS` e `VConfig.pVeOpsSelf` são os pontos de integração VE já identificados na API;
- `VeOpsS` expõe operações de inicialização, lock/unlock, reset e espera por interrupção;
- a documentação identifica `VE_MODE_VCUDEC` e o intervalo de registradores VE_DEC `0x01C0E000–0x01C0FFFF` no material técnico A733 fornecido anteriormente;
- a desmontagem atual revela chamadas internas do plugin H.264 que devem ser rastreadas até essa interface/registradores antes de concluir exatamente como o trabalho é submetido.

### Conclusões e limites

Confirmado:
- o endereço de implementação real é `0xfffff6f154c0`;
- o argumento `w3` é preservado como comprimento em bits;
- o campo de controle `w2` é usado para selecionar ramos específicos, incluindo SEI (6), IDR (5) e SPS (7), compatíveis com tipos NAL H.264;
- o valor observado `w2=1` segue o caminho genérico, não o ramo SEI;
- o ramo SEI (`w2==6`) alcança funções internas `H264CheckBsDmaBusy`, `H264VeIsr` e `H264GetDecodeMbNum`.

Ainda não confirmado:
- se `w2` é sempre o tipo NAL puro ou um código de controle transformado;
- semântica dos offsets `+9300`, `+9304`, `+9992`, `+9296`, `+152`, `+192`, `+200`;
- semântica da callback `[x2+40]`;
- onde ocorre a escrita efetiva dos parâmetros do slice nos registradores VE e o disparo da decodificação;
- se `H264VeIsr` é efetivamente uma rotina de serviço de interrupção ou uma consulta/rotina de tratamento chamada em polling (o nome sugere ISR, mas o corpo precisa confirmar).

### Próxima investigação recomendada

Para o caso atual (`w2=1`, compatível com NAL type 1), desmontar primeiro o ramo genérico `H264ProcessNaluUnit+688` em diante. Depois, analisar separadamente `H264CheckBsDmaBusy`, `H264VeIsr` e `H264GetDecodeMbNum`, lembrando que o trecho atual mostra essas chamadas no ramo SEI.

## 14.68 — Recuperação e cruzamento dos documentos técnicos originais Allwinner

Os documentos originais enviados anteriormente pelo usuário foram localizados na Library e materializados para consulta, sem necessidade de novo upload:

- `Allwinner_Developer_Guide_CedarX_zh.txt` — guia de desenvolvimento CedarX.
- `Allwinner_Developer_Guide_Decoder_zh.txt` — guia de desenvolvimento do decoder.
- `A733_User Manual_V1.00.txt` — manual específico do SoC A733, Revision 1.00.
- Também está disponível o PDF `OrangePi_4_Pro_A733_User Manual_v1.4.pdf`, relacionado à placa.

### Arquitetura do decoder documentada pelo fabricante

A seção 14.2 do manual A733 (páginas impressas 639–640) declara explicitamente que a decodificação de vídeo é composta por:

1. **Video Control Firmware (VCF), executado no ARM:** recebe o bitstream da camada superior, faz o parsing do bitstream e invoca o Video Engine.
2. **Video Engine (VE), hardware embarcado:** executa as operações de decodificação e gera a sequência de imagens decodificadas.
3. **Video Output Controller:** transmite a sequência de imagens ao dispositivo de display sob controle da camada superior.

Essa descrição é fundamental para interpretar a desmontagem: encontrar parsing H.264 e leitura de campos de NAL/slice em funções ARM como `H264ProcessNaluUnit()` não demonstra, por si só, decodificação integral por software. O fabricante descreve justamente uma arquitetura híbrida: parsing/controle no ARM e operações de decodificação no VE.

### SBM conforme o guia do decoder

A seção 3.6.4 do `Allwinner_Developer_Guide_Decoder_zh.txt` descreve o SBM como gestor do buffer de bitstream e dos metadados das unidades de dados. O texto cita explicitamente NALU H.264, FIFO e a necessidade de buffer fisicamente contíguo para o decoder de hardware.

Isso se alinha às observações já confirmadas em runtime:

- `SbmFrameAddStream()` armazena descritores de 64 bytes e atualiza o ring buffer, sem copiar o payload;
- `SbmFrameRequestFramePic()` fornece slots de metadados;
- os ponteiros dos dados observados pertencem a um mapeamento DMA-BUF;
- a estrutura auxiliar remove o start code Annex B de quatro bytes antes de entregar o payload ao processamento H.264.

### Orientações de diagnóstico no guia CedarX

O capítulo 4 do guia `Allwinner_Developer_Guide_CedarX_zh.txt` fornece verificações diretamente úteis:

- seção 4.2: usar `mtop` para observar o uso de banda do VE; o guia indica atividade de banda como evidência de uso de hardware;
- seção 4.4: `vdecoder_show_speed_info = 1` para acompanhar desempenho de decodificação;
- `ve_printf_register = 1` para habilitar impressão de registradores VE;
- `vdecoder_save_bitstream` e `vdecoder_save_special_bitstream` para salvar bitstreams;
- `vdecoder_save_picture_en` para salvar imagens decodificadas;
- logs de SBM/FBM e nível `cdc_log_level` para diagnóstico.

Essas opções devem ser tratadas como instrumentação documentada pelo fabricante. A disponibilidade efetiva e os caminhos dos arquivos de configuração precisam ser verificados na versão de bibliotecas instalada no A733.

### Consequência para a análise atual de `H264ProcessNaluUnit()`

O trecho de `H264ProcessNaluUnit()` em `+0..+320` identifica classificações de NAL e, no ramo de SEI (`nal_unit_type 6`), chamadas como `H264CheckBsDmaBusy()`, `H264VeIsr()` e `H264GetDecodeMbNum()`. Para a NAL atual, cujo primeiro byte é `0x41` (tipo 1), o fluxo segue para `+688` (`0xfffff6f15770`), ainda não desmontado.

À luz da arquitetura VCF/VE documentada, o próximo objetivo é identificar no ramo de tipo 1 onde o VCF prepara estado/parâmetros e aciona o VE. Não classificar `H264ProcessNaluUnit()` como software decoder apenas por conter parsing no ARM; tampouco afirmar que a NAL já foi submetida ao hardware antes de localizar a operação correspondente.

### Fontes primárias recuperadas

- `A733_User Manual_V1.00.txt`, seção 14.2, páginas impressas 639–640.
- `Allwinner_Developer_Guide_Decoder_zh.txt`, seção 3.6.4 (SBM) e tabela de configuração de `VConfig` imediatamente anterior.
- `Allwinner_Developer_Guide_CedarX_zh.txt`, capítulo 4, seções 4.2 e 4.4.

## 14.69 — Correção de método: documentação primária como referência obrigatória

**Motivação:** a análise acumulou várias horas de disassembly com os documentos do fabricante já disponíveis. A partir desta etapa, a desmontagem não deve ser conduzida como leitura isolada de assembly. Cada hipótese deve ser confrontada com os contratos e a arquitetura documentados pela Allwinner.

### Hierarquia das evidências

1. **Manual A733 V1.00:** autoridade para arquitetura do SoC, blocos VE/VCF, registradores, interrupções e capacidades específicas do A733.
2. **Allwinner Developer Guide — Decoder 1.5:** autoridade para API do decoder, ciclo de vida, SBM/FBM, estruturas e contratos de buffers.
3. **Allwinner Developer Guide — CedarX 1.3:** autoridade para integração CedarX, configuração, logs e procedimentos de diagnóstico.
4. **Runtime e disassembly do binário instalado:** evidência do comportamento efetivo desta compilação. Pode complementar ou divergir dos documentos, mas a divergência deve ser registrada explicitamente.

### Regra de interpretação

A arquitetura documentada pelo A733 separa parsing/controle no ARM (VCF) das operações de decodificação executadas pelo VE. Portanto:

- parsing de NAL, SPS/PPS, SEI, slice header, Exp-Golomb e controle de fluxo em funções ARM é compatível com aceleração de hardware;
- movimentação de bitstream para buffers DMA, configuração de parâmetros, escrita de registradores, submissão de trabalho e espera por conclusão são os pontos a rastrear para demonstrar a participação do VE;
- callbacks com nomes sugestivos, isoladamente, não comprovam submissão ao hardware;
- a confirmação deve combinar fluxo de chamadas, acesso à interface VE/registradores ou ioctl correspondente e uma medição independente de atividade do VE (por exemplo, `mtop`, conforme o guia CedarX), quando disponível.

### Protocolo para cada próxima etapa de engenharia reversa

Antes de propor um breakpoint ou interpretar uma função:

1. localizar no guia original a API, estrutura, opção de configuração ou subsistema relacionado;
2. identificar os campos/offsets documentados e confrontá-los com os offsets observados em runtime;
3. classificar cada conclusão como **documentada**, **observada em runtime**, **inferida** ou **ainda desconhecida**;
4. escolher o menor trecho de assembly capaz de testar a hipótese;
5. atualizar este relatório com comando, resultado bruto relevante, interpretação e grau de confiança.

### Reavaliação do ponto atual

O caminho em foco continua sendo `H264ProcessNaluUnit()` para `w2=1` (byte NAL `0x41`, tipo 1). O ramo genérico começa em `+688`, endereço `0xfffff6f15770`. As chamadas `H264CheckBsDmaBusy()`, `H264VeIsr()` e `H264GetDecodeMbNum()` vistas no trecho `+0..+320` pertencem ao ramo SEI (`w2==6`) e não devem ser usadas como prova de atividade do VE no caminho da slice tipo 1.

A próxima inspeção deve começar pelo ramo `+688`, mas sua leitura será orientada pelo fluxo VCF→VE e pelas interfaces descritas nos guias originais. A investigação não deve voltar a atribuir semântica a offsets ou callbacks sem cruzamento documental.

### Compromisso de rastreabilidade

Nenhuma etapa já confirmada será descartada. As observações anteriores de SBM, DMA-BUF, `VideoPicture`, `VConfig`, `VeOpsS`, seleção HW/SW e argumentos de `H264ProcessNaluUnit()` permanecem no histórico. Quando uma interpretação antiga for corrigida, a documentação deverá preservar a observação original e registrar claramente a correção, sem reescrever hipótese como fato.

## 14.70 — Início da análise de H264DecodeSliceHeader (22/11/2026)

### Documentação primária consultada antes do disassembly

Foi consultado o `Allwinner_Developer_Guide_Decoder_zh.txt`, Developer Guide Decoder v1.5:

- Seção 3.5, páginas impressas 10–12: fluxo público do decoder — inicialização (`InitializeVideoDecoder`), submissão (`SubmitVideoStreamData`), execução (`DecodeVideoStream`), obtenção/retorno de imagem (`RequestPicture`/`ReturnPicture`).
- Seção 3.6.1–3.6.3, páginas impressas 13–14: separação dos módulos VideoEngine, SBM e FBM e estruturas `VideoStreamInfo`, `VConfig`, `VideoStreamDataInfo`, `VideoPicture`.
- A documentação descreve o contrato público e a arquitetura modular, mas não especifica o algoritmo interno do parser H.264 nem o layout privado do contexto de `H264DecodeSliceHeader`. Portanto, não se deve atribuir semântica a offsets privados apenas com base nesse guia.
- O CedarX Developer Guide v1.3, seção 4.4, documenta `vdecoder_show_speed_info` e `ve_printf_register` como mecanismos de diagnóstico do decoder/VE. Isso oferece uma verificação independente posterior; não substitui a análise do caminho de execução.

### Evidência observada no início de `H264DecodeSliceHeader`

Símbolo: `H264DecodeSliceHeader`, endereço `0xfffff6f0d4b0`, biblioteca compilada sem símbolos de depuração.

Prólogo e primeiros blocos observados:

- Reserva 0x90 bytes de stack e salva registradores.
- Preserva `x1` em `x19` e `x0` em `x20`.
- Carrega um ponteiro de função de `[x0+456]` para `x1` e chama-o via `blr x1` em `+72`.
- Guarda o retorno em `w21`.
- Lê um inteiro de `[x19+152]` em `w1`.
- Compara `w1` com `w0`; se `w1 <= w0`, desvia para `+1332`.
- Compara `w1` com `w0 << 1`; se iguais, também desvia para `+1332`.
- Caso contrário, chama `H264CheckNewFrame(x19)` em `+104`.

**Interpretação limitada:** há uma chamada indireta de callback antes de duas verificações entre o valor retornado e o campo privado em `x19+152`; em seguida, o fluxo pode verificar início de novo frame. Ainda não está identificado o significado desses campos nem do callback. Não afirmar que são contagem de bits, posição do bitstream ou operação de hardware sem rastrear a origem e o contrato.

### Próximo passo de depuração — dump curto e contextual

Não usar `disassemble H264DecodeSliceHeader` completo. O próximo trecho útil é somente o bloco após `H264CheckNewFrame`:

```gdb
disassemble /r 0xfffff6f0d514, 0xfffff6f0d5b0
```

Se o bloco terminar antes, não ampliar automaticamente: primeiro analisar as instruções e identificar chamadas/branches. O objetivo imediato é descobrir o que `H264CheckNewFrame` decide e quais campos são lidos/escritos logo depois, não desmontar os 1.3 KB da função de uma vez.

### Classificação

- **Documentado:** fluxo público VideoEngine/SBM/FBM e APIs de entrada/saída descritos no Decoder Guide v1.5.
- **Observado:** endereços, registradores, offsets e chamadas listados acima, extraídos do GDB.
- **Inferido (baixa confiança):** o caminho testa alguma condição de novo frame antes de prosseguir com o cabeçalho do slice.
- **Desconhecido:** semântica de `[x0+456]`, `[x19+152]`, retorno do callback e significado exato do desvio `+1332`.

## 14.71 — Continuação controlada de H264DecodeSliceHeader (22/11/2026)

### Evidência adicional enviada pelo GDB

Trecho: `H264DecodeSliceHeader+108` até `+576`, endereço-base `0xfffff6f0d4b0`.

- `+108`: preserva em `w22` o retorno de `H264CheckNewFrame(x19)`. Se não zero, desvia para `+1300`.
- `+116`: zera `[x19+68]`.
- `+120`: se `w21` (retorno do callback carregado de `[x20+456]` em `+72`) não for zero, desvia para `+1336`.
- `+124..+140`: quando `w21==0`, grava zero em `[x19+148]` e chama novamente o callback `[x20+456]`.
- `+140..+176`: o retorno é validado contra `9`; valores acima de `9` desviam para `+1816`. O valor é transformado por uma seleção condicional e usado para indexar uma tabela local na pilha (`[sp+0x88 + index]`), cujo byte fica em `w23`.
- `+164`: consulta DWORD em `[x19+0xC000+9296]`; se igual a `1`, desvia para `+1372`.
- `+184..+232`: salva `x27/x28`; compara `w23` com `2`; consulta os campos privados `[x19+68]` e `[x19+72]`; para `w23==1`, verifica também o ponteiro em `[x19+0x8000+25472]`. Depois grava `w23` em `[x19+32]`.
- `+244..+268`: chama novamente o callback `[x20+456]`; valida o retorno (`<=255`) e usa-o para indexar uma tabela de ponteiros em `x19` (base calculada com escala 8 e deslocamento 600). O ponteiro selecionado é carregado em `x1`.
- `+272..+320`: lê campos da estrutura selecionada, atualiza dois ponteiros em `[x19+256]` e, sob uma condição, grava `1` em `[x19+0xC000+9308]`.
- `+324..+344`: chama um segundo callback, carregado de `[x20+448]`, com `x1=[x0+64]`; salva o retorno em `[x19+76]`.
- `+348..+388`: lê campos em `[x19+256]` e `[x19+30]`, consulta byte `[x1+3]` e chama novamente o callback `[x20+448]`, desta vez com argumento `1`.
- `+396..+416`: grava `3` em `[x19+30]` e copia o primeiro byte da estrutura apontada por `[x19+256]` para `[x19+104]`.
- `+440..+508`: sob as condições mostradas, quando `[x19+68]==0`, verifica `[x19+30]` e `[x19+29]`; se não tomar os desvios alternativos, grava `1` em `[x19+29]` e chama `H264FrameStart(x20,x19)` em `+492`. O retorno é salvo em `w22`; retorno igual a `4` desvia para `+3156`.
- `+512..+576`: carrega um ponteiro de contexto de `[x19+0x8000+25488]`. Se não nulo, grava nele valores derivados de `[x19+24]`, `[x19+76]`, `[x19+32]` e `[x19+30]`; também lê campos da estrutura apontada por `[x19+256]`. O trecho termina em `udiv w3,w21,w0` (`+576`), portanto a análise desse cálculo fica em aberto até recebermos as instruções seguintes.

### Cruzamento com documentação primária

Reconsultado `Allwinner_Developer_Guide_Decoder_zh.txt`, versão 1.5:

- Seção 3.5 descreve o fluxo público de inicialização, entrada do bitstream, processamento pelo decoder e retirada das imagens.
- Seções 3.6.1–3.6.3 descrevem os módulos VideoEngine/SBM/FBM e estruturas públicas `VideoStreamInfo`, `VConfig`, `VideoStreamDataInfo` e `VideoPicture`.
- O guia não documenta `H264DecodeSliceHeader`, `H264CheckNewFrame`, `H264FrameStart` nem os offsets privados observados neste trecho. Assim, não há base documental para dar nomes semânticos definitivos aos campos privados.

### Interpretação e limites

1. **Observado:** há dois callbacks distintos no contexto do decoder: `[x20+456]` (chamado em `+72`, `+136` e `+244`) e `[x20+448]` (chamado em `+332` e `+380`). Não presumir que sejam a mesma operação ou que submetam trabalho ao VE.
2. **Observado:** o caminho contém seleção de estado/tabela, atualização de estruturas e uma chamada condicional a `H264FrameStart(x20,x19)`.
3. **Inferido, confiança moderada:** `H264FrameStart` marca uma transição de estado de início de frame no fluxo interno do decoder, dado o nome da função e a posição no caminho. O nome não prova, por si só, que a função programe registradores ou inicie o VE.
4. **Desconhecido:** contrato dos callbacks, semântica dos campos privados, conteúdo da tabela local, significado do retorno `4` e relação do ponteiro `[x19+0x8000+25488]` com estruturas públicas.
5. **Não demonstrado neste trecho:** escrita de registradores VE, ioctl de submissão, chamada de `VeOpsS` para iniciar o hardware ou espera por interrupção.

### Próximo passo mínimo

Não continuar desmontando `H264DecodeSliceHeader` inteiro. O próximo trecho deve ser o corpo de `H264FrameStart`, em janela curta:

```gdb
disassemble /r H264FrameStart
```

Se for extenso, interromper após o prólogo e as primeiras chamadas/branches e enviar apenas esse bloco. A pergunta objetiva será: `H264FrameStart` apenas prepara estado/metadados do frame ou alcança a interface VE (VeOps, registradores ou ioctl)?

### Classificação

- **Documentado:** arquitetura pública VideoEngine/SBM/FBM e fluxo API no Decoder Guide v1.5.
- **Observado em runtime:** instruções, offsets, chamadas indiretas e desvios listados nesta seção.
- **Inferido:** `H264FrameStart` representa uma transição interna de início de frame (confiança moderada).
- **Desconhecido:** semântica dos callbacks e campos privados; eventual submissão ao hardware ainda não provada.

## 14.72 — Análise de H264FrameStart e confirmação do papel do FBM (22/11/2026)

### Documentação primária consultada antes da interpretação

Reconsultado `Allwinner_Developer_Guide_Decoder_zh.txt`, Developer Guide Decoder v1.5:

- Seção 3.5: o guia afirma que VideoEngine executa a função de decodificação, SBM gerencia o buffer do bitstream, memory gerencia memória e FBM gerencia buffers de frame. A interface FBM fica basicamente encapsulada dentro do subdecoder; o cliente não precisa chamar FBM diretamente na camada externa.
- Seções 3.6.1–3.6.2: classifica FBM como gerenciador de buffers de frame e associa a ele `VideoPicture` e `FbmBufInfo`; descreve esses buffers como parte do gerenciamento de frames decodificados.
- Seção 3.6.3.2: `VConfig.nFrameBufferNum` define a quantidade de frame buffers; `nAlignStride` define alinhamento; `bSupportPallocBufBeforeDecode` controla suporte à alocação antes da decodificação; os campos `nDisplayHoldingFrameBufferNum`, `nRotateHoldingFrameBufferNum` e `nDecodeSmoothFrameBufferNum` descrevem buffers retidos por etapas distintas.
- O guia documenta o papel arquitetural do FBM, mas não documenta o corpo privado de `H264FrameStart` nem os offsets internos do decoder.

### Evidência observada no GDB

Função: `H264FrameStart`, endereço `0xfffff6f0cab0`, dump completo fornecido.

**Prólogo e contexto**

- Reserva `0x61e0` (25.056) bytes de stack.
- Preserva `x1` em `x19`, `x0` em `x22` e calcula `x20=x1+0xC000`.
- Consulta `[x20+9332]`; quando o valor não é `-1`, chama `H264GetValidBufferIndex(x19, stack_arg)`.
- Há caminhos condicionados pelos campos `[x22+104]`, `[x22+108]` e bytes em `[x19+0xE000+1296]` / `[x19+0xE000+1297]`. Seus significados semânticos ainda não foram identificados.

**Solicitação e registro de buffers FBM**

- Em `+136..+144`, carrega um objeto de `[x22+25784]` e chama `FbmRequestBuffer()`.
- O ponteiro retornado é armazenado em uma entrada calculada da estrutura privada do contexto, no deslocamento `+3952` da entrada selecionada.
- Se o ponteiro retornado for nulo, o fluxo retorna código `4`.
- Em outro caminho (`+760..+820`), carrega um objeto de `[x22+25776]`, chama novamente `FbmRequestBuffer()` e armazena o retorno no deslocamento `+3944` de uma entrada calculada.
- Se essa segunda solicitação não produzir buffer válido, há caminho que retorna o buffer anterior via `FbmReturnBuffer()` (`+1300..+1312`) e retorna código `4`.
- O corpo também atualiza metadados privados por entrada, incluindo campos nos deslocamentos `3868`, `3872`, `3876`, `3880`, `3944`, `3952`, `3968`, `3972`, `3973` e `3974` relativos à entrada calculada. Não atribuir nomes a esses campos sem rastrear os consumidores.
- Em um caminho alternativo, chama `memset()` e `memcpy()` para inicializar/copiar uma região associada ao buffer. O tamanho copiado observado é `0x6118`; o conteúdo/estrutura dessa região ainda não foi identificado.
- As chamadas externas visíveis no dump são `H264GetValidBufferIndex`, `FbmRequestBuffer`, `FbmReturnBuffer`, `memset`, `memcpy` e `printf`. Não aparece chamada direta a `VeOpsS`, `H264VeIsr`, `H264CheckBsDmaBusy`, escrita identificável em registradores VE ou ioctl de submissão neste corpo.

### Conclusão técnica

1. **Documentado:** o FBM é responsável pelo gerenciamento dos buffers de imagem/frame decodificados; o VideoEngine é a camada de decodificação.
2. **Observado:** `H264FrameStart` solicita buffers por `FbmRequestBuffer()`, registra os ponteiros em slots privados, inicializa/atualiza metadados e pode devolver um buffer com `FbmReturnBuffer()` em um caminho de falha.
3. **Conclusão de alta confiança:** `H264FrameStart` executa preparação do estado do frame e aquisição/gestão de buffers FBM. Isso é coerente com a responsabilidade do FBM documentada pela Allwinner.
4. **Limite importante:** o dump não demonstra que `H264FrameStart` inicia a decodificação no hardware. Não há, no corpo mostrado, chamada identificável à interface VE nem escrita identificável em registradores VE. Isso não prova que o hardware não será usado mais adiante; apenas delimita o que esta função faz/mostra.
5. **Relação com a função chamadora:** em `H264DecodeSliceHeader+492`, a chamada a `H264FrameStart(x20,x19)` ocorre em um caminho condicional. O retorno `4` é tratado separadamente; o caminho normal observado retorna `0`.

### Próximo passo focado

Voltar ao ponto em que `H264DecodeSliceHeader` continua após a preparação do frame. O dump anterior terminou em `+576` (`udiv w3,w21,w0`). Solicitar somente o bloco seguinte:

```gdb
disassemble /r 0xfffff6f0d6f0, 0xfffff6f0d790
```

Objetivo: verificar se o código que consome os metadados e o buffer FBM chega à configuração/submissão do VideoEngine/VE. Não ampliar o intervalo antes de analisar esse bloco.

### Classificação

- **Documentado:** responsabilidades VideoEngine/SBM/FBM e parâmetros públicos de quantidade/alinhamento/retensão de frame buffers (Decoder Guide v1.5, seções 3.5 e 3.6.1–3.6.3.2).
- **Observado:** chamadas `FbmRequestBuffer`/`FbmReturnBuffer`, atualização de slots e metadados privados, e ausência de chamadas VE explícitas no corpo fornecido.
- **Inferido com alta confiança:** `H264FrameStart` prepara o estado do frame e adquire buffers do FBM.
- **Desconhecido:** semântica dos campos privados, finalidade exata dos dois objetos FBM (`[x22+25784]` e `[x22+25776]`), conteúdo da região copiada e ponto exato em que o VE recebe o trabalho.

## 14.73 — Continuação de H264DecodeSliceHeader: atualização do estado do slice (22/11/2026)

### Fonte documental consultada

Foi revisado o *Allwinner Decoder Developer Guide v1.5*, especialmente a separação entre VideoEngine, SBM e FBM (seção 3.5 e capítulo 3.6). O guia descreve o VideoEngine como módulo de execução do decoder, SBM como gestor do bitstream e FBM como gestor dos buffers de imagem; não documenta os offsets privados usados pelo parser H.264 desta versão da biblioteca. O *A733 User Manual*, seção 14.2, descreve a arquitetura híbrida: o firmware de controle no ARM recebe e analisa o bitstream e invoca o Video Engine, enquanto o VE executa o decode. Portanto, análise de cabeçalho H.264 no ARM não é, por si só, evidência de decodificação por software.

### Observação direta no GDB

Função: `H264DecodeSliceHeader`, base `0xfffff6f0d4b0`. Trecho observado: `+576` a `+732` (`0xfffff6f0d6f0`–`0xfffff6f0d78c`).

- `+576..+588`: executa `udiv w3,w21,w0`, calcula o resto com `msub`, desloca o quociente por `w4` e grava dois valores em `[x19+92]` e `[x19+96]` (`stp w0,w3,[x19,#92]`). A finalidade semântica exata desses campos não pode ser afirmada sem identificar a estrutura e os valores de runtime.
- `+592..+620`: lê `[x1+64]`; conforme os flags de condição preservados de uma instrução anterior, pode incrementar esse valor e calcular `2*w2+1` e `1<<w3`, gravando os resultados em `[x19+84]` e `[x19+88]`. O `b.eq` em `+596` usa flags produzidos antes deste bloco, pois `udiv`, `msub` e `lsl` não atualizam NZCV. Sem o bloco anterior completo e os registradores no ponto, não se deve inferir a condição lógica desse desvio.
- `+624..+652`: calcula `x21=x19+0xe000`, lê o byte em `[x21+1224]`; se for 1, chama o callback armazenado em `[x20+456]`. Depois carrega `[x19+256]` em `x1`.
- `+656..+684`: lê o byte `[x1+1]`. Se zero, lê `[x1+72]`, chama o callback `[x20+448]` e grava o retorno em `[x19+180]`.
- `+688..+700`: lê o byte `[x19+264]+24`; se for 1, desvia para `H264DecodeSliceHeader+1624`. Caso contrário, volta a usar a estrutura apontada por `[x19+256]`.
- `+704..+716`: lê o DWORD no início dessa estrutura, aplica `and w0,w0,#0xffff00` e compara com `0x100`; igualdade desvia para `+2128`.
- `+720..+732`: lê `[x19+68]`; se zero, desvia para `+2208`; caso contrário, carrega `[x19+264]` e lê seu byte de offset 25.

### Interpretação e limites

Este bloco continua a preparação/validação do estado do slice e consulta callbacks por meio da tabela de funções do contexto. Há cálculos e gravações no contexto privado do parser, além de desvios para caminhos alternativos dentro da própria `H264DecodeSliceHeader`.

Neste intervalo não aparece chamada explícita a `VeOpsS`, `H264VeIsr`, `H264CheckBsDmaBusy`, `ioctl`, nem escrita identificável em registrador VE. Assim, este trecho **não demonstra a submissão do slice ao hardware**. Também **não demonstra decode por software**: é compatível com a etapa ARM-side de parsing/controle descrita no manual A733, antes de uma operação VE que pode ocorrer em outro ponto da função ou em outra função.

Os offsets `[x19+84..96]`, `[x19+180]`, `[x19+256]`, `[x19+264]` e os bytes nos offsets relativos a `+0xe000` continuam sem nomes semânticos confirmados. Não atribuir nomes de campos H.264 apenas pela forma dos cálculos.

### Próximo passo

Inspecionar o bloco imediatamente seguinte, incluindo os desvios locais relevantes, para continuar seguindo o fluxo de controle sem saltar prematuramente para uma hipótese de submissão ao VE:

```gdb
disassemble /r 0xfffff6f0d790, 0xfffff6f0d930
```

## 14.74 — H264DecodeSliceHeader: reordenação de referências e configuração da tabela de pesos (+736 a +1148) (22/11/2026)

### Fonte documental consultada

Foi reconsultado o *Allwinner Decoder Developer Guide v1.5*, seção 3.5 e capítulo 3.6. O guia separa VideoEngine (execução da decodificação), SBM (gestão do bitstream) e FBM (gestão dos buffers de frame), mas não especifica as rotinas privadas `H264DecodeRefPicListReordering` e `H264CongigureWeightTableRegisters`, nem os offsets internos do contexto H.264. A arquitetura do A733 descrita no User Manual, seção 14.2, distingue o parsing/controle ARM-side da execução pelo Video Engine. Assim, encontrar parsing e preparação de parâmetros no ARM não permite concluir que a decodificação seja por software.

### Observação direta no GDB

Função `H264DecodeSliceHeader`, base `0xfffff6f0d4b0`; bloco fornecido: `+736` a `+1148` (`0xfffff6f0d790`–`0xfffff6f0d92c`).

**Callbacks e inicialização de campos (`+736..+868`)**

- `+736..+756`: se o DWORD previamente carregado em `w0` for zero, chama o callback em `[x20+456]` e grava o byte de retorno em `[x19+31]`. Em seguida, carrega o ponteiro `[x19+264]`.
- `+760..+776`: lê dois DWORDs da estrutura apontada por `[x19+264]`, offsets `+28` e `+32`, zera `[x19+44]`, e grava os dois valores em `[x19+56]` e `[x19+60]`.
- `+776..+864`: lê o byte `[x19+32]`, executa comparações com 1 e 3, e conforme o caso chama o callback `[x20+448]` sem argumento adicional ou com `w1=1`. Os retornos são gravados em `[x19+100]` e `[x19+132]`; o valor 1 em `[x19+132]` desvia para o caminho `+2664`. Em outro caminho, `[x19+44]` recebe 1 ou 2, conforme `[x19+32]`.
- `+868..+884`: se `w27` não for zero, chama `H264DecodeRefPicListReordering(x20,x19)`. Retorno com bit 31 definido (valor negativo) desvia para `+2060`, caminho de erro.

**Condições de frame e tabela de pesos (`+888..+972`)**

- `+888..+920`: consulta `[x19+68]`, `[x19+30]` e `[x19+29]` para escolher caminhos alternativos. No caminho comum, zera o byte `[x21+1452]` e grava `-1` em `[x28+25584]`.
- `+932..+956`: lê o DWORD `[x19+264]+36` e o byte `[x19+32]`. Só prossegue para a configuração da tabela quando as condições do código são satisfeitas (valor de tipo em `w1` igual a 0 ou 3 e campo DWORD não zero).
- `+960..+968`: chama `H264CongigureWeightTableRegisters(x20,x19)` por PLT (`0xfffff6f03860`). O nome da rotina sugere configuração de registradores associados à tabela de pesos H.264. **Ainda não está demonstrado que essa função escreva diretamente nos registradores físicos do VE**; é necessário seguir o destino real da PLT e examinar o corpo da função.

**Leitura de parâmetro adicional e callback (`+972..+1148`)**

- `+972..+976`: lê `[x19+24]`; se diferente de zero, desvia para `+1692`.
- `+980..+1044`: avalia o byte `[x19+32]` e o DWORD `[x19+264]+8`; em um dos caminhos chama o callback `[x20+456]`, preserva o retorno em `w21` e grava-o em `[x19+108]` quando não excede 2. Retorno maior que 2 desvia para `+2984`.
- `+1044..+1064`: carrega callback `[x20+480]`, passa `x0=x20`, chama-o e soma o retorno ao DWORD lido de `[x19+264]+12`. O resultado é mantido em `w21`.
- `+1068..+1092`: compara a soma com `0x33` (51). Se maior, consulta um global de log e pode desviar para `+2860`.
- `+1096..+1104`: grava `w21` em `[x19+116]` e zera o ponteiro/campo de 64 bits `[x19+136]`.
- `+1108..+1132`: se `[x19+32] == 3`, chama o callback `[x20+448]` com `w1=1`.
- `+1136..+1148`: subtrai 3 do byte `[x19+32]`, mascara para 8 bits e compara com 1; o resultado controla o próximo desvio, cujo destino está fora do bloco fornecido.

### Interpretação técnica

1. **Observado:** o parser executa `H264DecodeRefPicListReordering` e, em condições específicas, `H264CongigureWeightTableRegisters`.
2. **Interpretação consistente com H.264:** a primeira rotina trata a reordenação da lista de imagens de referência; a segunda está relacionada à configuração da tabela de pesos usada por slices que exigem weighted prediction. A semântica exata dos campos privados e dos argumentos deve ser confirmada no código das rotinas chamadas.
3. **Ponto de investigação prioritário:** `H264CongigureWeightTableRegisters` é a primeira função deste trecho cujo nome sugere uma possível ponte entre os parâmetros do slice e registradores do decoder. Entretanto, o nome não prova acesso a registradores VE. Precisamos resolver a PLT e inspecionar a implementação.
4. **Ainda não observado:** nenhuma chamada direta a `VeOpsS`, `H264VeIsr`, `H264CheckBsDmaBusy`, ioctl ou escrita identificável em registrador VE aparece diretamente neste bloco. A chamada à rotina `...WeightTableRegisters` é uma pista para investigação, não confirmação de submissão.
5. **Arquitetura:** o resultado é compatível com a etapa ARM-side de parsing e preparação dos parâmetros, descrita no manual A733, antes da execução pelo VE. Não é evidência de decode por software.

### Próximo passo focado

Resolver e examinar a implementação da rotina chamada em `+968`. No GDB:

```gdb
info functions H264CongigureWeightTableRegisters
```

Se o símbolo aparecer como função carregada, em seguida:

```gdb
disassemble /r H264CongigureWeightTableRegisters
```

Se GDB mostrar apenas o stub PLT ou não resolver o símbolo, enviar a saída de `info functions` para localizar o endereço real sem presumir que `0xfffff6f03860` seja o corpo da função.

### Classificação

- **Documentado:** separação entre VideoEngine, SBM e FBM no Decoder Guide v1.5; arquitetura ARM-side/VE no manual A733.
- **Observado:** chamadas e desvios listados no dump de `H264DecodeSliceHeader+736..+1148`.
- **Inferido:** `H264DecodeRefPicListReordering` implementa a etapa de reordenação de referências; `H264CongigureWeightTableRegisters` trata da tabela de pesos/registradores do decoder, inferência baseada no nome e no ponto de chamada.
- **Desconhecido:** destino real da PLT e se a rotina de tabela de pesos escreve em registradores físicos VE, em uma estrutura de shadow registers ou em outra camada de configuração.

## 14.75 — Resolução do símbolo H264CongigureWeightTableRegisters (22/11/2026)

### Evidência GDB

`info functions H264CongigureWeightTableRegisters` retornou dois endereços:

- `0xfffff6f03860`: `H264CongigureWeightTableRegisters@plt`, entrada PLT.
- `0xfffff6f07ba4`: implementação real, símbolo não-PLT.

A chamada observada em `H264DecodeSliceHeader+968` usa a entrada PLT. Para examinar o comportamento real da rotina, o próximo disassembly deve começar em `0xfffff6f07ba4`, não em `0xfffff6f03860`.

### Interpretação e limites

O nome do símbolo sugere configuração de registradores relacionados à tabela de pesos do H.264 (weighted prediction). Isso é apenas uma pista nominal: ainda não demonstra que a função escreva em registradores físicos do Video Engine. A confirmação depende do corpo da implementação e, em particular, de chamadas a operações VE, escritas MMIO ou configuração de uma estrutura de comandos.

### Próximo passo

Inspecionar a implementação real:

```gdb
disassemble /r 0xfffff6f07ba4
```

## 14.76 — Implementação parcial de H264CongigureWeightTableRegisters: buffer de comandos e imagem de registradores (22/11/2026)

### Fonte consultada antes da interpretação

O Decoder Developer Guide v1.5, seção 3.5, descreve a separação entre VideoEngine (execução da decodificação), SBM (gestão do bitstream), memória e FBM (buffers de frames). A seção 3.6.3 descreve `VideoEngine`/`VConfig` como configuração do decoder, mas não documenta os offsets privados observados abaixo nem a estrutura interna do buffer de comandos desta versão do plugin.

Portanto, a documentação pública confirma a arquitetura em camadas, mas não permite atribuir nomes oficiais aos campos privados `+24`, `+32`, `+4232`, `+26400` ou aos offsets relativos calculados no disassembly.

### Evidência GDB

Função real `H264CongigureWeightTableRegisters`, endereço `0xfffff6f07ba4`. O dump fornecido cobre `+0` a `+984` (endereço final `0xfffff6f07f7c`); a função continua até pelo menos `+1916`, portanto esta análise é parcial.

**Contextos e ponteiros (`+0..+52`)**

- `x25 = x0`: primeiro argumento, contexto do decoder.
- `x22 = x1`: segundo argumento, contexto do slice/parser.
- `x20 = [x25+400]`: ponteiro para contexto privado do decoder.
- `x19 = [x20+4232]`: ponteiro para uma estrutura privada usada como imagem/estado de registradores ou parâmetros.
- `[x22+264]` é carregado e o DWORD em `+40` é consultado para selecionar um caminho alternativo (`+1916`).

Os nomes semânticos desses campos não estão confirmados.

**Atualização de estrutura privada (`+108..+168`)**

- A função consulta duas vezes o callback `[x25+456]` e usa os retornos para compor campos de bits.
- O valor resultante é gravado em `[x19+16]` e copiado para `[x5+16]`, onde `x5 = [x20+24]`.
- O campo `[x19+180]` é zerado e seu valor é copiado para `[x5+224]`.

Isso demonstra a atualização de uma estrutura privada separada, mas não identifica por si só se `x5` é uma imagem completa de registradores, uma estrutura de parâmetros ou outro objeto.

**Evidência forte de append em buffer (`+180..+216`)**

- `x0 = [x20+32]`.
- `x6 = x0 + 0xC000 + 9992` (offset efetivo `0xE708` a partir de `x0`).
- Se o DWORD em `[x6]` for `1`, a função calcula `x1 = x0 + 0x8000`, carrega o ponteiro em `[x1+26400]` (offset efetivo `0xE720` a partir de `x0`), grava um DWORD no endereço apontado e avança o ponteiro em quatro bytes.

Sequência exata:

```asm
ldr x2, [x1, #26400]
str w3, [x2], #4
str x2, [x1, #26400]
```

A mesma forma de append reaparece em loops posteriores. Isso é evidência direta de uma fila/lista linear de palavras de 32 bits ou buffer de comandos/parâmetros, condicionado pelo flag privado em `x0+0xE708`. **Ainda não é prova isolada de que esse buffer seja submetido ao hardware**, pois não vimos aqui quem consome o ponteiro ou quem dispara a execução.

**Laços de preenchimento (`+240..+396`)**

- Um laço executa 32 iterações (`w4=0x20`) e, em cada iteração, grava `w24` no endereço `[x5]`; quando o flag de append está ativo, também acrescenta `w24` ao buffer apontado por `[x0+0x8000+26400]` e avança o ponteiro em 4 bytes.
- Outro laço executa 64 iterações (`w19=0x40`) e grava `w23` em `[x5]`; sob a mesma condição, acrescenta `w23` ao buffer e avança o ponteiro.
- Há ainda um append do valor `0xE0` em `+408..+424`.

O dump não mostra incremento do endereço `[x5]` dentro desses dois laços, mas o ponteiro do buffer de append é incrementado. Não devemos assumir que as escritas repetidas em `[x5]` representem uma matriz contígua; a sequência pode atualizar um registrador/slot fixo enquanto serializa valores em outro buffer.

**Parâmetros adicionais (`+428..+984`)**

- A função consulta `[x22+264]+36` e `[x22+32]` para decidir se percorre uma sequência de valores adicionais.
- No laço `+776..+884`, consulta callback `[x25+448]` com `w1=1`; se o retorno for diferente de zero, obtém dois valores por callback `[x25+480]`, combina-os em um DWORD (`low16 | high16<<16`) e grava o resultado em `[x1+228]`, com `x1` obtido de `[x20+24]`.
- Quando o flag privado de append está ativo, grava o DWORD combinado no buffer e avança o ponteiro. Em certos caminhos, também grava `0xE0` como palavra adicional.
- O laço usa `[x22+56]` como limite e atualiza os índices/offsets locais `w23`, `w24` e `w19`.
- As chamadas a `printf` em `+936` e `+984` são logs; não são submissão ao VE.

### Interpretação técnica — níveis de confiança

1. **Observado diretamente:** a função monta valores de 32 bits a partir de callbacks/valores privados, atualiza campos de uma estrutura em `[x20+24]` e, quando o DWORD privado em `x0+0xE708` vale `1`, serializa palavras de 32 bits em um buffer cujo ponteiro está em `x0+0xE720`.
2. **Inferência forte:** esse buffer é uma lista/stream de comandos ou parâmetros para uma camada inferior do decoder/VE. A escrita sequencial com incremento de 4 bytes e o nome `...WeightTableRegisters` tornam essa hipótese consistente, mas ainda falta localizar o consumidor e o mecanismo de execução.
3. **Não demonstrado:** que `x0+0xE720` seja diretamente um endereço MMIO; o código o trata como ponteiro para memória comum e atualiza o próprio ponteiro. Não há, neste trecho, `str` para endereço fixo de registrador físico, ioctl, chamada explícita a `VeOpsS` ou disparo de interrupção.
4. **Importante:** o fato de a função ser chamada durante `H264DecodeSliceHeader` reforça que o parser ARM prepara parâmetros que podem ser consumidos pelo hardware. Isso é compatível com a arquitetura híbrida ARM + VE do A733, e não caracteriza decodificação H.264 inteiramente por software.

### Próximo passo focado

O disassembly ainda está incompleto: a função termina em `+1916`, mas o dump recebido vai somente até `+984`. Continuar do endereço `0xfffff6f07f80`:

```gdb
disassemble /r 0xfffff6f07f80, 0xfffff6f08120
```

O objetivo é terminar o corpo da rotina e descobrir se há outras operações/callbacks que esclareçam o buffer e a eventual transferência para o VE. Depois disso, devemos procurar referências ao campo privado em `x0+0xE720` e ao flag em `x0+0xE708` para localizar quem consome a lista.

### Classificação

- **Documentado:** arquitetura modular VideoEngine/SBM/FBM no Decoder Developer Guide v1.5, seção 3.5; o guia não define os offsets privados desta implementação.
- **Observado:** gravações na estrutura apontada por `[x20+24]`; append sequencial de DWORDs pelo ponteiro em `x0+0xE720`, condicionado por `x0+0xE708 == 1`; callbacks e laços descritos acima.
- **Inferido:** buffer de comandos/parâmetros de registradores destinado à camada inferior/VE.
- **Desconhecido:** layout e tipo formal da estrutura privada; significado de cada palavra; função que consome o buffer; mecanismo final de kick/submissão ao VE; comportamento dos caminhos após `+984`.

## 14.77 — Continuação de H264CongigureWeightTableRegisters: serialização de parâmetros adicionais (22/11/2026)

### Fonte consultada antes da interpretação

O Decoder Developer Guide v1.5, seção 3.5, descreve o VideoEngine como responsável pela execução da decodificação e separa essa camada do SBM e do FBM. A seção 3.6.3 trata da configuração pública do VideoEngine, mas não especifica a rotina privada `H264CongigureWeightTableRegisters`, seus callbacks nem o layout do buffer observado abaixo. Portanto, a documentação confirma a arquitetura geral, mas não permite dar nomes oficiais aos offsets privados.

### Evidência GDB recebida

Continuação da implementação real `H264CongigureWeightTableRegisters`, endereço-base `0xfffff6f07ba4`. O trecho recebido cobre `+988` até `+2148` (não é o fim da função: há branches que retornam a blocos anteriores e o dump termina em `+2148`).

**1. Continuidade do buffer de palavras (`+1460..+1668`, `+1780..+1804`)**

O código continua a usar o mesmo mecanismo identificado na seção 14.76:

```asm
ldr x1, [x0, #26400]   // ponteiro corrente
str w23, [x1], #4      // acrescenta um DWORD
str x1, [x0, #26400]   // grava ponteiro avançado
```

No trecho atual, o endereço-base `x0` para essa estrutura é derivado de `[x20+32]`, com deslocamentos `0x8000`/`0xC000`; o ponteiro de escrita fica no campo efetivo `base+0xE720`, e o flag de habilitação consultado anteriormente fica em `base+0xE708`. Quando o flag vale 1, a função acrescenta valores como `w23` ou `w28` ao buffer e avança o ponteiro em 4 bytes. Em alguns caminhos acrescenta também o par de DWORDs `w28` e `0xE0` (instrução `stp w28,w3,[x1]`) e avança 8 bytes.

**2. Leitura/empacotamento de valores (`+1484..+1616`, `+1720..+1760`)**

A função chama repetidamente o callback carregado de `[x25+480]`, duas vezes por par. O primeiro retorno é mascarado para 16 bits (`and w28,w0,#0xffff`); o segundo retorno é deslocado 16 bits e combinado com o primeiro (`orr w28,w28,w0,lsl #16`). Portanto, o DWORD `w28` contém dois valores de 16 bits obtidos em chamadas sucessivas ao mesmo callback.

Esse DWORD combinado é armazenado em `[x1+228]`, onde `x1` é carregado de `[x20+24]`. Sob o flag de serialização ativo, também é acrescentado ao buffer em `base+0xE720`. O trecho não identifica a semântica dos dois valores de 16 bits; não se deve chamá-los de offsets, deltas ou coeficientes específicos sem validar o callback e o estado do parser.

**3. Iteração por entradas (`+1380..+1700`)**

- `x27 = [x21+3952]`; a função consulta o primeiro DWORD apontado por `x27` para escolher caminhos (`<=2` ou `>2`).
- `w23` inicia em `0x200` (512), `w24` em `0x180` (384) e `w19` em zero.
- O limite do laço vem de `[x22+60]`.
- A cada iteração normal, `w19` avança 1, `w23` avança 8 e `w24` avança 4.
- O código escreve `w23` em `[x1+224]` e o DWORD combinado `w28` em `[x1+228]`; em um caminho também escreve `0xE0` como terceiro DWORD.

Isso confirma uma sequência de registros/pares de parâmetros que é atualizada na estrutura privada e, quando habilitado, espelhada/serializada no buffer. Os valores iniciais 512 e 384 são constantes observadas; seu significado físico ainda não foi demonstrado.

**4. Callback de condição (`+1700..+1716`)**

Antes de processar cada entrada, a função chama o callback `[x25+448]` com `w1=1`. Retorno zero desvia para um caminho de tratamento/continuação; retorno não zero permite prosseguir com a leitura de valores pelo callback `[x25+480]`. O significado do callback ainda depende de resolver os slots da tabela de funções em `x25`.

**5. Caminho alternativo (`+1916..+2004`)**

O fluxo alternativo testa o byte `[x1+32]`. Se for 1, define `w23=0x20` e `w1=5`, prepara `w24=0x20` e retorna ao bloco comum em `+108`. A finalidade desse modo ainda não está identificada.

**6. Logs e branches**

As chamadas `printf@plt` nos offsets `+1068`, `+1120`, `+1212`, `+1260`, `+1316`, `+1864`, `+1908`, `+1996`, `+2048`, `+2096` e `+2144` são caminhos de log. Os branches ao bloco `+1404` e aos blocos anteriores indicam que a rotina continua em laços; o endereço final do dump (`+2148`) ainda não é o epílogo.

### Interpretação técnica — atualização

1. **Observado diretamente:** a função combina pares de retornos de 16 bits em DWORDs, atualiza campos em uma estrutura privada (`+224`, `+228`) e, quando um flag privado está ativo, serializa os DWORDs em um buffer por meio de um ponteiro incrementado em unidades de 4 bytes.
2. **Inferência forte, ainda não confirmação final:** o buffer tem formato compatível com uma lista de parâmetros/comandos de registradores para a camada inferior do decoder/Video Engine. O nome da função e a serialização condicional sustentam a hipótese, mas ainda falta identificar quem consome o buffer e como ele é submetido.
3. **Não demonstrado:** o buffer não foi provado como MMIO. Neste trecho, o código grava em memória apontada por um ponteiro armazenado numa estrutura privada; não aparece endereço físico fixo de registrador, ioctl ou chamada explícita a `VeOpsS`.
4. **Não confundir:** as chamadas ao callback `[x25+480]` são operações indiretas cuja identidade ainda não foi resolvida. Não podemos concluir apenas por este dump que sejam `H264GetBitsSw` ou leitura de bitstream.

### Próximo passo focado

O dump termina em `+2148`, mas ainda há branches para trás e não foi mostrado o epílogo. Obter o trecho seguinte para fechar a função:

```gdb
disassemble /r 0xfffff6f08408, 0xfffff6f08540
```

Depois de fechar o corpo, o passo mais útil será resolver os callbacks `[x25+448]` e `[x25+480]` e localizar referências cruzadas ao campo `base+0xE720` (ponteiro do buffer), para identificar o consumidor/submitter.

### Classificação

- **Documentado:** separação VideoEngine/SBM/FBM no Decoder Developer Guide v1.5, seção 3.5; a documentação não descreve esta rotina privada.
- **Observado:** empacotamento low16/high16, gravação nos offsets privados `+224`/`+228`, append de DWORDs e pares de DWORDs em buffer com ponteiro incrementado.
- **Inferido:** buffer de parâmetros/comandos associado à configuração do decoder/VE.
- **Desconhecido:** significado semântico dos valores; identidade dos callbacks; consumidor do buffer; mecanismo de submissão/execução no VE; fim da função e caminhos após `+2148`.

## 14.78 — Fim de H264CongigureWeightTableRegisters e identificação dos helpers seguintes (22/11/2026)

### Trecho recebido

O usuário executou:

```gdb
disassemble /r 0xfffff6f08408, 0xfffff6f08540
```

O intervalo começa na última instrução do fluxo de `H264CongigureWeightTableRegisters` (`+2148`) e avança por funções adjacentes.

### 1. Fechamento de H264CongigureWeightTableRegisters

Em `0xfffff6f08408` (`+2148`) há um branch incondicional para `0xfffff6f0814c` (`+1448`), que faz parte do laço de processamento já analisado. Em `0xfffff6f0840c` há um `nop`; a função seguinte começa em `0xfffff6f08410`, rotulada `H264AddSorted`.

Conclusão: o trecho não revela outro mecanismo de submissão ao VE. Ele confirma que o endereço `+2148` ainda é um retorno ao laço interno e que o código subsequente pertence a outras funções. O retorno da rotina ocorre pelos caminhos que saltam para o epílogo em `+1004..+1028` (`ldp` dos registradores preservados e `ret`).

### 2. H264AddSorted — `0xfffff6f08410`

A assinatura inferida pelo uso de registradores AArch64 é:
- `x0`: vetor/array de destino;
- `x1`: vetor/array de origem;
- `w2`: quantidade de elementos a percorrer;
- `w3`: chave/valor de comparação;
- `w4`: seletor de ordenação (controla o sentido da comparação).

Evidências:
- `ldr x7, [x1, x5, lsl #3]` percorre ponteiros de 8 bytes no array de origem.
- `ldr w6, [x7, #8]` lê um campo de 32 bits no offset `+8` de cada objeto.
- `cmp w3, w6` e `cmp w6, w9`, junto a `cset ... lt`, determinam a posição de inserção/ordenação.
- `str x7, [x11, x10, lsl #3]` grava ponteiros no array de destino.

Interpretação: helper genérico de inserção/ordenação de entradas referenciadas por ponteiros, comparando o campo DWORD em `objeto+8`. O dump não permite atribuir nome semântico ao campo.

### 3. H264RemoveShort_at_index — `0xfffff6f084a4`

Evidências:
- valida `w1 <= 31`; acima disso chama `abort()`.
- calcula o endereço de uma entrada com stride de 8 bytes (`add x0, x0, w1, sxtw #3`).
- zera um campo de 8 bytes em offset `+12920` relativo à estrutura-base/entrada calculada.
- decrementa o contador DWORD em `[x3+52]`.
- se ainda houver elementos, prepara endereços e chama `memmove@plt` para compactar o array.

Interpretação: remoção de uma entrada de uma lista de referências curtas, com compactação dos elementos subsequentes. O nome do símbolo sugere short-term reference picture; a implementação é consistente com gerenciamento de lista de referências H.264, mas a estrutura exata ainda não está identificada.

### 4. H264BuildDefList — início em `0xfffff6f08500`

O dump cobre somente `+0..+60`, insuficiente para caracterizar a função. Observa-se:
- `w4` é copiado para `w27` e comparado com 1 e 2;
- `w28 = w27 XOR 3`;
- `w6` recebe 1 ou 2 conforme `w27 == 2`;
- `x1` é preservado em `x26`, `w2` em `w25` e `w3` em `w23`.

O nome `H264BuildDefList` sugere construção da lista padrão de referências H.264. Isso é compatível com a proximidade de `H264AddSorted` e `H264RemoveShort_at_index`, mas ainda é uma inferência baseada nos nomes e no código parcial.

### 5. Relação com o objetivo CedarX/VE

Esses três helpers são de gerenciamento/ordenação de listas de referências de imagens H.264. Não há, no trecho fornecido, escrita em registradores VE, chamadas a `VeOpsS`, ioctl, nem acesso ao buffer condicional em `base+0xE720` identificado em `H264CongigureWeightTableRegisters`.

Portanto, não devemos confundir a preparação da lista de referências H.264 com a submissão de comandos ao hardware. O helper `H264BuildDefList` pode esclarecer a semântica dos parâmetros que a rotina de peso usa, mas não é, por si só, evidência de execução no VE.

### Próximo passo

Completar apenas o corpo de `H264BuildDefList`:

```gdb
disassemble /r 0xfffff6f08500, 0xfffff6f08700
```

Depois, voltar à investigação principal: resolver os callbacks em `[x25+448]` e `[x25+480]` e localizar o consumidor do buffer apontado por `base+0xE720`.

### Classificação

- **Observado:** fim do fluxo de `H264CongigureWeightTableRegisters` no intervalo; `H264AddSorted` percorre arrays de ponteiros e compara `objeto+8`; `H264RemoveShort_at_index` remove e compacta entradas; `H264BuildDefList` inicia com lógica de seleção de modo/lista.
- **Inferido:** os helpers pertencem ao gerenciamento de listas de referência H.264 (short-term/default list).
- **Não demonstrado:** qualquer ligação direta desses helpers com a submissão ao VE ou com o buffer de comandos.
- **Pendente:** corpo completo de `H264BuildDefList`; semântica dos campos e callbacks; consumidor do buffer condicional.

---

## 14.79 — Corpo completo de `H264BuildDefList` (GDB)

**Endereço:** `0xfffff6f08500`  
**Intervalo recebido:** `0xfffff6f08500`–`0xfffff6f086fc`  
**Objetivo desta etapa:** caracterizar a construção da lista padrão de referências H.264 e distinguir essa preparação da submissão ao Video Engine.

### 1. Argumentos e variáveis preservadas

A partir dos movimentos no prólogo, os argumentos são preservados assim:

| Entrada | Registrador preservado | Evidência |
|---|---|---|
| `x0` | `x21` | `mov x21, x0` em `+80` |
| `x1` | `x26` | `mov x26, x1` em `+52` |
| `w2` | `w25` | `mov w25, w2` em `+48` |
| `w3` | `w23` | `mov w23, w3` em `+44` |
| `w4` | `w27` | `mov w27, w4` em `+20` |

Outros valores calculados:
- `w28 = w27 XOR 3` (`+24`).
- `w22 = 280` (`0x118`, `+84`): tamanho de cada slot/objeto copiado na lista de destino.
- `w24` começa em zero e é incrementado quando uma entrada é incluída; é o valor retornado em `w0` (`+392..+420`).
- `w19` e `w20` funcionam como índices de varredura da lista de origem.

A interpretação de `x0` como base de destino e `x1` como array de origem é sustentada pelo cálculo de destino `x21 + índice*280` e pela leitura de ponteiros em `[x26 + índice*8]`. A assinatura C exata e os tipos formais ainda não foram recuperados.

### 2. Estrutura da lista de origem e filtros

A função percorre um array de ponteiros de 64 bits:

```asm
ldr x6, [x26, x5, lsl #3]
```

Para cada ponteiro não nulo, consulta um DWORD no deslocamento `+164`:

```asm
ldr w1, [x6, #164]
tst w28, w1
```

ou, em outro caminho:

```asm
ldr w0, [x1, #164]
tst w27, w0
```

Portanto, o campo `objeto+164` é usado como campo de flags/máscara, e `w27`/`w28` selecionam bits para filtrar entradas. O significado nominal dos bits ainda não está estabelecido.

O algoritmo tem duas fases de varredura/intercalação, com índices `w19` e `w20`. Ele pula ponteiros nulos e usa testes de máscara para decidir qual entrada considerar. Os saltos entre `+96..+220` e `+328..+388` mostram que a função não é apenas uma cópia linear: há seleção de entradas de acordo com o modo (`w27`) e as flags do objeto.

### 3. Campos modificados e cópia de objetos

Quando uma entrada é selecionada, a função calcula um valor a partir de `objeto+12` ou do índice da entrada, dependendo de `w23`:

```asm
cbnz w23, ...
ldr  w6, [x1, #12]   // executado quando w23 == 0
str  w6, [x1, #16]
```

Assim, quando `w23 == 0`, o valor de `objeto+12` é copiado para `objeto+16`; quando `w23 != 0`, o valor usado é o índice corrente. A função modifica a estrutura de origem antes ou durante a construção da lista.

A posição de destino é calculada com stride de 280 bytes:

```asm
smaddl x5, w5, w22, x21
```

E a estrutura inteira é copiada:

```asm
mov x0, x5
mov x2, #0x118
bl  memcpy@plt
```

Há caminhos que depois ajustam campos no objeto copiado:

- `destino+164` recebe `w28` ou `w27`, conforme o caminho;
- `destino+16` recebe `2*w6` ou `2*w6+1`;
- `destino+176` recebe um valor temporário guardado na pilha.

Isso demonstra que a lista de destino contém cópias de estruturas de 280 bytes, com campos de flags/índice recalculados para a lista construída. Não identifica, por si só, o tipo C completo da estrutura nem o significado semântico de cada campo.

### 4. Retorno

No caminho normal de saída, `w24` é movido para `w0` e a função restaura os registradores preservados antes de retornar. Portanto, o retorno é a quantidade de entradas adicionadas à lista de destino, conforme a interpretação mais direta do contador `w24` que é incrementado nas inclusões.

### 5. Relação com o fluxo de decodificação

O corpo é consistente com a construção/filtragem de uma lista de imagens de referência H.264. Isso se encaixa no contexto das funções vizinhas `H264AddSorted` e `H264RemoveShort_at_index`, mas a classificação como lista padrão de referências ainda se apoia também nos nomes dos símbolos; não foi identificado aqui o chamador nem os enum/defines que dariam nomes formais aos modos `w27` e `w23`.

**Não há evidência de submissão ao hardware neste corpo:** não aparecem chamadas `VeOpsS`, ioctl, escrita em registradores VE ou acesso ao buffer condicional `base+0xE720` identificado em `H264CongigureWeightTableRegisters`. Esta rotina deve ser tratada como preparação de metadados/listas do decodificador, não como execução do decode no VE.

### 6. Classificação das conclusões

- **Observado:** array de ponteiros em `x1`; contador de elementos em `w2`; campo de flags em `objeto+164`; cópias de 280 bytes para slots consecutivos na base `x0`; alterações nos campos `+16`, `+164` e `+176`; retorno do contador `w24`.
- **Inferido com boa confiança:** `x0` é a base da lista de destino; `x1` é a lista de origem; `w24` conta os elementos construídos; a função prepara uma lista filtrada/ordenada de referências H.264.
- **Ainda desconhecido:** nomes formais dos argumentos, enum/semântica de `w27` e `w23`, significado dos campos `+12`, `+16`, `+164` e `+176`, e chamador exato.
- **Não demonstrado:** qualquer submissão de comandos ao Video Engine ou relação direta com o buffer `base+0xE720`.

### 7. Próximo passo

A trilha principal continua sendo encontrar quem consome o ponteiro de escrita armazenado em `base+0xE720`. Em paralelo, para dar nomes concretos aos modos desta função, o próximo dado útil é identificar os chamadores de `H264BuildDefList` e os valores de `w4`/`w3` passados por eles. Não avançar para mais helpers de lista sem essa necessidade; priorizar a ligação com o consumidor do buffer de comandos.

## 14.80 — Chamada real de `H264BuildDefList` e análise de `H264FillDefaultRefList`

### 14.80.1 — Contexto de execução confirmado pelo GDB

Breakpoint em `H264BuildDefList` e backtrace:

```text
#0 H264BuildDefList
#1 H264FillDefaultRefList
#2 H264DecodeSliceHeader
#3 H264ProcessNaluUnit
#4 H264DecoderDecodeSbmFrame
#5 VideoEngineDecode
#6 DecodeVideoStream
```

Valores dos argumentos no breakpoint:

| Registrador | Valor | Interpretação |
|---|---:|---|
| `x0` | `0xfffff0030518` | base do array/lista de destino (`H264BuildDefList`) |
| `x1` | `0xfffff0030418` | base do array de ponteiros de origem |
| `w2` | `1` | quantidade de entradas examinadas |
| `w3` | `0` | modo de seleção/cálculo do campo |
| `w4` | `3` | máscara de flags usada na seleção |

A assinatura efetiva inferida diretamente dos movimentos de registradores e dos acessos da função é:

```c
int H264BuildDefList(void *dst, void **src, int count,
                     int mode, unsigned int flags);
```

Os nomes semânticos dos parâmetros `mode` e `flags` ainda não foram confirmados por símbolos ou documentação pública.

### 14.80.2 — Papel de `H264FillDefaultRefList`

O chamador imediato é `H264FillDefaultRefList`, executado por `H264DecodeSliceHeader`. A função recebe o contexto H.264 em `x0` e salva `x1` na pilha para reutilizá-lo como estrutura auxiliar/estado de referência.

A função calcula endereços internos a partir do contexto:

- `x26 = x0`: contexto H.264;
- `x27 = x26 + 12920` (`0x3278`): array/lista interna passado como origem em chamadas de `H264BuildDefList`;
- `x24 = x26 + 12664` (`0x3178`): outra lista interna usada como origem;
- `x21` é calculado a partir de `x26 + 13176` (`0x3378`) e de um índice multiplicado por `280` (`0x118`), servindo como região de destino das entradas construídas;
- `w22` recebe o byte de `w2` (`and w2,#0xff`), usado em decisões posteriores;
- `w23` recebe o argumento `w3` do chamador;
- `w26+30` é lido como seletor de modo; `w26+32` decide entre dois caminhos principais.

No caminho em que `[x26+32] == 1`, a função chama `H264AddSorted` duas vezes para montar arrays temporários ordenados e, em seguida, chama `H264BuildDefList` para produzir listas de referência. Uma chamada usa `mode=0`; a seguinte usa `mode=1` e `flags=0x10`. O caminho alternativo também constrói listas por chamadas a `H264BuildDefList`.

As entradas de referência têm stride de `0x118` (280 bytes), coerente com o tamanho usado em `H264BuildDefList` e nas operações `memcpy`/`memset` observadas.

### 14.80.3 — Outras operações observadas

- `H264AddSorted` recebe duas vezes arrays temporários e o campo de ordenação, sendo usado para ordenar/particionar referências.
- `H264BuildDefList` filtra entradas pelo DWORD em `objeto+164`, copia 280 bytes por entrada e ajusta campos em `+16`, `+164` e `+176`.
- `H264FillDefaultRefList` calcula contagens e compara-as com limites/contagens de referência no contexto; em caminhos específicos limpa entradas excedentes com `memset`.
- No final do caminho normal, `H264FillDefaultRefList` retorna zero. Há um caminho de validação adicional que inspeciona contadores e pode sair antecipadamente.

### 14.80.4 — Conclusão técnica e limites

O backtrace estabelece que `H264BuildDefList` é parte do processamento de cabeçalho de slice e da montagem de listas padrão de referências H.264, não uma rotina de submissão de comandos ao Video Engine. O fato de a pilha incluir `VideoEngineDecode` demonstra que a rotina está sendo executada dentro da chamada de decodificação, mas não significa que cada função nessa pilha programe diretamente o hardware.

Não há, no disassembly apresentado de `H264FillDefaultRefList`/`H264BuildDefList`, chamada a `VeOpsS`, ioctl, escrita em registradores VE ou mecanismo de kick. O buffer sequencial observado em `H264CongigureWeightTableRegisters` continua sendo a pista mais concreta para a ponte entre parser ARM e camada de hardware; seu consumidor ainda precisa ser localizado.

**Observado:** hierarquia de chamadas, valores dos argumentos no breakpoint, offsets, stride de 280 bytes, chamadas de ordenação/cópia/limpeza.

**Inferido:** finalidade de montagem das listas de referência padrão H.264, sustentada pelo nome das funções e pelo contexto de `H264DecodeSliceHeader`.

**Ainda desconhecido:** nomes formais dos campos privados, semântica exata de cada modo/flag e ligação entre o buffer montado por `H264CongigureWeightTableRegisters` e a rotina que efetivamente inicia o hardware.

## 14.81 — Continuação de `H264DecodeSliceHeader`: segunda chamada de listas padrão, FBM e FIFO

### 14.81.1 — Chamada de `H264FillDefaultRefList` neste caminho

O disassembly de `0xfffff6f0dd00` a `0xfffff6f0df00` mostra outra chamada a `H264FillDefaultRefList`, em `0xfffff6f0dd68`:

```asm
0xfffff6f0dd5c: ldrb w2, [x25, #48]
0xfffff6f0dd60: mov  x0, x19
0xfffff6f0dd64: ldr  x1, [x25, #56]
0xfffff6f0dd68: bl   H264FillDefaultRefList@plt
0xfffff6f0dd6c: b    0xfffff6f0d818
```

Assim, nesta chamada:
- `x0 = x19`: contexto/estrutura da slice;
- `w2 = *(uint8_t *)(x25+48)`;
- `x1 = *(uint64_t *)(x25+56)`.

Após a chamada, o fluxo salta para `H264DecodeSliceHeader+872` (`0xfffff6f0d818`). Isso confirma que a montagem da lista padrão é uma etapa do processamento do cabeçalho, e não uma chamada direta de programação do VE.

### 14.81.2 — Caminhos próximos: POC, PTS e estado da slice

O trecho também mostra:
- chamada a `H264InitPoc` em `+2212`;
- chamada indireta por `[x20+480]`, com retorno gravado em `[x19+36]` e, em outro caminho, em `[x19+40]`;
- chamada a `H264CalculatePicturePts` em `+2248`, com o byte de estado `[x19+33]` preparado antes;
- verificações de largura (`[x19+208] == 0x780`, isto é, 1920), estado de slice e flags antes de seguir para o caminho condicional iniciado em `+2296`.

Essas operações são consistentes com atualização de POC/PTS e estado temporal/metadados da imagem. Não são, isoladamente, evidência de submissão ao hardware.

### 14.81.3 — FBM: retorno de buffer e limpeza de estado

No caminho condicional entre `+2296` e `+2496`:
- carrega um objeto a partir de `[x28+25488]` e obtém outro objeto por `[objeto+248]`;
- lê um ponteiro em `[x22+64]` e compara seu endereço, acrescido de `0x1869f`, com `[x28+25920]`;
- sob as condições verificadas, chama `FbmReturnBuffer(x0, x22, 0)` em `+2396`;
- calcula endereços de estruturas usando stride `0x118` (280 bytes), zera campos no offset `+3952` e atualiza bytes nos offsets `+3976`, `+1450` e `+1452`;
- copia campos do contexto da slice para offsets `+160` e `+168` e grava um DWORD em `[objeto+172]`.

A chamada explícita `FbmReturnBuffer` confirma que esse ramo inclui devolução/reciclagem de um buffer gerenciado pelo FBM. As operações de limpeza e atualização de estruturas são gerenciamento de estado e de buffers; não identificam o ponto de execução do VE.

### 14.81.4 — Entrada em `FIFODequeue`

Em `+2512`, o código chama `FIFODequeue` sobre uma estrutura localizada em `x19+0xE6C8` (`59080` bytes). Se o retorno não for nulo, lê o objeto retornado e copia campos para o objeto apontado por `[x28+25488]`:
- dois pares de ponteiros a partir dos offsets `+0` e `+16` são gravados nos offsets `+200` e `+216` do destino;
- um DWORD do offset `+32` é gravado em `+232`;
- o valor de `[x26+9332]` é gravado no objeto dequeued em `+36`;
- o ponteiro do objeto de destino é gravado em `+40`.

O trecho termina em `+2636`, no início de verificações adicionais sobre o valor do campo `+36`. Portanto, o conteúdo visto comprova uma fila interna de objetos/eventos e transferência de metadados, mas ainda não revela o consumidor final nem uma submissão ao VE.

### 14.81.5 — Conclusão e próximo alvo

**Observado:** segunda chamada a `H264FillDefaultRefList`; chamadas a `H264InitPoc` e `H264CalculatePicturePts`; chamada explícita a `FbmReturnBuffer`; uso de estruturas com stride de 280 bytes; chamada a `FIFODequeue` e cópia de campos do objeto retornado.

**Inferido:** os ramos analisados gerenciam listas de referência, estado temporal, buffers FBM e objetos de uma FIFO no contexto do parser/decoder.

**Ainda não demonstrado:** que `FIFODequeue` ou qualquer uma dessas operações programe ou dispare o Video Engine. A trilha mais forte para investigar continua sendo o buffer de palavras de 32 bits preenchido por `H264CongigureWeightTableRegisters`, cujo consumidor ainda precisa ser identificado.

Próximo trecho recomendado: continuar a partir de `H264DecodeSliceHeader+2636` para acompanhar a validação e o uso dos objetos retirados da FIFO:

```gdb
disassemble /r 0xfffff6f0def8, 0xfffff6f0e180
```

## 14.82 — Continuação de H264DecodeSliceHeader: consumo da FIFO e atualização de metadados (2026-11-22)

### Trecho analisado

Desmontagem de `H264DecodeSliceHeader+2632` até `+3276`, endereços `0xfffff6f0def8` a `0xfffff6f0e180`.

### 1. Final do laço anterior: ajuste de identificadores

Em `+2632..+2660`, o código percorre entradas de uma tabela:

- `x1` percorre elementos com passo `0x20` (32 bytes).
- `x4` é o limite calculado para o laço.
- Cada elemento contém um ponteiro em `[x1]`; o código lê um DWORD em `[objeto+36]`.
- Se esse DWORD não for `-1`, soma `w5`, cujo valor construído imediatamente antes é `0x80000100`, e grava o resultado novamente em `[objeto+36]`.
- Ao terminar, retorna ao fluxo comum em `H264DecodeSliceHeader+1284`.

**Interpretação cautelosa:** é uma transformação de identificadores/índices associados aos objetos da tabela. O trecho isolado não permite atribuir significado semântico definitivo ao valor `0x80000100`.

### 2. Callback e contadores de slices

Em `+2664..+2720`, o código chama indiretamente o callback armazenado em `[x20+456]`:

- Primeiro resultado `w0` é incrementado e gravado em `[x19+56]`.
- Se o byte `[x19+32]` for `1`, o callback é chamado novamente; o resultado incrementado é gravado em `[x19+60]`.
- Os valores são comparados com `0x20` (32). Se ultrapassarem o limite, o código segue para a rotina de diagnóstico/erro em `+2728` ou para o caminho em `+3064`.

Isso aparenta contabilizar posições/elementos de slice ou unidades relacionadas ao parser. A identidade do callback ainda não foi resolvida.

### 3. Consumo de FIFO: cópia de metadados

O caminho iniciado em `H264DecodeSliceHeader+2512` (`FIFODequeue`) continua da seguinte forma:

- `FIFODequeue` retorna um nó; `[nó+8]` é tratado como ponteiro para o payload.
- `x1` recebe o ponteiro de destino carregado de `[x28+25488]`.
- Quatro campos de 64 bits do payload, offsets `0`, `8`, `16` e `24`, são copiados para offsets `200`, `208`, `216` e `224` do destino.
- O DWORD do payload em `+32` é copiado para destino `+232`.
- O DWORD `[x26+9332]` é gravado no payload `+36`.
- O ponteiro de destino `x1` é gravado no payload `+40`.

A operação caracteriza transferência/associação de metadados entre um item retirado da FIFO e uma estrutura do decoder. Não há, nesse bloco, escrita em registradores VE, chamada de ioctl, `VeOpsS` ou comando de início de decodificação.

### 4. Limite de sequência e tabela auxiliar

Depois da cópia, o código compara o DWORD de `[x26+9332]` com `0x7ffffffe`. Em um dos caminhos, consulta `[x26+9920]` e percorre uma tabela apontada por `[x28+26296]`, com elementos de 32 bytes. Para cada ponteiro encontrado, examina `[objeto+36]`; se o valor não for `-1`, soma `0x80000100` e atualiza o campo.

O padrão reforça que esse bloco faz bookkeeping de objetos/índices e valores sentinela. Ainda não há evidência de que a FIFO em questão seja uma fila de comandos de hardware; a cópia de campos e a atualização de referências são compatíveis com uma fila interna de metadados.

### 5. Conclusão desta amostra

**Observado:** `FIFODequeue` fornece um payload cujo conteúdo é copiado para offsets `+200..+232` de uma estrutura do decoder; o payload recebe de volta um identificador e um ponteiro para essa estrutura. Há também atualização condicional de campos `+36` em objetos referenciados por uma tabela auxiliar.

**Não demonstrado:** que essa FIFO seja o command buffer do Video Engine, ou que esse trecho submeta trabalho ao hardware. A hipótese de fila de metadados/objetos internos é mais consistente com as operações visíveis, mas permanece uma inferência.

**Direção de investigação:** manter como trilha prioritária o buffer sequencial de DWORDs identificado em `H264CongigureWeightTableRegisters` (seção anterior sobre `x0+0xE708` / ponteiro em `x0+0xE720`). O próximo passo deve identificar os leitores/consumidores desse ponteiro e o mecanismo de kick/execução, em vez de classificar esta FIFO como fila de comandos sem evidência.

## 14.83 — Isolamento do breakpoint em H264CongigureWeightTableRegisters

### Estado observado no GDB

Foi instalado o breakpoint 18 no endereço real da função:

```gdb
break *0xfffff6f07ba4
```

O GDB confirmou:

```text
Breakpoint 18 at 0xfffff6f07ba4 <H264CongigureWeightTableRegisters>
```

Após `continue`, a execução foi interrompida sucessivamente pelos breakpoints 17 (`H264BuildDefList`), 13 (`H264DetectIsFullFrame`, duas ocorrências) e 14 (`H264DecoderDecodeSbmFrame+412`). A tabela de breakpoints mostrou o breakpoint 18 habilitado (`Enb y`), mas sem indicação de que tenha sido atingido.

Os breakpoints 13, 14, 15, 16 e 17 continuam habilitados e interrompem o fluxo antes que seja possível observar se a execução chega ao endereço 18. Isso explica as interrupções intermediárias; não prova que a função-alvo seja chamada no processamento atual.

### Próxima ação de depuração

Desabilitar temporariamente apenas os breakpoints intermediários 13–17, mantendo o breakpoint 18 habilitado:

```gdb
disable 13 14 15 16 17
continue
```

Se o breakpoint 18 for atingido, inspecionar imediatamente `bt` e os registradores/argumentos. Se não for atingido e o decoder terminar ou avançar para outro fluxo, será necessário verificar o caminho condicional de `H264DecodeSliceHeader` e confirmar se a slice atual realmente executa a chamada a `H264CongigureWeightTableRegisters`.

**Conclusão:** o endereço do breakpoint 18 está correto e habilitado. A ausência de parada nele ainda não é evidência de que a função não seja chamada; os breakpoints 13–17 estão causando interrupções intermediárias e devem ser isolados primeiro.

## 14.84 — Breakpoint 18 não foi atingido antes do término normal do decoder

### Experimento

Com o breakpoint 18 habilitado em `0xfffff6f07ba4` (`H264CongigureWeightTableRegisters`), foram desabilitados os breakpoints intermediários 13–17 e executado `continue`:

```gdb
disable 13 14 15 16 17
continue
```

### Resultado observado

O processo `vdecoderdemo` terminou normalmente (`Inferior 1 ... exited normally`) sem parar no breakpoint 18. O log final informou:

- `decode frame: 32`
- `display frame: 31`
- encerramento das threads de display, parser e decoder.

### Interpretação

- O breakpoint 18 estava habilitado, conforme a listagem anterior de breakpoints.
- Nesta execução, o endereço de entrada de `H264CongigureWeightTableRegisters` não foi executado antes do fim do processo.
- O decoder completou o teste sem falha fatal; portanto, a ausência do breakpoint não indica, por si só, erro de decodificação.
- A hipótese mais provável é que o caminho de execução das slices deste bitstream não tenha satisfeito a condição que leva à chamada de `H264CongigureWeightTableRegisters`. Outra possibilidade a excluir é que o endereço observado corresponda a uma implementação/ramo diferente do efetivamente usado nesta execução.
- Isso não invalida a evidência estática de que a função contém escrita sequencial em um buffer apontado por `base + 0xE720`; apenas significa que ainda não observamos essa função em execução.

### Próxima verificação

Inspecionar a região do chamador `H264DecodeSliceHeader` ao redor do ponto conhecido da chamada (offset aproximado +960, endereço `0xfffff6f0d870`) para identificar a condição de controle e determinar por que o fluxo atual pode saltá-la:

```gdb
disassemble /r 0xfffff6f0d850, 0xfffff6f0d890
```

Não concluir que a função é dispensável ou que não participa do decode em geral até verificar esse ramo e, se necessário, repetir o teste com um bitstream que ative weighted prediction.

## 14.85 — Condições exatas para chamar `H264CongigureWeightTableRegisters`

### Disassembly fornecido

Foi inspecionado o trecho `H264DecodeSliceHeader+928` até `+988`, no endereço `0xfffff6f0d850`:

```asm
0xfffff6f0d854: ldr x0, [x19, #264]
0xfffff6f0d858: ldr w1, [x0, #36]
0xfffff6f0d85c: cbz w1, 0xfffff6f0db2c
0xfffff6f0d860: ldrb w1, [x19, #32]
0xfffff6f0d864: cmp w1, #0
0xfffff6f0d868: ccmp w1, #3, #0x4, ne
0xfffff6f0d86c: b.ne 0xfffff6f0db2c
0xfffff6f0d870: mov x1, x19
0xfffff6f0d874: mov x0, x20
0xfffff6f0d878: bl H264CongigureWeightTableRegisters@plt
```

### Condição deduzida diretamente das instruções

A chamada só acontece quando as duas condições abaixo são verdadeiras:

1. O DWORD em `*(uint64_t *)(x19 + 264) + 36` é diferente de zero. Em termos de acesso: carregar o ponteiro armazenado em `[x19+264]` e verificar se o DWORD em `[ponteiro+36]` não é zero.
2. O byte em `[x19+32]` é exatamente `0` ou `3`.

Se a primeira condição falhar, `cbz` desvia para `H264DecodeSliceHeader+1660`. Se a segunda falhar, `b.ne` também desvia para `+1660`. Só então são preparados os argumentos (`x0=x20`, `x1=x19`) e feita a chamada.

A instrução `ccmp` implementa a comparação alternativa: aceita `w1==0` ou `w1==3`. Ainda não está comprovado o significado semântico dos campos `[x19+264]+36` e `[x19+32]`; não os rotular como weighted-prediction ou tipo de slice sem identificar a estrutura/campo.

### Próximo experimento

Reexecutar `vdecoderdemo` com breakpoint no início da avaliação da condição. Em cada parada, registrar os dois valores usados pelo ramo:

```gdb
break *0xfffff6f0d854
run
```

Quando parar:

```gdb
set $slice = $x19
set $aux = *(void **)($slice + 264)
printf "slice=%p aux=%p campo_aux36=", $slice, $aux
x/wx $aux+36
printf "campo_slice32="
 x/1ub $slice+32
```

Repetir com `continue` para as próximas slices. Se o breakpoint não parar no início do trecho, confirmar novamente o endereço de carga/ASLR e o caminho de execução. O objetivo é registrar qual das duas condições elimina a chamada nesta amostra, sem inferir ainda a semântica dos campos.

## 14.86 — Breakpoints absolutos não podem ser inseridos antes do carregamento da biblioteca

### Resultado observado

Ao executar `run`, o GDB informou:

- Não conseguiu restaurar os breakpoints 9 e 10 porque os símbolos `SbmAddStream` e `SbmFrameAddStream` não estavam disponíveis naquele contexto.
- Não conseguiu inserir os breakpoints absolutos em `0xfffff6f07ba4` e `0xfffff6f0d854`, pois esses endereços ainda não estavam mapeados.

### Interpretação

Os endereços usados anteriormente pertencem a bibliotecas compartilhadas do CedarX/H.264. Antes de o processo carregar essas bibliotecas, os endereços não são válidos para inserir breakpoints. Além disso, a base de carga pode mudar entre execuções devido a ASLR. Portanto, não se deve reutilizar cegamente endereços absolutos de uma execução anterior.

Os avisos dos breakpoints 9 e 10 são referências simbólicas antigas que também precisam ser removidas ou recriadas quando a biblioteca estiver carregada.

### Procedimento corrigido

Remover os breakpoints antigos e configurar um breakpoint simbólico pendente no início de `H264DecodeSliceHeader`. O GDB deverá resolvê-lo quando a biblioteca for carregada:

```gdb
 delete 9 10 18 19
set breakpoint pending on
break H264DecodeSliceHeader
run
```

Quando parar no início da função, obter o endereço de execução atual e instalar o breakpoint relativo ao endereço real da função:

```gdb
set $slice_header = $pc
break *($slice_header + 944)
continue
```

O deslocamento `944` (decimal, `0x3b0`) corresponde à instrução `H264DecodeSliceHeader+944`, que lê o byte em `[x19+32]`. A avaliação anterior, em `+940`, lê o DWORD em `[[x19+264]+36]`. No breakpoint, registrar ambos os valores antes de continuar.

**Observação:** confirmar que a parada inicial ocorreu exatamente no prólogo/início de `H264DecodeSliceHeader` antes de calcular o endereço relativo. Não assumir que o endereço-base de uma execução anterior permanece válido.

## 14.87 — Condições runtime para H264CongigureWeightTableRegisters

### Breakpoint posicionado na primeira condição

Após reiniciar `vdecoderdemo`, o breakpoint simbólico em `H264DecodeSliceHeader` foi resolvido quando `libawh264.so` foi carregada. O endereço da função nesta execução foi `0xfffff6f0d4c0`.

Breakpoint em `0xfffff6f0d854` (`H264DecodeSliceHeader+932`), imediatamente antes da primeira condição:

```asm
ldr x0, [x19, #264]
ldr w1, [x0, #36]
cbz w1, ...
ldrb w1, [x19, #32]
cmp w1, #0
ccmp w1, #3, #0x4, ne
b.ne ...
```

Valores observados:

```text
x19 = 0xfffff002d1a0
x20 = 0xfffff000cc80
*(uint64_t *)(x19 + 264) = 0xffffe0000f40
*(uint8_t *)(x19 + 32) = 2
```

Interpretação confirmada para esta passagem:
- `x19` é o contexto/estrutura de slice usado por `H264DecodeSliceHeader`.
- `[x19+264]` contém o ponteiro `0xffffe0000f40` para a estrutura auxiliar consultada pela condição.
- O byte `[x19+32]` vale `2`.
- A segunda condição exige que esse byte seja `0` ou `3`. Como o valor observado é `2`, o branch `b.ne` desvia para `H264DecodeSliceHeader+1660` e a chamada a `H264CongigureWeightTableRegisters()` é ignorada nesta passagem, independentemente do resultado da primeira condição.

O DWORD em `aux+36` ainda não foi lido nesta sessão. Seu valor permanece desconhecido.

### Limites da conclusão

Esta observação demonstra por que a chamada foi ignorada nesta passagem específica. Não demonstra que a função nunca seja chamada durante o fluxo inteiro: outras slices podem apresentar valores diferentes em `[x19+32]` e no DWORD `aux+36`.

### Próximo passo

No breakpoint atual, ler o DWORD da primeira condição:

```gdb
set $aux = *(void **)($x19+264)
x/wx $aux+36
```

Depois, observar as próximas passagens no mesmo breakpoint para verificar se `[x19+32]` assume `0` ou `3` em alguma slice. Não atribuir significado semântico ao byte `2` ou ao DWORD de `aux+36` antes de rastrear sua origem/consumidores.

## 14.88 — Valores runtime das duas condições antes de H264CongigureWeightTableRegisters

No breakpoint em `H264DecodeSliceHeader+932` (`0xfffff6f0d854`), foram observados:

```text
x19 = 0xfffff002d1a0
*(uint64_t *)(x19 + 264) = 0xffffe0000f40
*(uint8_t *)(x19 + 32) = 2
```

Leitura adicional solicitada para a primeira condição:

```gdb
set $aux = *(void **)($x19+264)
x/wx $aux+36
```

Resultado:

```text
0xffffe0000f64: 0x00000000
```

### Interpretação direta do assembly

O trecho relevante é:

```asm
ldr x0, [x19, #264]
ldr w1, [x0, #36]
cbz w1, H264DecodeSliceHeader+1660
ldrb w1, [x19, #32]
cmp w1, #0
ccmp w1, #3, #0x4, ne
b.ne H264DecodeSliceHeader+1660
```

Nesta passagem, **as duas condições falham**:

1. `*(uint32_t *)(aux+36) == 0`; portanto, `cbz` desvia diretamente para `+1660`.
2. `[x19+32] == 2`, que também não satisfaz a condição de ser `0` ou `3`.

A primeira condição já é suficiente para impedir a chamada. A segunda também é incompatível com a chamada, mas não chega a ser avaliada em execução porque o primeiro branch já desvia.

### Conclusão limitada

Está confirmado que, nesta passagem de `H264DecodeSliceHeader`, `H264CongigureWeightTableRegisters()` não é chamada porque o DWORD em `aux+36` vale zero. O byte em `x19+32` também vale 2. Não atribuir significado semântico definitivo a esses campos sem rastrear quem os inicializa e quem os consome.

Isso ainda não prova que a função não seja chamada em outra slice. O próximo passo é observar novas entradas no mesmo ponto e registrar `aux+36` e `[x19+32]` para cada passagem; alternativamente, um breakpoint simbólico na própria função pode confirmar diretamente se alguma chamada ocorre durante o restante do fluxo.

## 14.89 — Reentrada no início de H264DecodeSliceHeader e breakpoints duplicados

Na sessão seguinte à inspeção das condições em `H264DecodeSliceHeader+932`, foi instalado novamente um breakpoint no endereço `0xfffff6f0d854`. O GDB informou que o breakpoint 21 já estava instalado nesse endereço e criou também o breakpoint 22:

```text
Note: breakpoint 21 also set at pc 0xfffff6f0d854.
Breakpoint 22 at 0xfffff6f0d854
```

Ao executar `continue`, a execução parou primeiro no breakpoint simbólico 20, no início de `H264DecodeSliceHeader` (`0xfffff6f0d4c0`). Isso não representa uma nova amostra das condições em `+932`; os breakpoints 21/22 ainda não foram atingidos nesta parada.

O log simultâneo informou uma imagem de 1920x816, pixel format 6. Esse log confirma a atividade do thread de display, mas não altera a interpretação dos campos privados analisados.

### Próxima ação

Eliminar a duplicação e evitar a parada no início da função:

```gdb
disable 20
delete 22
continue
```

Manter o breakpoint 21 em `0xfffff6f0d854`. Quando ele for atingido, coletar novamente:

```gdb
set $aux = *(void **)($x19+264)
x/wx $aux+36
x/1ub $x19+32
```

Não concluir que os valores das condições mudaram até obter essa nova leitura no breakpoint correto.

## 14.90 — Nova amostra em H264DecodeSliceHeader+932: slice+32 = 0, mas aux+36 = 0

Após desabilitar o breakpoint simbólico 20 (início de `H264DecodeSliceHeader`) e remover o breakpoint duplicado 22, a execução atingiu o breakpoint 21 em `0xfffff6f0d854`, correspondente a `H264DecodeSliceHeader+932`.

Leitura feita nesta passagem:

```gdb
set $aux = *(void **)($x19+264)
x/wx $aux+36
x/1ub $x19+32
```

Resultado:

```text
0xffffe0000f64: 0x00000000
0xfffff002d1c0: 0
```

Interpretação, considerando as instruções já disassembladas nesse ponto:

```asm
ldr x0, [x19, #264]
ldr w1, [x0, #36]
cbz w1, H264DecodeSliceHeader+1660
ldrb w1, [x19, #32]
cmp w1, #0
ccmp w1, #3, #0x4, ne
b.ne H264DecodeSliceHeader+1660
```

- `[aux+36] == 0`: o `cbz` desvia imediatamente para `+1660`.
- `[x19+32] == 0`: isoladamente, esse valor satisfaria a condição posterior, que aceita 0 ou 3.
- Como a primeira condição já desvia, a execução não alcança a chamada de `H264CongigureWeightTableRegisters()` nesta passagem.

Esta amostra é útil porque mostra que o campo `slice+32` pode assumir 0, ao contrário da amostra anterior em que valia 2. Porém, o campo `aux+36` continua zerado e impede a chamada. Portanto, a condição que ainda precisamos observar é `aux+36 != 0` numa passagem em que `slice+32` seja 0 ou 3.

**Conclusão limitada à amostra:** `H264CongigureWeightTableRegisters()` não é chamada nesta passagem de `H264DecodeSliceHeader+932`. Isso não exclui chamadas em outras slices ou em outros fluxos.

**Próximo passo:** continuar a execução mantendo o breakpoint 21 e registrar os dois campos em cada parada. Não é necessário reinstalar o breakpoint 22.

## 14.91 — Nova passagem em H264DecodeSliceHeader+932: condições novamente bloqueiam a chamada

Nova execução, mantendo o breakpoint 21 em `0xfffff6f0d854`:

```gdb
(gdb) continue
Continuing.

Thread 5 "vdecoderdemo" hit Breakpoint 21, 0x0000fffff6f0d854 in H264DecodeSliceHeader ()
(gdb) x/wx $aux+36
0xffffe0000f64: 0x00000000
(gdb) set $aux = *(void **)($x19+264)
(gdb) x/wx $aux+36
0xffffe0000f64: 0x00000000
(gdb) x/1ub $x19+32
0xfffff002d1c0: 0
```

Valores observados nesta passagem:

- `*(uint32_t *)(aux+36) == 0`.
- `*(uint8_t *)(x19+32) == 0`.

A primeira leitura de `aux+36` foi feita antes de atualizar `$aux`; a leitura seguinte, após recalcular `$aux` a partir de `[x19+264]`, confirmou o mesmo valor zero no endereço `0xffffe0000f64`.

A interpretação do fluxo permanece a mesma: o `cbz` após carregar `aux+36` desvia para `H264DecodeSliceHeader+1660`, antes de a condição baseada em `slice+32` poder permitir a chamada de `H264CongigureWeightTableRegisters()`. O valor zero em `slice+32` seria compatível com a condição posterior, mas não supera o desvio anterior.

### Contagem de amostras documentadas

Até esta atualização, temos três passagens observadas nesse ponto:

| Amostra | `aux+36` | `slice+32` | Resultado |
|---|---:|---:|---|
| 1 | 0 | 2 | Desvio pelo `cbz`; chamada não alcançada |
| 2 | 0 | 0 | Desvio pelo `cbz`; chamada não alcançada |
| 3 (esta seção) | 0 | 0 | Desvio pelo `cbz`; chamada não alcançada |

**Conclusão:** três passagens consecutivas observadas não alcançaram a chamada de `H264CongigureWeightTableRegisters()`. Isso não demonstra que a função nunca seja chamada; ainda precisamos capturar uma passagem com `aux+36 != 0` e `slice+32` igual a 0 ou 3.

**Próximo passo:** execute `continue` e, em cada parada no breakpoint 21, atualize `$aux` antes de ler os dois campos:

```gdb
set $aux = *(void **)($x19+264)
x/wx $aux+36
x/1ub $x19+32
```

## 14.92 — Quarta amostra em H264DecodeSliceHeader+932

Nova execução do breakpoint 21 em `0xfffff6f0d854`:

```gdb
set $aux = *(void **)($x19+264)
x/wx $aux+36
x/1ub $x19+32
```

Resultados:

- `aux+36 = 0x00000000`
- `slice+32 = 0`

A chamada a `H264CongigureWeightTableRegisters()` permanece inelegível nesta passagem: o `cbz` após carregar `[aux+36]` desvia antes de avaliar a condição sobre `[x19+32]`.

Amostras observadas até agora:

| Amostra | `aux+36` | `slice+32` | Resultado |
|---|---:|---:|---|
| 1 | 0 | 2 | Bloqueada pelo primeiro teste |
| 2 | 0 | 0 | Bloqueada pelo primeiro teste |
| 3 | 0 | 0 | Bloqueada pelo primeiro teste |
| 4 | 0 | 0 | Bloqueada pelo primeiro teste |

Ainda não foi observada uma passagem com `aux+36 != 0`. Para alcançar a chamada, é necessário que esse DWORD seja diferente de zero e que `slice+32` seja 0 ou 3.

## 14.93 — Quinta amostra em H264DecodeSliceHeader+932

Nova execução do breakpoint 21 em `0xfffff6f0d854`:

```gdb
set $aux = *(void **)($x19+264)
x/wx $aux+36
x/1ub $x19+32
```

Resultados informados:

- `aux+36 = 0x00000000`
- `slice+32 = 0`

A chamada a `H264CongigureWeightTableRegisters()` continua inelegível nesta passagem. O `cbz` aplicado ao DWORD `[aux+36]` desvia antes de a condição sobre `[x19+32]` poder permitir a chamada.

Amostras observadas:

| Amostra | `aux+36` | `slice+32` | Resultado |
|---|---:|---:|---|
| 1 | 0 | 2 | Bloqueada pelo primeiro teste |
| 2 | 0 | 0 | Bloqueada pelo primeiro teste |
| 3 | 0 | 0 | Bloqueada pelo primeiro teste |
| 4 | 0 | 0 | Bloqueada pelo primeiro teste |
| 5 | 0 | 0 | Bloqueada pelo primeiro teste |

Ainda não foi observada uma passagem com `aux+36 != 0`. A condição necessária para chegar à chamada continua sendo `aux+36 != 0` e `slice+32` igual a 0 ou 3.

**Próximo passo:** continuar a execução e, em cada parada no breakpoint 21, atualizar `$aux` antes de consultar os campos:

```gdb
continue
set $aux = *(void **)($x19+264)
x/wx $aux+36
x/1ub $x19+32
```

## 14.94 — Repetição da condição em H264DecodeSliceHeader e mudança de estratégia

Nova passagem no breakpoint `0xfffff6f0d854` (`H264DecodeSliceHeader+932`) apresentou novamente:

- `aux = *(void **)(x19+264) = 0xffffe0000f40` (endereço inferido do campo observado nesta passagem).
- DWORD em `aux+36` = `0`.
- byte em `x19+32` = `0`.

A primeira condição (`cbz` após carregar `aux+36`) continua desviando antes da chamada a `H264CongigureWeightTableRegisters()`. Esta amostra repete o estado das anteriores e não constitui evidência nova sobre a origem do campo.

**Mudança metodológica:** interromper a amostragem repetitiva no breakpoint de `H264DecodeSliceHeader+932` e instalar um watchpoint de escrita sobre `*(uint32_t *)(aux+36)`, desabilitando o breakpoint 21. Objetivo: capturar a instrução e a pilha que alterem o valor do campo, caso ele seja escrito durante a execução restante. Se não houver escrita após a instalação, será necessário rastrear sua inicialização em uma execução desde o início ou localizar estaticamente os acessos ao deslocamento `+36`.

Comandos propostos no GDB, a partir da parada atual:

```gdb
watch -l *(unsigned int *)($aux+36)
disable 21
continue
```

Aguardar uma eventual parada do watchpoint. Não repetir a leitura manual no breakpoint antigo.

## 14.95 — Watchpoint em `aux+36` não disparou antes do encerramento

Foi instalado um watchpoint de hardware de escrita no endereço atual `aux+36`:

```gdb
watch -l *(unsigned int *)($aux+36)
disable 21
continue
```

O GDB aceitou o watchpoint 23. Após continuar, **não houve parada no watchpoint**. O processo `vdecoderdemo` encerrou normalmente e as threads de parser/display terminaram. O log final informou `decode frame: 32`, `display frame: 31`, seguido da saída normal do inferior.

### Conclusão estrita

- Não foi observada nenhuma escrita no endereço monitorado entre a instalação do watchpoint e o encerramento do processo.
- Isso não prova que o campo nunca seja escrito em outras execuções, nem que todos os objetos apontados por `x19+264` usem o mesmo endereço.
- O watchpoint monitorou o endereço concreto de `aux+36` capturado na parada anterior; não acompanha automaticamente alterações posteriores do ponteiro `aux` para outro objeto.
- O encerramento normal após 32 frames confirma que o teste de decodificação chegou ao fim; não demonstra, por si só, a função de `aux+36` nem o caminho de submissão ao hardware.

### Próxima estratégia

Não repetir a sequência de amostragem em `H264DecodeSliceHeader+932`. Na próxima execução, investigar a origem e os escritores do campo:

1. Capturar o ponteiro `aux` desde a criação/inicialização da estrutura, antes do processamento das slices.
2. Identificar estaticamente as instruções que escrevem no deslocamento `+36` da estrutura apontada por `x19+264` (considerando que o ponteiro pode variar entre chamadas).
3. Se necessário, instalar o watchpoint logo após a estrutura ser inicializada, antes de começar a decodificação, e confirmar que o endereço monitorado continua sendo o objeto relevante.

O resultado atual é inconclusivo sobre a origem do campo, mas elimina a expectativa de que ele seria alterado mais adiante no restante desta execução específica.

## 14.96 — Watchpoint em `aux+36` não disparou; mudança de estratégia

### Resultado observado

Na execução monitorada, foi instalado um hardware watchpoint sobre:

```gdb
watch -l *(unsigned int *)($aux+36)
```

O breakpoint 21 em `H264DecodeSliceHeader+932` foi desabilitado e a execução continuou. O watchpoint 23 não disparou antes do encerramento normal do processo. O log final informou:

```text
demoDecoder finish.decode frame: 32, display frame: 31
[Inferior 1 (process 44176) exited normally]
```

### Interpretação e limite

- Não foi observada escrita no endereço monitorado durante o intervalo em que o watchpoint esteve ativo.
- Isso não prova que o campo nunca seja escrito: o watchpoint foi instalado depois da inicialização da estrutura e monitorava um endereço fixo. Se o ponteiro `aux` mudar em outra chamada, o novo endereço não estará coberto.
- As amostras anteriores mostraram repetidamente `aux+36 == 0`; não há justificativa para continuar repetindo a leitura no mesmo breakpoint.

### Nova estratégia

Na próxima execução, voltar ao breakpoint simbólico no início de `H264DecodeSliceHeader`, não ao endereço absoluto antigo. Inspecionar o prólogo da função e identificar quando o registrador que origina `x19` é preservado e como o campo `[x19+264]` é obtido. A partir daí, rastrear a criação/atribuição do ponteiro `aux` e localizar a inicialização do campo `aux+36` antes de decidir onde instalar um watchpoint antecipado.

Comando inicial proposto após reiniciar o processo:

```gdb
 delete 21
 set breakpoint pending on
 break H264DecodeSliceHeader
 run
```

Ao parar na entrada simbólica, obter o prólogo sem executar passos repetitivos:

```gdb
 disassemble /r H264DecodeSliceHeader, H264DecodeSliceHeader+120
 info registers x0 x1 x19 x20
```

A análise deve distinguir fatos observados de inferências. O não disparo do watchpoint não identifica, por si só, o escritor de `aux+36`.

## 14.97 — Origem da zeragem de `aux+36`: inicialização em `H264DecodePps`

Na execução reiniciada, o watchpoint 23 sobre o endereço anteriormente observado como `aux+36` disparou antes de o breakpoint 24 em `H264DecodeSliceHeader` ser atingido. A parada ocorreu dentro da libc:

```text
Hardware watchpoint 23: -location *(unsigned int *)($aux+36)
Old value = <unreadable>
New value = 0
PC = 0xfffff7c442f4 (libc.so.6)
```

A pilha capturada nessa interrupção foi:

```text
#0  0xfffff7c442f4  libc.so.6
#1  0xfffff6f0bdf4  H264DecodePps() [libawh264.so]
#2  0xfffff6f1575c  H264ProcessNaluUnit() [libawh264.so]
#3  0xfffff6f108b4  H264DecoderDecodeSbmFrame() [libawh264.so]
#4  0xfffff7d63528  VideoEngineDecode() [libvideoengine.so]
#5  DecodeVideoStream() [vdecoder.c:1285]
#6  DecodeThread()
```

No PC da interrupção, a desmontagem mostra instruções de armazenamento vetorial (`stp q0, ...`) da rotina de preenchimento da libc. Os registradores eram:

```text
x0 = 0xffffe0000f40
x1 = 0
x2 = 0x120
```

Isso corresponde à operação de zerar 0x120 bytes (288 bytes) a partir de `0xffffe0000f40`. O endereço observado pelo watchpoint era `0xffffe0000f64`, isto é, início do bloco + 0x24 (36 bytes). Logo, o DWORD em `aux+36` foi zerado como parte da inicialização integral desse bloco por `H264DecodePps()`.

### Conclusão comprovada

- A escrita que disparou o watchpoint ocorreu durante `H264DecodePps()`.
- A operação observada preencheu com zero 288 bytes a partir do endereço monitorado como início de `aux`.
- O deslocamento `+36` está dentro dessa região e, portanto, recebeu zero nessa inicialização.
- A sequência `H264ProcessNaluUnit -> H264DecoderDecodeSbmFrame -> VideoEngineDecode -> DecodeVideoStream` situa essa inicialização no processamento do NAL/PPS, antes do processamento posterior das slices.

### O que ainda não está determinado

- O significado semântico do campo de 32 bits no deslocamento `+36`.
- Se esse campo é posteriormente preenchido em outro caminho de PPS/slice ou permanece zero durante todo o fluxo testado.
- Se o objeto de 288 bytes corresponde exatamente à estrutura apontada por `[x19+264]` em todas as chamadas; nesta execução, a coincidência de endereço e o watchpoint sustentam essa associação para o objeto observado.

### Próximo passo: identificar o local exato da chamada de zeragem

A pilha fornece o endereço de retorno em `H264DecodePps`: `0xfffff6f0bdf4`. Inspecionar as instruções imediatamente anteriores para localizar a preparação dos argumentos e a chamada à rotina de memória:

```gdb
x/20i 0xfffff6f0bdd4
```

A partir dessa sequência, identificar de onde veio o ponteiro do bloco e se o objeto é alocado/associado ao contexto PPS. Não atribuir ainda um nome funcional ao campo `+36` sem examinar seus demais leitores e escritores.

---

## 14.98 — Alocação e inicialização do objeto auxiliar em `H264DecodePps`

### Evidência GDB

O trecho fornecido pelo usuário em `H264DecodePps+1744` a `+1784`:

```asm
0xfffff6f0bdd4: mov w0, #0x120
0xfffff6f0bdd8: str x3, [sp, #128]
0xfffff6f0bddc: bl  cdc_malloc@plt
0xfffff6f0bde0: ldp x3, x23, [sp, #128]
0xfffff6f0bde4: mov x2, #0x120
0xfffff6f0bde8: mov w1, #0
0xfffff6f0bdec: str x0, [x3, x23, lsl #3]
0xfffff6f0bdf0: bl  memset@plt
0xfffff6f0bdf4: ldr x3, [sp, #128]
0xfffff6f0bdf8: ldr x23, [x3, x23, lsl #3]
```

### Interpretação comprovada

1. `H264DecodePps` solicita `0x120` bytes (288 bytes) a `cdc_malloc`.
2. O ponteiro retornado em `x0` é armazenado em um vetor de ponteiros no endereço `x3 + (x23 * 8)`, ou seja, índice `x23` de uma tabela com entradas de 8 bytes.
3. Os argumentos de `memset` são preparados como `x0` = ponteiro recém-alocado, `x1` = 0 e `x2` = `0x120`; portanto, o bloco recém-alocado é integralmente zerado.
4. O endereço observado no watchpoint (`0xffffe0000f40`) é o ponteiro passado a `memset`; o endereço monitorado (`0xffffe0000f64`) é o deslocamento `+0x24` dentro desse bloco.

Isso explica diretamente por que o DWORD em `aux+36` recebeu zero: ele faz parte de um objeto de 288 bytes recém-alocado e inicializado com zero durante `H264DecodePps`.

### Limites da conclusão

O trecho demonstra a alocação e o armazenamento do ponteiro em uma tabela indexada por `x23`, mas ainda não revela, isoladamente, o significado semântico do objeto ou do campo `+36`. Também não demonstra se o mesmo objeto é o apontado por `[parser_context+264]` em todos os casos; essa associação deve ser confirmada seguindo o vetor/tabela e os leitores posteriores.

### Próxima inspeção

Inspecionar o contexto do frame `H264DecodePps` e as instruções que calculam `x3` e `x23`, para identificar a tabela e o índice PPS:

```gdb
frame 1
info registers x3 x20 x23 x28
x/40i 0xfffff6f0bd00
```

Se os registradores não forem recuperados pelo GDB no frame selecionado, usar a desassemblagem para trás e localizar as atribuições de `x3` e `x23`; não inferir o nome da estrutura apenas pelo tamanho de 288 bytes.

---

## 14.99 — Contexto da tabela de PPS em `H264DecodePps` (22/11/2026)

### Novas observações do GDB

Após parar no `memset` chamado por `H264DecodePps`, o usuário selecionou o frame 1 e observou:

```text
x3  = 0xffffe0000f40
x20 = 0xfffff002d1a0
x23 = 0
x28 = 0xfffff0028070
```

A desassemblagem de `H264DecodePps+1532` até `+1816` mostrou, entre outros, caminhos de diagnóstico e o caminho de alocação:

```asm
H264DecodePps+1744: mov  w0, #0x120
H264DecodePps+1748: str  x3, [sp, #128]
H264DecodePps+1752: bl   cdc_malloc@plt
H264DecodePps+1756: ldp  x3, x23, [sp, #128]
H264DecodePps+1760: mov  x2, #0x120
H264DecodePps+1764: mov  w1, #0
H264DecodePps+1768: str  x0, [x3, x23, lsl #3]
H264DecodePps+1772: bl   memset@plt
H264DecodePps+1780: ldr  x23, [x3, x23, lsl #3]
```

### Interpretação e ressalvas

- `x23` é um registrador preservado pelo ABI AArch64 (callee-saved); seu valor observado como zero após a chamada à libc é compatível com o índice zero da tabela usada no armazenamento `[x3 + x23*8]`.
- `x20` também é callee-saved e continua apontando para o contexto `0xfffff002d1a0` observado anteriormente como contexto de parser H.264.
- `x3` é caller-saved. Seu valor observado durante a execução interna de `memset` corresponde ao destino do preenchimento e **não deve ser interpretado como a base da tabela**. A base da tabela foi preservada temporariamente em `[sp+128]` e restaurada no `ldp` anterior ao `memset`.
- `x28` é preservado, mas o trecho disponível ainda não permite atribuir-lhe um papel semântico seguro.
- A desassemblagem `0xfffff6f0bd00`–`0xfffff6f0bd9c` mostra caminhos anteriores de controle/diagnóstico; não basta, por si só, para identificar a tabela. É necessário examinar as instruções imediatamente anteriores à preparação da alocação para localizar a origem de `x3` e `x23`.

### Conclusão atualizada

Está comprovado que `H264DecodePps` aloca um bloco de `0x120` bytes, grava o ponteiro em uma tabela indexada por `x23` e zera o bloco. O índice observado é zero. A identidade/base da tabela e o significado dos campos do objeto continuam em aberto. Não se deve chamar o objeto de PPS específico ou atribuir semântica a `+36` sem rastrear os usos e a origem da tabela.

### Próxima inspeção

Para revelar como `x3` e `x23` são preparados no caminho de alocação, inspecionar a região anterior à chamada de `cdc_malloc`:

```gdb
x/55i 0xfffff6f0bcf0
```

---

## 14.100 — Recuperação da base da tabela de PPS (22/11/2026)

### Evidência observada

O watchpoint em `0xffffe0000f64` disparou durante `H264DecodePps()`. O backtrace mostrou a rotina de memória da libc chamada diretamente por `H264DecodePps()` (`libawh264.so`). No instante da interrupção:

- `x0 = 0xffffe0000f40`: destino do preenchimento;
- `x1 = 0`: byte de preenchimento;
- `x2 = 0x120`: tamanho de 288 bytes;
- endereço monitorado `0xffffe0000f64 = destino + 0x24`.

A sequência em `H264DecodePps+1744` é:

```asm
mov w0, #0x120
str x3, [sp, #128]
bl cdc_malloc@plt
ldp x3, x23, [sp, #128]
mov x2, #0x120
mov w1, #0
str x0, [x3, x23, lsl #3]
bl memset@plt
```

Isto comprova que a função aloca 0x120 bytes, preserva temporariamente a base da tabela em `[sp+128]` e o índice em `[sp+136]` (par de registradores `x3`, `x23`), grava o ponteiro alocado na entrada `x3 + 8*x23` e zera a estrutura.

### Limites da conclusão

O valor `x23=0` observado no frame da libc é compatível com índice zero, pois `x23` é callee-saved no ABI AArch64. Entretanto, o valor atual de `x3` dentro da libc não identifica a base da tabela: `x3` é caller-saved e foi alterado pela rotina de memória. O endereço da tabela deve ser lido do slot salvo na pilha do frame `H264DecodePps`, `[sp+128]` (com o `sp` correspondente a esse frame), e a origem do índice deve ser confirmada no código imediatamente anterior a `H264DecodePps+1728`.

O trecho `H264DecodePps+1516` a `+1724` exibido pelo usuário contém caminhos de diagnóstico e tratamento de bytes, mas não mostra a preparação da alocação; o dump terminou em `+1728`, justamente antes dela.

### Próxima investigação

Inspecionar as instruções a partir de `H264DecodePps+1728` e recuperar os valores salvos no frame 1 com `info frame` e leitura de `[sp+128]` e `[sp+136]`. Não inferir o significado semântico da estrutura de 288 bytes apenas pelo tamanho ou pelo offset 36.

## 14.101 — H264DecodePps: identificação da tabela de ponteiros da alocação de 0x120 bytes

Na interrupção do watchpoint em `memset`, o frame 1 era `H264DecodePps` e a sequência em `H264DecodePps+1728..+1784` mostrou:

```asm
+1728 adrp x1, ...
+1732 add  x1, x1, #0x480
+1736 add  x1, x1, #0x128
+1744 mov  w0, #0x120
+1748 str  x3, [sp, #128]
+1752 bl   cdc_malloc@plt
+1756 ldp  x3, x23, [sp, #128]
+1760 mov  x2, #0x120
+1764 mov  w1, #0
+1768 str  x0, [x3, x23, lsl #3]
+1772 bl   memset@plt
+1776 ldr  x3, [sp, #128]
+1780 ldr  x23, [x3, x23, lsl #3]
```

O dump da pilha em `x/2gx $sp+128` retornou:

- base da tabela salva: `0xfffff002d3f8`;
- índice salvo (`x23`): `0`.

O registrador `x20` vale `0xfffff002d1a0`, portanto a base da tabela é `x20 + 0x258`. A alocação de 288 bytes é armazenada no elemento zero dessa tabela, no endereço `0xfffff002d3f8`. A chamada seguinte zera o bloco recém-alocado.

**Conclusão observada:** a alocação de 0x120 bytes é registrada numa tabela de ponteiros embutida no contexto apontado por `x20`, no deslocamento `0x258`, usando índice zero nesta execução. Isso identifica a tabela de destino, mas ainda não prova a relação entre esse elemento e o ponteiro lido por `H264DecodeSliceHeader` em `[x19+264]` (deslocamento `0x108`). Essa relação deve ser verificada diretamente na memória antes de atribuir significado ao campo `+36`.

## 14.102 — Comparação do campo `contexto+0x108` com a tabela em `contexto+0x258`

O usuário examinou `x/8gx 0xfffff002d2a8`. Como o contexto em questão é `0xfffff002d1a0`, o endereço `0xfffff002d2a8` corresponde exatamente a `contexto+0x108`, campo lido por `H264DecodeSliceHeader` com `ldr x0, [x19,#264]`.

Conteúdo observado:

```text
0xfffff002d2a8: 0x0000000000000000  0x0000000000000101
0xfffff002d2b8: 0x0000000000000000  0x0000000000000000
0xfffff002d2c8: 0x0000000000000000  0x0000000000000000
0xfffff002d2d8: 0x0000000000000000  0x0000000000000000
```

**Conclusão observada:** no instante dessa leitura, o ponteiro em `contexto+0x108` é nulo. O valor no campo seguinte (`contexto+0x110`) é `0x101`. Isso não estabelece o significado desse segundo campo.

A tabela identificada em `contexto+0x258` (base `0xfffff002d3f8`) é uma região distinta. Ainda não há evidência suficiente para afirmar que o objeto de 0x120 bytes alocado por `H264DecodePps` seja o objeto que `H264DecodeSliceHeader` espera encontrar em `contexto+0x108`. Também não se deve assumir que as observações feitas em instantes diferentes representam o mesmo estado de inicialização.

### Próximo passo de depuração

Inspecionar o conteúdo da tabela e a região em torno do campo `+0x108` no mesmo estado de execução, sem avançar o processo:

```gdb
x/8gx 0xfffff002d3f8
x/16gx 0xfffff002d2a8
```

A primeira leitura deve confirmar o ponteiro atualmente armazenado no elemento zero da tabela. A segunda amplia a vizinhança do campo `+0x108`. Se o ponteiro em `+0x108` continuar nulo, será necessário localizar a instrução que o preenche antes de `H264DecodeSliceHeader` tentar desreferenciá-lo. Até lá, a relação entre a alocação de `H264DecodePps` e o campo `+0x108` permanece **não demonstrada**.

## 14.103 — Confirmação da tabela de PPS e distinção do campo usado pelo slice header

Na execução interrompida dentro de `H264DecodePps`, o contexto preservado em `x20` é `0xfffff002d1a0`. A base salva da tabela é `0xfffff002d3f8`, correspondente a `contexto + 0x258`. O índice salvo em `x23` é zero.

Leitura de `0xfffff002d3f8`:

- elemento `[0]` = `0xffffe0000f40`, ponteiro alocado por `cdc_malloc(0x120)` e zerado por `memset`;
- elemento `[1]` e seguintes = zero no instante da observação.

Leitura de `contexto + 0x108` (`0xfffff002d2a8`): primeiro qword = zero. Portanto, no instante observado em `H264DecodePps`, o campo `+0x108` está nulo, enquanto a tabela em `+0x258` já contém o ponteiro da alocação de 288 bytes.

A região `contexto + 0x150` (`0xfffff002d2f0`) contém, no segundo qword exibido (`0xfffff002d2f8`), o ponteiro `0xffffe0000b70`; seu significado ainda não foi identificado.

Conclusão: a alocação de 288 bytes criada por `H264DecodePps` está armazenada em `contexto+0x258` (elemento zero da tabela). Não há evidência de que seja o objeto acessado por `H264DecodeSliceHeader` através de `contexto+0x108`; este último está nulo neste ponto do fluxo. A relação temporal/funcional entre os campos permanece desconhecida.

Próximo experimento: instalar um watchpoint de escrita no qword `contexto+0x108` (`0xfffff002d2a8`) e continuar a execução para capturar a função que o inicializa, caso seja escrito. Comando GDB sugerido:

`watch -l *(unsigned long long *)0xfffff002d2a8`

Depois, `continue`; ao disparar, coletar `bt`, `info registers x0 x1 x2 x3 x19 x20 x29 x30` e `x/8i $pc-12`. Se o watchpoint não disparar até o fim do teste, registrar apenas que o campo permaneceu nulo durante esse percurso, sem generalizar para outros fluxos.

## 14.104 — Watchpoint confirma a origem de `parser_context+0x108`

O usuário instalou um watchpoint de escrita em `0xfffff002d2a8`, endereço correspondente a `parser_context+264` (`+0x108`), e a execução parou em `H264DecodePictureScanType`:

```text
Hardware watchpoint 25
Old value = 0
New value = 281474439843648  (0xffffe0000f40)
PC = 0xfffff6f0e5b4
```

Backtrace observado:

```text
#0 H264DecodePictureScanType
#1 H264DecoderDecodeSbmFrame
#2 VideoEngineDecode
#3 DecodeVideoStream
#4 DecodeThread
```

Registradores no ponto da interrupção:

```text
x0  = 0xfffff000cc80
x1  = 0xffffe0000f40
x2  = 0
x3  = 0xfffff6f06e14
x19 = 0xfffff000cc80
x20 = 0xfffff002d1a0
```

Instruções ao redor da escrita:

```asm
0xfffff6f0e5a8: mov x0, x19
0xfffff6f0e5ac: ldr x3, [x19, #448]
0xfffff6f0e5b0: str x1, [x20, #264]
0xfffff6f0e5b4: add x1, x20, x2, lsl #3
0xfffff6f0e5b8: ldr x1, [x1, #344]
0xfffff6f0e5bc: str x1, [x20, #256]
0xfffff6f0e5c0: ldr w1, [x1, #64]
0xfffff6f0e5c4: blr x3
```

**Conclusão diretamente demonstrada:** `H264DecodePictureScanType` grava o ponteiro recebido em `x1` no campo `parser_context+0x108`. O valor gravado (`0xffffe0000f40`) é exatamente o ponteiro que `H264DecodePps` alocou com `cdc_malloc(0x120)`, zerou com `memset` e armazenou no elemento zero da tabela em `parser_context+0x258`. Assim, a relação entre a alocação PPS e o campo usado por `H264DecodeSliceHeader` está confirmada nesta execução.

O `x2=0` também seleciona o elemento zero de outra tabela/região: `ldr x1,[x20,#344]` lê `parser_context+0x158` neste caso, e o resultado é armazenado em `parser_context+0x100` (`+256`). Esse ponteiro é distinto do PPS em `+0x108`; seu significado ainda não foi identificado. O registrador `x3` contém o endereço da rotina de callback `H264GetBitsSw` (endereço observado `0xfffff6f06e14`), chamada após a leitura do DWORD em `[x1+64]`.

### Consequência para o campo PPS+36

A cadeia de ponteiros agora está estabelecida:

```text
H264DecodePps
  └─ cdc_malloc(0x120)
      └─ memset(ponteiro, 0, 0x120)
          └─ parser_context+0x258[0] = ponteiro

H264DecodePictureScanType
  └─ parser_context+0x108 = parser_context+0x258[0]

H264DecodeSliceHeader
  └─ pps = *(parser_context+0x108)
      └─ consulta DWORD em pps+36
```

A inicialização por `memset` explica o valor zero inicial de `PPS+36`, mas ainda não demonstra se, nem quando, outro caminho o altera. Também não identifica o significado semântico do DWORD em `+36`.

### Próximo passo de depuração

Sem avançar a execução ainda, examinar a função e o objeto PPS no estado atual:

```gdb
disassemble /r H264DecodePictureScanType
x/24wx 0xffffe0000f40
```

A desmontagem completa permitirá identificar de onde vem o argumento `x1` e como a função seleciona a entrada PPS. A leitura do objeto permitirá confirmar os DWORDs iniciais, inclusive `PPS+36` (décimo DWORD, offset `0x24`). Depois disso, a investigação deve procurar escritas posteriores especificamente em `PPS+36`, em vez de continuar monitorando o ponteiro inteiro ou repetir leituras do mesmo campo.

## 14.102 — Verificação do campo PPS+0x24 imediatamente antes do memcpy (22/11/2026)

### Evidência observada no GDB

Breakpoint temporário em `H264DecodePps+0x470` (endereço `0xfffff6f0bb74`), imediatamente antes da chamada a `memcpy`, com os seguintes registradores:

```text
x0  = 0xffffe0000f40
x1  = 0xfffff54cd600
x2  = 0x120 (288)
x23 = 0xffffe0000f40
x25 = 0xfffff54cd600
```

Leitura dos dois objetos no offset `+36` (`+0x24`):

```text
[x25+36] = 0x00000000   # objeto temporário, origem do memcpy
[x23+36] = 0x00000000   # objeto PPS de destino
```

A instrução de cópia usa `x0=x23`, `x1=x25` e `x2=0x120`; portanto, nesse ponto, o campo `+0x24` do objeto temporário já está zerado e o campo correspondente do objeto PPS de destino também está zerado. O `memcpy` copiará 288 bytes da origem para o destino.

### Conclusão sustentada

Para este PPS e esta execução, o campo `PPS+0x24` chega a `memcpy` com valor zero na origem. Isso confirma que o zero observado anteriormente no objeto de destino não é apenas um artefato da alocação do destino: a estrutura temporária que será copiada também contém zero nesse offset imediatamente antes da cópia.

Isso **não** identifica o significado semântico do campo, nem prova que ele será sempre zero para outros PPS/bitstreams. Também não identifica qual instrução ou helper deixou o campo zerado: ele pode ter permanecido no estado inicial de `memset` ou ter sido explicitamente escrito com zero.

### Próximo passo de maior valor

Para descobrir se algum trecho da função escreve nesse DWORD, repetir a execução colocando um hardware watchpoint em `x25+0x24` logo após o `memset` do objeto temporário e antes do parsing dos campos do PPS. O watchpoint deve ser instalado usando o endereço real do objeto temporário daquela execução; quando disparar, registrar PC, backtrace e instrução responsável. Se não disparar até o `memcpy`, fica demonstrado que o campo permaneceu zero desde a inicialização nessa amostra.

## 14.105 — Localização exata do memset da estrutura PPS temporária (22/11/2026)

### Evidência observada no GDB

O breakpoint 27 foi atingido no início de `H264DecodePps`, no endereço `0xfffff6f0b708` (`H264DecodePps+4`). A desmontagem de 100 instruções identificou a preparação da estrutura temporária e a chamada ao `memset`:

```asm
0xfffff6f0b84c <H264DecodePps+328>: add x25, sp, #0x90
0xfffff6f0b850 <H264DecodePps+332>: blr x1
0xfffff6f0b854 <H264DecodePps+336>: mov w23, w0
0xfffff6f0b858 <H264DecodePps+340>: mov x2, #0x120       // 288 bytes
0xfffff6f0b85c <H264DecodePps+344>: mov w1, #0
0xfffff6f0b860 <H264DecodePps+348>: mov x0, x25
0xfffff6f0b864 <H264DecodePps+352>: bl memset@plt
0xfffff6f0b868 <H264DecodePps+356>: mov w2, w19
```

`x25` recebe o endereço da estrutura temporária na pilha (`sp+0x90`). O `memset` recebe `x0=x25`, `x1=0` e `x2=0x120`, portanto zera exatamente 288 bytes. A primeira instrução após o retorno é `0xfffff6f0b868`.

### Conclusão sustentada

O ponto seguro para instalar um watchpoint no DWORD `PPS+0x24` da estrutura temporária é imediatamente após o `memset`, em `0xfffff6f0b868`. Nesse ponto, `x25` ainda identifica a estrutura temporária e o campo `x25+36` acaba de ser zerado, antes das chamadas subsequentes de parsing (`H264GetVLCSymbol` e outros helpers).

### Próximo passo

Na execução atual, instalar um breakpoint temporário na instrução após o `memset` e continuar até ele:

```gdb
tbreak *0xfffff6f0b868
continue
```

Ao parar, conferir o endereço e o valor inicial, sem continuar a execução:

```gdb
p/x $x25
x/wx $x25+36
```

Com esses valores confirmados, instalar `watch -l *(unsigned int *)($x25+36)` e continuar para capturar a primeira escrita posterior no campo. Ainda não há evidência de escrita nesse DWORD após o `memset`, nem de seu significado semântico.

## 14.106 — Confirmação do endereço e do valor inicial de PPS temporário+0x24 (22/11/2026)

### Evidência observada no GDB

Após o breakpoint temporário na instrução imediatamente posterior ao `memset` (`0xfffff6f0b868`), o GDB parou em `H264DecodePps` e retornou:

```text
$ x25 = 0xfffff54cd600
0xfffff54cd624: 0x00000000
```

Assim, nesta execução, `x25` aponta para a estrutura temporária na pilha e `x25+36` (`x25+0x24`, endereço `0xfffff54cd624`) contém zero imediatamente após o `memset` de 288 bytes.

### Conclusão sustentada

A localização dinâmica do DWORD monitorado foi confirmada. O valor inicial é zero antes de continuar o parsing do PPS. Isso estabelece o estado de referência para o watchpoint, mas ainda não informa se o campo será escrito posteriormente.

### Próximo passo

Instalar um watchpoint de escrita no endereço confirmado e verificar sua ativação antes de prosseguir:

```gdb
watch -l *(unsigned int *)($x25+36)
info breakpoints
continue
```

Se houver disparo, registrar PC, registradores, backtrace e instrução responsável. Se não houver disparo até o fim de `H264DecodePps`/cópia para o PPS de destino, o campo terá permanecido zero nessa amostra desde a inicialização observada.

## 14.107 — Primeira escrita observada em PPS temporário+0x24 (22/11/2026)

### Evidência observada no GDB

Após instalar o hardware watchpoint 32 no endereço dinâmico `x25+36` (`0xfffff54cd624`), a execução parou com:

```text
Thread 5 "vdecoderdemo" hit Hardware watchpoint 32:
  -location *(unsigned int *)($x25+36)
Old value = 0
New value = 65535
0x0000fffff6f09218 in H264DecodeRefPicListReordering ()
  from /usr/lib/aarch64-linux-gnu/libawh264.so
```

O watchpoint foi criado como hardware watchpoint e estava habilitado. O DWORD monitorado mudou de `0x00000000` para `0x0000ffff`. O PC reportado pelo GDB no evento é `0xfffff6f09218`, dentro de `H264DecodeRefPicListReordering`.

### Conclusão sustentada

O campo no offset `+0x24` da estrutura temporária PPS não permanece no zero inicial: durante a execução monitorada, houve uma escrita que alterou seu valor para `65535` (`0x0000ffff`). O evento está associado à execução de `H264DecodeRefPicListReordering`. Isso refuta a hipótese anterior de que o campo permaneceria necessariamente zero desde o `memset` nesta amostra.

Ainda não se deve atribuir significado semântico ao valor `0xffff`, nem afirmar que a rotina inteira é sua única origem sem examinar a instrução de escrita e o contexto. O PC reportado pelo GDB precisa ser correlacionado com as instruções anteriores e posteriores para determinar a instrução exata que disparou o watchpoint, os registradores usados e o endereço efetivamente escrito.

### Próximo passo

Com a execução ainda parada no watchpoint, coletar:

```gdb
info registers
bt
x/12i $pc-24
x/wx 0xfffff54cd624
```

A desmontagem ao redor do PC e os registradores permitirão identificar a instrução que alterou o campo e os argumentos/estado de `H264DecodeRefPicListReordering`. Não executar `continue` antes de capturar esses dados.

## 14.108 — Correção: disparo do watchpoint causado pela reutilização da pilha (22/11/2026)

### Evidência adicional do GDB

A coleta completa no evento do hardware watchpoint 32 mostrou:

```text
sp  = 0xfffff54cd600
x21 = 0xfffff003b1a0
x22 = 0x0
pc  = 0xfffff6f09218 <H264DecodeRefPicListReordering+24>
```

O backtrace foi:

```text
#0 H264DecodeRefPicListReordering
#1 H264DecodeSliceHeader
#2 H264ProcessNaluUnit
#3 H264DecoderDecodeSbmFrame
#4 VideoEngineDecode
#5 DecodeVideoStream
```

A desmontagem no início da função mostrou:

```asm
0xfffff6f09200 <H264DecodeRefPicListReordering>: stp x29, x30, [sp, #-144]!
0xfffff6f09204 <+4>:  mov x29, sp
0xfffff6f09208 <+8>:  stp x23, x24, [sp, #48]
0xfffff6f0920c <+12>: mov x23, x1
0xfffff6f09210 <+16>: mov w1, #0xfd
0xfffff6f09214 <+20>: stp x21, x22, [sp, #32]
0xfffff6f09218 <+24>: mov x21, x0
```

O endereço observado pelo watchpoint era `0xfffff54cd624`. No momento da parada, `sp=0xfffff54cd600`, portanto o endereço monitorado coincide com `sp+36`. A instrução `stp x21, x22, [sp,#32]` grava `x21` nos bytes `sp+32..sp+39` e `x22` nos bytes `sp+40..sp+47`. Como `x21=0xfffff003b1a0`, os quatro bytes superiores dessa palavra de 64 bits são `0x0000ffff`, precisamente o novo valor que o watchpoint reportou em `sp+36`.

### Correção da interpretação anterior

A estrutura temporária PPS havia sido observada em `0xfffff54cd600`, com o campo `+36` em `0xfffff54cd624`. Esse endereço era uma área temporária da pilha de `H264DecodePps`. O watchpoint foi mantido após o fim da vida útil dessa estrutura; posteriormente, a mesma região da pilha foi reutilizada como área local de `H264DecodeRefPicListReordering`.

Assim, o evento `Old value=0 / New value=65535` é explicado pelo salvamento do registrador `x21` no prólogo da função, não por uma escrita semântica no campo PPS. O PC reportado (`+24`) aparece imediatamente após a instrução `stp` em `+20`, consistente com a parada do watchpoint após a gravação.

**A conclusão da seção 14.107 fica expressamente revogada:** o disparo não demonstra que `H264DecodeRefPicListReordering` alterou `PPS+0x24`, nem que o campo PPS assumiu `0xffff`. A única observação válida até aqui é que o campo estava zerado imediatamente após o `memset` em `H264DecodePps`. Ainda não foi capturada nenhuma escrita válida nesse campo enquanto a estrutura temporária estava viva.

### Próximo passo

Repetir a observação com um watchpoint cujo ciclo de vida esteja limitado à execução de `H264DecodePps`: instalar o watchpoint após o `memset`, mas removê-lo/desabilitá-lo antes de a função retornar e antes de a pilha ser reutilizada. Alternativamente, usar breakpoint na instrução imediatamente posterior ao `memcpy` (`H264DecodePps+1136`, endereço a confirmar na execução) para verificar o valor copiado ao objeto PPS de destino. Não interpretar watchpoints em endereços de pilha após o retorno da função que alocou o objeto temporário.

## 14.109 — Confirmação do salvamento de x21 sobre o endereço monitorado (22/11/2026)

### Saída adicional do GDB

O usuário confirmou o conteúdo da pilha:

```text
(gdb) x/wx 0xfffff54cd624
0xfffff54cd624: 0x0000ffff
(gdb) x/2gx $sp+32
0xfffff54cd620: 0x0000fffff003b1a0  0x0000000000000000
(gdb) disable 32
```

O DWORD em `sp+36` corresponde aos bytes superiores da palavra `x21` salva por `stp x21, x22, [sp,#32]`. O valor observado `0x0000ffff` é, portanto, explicado integralmente pelo salvamento do registrador `x21` na pilha reutilizada. O watchpoint 32 foi desabilitado.

### Conclusão consolidada

Não houve evidência de escrita semântica em `PPS+0x24`. O watchpoint atravessou o fim da vida útil da estrutura temporária local de `H264DecodePps`; quando `H264DecodeRefPicListReordering` iniciou, seu prólogo reutilizou a mesma região de pilha e sobrescreveu o endereço monitorado. A interpretação de `0xffff` como valor do campo PPS está descartada.

O campo temporário foi observado como zero logo após o `memset`; o objeto PPS de destino também foi observado como zero no instante posterior ao `memcpy` (evidência anterior). Ainda não há captura de alteração desse campo enquanto a estrutura temporária está viva.

### Próximo passo de verificação

Com o GDB ainda parado em `H264DecodeRefPicListReordering`, o registrador `x2` contém o ponteiro do objeto PPS de destino `0xffffe0000f40`, conforme a captura de registradores. Ler o campo no objeto persistente, que não depende da validade da pilha temporária:

```gdb
x/wx $x2+36
```

O resultado esperado, conforme a observação imediatamente posterior ao `memcpy`, é `0x00000000`. Essa leitura deve ser feita antes de avançar a execução.

## 14.110 — Confirmação de PPS+0x24 no objeto persistente após o falso positivo do watchpoint (22/11/2026)

### Saída adicional do GDB

Após confirmar que o hardware watchpoint 32 havia observado a reutilização da pilha pelo prólogo de `H264DecodeRefPicListReordering`, o usuário desabilitou o watchpoint e consultou o campo no objeto PPS de destino apontado por `x2`:

```text
(gdb) x/wx $x2+36
0xffffe0000f64: 0x00000000
```

Na captura anterior, `x2=0xffffe0000f40`; portanto, a leitura corresponde a `*(uint32_t *)(0xffffe0000f40 + 0x24)`, isto é, `PPS_destino+0x24`.

### Conclusão sustentada

1. O campo `+0x24` do objeto PPS persistente de destino está em `0` no momento da leitura, durante o processamento do slice.
2. Isso é consistente com a observação anterior de que o campo estava em `0` no objeto temporário imediatamente após o `memset` e também no destino no instante posterior ao `memcpy`.
3. O valor `0x0000ffff` observado pelo watchpoint 32 não pertenceu ao campo PPS persistente: foi produzido pelo salvamento de `x21` na pilha reutilizada por `H264DecodeRefPicListReordering`.
4. Ainda não foi observada uma escrita válida em `PPS+0x24` enquanto a estrutura temporária estava viva. Também não foi identificado o significado semântico desse campo.

### Próximo passo

Para avançar sem repetir o falso positivo, a próxima captura deve monitorar o objeto PPS persistente (`0xffffe0000f40+36`) durante a vida útil do objeto, ou instalar um watchpoint temporário na estrutura local e removê-lo antes do retorno de `H264DecodePps`. A opção mais direta é observar o destino persistente, pois seu endereço não depende da reutilização da pilha.

---

## 14.111 — Watchpoint no PPS persistente sem disparo durante a decodificação (22/11/2026)

### Configuração

Após desabilitar o watchpoint 32, que havia monitorado incorretamente uma área de pilha já reutilizada, foi instalado um hardware watchpoint diretamente no campo `+0x24` do objeto PPS persistente:

```gdb
(gdb) watch -l *(unsigned int *)0xffffe0000f64
Hardware watchpoint 33: -location *(unsigned int *)0xffffe0000f64
(gdb) continue
```

O endereço-base conhecido do objeto PPS é `0xffffe0000f40`; logo, `0xffffe0000f64 = base + 36 = base + 0x24`.

### Resultado da execução

O programa prosseguiu até o encerramento normal, sem mensagem de disparo do hardware watchpoint 33:

```text
ERROR : cedarc <displayPictureThreadFunc:677>: ============> Picture Size: 1920x816
ERROR : cedarc <displayPictureThreadFunc:678>: ============> Picture PixelFormat: 6
ERROR : cedarc <displayPictureThreadFunc:695>: display thread get enungh frame, exit ...
[Thread ... exited]
ERROR : cedarc <parserThreadFunc:785>: hevc parser receive other thread error. exit flag
ERROR : cedarc <ChannelThread:1095>: demoDecoder finish.decode frame: 32, display frame: 31, cost 428 s
[Thread ... exited]
[Inferior 1 (process 46447) exited normally]
```

### Conclusão sustentada e limites

- Nesta execução, não foi detectada nenhuma escrita no DWORD `0xffffe0000f64` enquanto o watchpoint 33 esteve ativo.
- O processo terminou normalmente e o demo reportou 32 frames decodificados e 31 exibidos, com duração de 428 segundos.
- O resultado é consistente com as leituras anteriores de `PPS+0x24 = 0` no objeto temporário após o `memset`, no destino após o `memcpy` e no objeto persistente durante o processamento do slice.
- Isso não prova que o campo seja universalmente irrelevante ou que nunca seja escrito em outros fluxos, PPS, slices ou caminhos de execução. A conclusão é restrita a este objeto e a esta execução de teste.
- As mensagens `exit flag` e `display thread ... exit` ocorreram no encerramento do pipeline; isoladamente, não demonstram falha do decoder. O GDB confirmou `exited normally`.

### Próxima investigação

O monitoramento do PPS persistente não encontrou escrita. Não há motivo para repetir imediatamente o mesmo teste com o mesmo bitstream. O próximo avanço útil é comparar o comportamento com outro bitstream/stream que use parâmetros PPS ou slice distintos, ou investigar o consumidor de `PPS+0x24` em `H264DecodeSliceHeader` e determinar em quais condições a ramificação que consulta esse campo é necessária. O significado semântico do DWORD ainda permanece não identificado.

## 14.112 — Geração de bitstream H.264 com weighted P prediction efetiva (22/11/2026)

### Objetivo

Criar um fluxo de teste que tenha maior probabilidade de exercitar a condição observada em `H264DecodeSliceHeader`, que exige o DWORD privado em `PPS+36` diferente de zero e `slice_type` igual a 0 ou 3 antes de chamar `H264CongigureWeightTableRegisters`.

### Primeiro arquivo: ponderação habilitada, mas não utilizada

Foi gerado `/tmp/cedar-weighted.h264` com `testsrc2`, `libx264`, `weightp=1`, `bframes=0`, 90 quadros, 1280×720. O encoder informou:

```text
Weighted P-Frames: Y:0.0% UV:0.0%
```

Conclusão: habilitar `weightp` não garante que a sequência use ponderação efetiva. Esse arquivo não era um candidato adequado para testar o ramo.

### Segundo arquivo: fade para preto e weighted P adaptativo

Comando executado:

```bash
ffmpeg -hide_banner \\
  -f lavfi -i "testsrc2=size=1280x720:rate=30:duration=4" \\
  -vf "fade=t=out:st=1:d=3" \\
  -frames:v 120 \\
  -an \\
  -c:v libx264 \\
  -preset medium \\
  -pix_fmt yuv420p \\
  -x264-params "weightp=2:bframes=0:keyint=120:min-keyint=120:scenecut=0:repeat-headers=1" \\
  -f h264 /tmp/cedar-weighted.h264
```

Estatísticas retornadas pelo x264:

```text
profile High, level 3.1, 4:2:0, 8-bit
frame I:1
frame P:119
Weighted P-Frames: Y:74.8% UV:73.9%
```

### Interpretação

- **Observado:** o encoder reportou uso efetivo de weighted P prediction em 74,8% dos quadros P para luma e 73,9% para crominância.
- **Observado:** a sequência contém 1 quadro I e 119 quadros P; não foram configurados B-frames.
- **Conclusão:** este arquivo é um candidato significativamente melhor para investigar a ramificação, pois contém ponderação P efetivamente usada pelo encoder.
- **Ainda não confirmado:** não se sabe se o parser CedarX traduzirá essa sintaxe para um valor não zero no campo privado `PPS+36`. O campo não é documentado publicamente e não se deve equiparar automaticamente a sintaxe H.264 `weighted_pred_flag` ao offset privado.

### Próxima etapa

Executar o `vdecoderdemo` com `/tmp/cedar-weighted.h264` sob GDB, parar em `H264DecodeSliceHeader+932` (`0xfffff6f0d854`) e observar:

1. o ponteiro PPS em `[x19+264]`;
2. o valor DWORD em `PPS+36`;
3. o byte de tipo de slice em `[x19+32]`;
4. se a execução atravessa a condição e chega à chamada de `H264CongigureWeightTableRegisters`.

Usar o arquivo de 120 quadros, mas interromper a análise assim que os valores do primeiro slice P relevante forem capturados; não é necessário esperar o decode completo.


## 14.71 Teste com fluxo H.264 contendo weighted P-frames — entrada em H264DecodeSliceHeader

O segundo fluxo de teste `/tmp/cedar-weighted.h264` foi gerado com `libx264`, `weightp=2`, sem B-frames, contendo 120 quadros (1 I e 119 P). O encoder reportou `Weighted P-Frames: Y:74.8% UV:73.9%`, portanto este arquivo efetivamente contém P-frames ponderados.

O `vdecoderdemo` foi reiniciado sob GDB com esse arquivo. Após desativar o breakpoint 30 (`H264DecodePps+4`), a execução atingiu o breakpoint 34 na entrada de `H264DecodeSliceHeader` (`0xfffff6f0d4c0`). Registradores observados nesse ponto:

```text
x0 = 0xfffff000cc80
x1 = 0xfffff002d1a0
x19 = 0xfffff002d1a0
x20 = 0xfffff000cc80
```

Atingir a entrada da função confirma que o parser avançou até o processamento de uma slice do novo fluxo; ainda não identifica, por si só, se a slice atual é P nem se o PPS ativa a tabela de pesos. Próxima inspeção: ler o ponteiro PPS em `[x19+264]`, o DWORD `PPS+36` e o byte `[x19+32]` (tipo de slice), antes de avançar a execução.

## 14.72 PPS com campo +36 ativo em slice P do fluxo weighted

Na execução de `/tmp/cedar-weighted.h264`, após parar na entrada de `H264DecodeSliceHeader` (`0xfffff6f0d4c0`), foram obtidos:

```text
[x19+264] = 0x0000ffffe8002730
*(uint8_t *)(x19+32) = 0
```

O ponteiro PPS selecionado é, portanto, `0x0000ffffe8002730`, e o byte de tipo de slice é `0` (slice P, conforme a enumeração H.264 usada pelo parser). A leitura direta do DWORD no offset `+36` desse PPS retornou:

```text
endereço: 0xffffe8002754
valor:    0x00000001
```

Esta é a primeira observação positiva, nesta investigação, de um objeto PPS persistente com `PPS+36 = 1` associado a uma slice P do fluxo gerado com weighted prediction. Diferentemente do fluxo anterior, em que o campo era zero, os valores observados agora satisfazem a condição examinada em `H264DecodeSliceHeader`: campo PPS não zero e `slice_type` igual a 0 (ou 3). Isso indica que o caminho condicional para `H264CongigureWeightTableRegisters` deve ser seguido, salvo alguma condição adicional ou alteração de estado no restante da função.

**Ainda não confirmado:** a execução ainda não foi observada entrando em `H264CongigureWeightTableRegisters`, nem foi demonstrado que essa rotina submete comandos diretamente ao hardware. O próximo teste é instalar breakpoint em `0xfffff6f07ba4` (entrada observada da rotina) e continuar a execução a partir da entrada atual de `H264DecodeSliceHeader`.


## 14.73 Continuação do fluxo weighted: nova interrupção em H264DecodeSliceHeader

Após instalar o breakpoint 36 na entrada de `H264CongigureWeightTableRegisters` (`0xfffff6f07ba4`) e executar `continue`, o processo imprimiu informações de saída do decoder:

```text
Picture Size: 1280x720
Picture PixelFormat: 6
```

Em seguida, a execução voltou a parar no breakpoint 34, na entrada de `H264DecodeSliceHeader` (`0xfffff6f0d4c0`). Isso é compatível com o breakpoint 34 ainda estar habilitado e interromper a execução em outra chamada da função. **Não é evidência de que `H264CongigureWeightTableRegisters` tenha sido chamada ou ignorada**: o fluxo foi interrompido novamente antes de alcançar o breakpoint 36.

Próximo passo: desabilitar somente o breakpoint 34 e continuar, mantendo o breakpoint 36 ativo, para observar se o parser entra na rotina de configuração da tabela de pesos.

## 14.74 Confirmação de entrada em H264CongigureWeightTableRegisters

Após desabilitar o breakpoint 34 (entrada de `H264DecodeSliceHeader`) e continuar a execução do decoder, o GDB parou no breakpoint 36, endereço `0xfffff6f07ba4`:

```text
Thread 5 "vdecoderdemo" hit Breakpoint 36,
0x0000fffff6f07ba4 in H264CongigureWeightTableRegisters ()
   from /usr/lib/aarch64-linux-gnu/libawh264.so
```

Este é o primeiro resultado experimental que confirma a entrada efetiva em `H264CongigureWeightTableRegisters` durante a decodificação do arquivo `/tmp/cedar-weighted.h264`. A observação é coerente com a slice P (`slice_type = 0`) e com o PPS selecionado cujo DWORD no offset `+36` foi lido como `1` (`0xffffe8002754`). Portanto, a condição condicional identificada em `H264DecodeSliceHeader` foi satisfeita e o fluxo alcançou a rotina de configuração da weighted prediction.

O decoder também havia reportado quadro de saída de 1280x720 e `PixelFormat: 6` (NV12) nesta execução.

**Limite da conclusão:** atingir a entrada da rotina confirma que ela é executada; ainda não prova que seus dados sejam escritos em registradores MMIO nem que a rotina, isoladamente, efetue submissão de comandos ao VE. Para investigar o que ela configura, o próximo passo é capturar os registradores e o backtrace exatamente na entrada, antes de executar instruções (`ni`/`si`).

## 14.73 Análise do corpo de H264CongigureWeightTableRegisters — preparação de comandos ponderados

Foi fornecido o disassembly de `H264CongigureWeightTableRegisters`, endereços `0xfffff6f080fc` a `0xfffff6f08408` (offsets `+1368` a `+2148`). O trecho mostra um loop que percorre entradas de pesos e constrói valores de configuração para luma/crominância.

### Evidência observada

- Em `+1400`, inicializa o índice `w19=0`; `w23=0x200` e `w24=0x180` são valores-base incrementados a cada iteração (`w23 += 8`, `w24 += 4`). O limite do loop é lido de `[x22+60]`.
- Em `+1424..+1436`, chama uma função indireta por `[x25+448]`, com argumentos `x0=x25`, `w1=1`. Se o retorno for diferente de zero, grava `w23` em `[objeto+224]`.
- Em `+1440..+1464`, verifica um estado em `[x21+3952]` e uma condição no endereço derivado de `x20+32` (`[base+0xE708]`).
- Quando essa condição vale `1`, em `+1468..+1480`, carrega o ponteiro em `[base+0xE720]`, grava `w23` no endereço apontado e avança o ponteiro em quatro bytes.
- Em `+1484..+1516`, chama duas vezes a função indireta em `[x25+480]`, combina os retornos em `w28` (16 bits baixos e altos) e grava esse valor em `[objeto+228]`.
- Em `+1528..+1572`, sob a mesma condição de estado, grava `w28` no buffer apontado por `[base+0xE720]` e avança o ponteiro em quatro bytes.
- O bloco `+1576..+1672` repete o padrão de leitura/combinação e, em um caminho, grava pares de words com `stp` no buffer de comandos, avançando o ponteiro em oito bytes.
- O loop incrementa `w19`, `w23` e `w24` em `+1680..+1692` e compara o índice com `[x22+60]` em `+1692..+1696`.
- Outros caminhos (`+1700..+1804`) também gravam `w24`, `w28` e o literal `0xE0` no buffer apontado por `[base+0xE720]`.

### Interpretação delimitada

O código confirma que a função serializa valores calculados de ponderação em uma área de memória cujo endereço é mantido em `base+0xE720`, atualizando o ponteiro após cada escrita. A condição consultada em `base+0xE708` controla essas gravações. Isso é compatível com a construção de uma lista de comandos/parâmetros para o mecanismo VE.

**Não está provado ainda** que `base+0xE720` seja MMIO, nem que este trecho efetue a submissão final da lista ao hardware. A instrução `str` escreve através de um ponteiro carregado de memória; para classificar o destino é necessário observar o valor real do ponteiro em execução e rastrear quem consome ou submete o buffer. Também não foram fornecidos valores runtime de `x20`, `x21`, `x22`, `x25`, nem dos ponteiros e flags envolvidos.

### Próxima inspeção recomendada

Quando a execução atingir `H264CongigureWeightTableRegisters`, capturar os registradores `x20`, `x21`, `x22`, `x25` e os valores de memória:

```gdb
p/x $x20
p/x $x21
p/x $x22
p/x $x25
set $base = *(unsigned long *)($x20+32)
p/x $base
x/wx $base+0xe708
x/gx $base+0xe720
set $cmd = *(unsigned long *)($base+0xe720)
p/x $cmd
x/8wx $cmd
x/gx $x25+448
x/gx $x25+480
```

`[x20+32]` contém o ponteiro-base usado pelas operações do trecho; `base+0xE708` é a flag consultada e `base+0xE720` contém o ponteiro para o buffer. `x/8wx $cmd` inspeciona o conteúdo apontado, desde que `$cmd` seja válido e legível. Depois, examinar o código chamador e os consumidores desse buffer para localizar a submissão efetiva ao VE.

## 14.74 Rastreamento inicial do cursor do buffer em H264CongigureWeightTableRegisters

Na execução pausada na entrada de `H264CongigureWeightTableRegisters`, foram consultados os registradores preservados e a estrutura usada pelo código para localizar os campos de controle e o cursor:

```text
x20 = 0xfffff000cc80
x21 = 0xfffff003b1a0
x22 = 0x0
x25 = 0xfffff0023a00
*(uint64_t *)(x20 + 32) = 0xfffff6f0e990
base = 0xfffff6f0e990
*(uint32_t *)(base + 0xe708) = 0x200e4100
*(uint64_t *)(base + 0xe720) = 0x00000ed4d3ddde5c
```

O endereço `base+0xe708` corresponde ao DWORD que o disassembly compara com `1` antes de executar as gravações condicionais no buffer. O valor observado foi `0x200e4100`, portanto **não satisfaz a condição `== 1` nessa amostra**. Isso é importante: embora a função contenha instruções que acrescentam valores ao buffer quando a condição é verdadeira, não se pode concluir que essas gravações tenham ocorrido nesta execução.

O campo `base+0xe720` contém `0x00000ed4d3ddde5c`. Uma tentativa de ler esse valor como endereço virtual com `x/8wx` falhou (`Cannot access memory`). Ele não se parece com os endereços virtuais usuais das estruturas do processo, que nesta sessão aparecem no formato `0xffff...`. Pode ser um endereço de outro espaço (por exemplo DMA/físico), um valor codificado ou um campo com semântica diferente; ainda não há evidência suficiente para classificá-lo.

### Próxima inspeção

Inspecionar os DWORDs e QWORDs vizinhos em `base+0xe700` para entender a disposição dos campos e confirmar se `base+0xe708` é realmente a flag ativa nesta execução. Consultar também `info proc mappings` e `info symbol 0xed4d3ddde5c`; a ausência de mapeamento virtual reforçaria que o valor não é um ponteiro de CPU diretamente dereferenciável, mas não provaria por si só que seja DMA.

## 14.75 Inspeção dos campos vizinhos do cursor em `base+0xe720`

Na sessão GDB em `H264CongigureWeightTableRegisters`, foram obtidos:

```text
base = 0xfffff6f0e990
x/8gx base+0xe700:
0xfffff6f1d090: 0x00000188ffff2740 0x039e049d200e4100
0xfffff6f1d0a0: 0x0a41020194029342 0x0b41000ed4d3ddde
0xfffff6f1d0b0: 0x00000ed4d3ddde5c 0x000013ec0000004c
0xfffff6f1d0c0: 0x00000410ffff28a0 0x079e089d400e4100
```

A leitura em DWORDs (`x/16wx`) confirma a decomposição little-endian dos mesmos dados. O QWORD em `base+0xe720` continua sendo `0x00000ed4d3ddde5c`; `info symbol 0xed4d3ddde5c` retorna `No symbol matches`.

**Interpretação cautelosa:** `base+0xe720` contém um valor que a função anterior usa como cursor de gravação quando a condição de controle está ativa. Entretanto, a amostra atual em `base+0xe708` é `0x200e4100`, não `1`; portanto, as gravações condicionais descritas no disassembly não estão habilitadas por essa condição nesta amostra. O valor em `base+0xe720` não foi validado como ponteiro virtual CPU. A falha de `x/8wx` e a ausência de símbolo não bastam para concluir que seja endereço DMA/físico; também pode ser outro campo, endereço codificado ou referência de espaço distinto.

**Sobre leitura via `memcpy`:** não executar `memcpy` ainda. Se a origem não estiver mapeada no espaço virtual do processo, `memcpy` fará loads da mesma origem e poderá provocar `SIGSEGV`; não contorna a inacessibilidade observada pelo GDB. Antes, verificar os mapeamentos do processo (`info proc mappings`) e procurar se o intervalo que contém `0xed4d3ddde5c` está mapeado. Se não estiver, será necessário identificar a API/IOCTL de mapeamento ou a semântica do endereço no caminho CedarX, em vez de tentar copiá-lo diretamente.

### Próxima inspeção

Executar `info proc mappings` e enviar o resultado. Não chamar `memcpy` nem dereferenciar `0xed4d3ddde5c` até verificar o mapa de memória.

## 14.75 — Mapeamentos do processo e avaliação de `memcpy` para `base+0xe720`

O usuário forneceu a saída completa de `info proc mappings` para o processo 47746 (`vdecoderdemo`). Entre os mapeamentos observados estão regiões de heap, bibliotecas CedarX, várias regiões `rw-s /dmabuf:`, e o mapeamento de registradores `rw-s /dev/cedar_dev` em `0xfffff7ff0000–0xfffff7ff1000`.

O valor previamente lido em `base+0xe720` foi `0x00000ed4d3ddde5c`. Esse endereço não pertence a nenhum dos intervalos apresentados em `info proc mappings`: os mapeamentos do processo estão em regiões `0x0000aaaa...`, `0x0000ffff...` e stack `0x0000ffff...`; nenhum cobre `0x00000ed4d3ddde5c`.

**Conclusão:** não é seguro nem útil tentar `memcpy` diretamente a partir de `0xed4d3ddde5c` neste processo. `memcpy` executaria loads normais da CPU e não cria um mapeamento; a origem não mapeada pode causar `SIGSEGV`. A ausência de região mapeada também não identifica, por si só, o valor como endereço físico ou DMA. Ele pode ser um endereço em outro domínio, um valor codificado ou simplesmente um campo que não deve ser interpretado como ponteiro.

As regiões `/dmabuf:` mapeadas são candidatas a buffers CPU-accessible, mas não há evidência suficiente para associar o valor `0xed4d3ddde5c` a qualquer uma delas. O mapeamento de `/dev/cedar_dev` também não deve ser confundido com o endereço armazenado em `base+0xe720`.

A leitura anterior mostrou ainda `base+0xe708 = 0x200e4100`, enquanto o disassembly compara esse campo com `1` para selecionar as gravações condicionais no buffer. Logo, nessa amostra, essa condição não está satisfeita. A interpretação de `base+0xe720` como cursor/pointer permanece provisória; não foi demonstrado que o caminho de gravação esteja ativo nem que o conteúdo seja um ponteiro virtual.

**Próximo passo:** não chamar `memcpy` e não dereferenciar `0xed4d3ddde5c`. Continuar a investigação pelo fluxo de alocação/mapeamento dos buffers DMA e pela semântica do campo `base+0xe720` (incluindo quem inicializa e atualiza esse campo), correlacionando-o com as regiões `/dmabuf:` já mapeadas.


## 14.76 — Correção: `base+0xe720` está no mapeamento executável de `libawh264.so`

Reavaliando a saída de `info proc mappings`, há uma correção importante à interpretação anterior. O valor de `base` obtido no GDB era `0xfffff6f0e990`. Portanto:

- `base` = `0xfffff6f0e990`, dentro do mapeamento de `libawh264.so` (`0xfffff6f00000–0xfffff6f1e000`, `r-xp`).
- `base+0xe708` = `0xfffff6f1d098`, também dentro desse mapeamento.
- `base+0xe720` = `0xfffff6f1d0b0`, também dentro desse mapeamento.

Assim, os dados vistos em `x/8gx $base+0xe700` e `x/16wx $base+0xe700` estão sendo lidos de uma região executável/read-only da própria biblioteca, não de um buffer dinâmico em `/dmabuf`. A hipótese anterior de que `$base+0xe720` seria um cursor de buffer de comandos DMA foi precipitada e deve ser considerada **não comprovada**. O valor `0x00000ed4d3ddde5c` lido em `base+0xe720` é conteúdo armazenado nessa região da biblioteca; não deve ser tratado como ponteiro nem passado a `memcpy`.

A presença de dados em uma região `r-xp` é compatível com código, constantes ou tabelas estáticas. A comparação com `1` feita pelo disassembly continua sendo uma comparação sobre o DWORD em `base+0xe708`, mas, como esse endereço cai no mapeamento da biblioteca, não se deve interpretá-lo como flag mutável de um objeto sem primeiro identificar o símbolo/objeto base e a semântica dos offsets.

### Próxima inspeção

Identificar a que símbolo pertencem os endereços, sem alterar o estado do processo:

```gdb
info symbol 0xfffff6f0e990
info symbol 0xfffff6f1d0b0
```

Com esses resultados será possível determinar se `base` é uma função, uma tabela ou outro objeto estático e corrigir a interpretação do acesso `[x20+32]` no disassembly. Não instalar watchpoint de escrita nem chamar `memcpy` sobre o valor `0xed4d3ddde5c` nesta etapa.


## 14.76 — Correção da identificação de `base` após `info symbol` (22/11/2026)

O usuário executou:

```gdb
info symbol 0xfffff6f0e990
H264DecoderGetFbm in section .text of /usr/lib/aarch64-linux-gnu/libawh264.so
info symbol 0xfffff6f1d0b0
No symbol matches 0xfffff6f1d0b0.
```

**Correção importante:** o valor calculado anteriormente como `$base = *(unsigned long *)($x20+32)` é o endereço de código da função `H264DecoderGetFbm`, não um objeto de dados ou base de buffer. Consequentemente, os endereços `$base+0xe708` e `$base+0xe720` caem dentro da região executável `.text` de `libawh264.so`. Os valores `0x200e4100` e `0x00000ed4d3ddde5c` lidos nesses endereços são bytes/instruções da região de código interpretados como dados pelo comando `x`; não devem ser tratados como flag/cursor de buffer DMA. A conclusão anterior de que `base+0xe708` era uma flag de controle de buffer e `base+0xe720` era um cursor candidato estava incorreta e fica revogada.

A ausência de símbolo exato para `0xfffff6f1d0b0` é compatível com um endereço interno à seção `.text` sem símbolo nomeado. A próxima inspeção deve ser de instruções, não de ponteiros:

```gdb
x/12i 0xfffff6f1d090
x/12i 0xfffff6f1d0b0
```

Também é necessário rever o disassembly original e confirmar a cadeia de registradores que leva a `x20` e ao carregamento em `[x20+32]`, sem assumir que o ponteiro resultante seja base de estrutura. Não usar `memcpy`, não dereferenciar `0xed4d3ddde5c` e não instalar watchpoints sobre esses endereços até essa revisão.


## 14.77 Revisão da leitura de registradores na entrada de H264CongigureWeightTableRegisters (22/11/2026)

O usuário executou:

```gdb
x/12i 0xfffff6f1d090
x/12i 0xfffff6f1d0b0
```

As palavras exibidas são majoritariamente `.inst`/`udf`, com poucas instruções AArch64 aparentemente válidas. Isso não constitui evidência de uma sequência de código funcional nem de um buffer de comandos. Os endereços `0xfffff6f1d098` e `0xfffff6f1d0b0` resultaram da soma arbitrária de `0xe708`/`0xe720` ao ponteiro de função `0xfffff6f0e990`; essa interpretação como offsets de uma estrutura permanece inválida.

**Correção adicional importante:** o breakpoint `0xfffff6f07ba4` está na entrada de `H264CongigureWeightTableRegisters`, antes da execução do prólogo. Portanto, os registradores observados nesse instante não refletem necessariamente os valores locais que o prólogo atribui. O assembly começa com:

```asm
mov x25,x0
ldr x0,[x1,#264]
ldr x20,[x25,#400]
ldr w0,[x0,#40]
mov x22,x1
ldr x19,[x20,#4232]
```

Na entrada, `x0=0xfffff000cc80` e `x1=0xfffff002d1a0`. Após o prólogo, `x25` recebe o valor de entrada `x0`; somente então `x20` recebe o ponteiro armazenado em `[x25+400]`. Assim, o `x20` lido no breakpoint de entrada (`0xfffff000cc80`) não deve ser usado como se já fosse o `x20` local carregado por `ldr x20,[x25,#400]`. A leitura anterior de `*(uint64_t *)($x20+32)` na entrada não identifica o callback/objeto usado pela função.

A cadeia de registradores deve ser reconstituída a partir dos argumentos de entrada e do prólogo, sem reutilizar os valores pré-prólogo como se fossem os valores pós-prólogo. A evidência que permanece válida é: o breakpoint foi atingido, a rotina é chamada pelo parser H.264, e o PPS+36/slice_type observados anteriormente satisfazem a condição de entrada. Ainda não há prova de que o VE tenha consumido os parâmetros de weighted prediction.

Não dereferenciar `0xed4d3ddde5c`, não tratar `0xfffff6f1d090`/`0xfffff6f1d0b0` como buffer de comandos e não instalar watchpoints nesses endereços.

## 14.77 — Prólogo confirmado de `H264CongigureWeightTableRegisters` e breakpoint pós-carregamento

O dump completo do prólogo, obtido com:

```gdb
disassemble /r H264CongigureWeightTableRegisters, H264CongigureWeightTableRegisters+64
```

confirma os seguintes offsets:

```asm
+0:  stp x29,x30,[sp,#-112]!
+4:  mov x29,sp
+8:  stp x25,x26,[sp,#64]
+12: mov x25,x0
+16: ldr x0,[x1,#264]
+20: stp x19,x20,[sp,#16]
+24: ldr x20,[x25,#400]
+28: stp x21,x22,[sp,#32]
+32: ldr w0,[x0,#40]
+36: stp x23,x24,[sp,#48]
+40: mov x22,x1
+44: stp x27,x28,[sp,#80]
+48: ldr x19,[x20,#4232]
+52: cmp w0,#2
```

Conclusões:

- `x25` recebe o primeiro argumento (`x0`) em `+12`.
- `x0` recebe o ponteiro PPS de `[x1+264]` em `+16`; o campo PPS em `[PPS+40]` é lido em `+32`.
- `x20` só recebe o ponteiro do campo `[x25+400]` em `+24`.
- `x22` recebe o argumento de slice (`x1`) em `+40`.
- `x19` recebe `[x20+4232]` em `+48`.

Portanto, os valores de `x20` observados no breakpoint na entrada da função não eram o valor local após a instrução `ldr x20,[x25,#400]`. A leitura anterior de `[x20+32]` feita antes do prólogo foi inválida para identificar a tabela/objeto interno. A interpretação dos bytes `0xfffff6f1d090` e `0xfffff6f1d0b0` como código também não é sustentada: a região produziu palavras inválidas para AArch64 e continua sem identificação como dados funcionais.

Próximo ponto de inspeção recomendado: breakpoint temporário em `H264CongigureWeightTableRegisters+52`, desabilitando o breakpoint permanente 36 para evitar parar novamente na entrada:

```gdb
tbreak *H264CongigureWeightTableRegisters+52
disable 36
continue
```

Após parar, inspecionar sem avançar a execução:

```gdb
info registers x0 x1 x19 x20 x22 x25
x/gx $x20+32
x/gx $x20+4232
```

A inspeção deve estabelecer a identidade dos ponteiros pós-prólogo antes de qualquer dereferência adicional. Não concluir ainda que a função envia comandos ao VE: a chamada da rotina no parser ARM-side está comprovada, mas o consumo dos parâmetros pelo hardware ainda não.

# Stremio / CedarX — adendo técnico v123

## Inspeção de registradores em H264CongigureWeightTableRegisters

A função começa em `0xfffff6f07ba4`. O prólogo relevante é:

```asm
+12  mov x25, x0
+16  ldr x0, [x1, #264]
+24  ldr x20, [x25, #400]
+40  mov x22, x1
+48  ldr x19, [x20, #4232]
```

Na inspeção, foram observados:

```text
x0  = 0
x1  = 0xfffff002d1a0
x19 = 0xfffff000a8e0
x20 = 0xfffff0023a00
x22 = 0xfffff002d1a0
x25 = 0xfffff000cc80

[x20 + 32]   = 0xfffff002d1a0
[x20 + 4232] = 0xfffff000a8e0
```

O campo `x20+32` contém o mesmo endereço de `x1`/`x22`, compatível com um ponteiro para o contexto da slice. O campo `x20+4232` contém o mesmo endereço de `x19`, consistente com a instrução `ldr x19, [x20,#4232]` do prólogo.

**Ressalva:** o dump do GDB mostra o PC na entrada da função (`+0`). Portanto, não se pode assumir que `x25`, `x20` e `x19` já foram atualizados pelo prólogo desta chamada. Os registradores callee-saved podem ainda conter valores da função chamadora ou de uma chamada anterior. A interpretação estrutural acima é provisória até observar os registradores após a execução de `+48`.

### Próxima verificação

Interromper imediatamente após o `ldr x19`:

```gdb
tbreak *0xfffff6f07bd8
continue
info registers x0 x1 x19 x20 x22 x25
x/gx $x25+400
x/gx $x20+32
x/gx $x20+4232
```

Nesse ponto, `x25` deverá ter recebido o argumento `x0`, `x20` deverá ter sido carregado de `[x25+400]`, e `x19` deverá ter sido carregado de `[x20+4232]`. Se os valores coincidirem, teremos confirmado a cadeia de ponteiros sem confundir registradores de entrada com variáveis locais.

## Correção de interpretação anterior

A leitura de `*(uint64_t *)(x20+32)` feita quando `x20` ainda continha o valor anterior ao prólogo não identificava um callback nem um endereço de código. A análise deve usar os registradores após as instruções que os inicializam. Não inferir que o buffer de comandos de hardware foi localizado: isso ainda não foi demonstrado.


## Atualização v124 — cadeia de ponteiros confirmada após o prólogo

O breakpoint temporário foi colocado em `0xfffff6f07bd8`, imediatamente após a instrução `ldr x19, [x20, #4232]` (+48). Após `continue`, o GDB parou nesse endereço e mostrou:

```text
x0  = 0x0000000000000000
x1  = 0xfffff002d1a0
x19 = 0xfffff000a8e0
x20 = 0xfffff0023a00
x22 = 0xfffff002d1a0
x25 = 0xfffff000cc80

[x25 + 400]  = 0xfffff0023a00
[x20 + 32]   = 0xfffff002d1a0
[x20 + 4232] = 0xfffff000a8e0
```

A observação ocorreu depois das instruções do prólogo que inicializam esses registradores. Portanto, a cadeia está confirmada nesta chamada:

1. `x25` recebeu o argumento `x0` (`mov x25, x0`, offset +12).
2. `x20` recebeu `[x25+400]` (`ldr x20, [x25,#400]`, offset +24).
3. `x19` recebeu `[x20+4232]` (`ldr x19, [x20,#4232]`, offset +48).
4. `[x20+32]` contém o mesmo endereço de `x1`/`x22`, `0xfffff002d1a0`, o contexto da slice passado à função.

### Interpretação corrigida

A ressalva da versão anterior — de que os registradores poderiam ainda conter valores da função chamadora — fica superada para esta captura específica. O breakpoint foi atingido depois do carregamento de `x19`, e as leituras de memória corroboram as instruções do prólogo. `x19` (`0xfffff000a8e0`) é, portanto, o ponteiro carregado do campo `[x20+4232]` nesta execução.

Ainda não foi identificado o tipo semântico da estrutura apontada por `x19`, nem demonstrado que ela seja um buffer de comandos do VE ou um buffer DMA. Não interpretar seu conteúdo como tal sem rastrear os campos e seus consumidores.

### Próxima inspeção recomendada

Examinar uma janela pequena da estrutura apontada por `x19`:

```gdb
x/16gx $x19
```

Isso permite observar os primeiros 128 bytes como palavras de 64 bits e decidir quais campos merecem inspeção adicional, sem assumir previamente o layout da estrutura.

## v125 — inspeção inicial da estrutura apontada por x19

No breakpoint temporário em `H264CongigureWeightTableRegisters+52` (`0xfffff6f07bd8`), a cadeia de ponteiros foi confirmada após a execução do prólogo:

- `x25 = 0xfffff000cc80` (argumento original `x0`)
- `[x25+400] = 0xfffff0023a00`, carregado em `x20`
- `[x20+4232] = 0xfffff000a8e0`, carregado em `x19`
- `[x20+32] = 0xfffff002d1a0`, igual a `x1` (contexto da slice)

Leitura de 16 qwords em `x19 = 0xfffff000a8e0`:

```text
+0x00  0x00008811000d4f2c
+0x08  0x0000100000001020
+0x10  0x013e3e1800000000
+0x18  0x0000000000000000
+0x20  0x0000000000000000
+0x28  0x0000000000000000
+0x30  0x0000000000000000
+0x38  0x0000000000000000
+0x40  0x0000000100000000
+0x48  0xf7fb0000f4fc0000
+0x50  0x0000000000000000
+0x58  0x0000000000000000
+0x60  0x0000000000000000
+0x68  0x0000000000000000
+0x70  0x0000000000000000
+0x78  0x0000000000000000
```

Os valores observados não se parecem com ponteiros AArch64 válidos nos primeiros campos; há valores compactos/empacotados e muitos campos zerados. Isso é compatível com uma estrutura de estado ou parâmetros, mas o layout e o significado dos campos ainda não foram identificados. Não classificar `x19` como buffer DMA ou buffer de comandos com base apenas nesta leitura.

Próxima inspeção proposta: visualizar a mesma região como DWORDs (`x/32wx $x19`) e confirmar em que mapeamento de memória ela reside (`info proc mappings`). Isso ajuda a distinguir campos de 32 bits empacotados de possíveis ponteiros e a entender a natureza da alocação.


---

## Evidência v126 — classificação da estrutura `x19` por mapeamento

### Estado observado

No breakpoint temporário em `H264CongigureWeightTableRegisters+52` (`0xfffff6f07bd8`), após o prólogo:

- `x25 = 0xfffff000cc80`
- `[x25 + 400] = 0xfffff0023a00`, carregado em `x20`
- `[x20 + 4232] = 0xfffff000a8e0`, carregado em `x19`
- `[x20 + 32] = 0xfffff002d1a0`, igual ao contexto da slice em `x1`/`x22`

A leitura de `x/32wx $x19` mostrou:

```text
0xfffff000a8e0: 0x000d4f2c 0x00008811 0x00001020 0x00001000
0xfffff000a8f0: 0x00000000 0x013e3e18 0x00000000 0x00000000
0xfffff000a900: 0x00000000 0x00000000 0x00000000 0x00000000
0xfffff000a910: 0x00000000 0x00000000 0x00000000 0x00000000
0xfffff000a920: 0x00000000 0x00000001 0xf4fc0000 0xf7fb0000
0xfffff000a930: 0x00000000 0x00000000 0x00000000 0x00000000
0xfffff000a940: 0x00000000 0x00000000 0x00000000 0x00000000
0xfffff000a950: 0x00000000 0x00000000 0x00000000 0x00000000
```

### Classificação pelo mapa de memória

`info proc mappings` coloca `x19` dentro desta região:

```text
0x0000fffff0000000 0x0000fffff003c000 0x3c000 0x0 rw-p
```

Logo, o endereço está em uma região anônima, privada e gravável (`rw-p`), não em uma das regiões explicitamente mapeadas como `/dmabuf` (`rw-s`). A região não aparece identificada como `[heap]`; portanto, não a chamar de heap convencional. Pode ser uma alocação privada via `mmap` ou uma arena interna de alocação, mas isso ainda não foi demonstrado.

Os valores `0xf4fc0000` e `0xf7fb0000` aparecem como duas palavras de 32 bits nos offsets `x19+0x48` e `x19+0x4c`. Não há evidência suficiente para classificá-los como endereços físicos, endereços DMA, registradores VE ou ponteiros. Em AArch64, um endereço virtual completo deste processo tem a forma observada `0x0000ffff...`; esses valores de 32 bits não devem ser dereferenciados como ponteiros sem prova adicional.

### Conclusões permitidas

1. `x19` aponta para dados dentro de uma região virtual anônima `rw-p`.
2. O objeto contém campos escalares/empacotados e vários campos zerados nos primeiros 128 bytes.
3. O mapeamento não prova que a estrutura seja compartilhada com o VE ou diretamente acessível por DMA.
4. A interpretação anterior dos valores como buffer de comandos ou ponteiros DMA permanece não comprovada.

### Próxima investigação recomendada

Acompanhar as instruções reais que leem ou escrevem em offsets de `x19`, especialmente os acessos a `[x19,#16]` vistos no disassembly de `H264CongigureWeightTableRegisters`. Usar breakpoints/watchpoints de leitura e escrita apenas em offsets confirmados pelo assembly e, antes disso, verificar se o GDB/CPU suporta watchpoints de hardware adequados. Não instalar watchpoint nos valores `0xf4fc0000` ou `0xf7fb0000` como se fossem endereços.

Uma leitura adicional útil é `x/64wx $x19`, para ampliar a amostra da estrutura, mas a identificação semântica deve vir do código que consome seus campos e de quem inicializa a estrutura.

### LAST COMMANDS RESULTS:

(gdb) x/64wx $x19
0xfffff000a8e0:	0x000d4f2c	0x00008811	0x00001020	0x00001000
0xfffff000a8f0:	0x00000000	0x013e3e18	0x00000000	0x00000000
0xfffff000a900:	0x00000000	0x00000000	0x00000000	0x00000000
0xfffff000a910:	0x00000000	0x00000000	0x00000000	0x00000000
0xfffff000a920:	0x00000000	0x00000001	0xf4fc0000	0xf7fb0000
0xfffff000a930:	0x00000000	0x00000000	0x00000000	0x00000000
0xfffff000a940:	0x00000000	0x00000000	0x00000000	0x00000000
0xfffff000a950:	0x00000000	0x00000000	0x00000000	0x00000000
0xfffff000a960:	0x00000000	0x00000000	0x00000000	0x00000000
0xfffff000a970:	0x00000000	0x00000000	0x00000000	0x00000000
0xfffff000a980:	0x00000000	0x00000000	0x00000000	0x00000000
0xfffff000a990:	0x00000000	0x00000000	0x00000000	0x00000000
0xfffff000a9a0:	0x6c0a0028	0x73676269	0x00000041	0x00000000
0xfffff000a9b0:	0xf00000c0	0x0000ffff	0xf0025a10	0x0000ffff
0xfffff000a9c0:	0xf000aa30	0x0000ffff	0xf0000090	0x0000ffff
0xfffff000a9d0:	0x6c080028	0x70676269	0x2e656d67	0x312e6f73
(gdb) x/64bx $x19+0xc0
0xfffff000a9a0:	0x28	0x00	0x0a	0x6c	0x69	0x62	0x67	0x73
0xfffff000a9a8:	0x41	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0xfffff000a9b0:	0xc0	0x00	0x00	0xf0	0xff	0xff	0x00	0x00
0xfffff000a9b8:	0x10	0x5a	0x02	0xf0	0xff	0xff	0x00	0x00
0xfffff000a9c0:	0x30	0xaa	0x00	0xf0	0xff	0xff	0x00	0x00
0xfffff000a9c8:	0x90	0x00	0x00	0xf0	0xff	0xff	0x00	0x00
0xfffff000a9d0:	0x28	0x00	0x08	0x6c	0x69	0x62	0x67	0x70
0xfffff000a9d8:	0x67	0x6d	0x65	0x2e	0x73	0x6f	0x2e	0x31
(gdb) x/s $x19+0xf3
0xfffff000a9d3:	"libgpgme.so.1@"
(gdb) p/x *(unsigned long *)($x19+0xd0)
$71 = 0xfffff00000c0
(gdb) p/x *(unsigned long *)($x19+0xd8)
$72 = 0xfffff0025a10
(gdb) p/x *(unsigned long *)($x19+0xe0)
$73 = 0xfffff000aa30
(gdb) p/x *(unsigned long *)($x19+0xe8)
$74 = 0xfffff0000090
