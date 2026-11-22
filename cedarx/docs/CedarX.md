# Relatório técnico — Aceleração de vídeo CedarX no Stremio

**Plataforma:** Orange Pi 4 Pro  
**Arquitetura:** ARM64 (`aarch64`)  
**Sistema:** Armbian/Debian Trixie  
**Kernel identificado:** `6.6.98-vendor-sun60iw2`  
**Objetivo:** integrar a decodificação de vídeo por hardware CedarX ao `libmpv` utilizado pelo Stremio.

## 1. Objetivo e estratégia adotada

O objetivo é fazer com que o Stremio utilize o mecanismo de decodificação de vídeo por hardware CedarX disponível no Orange Pi 4 Pro, em vez de depender exclusivamente da decodificação por software.

O Stremio em questão é instalado como Flatpak:

```text
com.stremio.Stremio.Devel
```

Ele utiliza uma versão embutida do `libmpv`, portanto não basta instalar um decoder no FFmpeg do sistema: é necessário integrar o decoder à cadeia de bibliotecas utilizada pelo próprio Stremio.

A estratégia definida foi:

1. Investigar e validar o funcionamento da API CedarX no sistema.
2. Entender como o decoder oficial da Allwinner é inicializado.
3. Reproduzir esse processo em um decoder personalizado para FFmpeg.
4. Construir um FFmpeg com suporte CedarX.
5. Construir o `libmpv` contra esse FFmpeg.
6. Integrar a biblioteca ao ambiente do Stremio, mantendo a instalação original preservada até a validação.

**Não substituímos o FFmpeg do sistema nem modificamos a instalação existente do Stremio.**

---

## 2. Hardware gráfico e situação dos dispositivos

O sistema apresenta os seguintes dispositivos:

| Dispositivo | Função |
|---|---|
| `/dev/dri/card0` | DRM/KMS |
| `/dev/dri/card1` | DRM/KMS |
| `/dev/dri/renderD128` | Renderização DRM |
| `/dev/cedar_dev` | Interface CedarX |
| `/dev/cedar_dev_ve2` | Interface adicional do mecanismo VE |

Os módulos do kernel identificados incluem:

```text
sunxi_stmmac
sunxi_ve
```

Não existem dispositivos V4L2 disponíveis:

```text
/dev/video*
/dev/media*
```

As respectivas classes em `/sys/class/video4linux/` e `/sys/class/media/` também estavam vazias.

Isso é importante porque indica que a integração não deve partir da hipótese de que existe um decoder V4L2 acessível. O caminho disponível e comprovado é a API proprietária CedarX.

### 2.1. GPU e Vulkan

O sistema reconhece a GPU:

```text
PowerVR B-Series BXM-4-64 MC1
```

O driver Vulkan identificado foi:

```text
DRIVER_ID_IMAGINATION_PROPRIETARY
driverInfo 24.2@6603887
Vulkan API 1.3.277
```

O Vulkan funciona no host, mas apresentou:

```text
VK_ERROR_INCOMPATIBLE_DRIVER
```

dentro do Flatpak.

Também foi identificado o fallback `llvmpipe`, que é renderização por software.

### 2.2. VAAPI

O VAAPI não está funcional:

```text
Failed to initialise VAAPI connection: -1
Device creation failed: -5
```

O único driver VAAPI encontrado foi:

```text
radeonsi_drv_video.so
```

Esse driver é destinado a GPUs AMD e não é compatível com a GPU PowerVR.

Portanto, não devemos configurar:

```bash
LIBVA_DRIVER_NAME=radeonsi
```

Isso não resolveria a decodificação.

**Conclusão:** nem VAAPI nem Vulkan forneceram um caminho funcional de decodificação por hardware dentro do Stremio. A API CedarX é a alternativa concreta identificada.

---

## 3. Bibliotecas e componentes CedarX instalados

Foram encontrados os seguintes pacotes:

```text
libcedarc-dev-2.0.0-arm64 1.0.4
gst-omx-listcomponents 1.18.3-1
```

O sistema possui o arquivo:

```text
/etc/cedarc.conf
```

Também possui o executável de demonstração:

```text
/usr/bin/vdecoderdemo
```

Esse executável é especialmente importante porque permite observar como a biblioteca oficial CedarX inicializa e utiliza o decoder.

### 3.1. Bibliotecas disponíveis

Entre as bibliotecas instaladas em:

```text
/usr/lib/aarch64-linux-gnu/
```

estão:

```text
libMemAdapter.so
libOmxCore.so
libOmxVdec.so
libOmxVenc.so
libVE.so
libawavs.so
libawavs2.so
libawh264.so
libawh265.so
libawmjpeg.so
libawmjpegplus.so
libawmpeg2.so
libawmpeg4base.so
libawmpeg4dx.so
libawmpeg4h263.so
libawmpeg4normal.so
libawmpeg4vp6.so
libawvp6soft.so
libawvp8.so
libawvp9HwAL.so
libcdc_base.so
libfbm.so
libsbm.so
libscaledown.so
libvdecVcs.so
libvdecoder.so
libvenc_base.so
libvenc_codec.so
libvencoder.so
libvideoengine.so
```

