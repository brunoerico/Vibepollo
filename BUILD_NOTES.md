# host-vibepollo — avaliação em paralelo do Vibepollo

## O que é isso

[Vibepollo](https://github.com/Nonary/Vibepollo) é outro fork do mesmo Apollo que usamos
(`ClassicOldSong/Apollo` — mesma origem do nosso `host-apollo`), mantido por uma pessoa
(Nonary) com a maior parte do código gerado por IA (Codex/GPT-5), guiado por ele na
arquitetura. Diverge muito mais do upstream que o nosso fork: **1443 commits à frente**
contra os nossos 14.

Este submódulo existe **só pra avaliação em paralelo** — decisão de 2026-08-21, avaliando
se vale a pena adotar (no todo ou em parte) por causa de dois pontos onde ele investiu
pesado e nós travamos:

1. **Encoder AMD/AMF** — feature própria de "native AMD AMF encoder" (#342) com uma
   sequência longa de fixes de lifecycle/HDR/timeout. É exatamente a área do nosso bug
   ainda não resolvido (pipeline HEVC_AMF reiniciando sozinho mid-sessão na RX 7600,
   contornado hoje forçando H.264 — ver `host-apollo/BUILD_NOTES.md`). **Atenção:** a
   issue [#377 "AMD new encoder not working"](https://github.com/Nonary/Vibepollo/issues/377)
   está aberta — não é uma área resolvida nem pra eles, é ativa mas instável.

2. **WebRTC nativo no host** — Vibepollo tem um cliente WebRTC de navegador embutido
   direto no Apollo (desde a v1.14.0, "webRTC and display refactor"), com negociação
   ICE/SDP, HDR sobre WebRTC, fullscreen em iOS, trava de teclado, tudo integrado no
   mesmo pipeline de captura que o streaming nativo (Moonlight/RTSP). Isso é
   potencialmente relevante pro nosso próprio bridge WebRTC separado
   (`dev/lanhouse-stream/client`, ver `project_lanhouse_stream` na memória) — se
   funcionar bem, pode ser um caminho pra simplificar nossa arquitetura em vez de manter
   uma ponte própria.

## Por que um repositório próprio em vez de "fork" de verdade no GitHub

O GitHub não deixa uma conta ter dois forks na mesma "rede" de um repositório — como já
temos `brunoerico/Apollo` (fork de `ClassicOldSong/Apollo`), e Vibepollo também descende
de `ClassicOldSong/Apollo`, não dá pra criar um fork de verdade do Vibepollo sem colidir.
`brunoerico/Vibepollo` é uma cópia normal (clone --mirror + push), com um remote
`upstream` apontando pro Vibepollo real, pra continuar puxando atualizações do jeito que
já fazemos com o `host-apollo`.

## Status

- [x] Repositório próprio criado (`brunoerico/Vibepollo`), submódulo registrado aqui.
- [ ] Build local/numa máquina Windows de teste (não é possível a partir do macOS —
      mesma limitação já documentada pro app Windows nativo).
- [ ] Instalar numa máquina separada (ou em janela de tempo isolada na máquina de teste
      atual) — **sem tocar no `host-apollo` que já está em uso**.
- [ ] Confirmar que o `lanhouse-host-agent.ps1` funciona sem modificação contra ele —
      deve funcionar, já que o agente fala só com a API HTTP local padrão do Apollo/
      Sunshine (`/api/login`, `/api/pin`, `/api/clients/*` em `localhost:47990`), sem
      nenhum patch de código-fonte do nosso lado (conferido: nosso `host-apollo` só tem
      9 linhas de diff real em `src/video.cpp` além de docs/scripts — nenhuma lógica de
      pareamento/acesso vive dentro do Apollo em si).
- [ ] Testar o encoder AMD/AMF na RX 7600 real, comparando com o comportamento
      HEVC_AMF/H.264 já documentado no `host-apollo`.
- [ ] Testar o WebRTC nativo dele como alternativa/complemento à nossa ponte própria.