O conjunto comprova que existe uma implementação CedarX relativamente completa instalada, incluindo os plugins específicos de codecs.

### 3.2. Dependências de `libvdecoder.so`

A análise com `ldd` mostrou que `libvdecoder.so` depende de:

```text
libcdc_base.so
libMemAdapter.so
libfbm.so
libsbm.so
libvideoengine.so
libVE.so
libm.so.6
libpthread.so.0
librt.so.1
libc.so.6
```

Isso demonstra que o decoder não é uma biblioteca isolada. Ele depende de componentes de gerenciamento de memória, buffers e acesso ao mecanismo de vídeo.

### 3.3. Plugins OMX

O comando:

```bash
gst-omx-listcomponents /usr/lib/aarch64-linux-gnu/libOmxCore.so
```

identificou 24 componentes OMX, incluindo decoders H.264, HEVC e VP9.

Entretanto, não foi encontrado um plugin GStreamer OMX instalado:

```text
Nenhum *omx* ou *cedar* encontrado em
/usr/lib/aarch64-linux-gnu/gstreamer-1.0/
```

Também não foi possível confirmar um pipeline GStreamer funcional.

**Conclusão:** a existência de componentes OMX não demonstra que o Stremio possa utilizá-los diretamente. O caminho escolhido continua sendo a integração direta com a API CedarX.

---

## 4. Teste real de decodificação por hardware

Esta foi uma das descobertas mais importantes: conseguimos executar uma decodificação real usando o hardware CedarX, sem depender de uma hipótese ou de documentação.

### 4.1. Geração do vídeo de teste

Foi criado um vídeo H.264 de 1920 × 816, com 24 quadros por segundo e duração de 10 segundos:

```bash
ffmpeg -hide_banner \
  -f lavfi -i "testsrc2=size=1920x816:rate=24" \
  -t 10 \
  -c:v libx264 \
  -preset ultrafast \
  -profile:v high \
  -pix_fmt yuv420p \
  -an \
  -movflags +faststart \
  ~/Vídeos/cedar-test.mp4
```

Apesar da opção `-profile:v high`, o encoder `ultrafast` produziu um fluxo Constrained Baseline, pois desativou recursos como CABAC e B-frames.

Isso não invalida o teste: o objetivo inicial era verificar a comunicação entre o decoder CedarX e o mecanismo VE.

### 4.2. Conversão para Annex B

O fluxo H.264 foi extraído do contêiner MP4:

```bash
ffmpeg -hide_banner \
  -i ~/Vídeos/cedar-test.mp4 \
  -map 0:v:0 -c:v copy \
  -bsf:v h264_mp4toannexb \
  -an -f h264 /tmp/cedar-test.h264
```

O arquivo resultante tinha aproximadamente 12,7 MB e continha 240 quadros.

A conversão foi necessária porque o programa de demonstração recebeu um fluxo H.264 elementar, e não um contêiner MP4.

### 4.3. Execução do decoder

Foi executado:

```bash
sudo vdecoderdemo \
  -i /tmp/cedar-test.h264 \
  -codFmat 1 \
  -n 31 \
  -ss 1 \
  -sn 1 \
  -o cedar-frame.nv12 \
  -outFmat 6
```

O resultado foi:

```text
Picture Size: 1920x816
Picture PixelFormat: 6
display thread get enungh frame, exit ...
hevc parser receive other thread error. exit flag
demoDecoder finish.decode frame: 31, display frame: 31, cost 0 s
```

Código de saída:

```text
0
```

Arquivo produzido:

```text
cedar-frame.nv12
```

Tamanho:

```text
2.350.080 bytes
```

Esse tamanho corresponde exatamente a:

```text
1920 × 816 × 1,5 = 2.350.080
```

Ou seja, um quadro completo no formato NV12, com resolução 1920 × 816.

### 4.4. O que esse teste comprova

O teste confirma que:

- O mecanismo CedarX VE pode ser inicializado.
- O acesso ao dispositivo `/dev/cedar_dev` funciona.
- O plugin H.264 é carregado.
- O decoder consegue processar o fluxo H.264.
- Foram decodificados e apresentados 31 quadros.
- Foi possível exportar um quadro NV12 completo.
- O programa terminou com código de saída zero.

Há, porém, uma ressalva importante: a saída contém:

```text
hevc parser receive other thread error. exit flag
```

Esse aviso aparece no encerramento, mesmo no teste H.264. Ele não impediu a decodificação dos 31 quadros nem a geração do arquivo NV12, mas ainda precisa ser investigado.

**Conclusão:** a decodificação H.264 por hardware CedarX está comprovada no sistema. O problema restante é integrar essa capacidade à cadeia FFmpeg/libmpv utilizada pelo Stremio.

---

## 5. FFmpeg e libmpv do Stremio

O FFmpeg do sistema é:

```text
ffmpeg version 7.1.5-0+deb13u1
```

Versões das bibliotecas:

```text
libavutil      59.39.100
libavcodec     61.19.101
libavformat     61.7.103
libavfilter    10.5.103
```

Os pacotes de desenvolvimento correspondentes estão instalados:

```text
libavcodec-dev:arm64
libavutil-dev:arm64
libswscale-dev:arm64
```

### 5.1. Configuração do FFmpeg

A configuração de compilação contém:

```text
--disable-omx
--enable-libdrm
```

O comando:

```bash
ffmpeg -hwaccels
```

lista:

```text
vdpau
cuda
vaapi
drm
vulkan
```

Não existe suporte CedarX nessa compilação.

É importante distinguir duas coisas:

- `--enable-omx` habilita a integração do FFmpeg com OpenMAX IL.
- A API CedarX utilizada pelo `libvdecoder.so` é uma interface proprietária própria da Allwinner.

Portanto, habilitar OMX não equivale a habilitar CedarX.

### 5.2. libmpv embutido no Stremio

O Stremio utiliza um `libmpv` embutido, versão:

```text
libmpv 0.41.0
```

Ele foi compilado com suporte a várias tecnologias, incluindo Vulkan, GL/EGL, VAAPI, VDPAU e CUDA.

Entretanto, a presença dessas opções de compilação não significa que elas funcionem no hardware disponível.

Durante os testes:

- O Stremio não carregou bibliotecas CedarX.
- Não foram observados descritores de arquivo abertos para `/dev/cedar_dev` ou `/dev/cedar_dev_ve2`.
- Não foram observados acessos a `/dev/video*` ou `/dev/media*`.
- O processo apresentou uso elevado de CPU durante a reprodução.
- O acesso a `/dev/dri/renderD128` estava relacionado à renderização, não constituindo evidência de decodificação CedarX.

**Conclusão:** a versão atual do `libmpv` não está utilizando o decoder CedarX.

A integração exige disponibilizar um decoder CedarX ao FFmpeg que será usado na construção do `libmpv`.

---

## 6. API CedarX identificada nos headers

A análise dos headers instalados em `/usr/include/` permitiu identificar a interface necessária para construir um decoder próprio.

Os arquivos mais importantes são:

```text
cdc_config.h
memoryAdapter.h
sc_interface.h
sdecoder.h
typedef.h
vbasetype.h
vdecoder.h
veAdapter.h
veInterface.h
vencoder.h
vencoder_platform_v1.h
vencoder_platform_v2.h
```

### 6.1. Estrutura `VideoStreamInfo`

A estrutura que descreve o fluxo de vídeo é:

```c
typedef struct VIDEOSTREAMINFO {
    int eCodecFormat;
    int nWidth;
    int nHeight;
    int nFrameRate;
    int nFrameDuration;
    int nAspectRatio;
    int bIs3DStream;
    int nCodecSpecificDataLen;
    char* pCodecSpecificData;
    int bSecureStreamFlag;
    int bSecureStreamFlagLevel1;
    int bIsFramePackage;
    int h265ReferencePictureNum;
    int bReOpenEngine;
    int bIsFrameCtsTestFlag;
    int bCheckInitFlag;
} VideoStreamInfo;
```

Essa estrutura é utilizada para informar ao decoder:

- Codec.
- Resolução.
- Taxa de quadros.
- Dados específicos do codec.
- Características do fluxo.

Para o primeiro protótipo, o foco será H.264.

### 6.2. Estrutura `VConfig`

A estrutura de configuração do decoder é:

```c
typedef struct VCONFIG {
    int bScaleDownEn;
    int bRotationEn;
    int bSecOutputEn;
    int nHorizonScaleDownRatio;
    int nVerticalScaleDownRatio;
    int nSDWidth;
    int nSDHeight;
    int bAnySizeSD;
    int nSecHorizonScaleDownRatio;
    int nSecVerticalScaleDownRatio;
    int nRotateDegree;
    int bThumbnailMode;
    int eOutputPixelFormat;
    int eSecOutputPixelFormat;
    int bNoBFrames;
    int bDisable3D;
    int bSupportMaf;
    int bDispErrorFrame;
    int nVbvBufferSize;
    int nFrameBufferNum;
    int bSecureosEn;
    int bGpuBufValid;
    int nAlignStride;
    int bIsSoftDecoderFlag;
    int bVirMallocSbm;
    int bSupportPallocBufBeforeDecode;
    int nDeInterlaceHoldingFrameBufferNum;
    int nDisplayHoldingFrameBufferNum;
    int nRotateHoldingFrameBufferNum;
    int nDecodeSmoothFrameBufferNum;
    int bIsTvStream;
    int bAdapteDropFrame;
    struct ScMemOpsS *memops;
    eControlAfbcMode eCtlAfbcMode;
    eControlAfbcMode nAfbcSecMode;
    eControlIptvMode eCtlIptvMode;
    VeOpsS *veOpsS;
    void *pVeOpsSelf;
    int bConvertVp910bitTo8bit;
    unsigned int nVeFreq;
    int bCalledByOmxFlag;
    int bSetProcInfoEnable;
    int nSetProcInfoFreq;
    int nChannelNum;
    int nSupportMaxWidth;
    int nSupportMaxHeight;
    eCommonConfigFlag commonConfigFlag;
    int bATMFlag;
    /* campos adicionais de configuração */
} VConfig;
```

A definição completa inclui campos adicionais para determinadas configurações de decodificação.

A estrutura tem tamanho:

```text
sizeof(VConfig) = 224 bytes
```

Esse tamanho foi confirmado diretamente pelo compilador.

### 6.3. Estrutura `VideoStreamDataInfo`

O fluxo comprimido é enviado ao decoder por meio de:

```c
typedef struct {
    char *pData;
    int nLength;
    int64_t nPts;
    int64_t nPcr;
    int bIsFirstPart;
    int bIsLastPart;
    int nID;
    int nStreamIndex;
    int bValid;
    unsigned int bVideoInfoFlag;
    void *pVideoInfo;
} VideoStreamDataInfo;
```

Ela permite fornecer:

- Dados comprimidos.
- Tamanho do pacote.
- PTS e PCR.
- Informações sobre fragmentação do fluxo.
- Identificação do pacote e da faixa.

### 6.4. Estrutura `VideoPicture`

O decoder entrega os quadros por meio da estrutura `VideoPicture`.

Entre os campos identificados estão:

```c
nID
nStreamIndex
ePixelFormat
nWidth
nHeight
nLineStride
nTopOffset
nLeftOffset
nBottomOffset
nRightOffset
nFrameRate
nAspectRatio
nPts
nPcr
pData0
pData1
pData2
pData3
pMafData
nBufId
phyYBufAddr
phyCBufAddr
pPrivate
nBufFd
nBufStatus
```

Esses campos serão fundamentais para transferir os quadros decodificados do CedarX para as estruturas de imagem do FFmpeg.

Precisaremos verificar cuidadosamente:

- Formato de pixel.
- Stride.
- Dimensões efetivas.
- Endereços de memória.
- Possibilidade de compartilhamento de buffers.
- Sincronização entre o decoder e o consumidor dos quadros.

Ainda não está comprovado que seja possível entregar diretamente ao FFmpeg os buffers internos do CedarX sem cópia. Essa questão deverá ser investigada durante a implementação.

### 6.5. API de decodificação

As principais funções exportadas por `libvdecoder.so` são:

```c
VideoDecoder* CreateVideoDecoder(void);

void DestroyVideoDecoder(VideoDecoder* pDecoder);

int InitializeVideoDecoder(
    VideoDecoder* pDecoder,
    VideoStreamInfo* pVideoInfo,
    VConfig* pVconfig
);

void ResetVideoDecoder(VideoDecoder* pDecoder);

int DecodeVideoStream(
    VideoDecoder* pDecoder,
    int bEndOfStream,
    int bDecodeKeyFrameOnly,
    int bDropBFrameIfDelay,
    int64_t nCurrentTimeUs
);

int RequestVideoStreamBuffer(
    VideoDecoder* pDecoder,
    int nRequireSize,
    char** ppBuf,
    int* pBufSize,
    char** ppRingBuf,
    int* pRingBufSize,
    int nStreamBufIndex
);

int SubmitVideoStreamData(
    VideoDecoder* pDecoder,
    VideoStreamDataInfo* pDataInfo,
    int nStreamBufIndex
);

VideoPicture* NextPictureInfo(
    VideoDecoder* pDecoder,
    int nStreamIndex
);

VideoPicture* RequestPicture(
    VideoDecoder* pDecoder,
    int nStreamIndex
);

int ReturnPicture(
    VideoDecoder* pDecoder,
    VideoPicture* pPicture
);

int GetVideoStreamInfo(
    VideoDecoder* pDecoder,
    VideoStreamInfo* pVideoInfo
);
```

Um detalhe importante da declaração:

```c
typedef void* VideoDecoder;
```

Assim, `VideoDecoder*` equivale a um ponteiro para o handle do decoder, isto é, `void**`.

Esse detalhe deve ser respeitado na implementação C.

---

## 7. Interface de memória e mecanismo VE

A análise também identificou as interfaces de gerenciamento de memória e inicialização do mecanismo de vídeo.

### 7.1. Adaptador de memória

Em `memoryAdapter.h` existe:

```c
struct ScMemOpsS* MemAdapterGetOpsS(void);
```

A estrutura `ScMemOpsS` oferece operações como:

- Abrir e fechar o adaptador.
- Alocar e liberar memória.
- Alocar memória sem cache.
- Sincronizar cache.
- Obter endereços virtuais e físicos.
- Mapear descritores DMA.
- Configurar e encerrar operações de memória.

O programa de demonstração utiliza essa interface antes de criar o decoder.

### 7.2. Interface VE

Em `veAdapter.h`:

```c
VeOpsS* GetVeOpsS(int type);
```

Em `veInterface.h`, a estrutura `VeConfig` contém:

```c
typedef struct VeConfig {
    int nDecoderFlag;
    int nEncoderFlag;
    int nFormat;
    int nWidth;
    int nEnableAfbcFlag;
    int nResetVeMode;
    unsigned int nVeFreq;
    struct ScMemOpsS *memops;
    int nJustIspFlag;
    int bNotSetVeFreq;
    unsigned int EnableVcu;
} VeConfig;
```

A interface oferece operações como:

```text
init
release
lock
unlock
reset
waitInterrupt
```

Também foi identificada a enumeração:

```c
enum VE_MODE {
    VE_MODE_NULL = -1,
    VE_MODE_ENCPP = 0,
    VE_MODE_ENC = 1,
    VE_MODE_DE = 2,
    VE_MODE_VCUENC = 3,
    VE_MODE_VCUDEC = 4,
    VE_MODE_CNT = 5
};
```

E os tipos de operação:

```c
enum EVEOPSTYPE {
    VE_OPS_TYPE_NORMAL = 0,
    VE_OPS_TYPE_VP9 = 1
};
```

As bibliotecas exportam:

```text
libMemAdapter.so:
    MemAdapterGetDramFreq
    MemAdapterGetOpsS

libVE.so:
    GetVeOpsS
    VeInitialize
    VeRelease
```

### 7.3. Uma descoberta importante

Apesar da existência dessas interfaces, a análise do executável `vdecoderdemo` não encontrou chamadas diretas a:

```text
GetVeOpsS
VeInitialize
VeRelease
```

O demo chama `InitializeVideoDecoder()` e deixa a biblioteca `libvdecoder.so` cuidar da inicialização interna do mecanismo VE.

Isso significa que nossa implementação não deve presumir que precisa inicializar o VE manualmente.

A primeira abordagem deve reproduzir o caminho do programa de demonstração, deixando a biblioteca CedarX administrar o mecanismo de vídeo.

---

## 8. Engenharia reversa do `vdecoderdemo`

Como não temos o código-fonte do programa de demonstração, começamos a reconstruir seu comportamento por meio de:

```bash
nm -C
objdump
readelf
addr2line
strings
```

Também foi analisado o arquivo de texto enviado anteriormente com as saídas dos comandos.

A análise identificou as seguintes funções internas:

| Função | Endereço |
|---|---:|
| `initDecoder` | `0x4108` |
| `DecodeThread` | `0x4898` |
| `ReturnAllPicture` | `0x4834` |
| `displayPictureThreadFunc` | `0x4fbc` |
| `parserThreadFunc` | `0x55d8` |
| `initParams` | `0x6040` |
| `ChannelThread` | `0x64b8` |
| `main` | `0x6c48` |
| `supplyData` | `0x2264` |
| `CdcMemOpen` | `0x3090` |
| `CdcMemClose` | `0x30dc` |
| `CdcMemFlushCache` | `0x312c` |
| `addPictureToList` | `0x45c0` |
| `savePicture` | `0x4c80` |

O executável possui informações de depuração parciais, mas não foi possível recuperar o código-fonte original.

### 8.1. Comportamento de `main()`

A análise de `main()` mostrou que:

1. O programa aloca um contexto de 616 bytes com `calloc`.
2. Chama `initParams()` para interpretar os parâmetros da linha de comando.
3. Limita o número de decoders a cinco.
4. O padrão é um decoder.
5. Cria contextos de canal.
6. Cria threads para cada decoder.

Isso indica que a arquitetura do programa é multithread e que a inicialização do decoder é apenas uma parte do processo.

### 8.2. Comportamento de `initDecoder()`

Essa é a função mais importante para a implementação inicial.

A desmontagem revelou a seguinte sequência:

**1. Carregamento dos plugins**

```c
AddVDPlugin();
```

O carregamento dos plugins ocorre antes da criação do decoder.

**2. Inicialização das estruturas**

O programa zera:

```text
VConfig:          224 bytes
VideoStreamInfo:   72 bytes
```

A estrutura `VConfig` começa em:

```text
sp + 0x78
```

A estrutura `VideoStreamInfo` começa em:

```text
sp + 0x30
```

**3. Obtenção da interface de memória**

```c
MemAdapterGetOpsS();
```

O resultado é armazenado no contexto do decoder.

**4. Abertura do adaptador**

```c
CdcMemOpen(memops);
```

**5. Inicialização do mutex**

O programa chama:

```c
pthread_mutex_init();
```

**6. Criação do decoder**

```c
CreateVideoDecoder();
```

O handle retornado é armazenado no contexto.

**7. Seleção do codec**

A desmontagem mostra que o programa trata os seguintes valores:

```text
1 → VIDEO_CODEC_FORMAT_H264 (0x115)
2 → VIDEO_CODEC_FORMAT_H265 (0x116)
3 → VIDEO_CODEC_FORMAT_H264 (0x115)
```

O terceiro caso é curioso, pois as strings associadas ao programa sugerem AV1, mas a instrução observada seleciona o identificador H.264.

Isso deve ser tratado como comportamento específico do demo, possivelmente uma particularidade ou erro da implementação, e não como suporte AV1 comprovado.

**8. Leitura das configurações**

O programa consulta quatro parâmetros de `cedarc.conf`.

**9. Preparação de `VConfig`**

Além dos parâmetros de buffers, copia o ponteiro `memops` e um valor adicional do contexto para a estrutura.

**10. Inicialização final**

```c
InitializeVideoDecoder(
    decoder,
    &VideoStreamInfo,
    &VConfig
);
```

Se a inicialização falhar, o programa chama:

```c
DestroyVideoDecoder();
```

Essa sequência constitui a principal referência para reproduzir a inicialização do CedarX.

### 8.3. Comportamento de `parserThreadFunc()`

A análise identificou que essa thread:

1. Lê o fluxo por meio de `RawParserRead`.
2. Solicita espaço ao decoder usando `RequestVideoStreamBuffer`.
3. Prepara uma estrutura `VideoStreamDataInfo`.
4. Envia os dados com `SubmitVideoStreamData`.

Isso mostra que o demo não entrega simplesmente um ponteiro arbitrário de dados comprimidos ao decoder: ele utiliza o mecanismo de buffer disponibilizado pela própria API CedarX.

### 8.4. Comportamento de `DecodeThread()`

A thread de decodificação consulta informações sobre a quantidade de dados disponíveis e chama:

```c
DecodeVideoStream();
```

Ela é responsável por alimentar continuamente o mecanismo de decodificação.

### 8.5. Comportamento de `displayPictureThreadFunc()`

A thread de apresentação:

1. Solicita quadros por meio de `RequestPicture()`.
2. Adiciona os quadros à lista interna.
3. Executa a rotina de gravação quando necessário.
4. Devolve os quadros ao decoder por meio de `ReturnPicture()`.

Ao terminar, chama a rotina `ReturnAllPicture()`.

Isso é relevante porque o FFmpeg precisará implementar um ciclo de aquisição e devolução de quadros equivalente. Não podemos simplesmente obter um `VideoPicture` e ignorar seu ciclo de vida.

### 8.6. Chamadas da API confirmadas no executável

A desmontagem identificou chamadas para:

```text
InitializeVideoDecoder
ReturnPicture
DecodeVideoStream
RequestPicture
RequestVideoStreamBuffer
SubmitVideoStreamData
```

Não foram identificadas chamadas diretas a:

```text
GetVeOpsS
VeInitialize
VeRelease
```

Isso reforça a conclusão de que o gerenciamento VE é realizado internamente pela biblioteca de decodificação.

---

## 9. Descoberta dos parâmetros de `cedarc.conf`

Esta foi a parte mais recentemente concluída da engenharia reversa.

Na desmontagem de `initDecoder()`, encontramos quatro chamadas consecutivas a:

```c
CdcGetConfigParamterInt();
```

Os endereços das strings foram identificados e posteriormente confirmados com:

```bash
strings -a -t x /usr/bin/vdecoderdemo
```

Resultado:

```text
9d88 pic_4di_num
9d98 pic_4list_num
9da8 pic_4rotate_num
9db8 pic_4smooth_num
```

Os valores padrão também estão explícitos nas instruções AArch64:

| Chave | Valor padrão |
|---|---:|
| `pic_4di_num` | 2 |
| `pic_4list_num` | 3 |
| `pic_4rotate_num` | 0 |
| `pic_4smooth_num` | 3 |

### 9.1. Confirmação por desmontagem

As instruções relevantes são:

```asm
43d4: mov w1, #2
43d8: adrp x0, 9000
43dc: add x0, x0, #0xd88
43e0: bl CdcGetConfigParamterInt
43e4: str w0, [sp, #224]
```

```asm
43e8: mov w1, #3
43ec: adrp x0, 9000
43f0: add x0, x0, #0xd98
43f4: bl CdcGetConfigParamterInt
43f8: str w0, [sp, #228]
```

```asm
43fc: mov w1, #0
4400: adrp x0, 9000
4404: add x0, x0, #0xda8
4408: bl CdcGetConfigParamterInt
440c: str w0, [sp, #232]
```

```asm
4410: mov w1, #3
4414: adrp x0, 9000
4418: add x0, x0, #0xdb8
441c: bl CdcGetConfigParamterInt
4420: str w0, [sp, #236]
```

Portanto, não estamos inferindo os parâmetros pelos nomes: as chamadas e os valores padrão foram encontrados diretamente no código executável.

### 9.2. Confirmação dos campos com `offsetof()`

Para evitar erros de interpretação dos offsets, criamos um pequeno programa C que utiliza:

```c
offsetof(VConfig, campo)
```

O compilador retornou:

```text
nFrameBufferNum                            0x04c (76)
nVbvBufferSize                             0x048 (72)
nDeInterlaceHoldingFrameBufferNum          0x068 (104)
nDisplayHoldingFrameBufferNum              0x06c (108)
nRotateHoldingFrameBufferNum               0x070 (112)
nDecodeSmoothFrameBufferNum                0x074 (116)
memops                                     0x080 (128)
veOpsS                                     0x098 (152)
pVeOpsSelf                                 0x0a0 (160)
sizeof(VConfig) = 224
```

Essa saída permitiu fazer a associação exata entre os endereços da pilha e os campos da estrutura.

Como `VConfig` começa em `sp + 0x78`, temos:

| Parâmetro | Endereço na pilha | Offset relativo | Campo real |
|---|---:|---:|---|
| `pic_4di_num` | `sp + 224` | `0x68` | `nDeInterlaceHoldingFrameBufferNum` |
| `pic_4list_num` | `sp + 228` | `0x6C` | `nDisplayHoldingFrameBufferNum` |
| `pic_4rotate_num` | `sp + 232` | `0x70` | `nRotateHoldingFrameBufferNum` |
| `pic_4smooth_num` | `sp + 236` | `0x74` | `nDecodeSmoothFrameBufferNum` |

### 9.3. Mapeamento definitivo

| Configuração | Campo de `VConfig` | Offset | Padrão |
|---|---|---:|---:|
| `pic_4di_num` | `nDeInterlaceHoldingFrameBufferNum` | `0x68` | 2 |
| `pic_4list_num` | `nDisplayHoldingFrameBufferNum` | `0x6C` | 3 |
| `pic_4rotate_num` | `nRotateHoldingFrameBufferNum` | `0x70` | 0 |
| `pic_4smooth_num` | `nDecodeSmoothFrameBufferNum` | `0x74` | 3 |

**Esse mapeamento está confirmado tanto pelo código de máquina quanto pela definição C da estrutura.**

---

## 10. Outros campos de `VConfig` identificados

A análise de `initDecoder()` também revelou dois outros valores importantes.

### 10.1. Ponteiro `memops`

A desmontagem contém:

```asm
4424: ldr x0, [sp, #24]
4428: ldr x0, [x0, #232]
442c: str x0, [sp, #248]
```

Como `VConfig` começa em `sp + 0x78`, o destino `sp + 248` corresponde a:

```text
248 - 120 = 128 = 0x80
```

E o compilador confirmou:

```text
offsetof(VConfig, memops) = 0x80
```

Portanto, o programa grava corretamente o ponteiro obtido por `MemAdapterGetOpsS()` no campo:

```c
VConfig.memops
```

### 10.2. Outro valor do contexto

Também foi encontrada esta sequência:

```asm
4430: ldr x0, [sp, #24]
4434: ldr w0, [x0, #240]
4438: str w0, [sp, #292]
```

Como `VConfig` começa em `sp + 120`:

```text
292 - 120 = 172 = 0xAC
```

O campo correspondente, conforme a estrutura, é:

```c
VConfig.nSetProcInfoFreq
```

Portanto, o valor armazenado em `params + 240` é copiado para esse campo.

Isso é uma descoberta adicional, mas ainda não identificamos a origem e o significado operacional desse valor no contexto do demo.

### 10.3. Campos ainda não investigados

O tamanho de `VConfig` é de 224 bytes, mas ainda não reconstruímos todos os valores efetivamente utilizados pelo demo.

Precisamos investigar especialmente:

| Campo | Offset | Situação |
|---|---:|---|
| `eOutputPixelFormat` | A confirmar | Precisamos identificar o valor usado |
| `nFrameBufferNum` | `0x4C` | Campo identificado, valor ainda não confirmado |
| `bGpuBufValid` | A confirmar | Precisamos entender a configuração |
| `nAlignStride` | A confirmar | Precisamos verificar o comportamento |
| `bVirMallocSbm` | A confirmar | Precisamos verificar o comportamento |
| `bSupportPallocBufBeforeDecode` | A confirmar | Precisamos verificar o comportamento |
| `memops` | `0x80` | Confirmado |
| `veOpsS` | `0x98` | Não há atribuição direta identificada no trecho analisado |
| `pVeOpsSelf` | `0xA0` | Não há atribuição direta identificada no trecho analisado |
| `nVeFreq` | `0xA8` | Ainda não investigado |
| `nSetProcInfoFreq` | `0xAC` | Recebe valor de `params + 240` |

A ausência de atribuição direta em `initDecoder()` não significa necessariamente que o campo não seja utilizado. Ele pode permanecer zerado ou ser configurado internamente pela biblioteca.

---

# 11. Estado atual do projeto

Podemos separar as descobertas em três categorias.

### A. Comprovado experimentalmente

- [x] O hardware CedarX está acessível.
- [x] O dispositivo `/dev/cedar_dev` funciona.
- [x] O plugin H.264 é carregado.
- [x] O decoder H.264 inicializa.
- [x] O hardware decodifica um fluxo H.264 real.
- [x] Foram decodificados e apresentados 31 quadros.
- [x] Foi possível obter um quadro NV12 completo.
- [x] A biblioteca `libvdecoder.so` e suas dependências estão instaladas.
- [x] A API de decodificação foi identificada.

### B. Comprovado por engenharia reversa

- [x] Sequência de inicialização do `vdecoderdemo`.
- [x] Utilização de `AddVDPlugin()`.
- [x] Utilização de `MemAdapterGetOpsS()`.
- [x] Utilização de `CdcMemOpen()`.
- [x] Utilização de `CreateVideoDecoder()`.
- [x] Utilização de `InitializeVideoDecoder()`.
- [x] Fluxo de envio de dados com `RequestVideoStreamBuffer()` e `SubmitVideoStreamData()`.
- [x] Fluxo de decodificação com `DecodeVideoStream()`.
- [x] Aquisição e devolução de quadros com `RequestPicture()` e `ReturnPicture()`.
- [x] Identificação dos quatro parâmetros de buffers em `cedarc.conf`.
- [x] Associação dos quatro parâmetros aos campos corretos de `VConfig`.
- [x] Confirmação do offset de `VConfig.memops`.
- [x] Identificação de uma atribuição a `VConfig.nSetProcInfoFreq`.

### C. Ainda não implementado ou validado

- [ ] Decoder CedarX personalizado para FFmpeg 7.1.5.
- [ ] Conversão de pacotes FFmpeg para o mecanismo de entrada CedarX.
- [ ] Tratamento correto de PTS e DTS.
- [ ] Tratamento de quadros incompletos e fragmentados.
- [ ] Gerenciamento de buffers e sincronização.
- [ ] Conversão ou compartilhamento dos quadros NV12.
- [ ] Tratamento de flush, seek e fim de fluxo.
- [ ] Suporte a B-frames e fluxos H.264 mais complexos.
- [ ] Construção do `libmpv` com o FFmpeg personalizado.
- [ ] Integração e teste no Stremio.

---

# 12. Próxima etapa técnica

O próximo trabalho deve ser a reconstrução mais completa da inicialização do decoder, antes de escrever a integração FFmpeg.

Minha proposta é seguir esta ordem:

**Etapa 1 — Completar a análise de `VConfig`**

Precisamos descobrir como o demo configura:

- `eOutputPixelFormat`;
- `nFrameBufferNum`;
- `bGpuBufValid`;
- `nAlignStride`;
- `bVirMallocSbm`;
- `bSupportPallocBufBeforeDecode`.

Também devemos verificar os valores de `VideoStreamInfo`, especialmente `eCodecFormat`, resolução, taxa de quadros e dados específicos do codec.

**Etapa 2 — Criar um protótipo C independente**

Antes de modificar o FFmpeg, criar um pequeno programa que reproduza o fluxo de inicialização do demo:

```text
AddVDPlugin
    ↓
MemAdapterGetOpsS
    ↓
CdcMemOpen
    ↓
CreateVideoDecoder
    ↓
InitializeVideoDecoder
    ↓
RequestVideoStreamBuffer
    ↓
SubmitVideoStreamData
    ↓
DecodeVideoStream
    ↓
RequestPicture
    ↓
ReturnPicture
    ↓
DestroyVideoDecoder
```

O objetivo é validar que conseguimos reproduzir a decodificação sem depender do executável `vdecoderdemo`.

**Etapa 3 — Integrar ao FFmpeg**

Depois do protótipo, implementar um decoder próprio em `libavcodec`, inicialmente para H.264.

**Etapa 4 — Integrar ao libmpv**

Construir o `libmpv` utilizando o FFmpeg personalizado e testar a reprodução fora do Stremio.

**Etapa 5 — Integrar ao Stremio**

Somente depois dos testes anteriores, preparar a integração com o Flatpak.

---

## 13. Conclusão geral

O avanço mais significativo até aqui é que deixamos de trabalhar com hipóteses sobre a existência de aceleração CedarX e comprovamos que ela funciona no Orange Pi 4 Pro.

Também conseguimos reconstruir uma parte substancial do funcionamento interno do `vdecoderdemo`, incluindo a sequência de inicialização, o fluxo de envio de dados, a aquisição de quadros e os parâmetros de buffers.

A descoberta dos quatro parâmetros de `VConfig` é particularmente importante porque elimina a necessidade de adivinhar valores para o gerenciamento de quadros.

**O principal obstáculo agora não é mais fazer o hardware decodificar. É reproduzir corretamente a interface CedarX dentro de um decoder FFmpeg.**

Minha recomendação é não avançar ainda para a compilação do FFmpeg. Primeiro, devemos completar os campos de inicialização e construir o protótipo C independente. Isso reduz bastante o risco de confundir problemas da API CedarX com problemas internos do FFmpeg ou do libmpv.

---

