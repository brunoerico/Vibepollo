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
- [x] Instalado na máquina de teste (release oficial `-stable`, via `/qn` silencioso —
      ver seção de risco/runbook acima) em 2026-08-21. Confirmado sem tocar no que já
      estava lá: `uniqueid`, certificado do cliente pareado e `apps.json` sobreviveram
      intactos, sem precisar restaurar o backup nem ressincronizar o Supabase.
- [x] Confirmado que o `lanhouse-host-agent.ps1` funciona sem nenhuma modificação contra
      ele — heartbeat chegou fresco no Supabase logo após a instalação, sem restart da
      Scheduled Task nem do agente.
- [ ] Testar o encoder AMD/AMF na RX 7600 real, comparando com o comportamento
      HEVC_AMF/H.264 já documentado no `host-apollo`.
- [ ] Testar o WebRTC nativo dele como alternativa/complemento à nossa ponte própria.

## Risco real descoberto (2026-08-21): o instalador do Vibepollo compartilha o serviço `ApolloService`

O instalador oficial do Vibepollo (`packaging/windows/wix/custom_actions.wxs`) detecta uma
instalação existente do Apollo e **pergunta** (não força) se quer desinstalá-la antes de
prosseguir — mensagem "Existing Apollo installation detected. Would you like to
uninstall it now? [...] keeping your configuration". Não é confiável assumir que ele
preserva tudo que a gente precisa (`uniqueid`, `apps.json`, resolução fixa do display
virtual) — já vivemos exatamente essa classe de perda antes, ver a seção "Apagar
`C:\Program Files\Apollo\config` inteiro" no `host-apollo/BUILD_NOTES.md` (incidente
2026-08-15: reinstalação mudou o `uniqueid`, apagou `sunshine.conf` e exigiu re-parear
os 3 clientes e resincronizar `hosts.moonlight_host_id` manualmente).

**Não instalar o Vibepollo na máquina de teste sem seguir o runbook abaixo primeiro.**

**Atualização (2026-08-21, testado de verdade): o instalador tem um modo silencioso que evita
esse risco por completo.** `VibeshineInstaller.cs` (o bootstrapper `.exe`) tem dois caminhos
bem diferentes: sem argumentos ele abre a UI gráfica com o `MsgBox`/overlay perguntando sobre
o Apollo existente (o cenário arriscado descrito acima); com qualquer argumento reconhecido
como MSI (ex: `/qn`) ele entra em `RunCli()`, que **detecta e desinstala Apollo/Sunshine/
Vibeshine concorrentes de forma totalmente automática e silenciosa** (`UninstallCompetingProducts`),
sem abrir diálogo nenhum. Rodado assim (`VibepolloSetup-vX.Y.Z-stable.N.exe /qn` via
`Start-Process -Wait -PassThru`), terminou com `ExitCode=0` em ~1min, sem travar esperando
clique. Verificado depois: `ApolloService` `Running`, porta 47990 respondendo, **config
100% preservada** (`sunshine_state.json` byte-idêntico — mesmo `uniqueid`, mesmo certificado
do cliente pareado "LanHouse Native", mesma senha/salt; `apps.json` e `sunshine.conf`
também intocados), nenhuma flag de reboot pendente no registro, e o
`lanhouse-host-agent.ps1` mandou heartbeat fresco pro Supabase sem nenhuma mudança de
código — confirma que a API REST local (`localhost:47990`) do Vibepollo é mesmo compatível
com o Apollo puro. Cria um arquivo novo, `vibeshine_state.json`, só com `app_id_aliases`
(feature própria dele de manter o `moonlight_app_id` estável entre trocas de capa) — não
mexe no `uniqueid` nem em nada que a LanHouse dependa.

**Runbook revisado:** usar `/qn` em vez do fluxo "clicar OK" abaixo — mais seguro (sem UI
pra travar) e mais rápido. O passo 1 (backup) continua obrigatório mesmo assim, como rede
de segurança caso uma versão futura do instalador se comporte diferente.

## Runbook de teste seguro (revezando na mesma máquina)

### 1. Backup, com o Apollo atual ainda rodando

Via SSH (técnica base64, ver `.claude/agents/maquinista.md`), copiar pra fora da pasta
`config/` (que o instalador do Vibepollo pode apagar):

```powershell
$backup = "C:\Users\Administrator\apollo-backup-$(Get-Date -Format 'yyyyMMdd-HHmm')"
New-Item -ItemType Directory -Path $backup -Force | Out-Null
Copy-Item "C:\Program Files\Apollo\config\sunshine_state.json" "$backup\" -Force
Copy-Item "C:\Program Files\Apollo\config\apps.json" "$backup\" -Force
Copy-Item "C:\Program Files\Apollo\config\sunshine.conf" "$backup\" -Force
Copy-Item "C:\Program Files\Apollo\config\covers" "$backup\covers" -Recurse -Force -ErrorAction SilentlyContinue
reg export "HKLM\SOFTWARE\SudoMaker\SudoVDA" "$backup\SudoVDA.reg" /y
```

Também anotar à mão (não está em arquivo, é estado do Supabase):
- `hosts.moonlight_host_id` atual pra essa máquina (comparar depois do teste — se mudou,
  precisa atualizar).
- Lista de `host_games` cadastrados (game_id → moonlight_app_id/moonlight_app_name).

### 2. Instalar o Vibepollo

Baixar o instalador oficial (assinado digitalmente via SignPath, sem aviso do
SmartScreen) da [página de releases](https://github.com/Nonary/Vibepollo/releases) —
usar a última `-stable`, não uma `-beta`/`-alpha`, pra essa primeira avaliação. Quando
o instalador perguntar sobre desinstalar o Apollo existente, **clicar OK** (é a única
forma de instalar, já que dividem o mesmo nome de serviço) — o backup do passo 1 já
cobre a perda.

### 3. Testar

- Conferir `Get-Service ApolloService` voltou a `Running` depois da instalação.
- Recriar os apps a partir do `apps.json` salvo no backup (nomes idênticos = mesmo
  `moonlight_app_id`, calculável com `computeMoonlightAppId` se precisar).
- Parear pelo menos um cliente (LanHouse Native ou a ponte WebRTC) e jogar uma sessão
  real com um jogo que usa HEVC/AMF pra comparar com o comportamento documentado no
  `host-apollo` (pipeline reiniciando sozinho, ver seção correspondente lá).
- Testar o WebRTC nativo do Vibepollo separadamente (ele expõe uma URL de stream de
  navegador própria — checar a documentação/WebUI dele pra achar o endpoint).
- `lanhouse-host-agent.ps1` deve continuar funcionando sem mudança nenhuma (mesma API
  em `localhost:47990`) — confirmar heartbeat chegando fresco no Supabase.

### 4. Restaurar o Apollo depois do teste

1. Desinstalar o Vibepollo (Painel de Controle ou `Uninstall.exe /S` dele).
2. Reinstalar o Apollo a partir do instalador vendorizado
   (`host-apollo/vendor/Apollo-*.exe`) — nunca a última release do upstream.
3. Parar o serviço (`Stop-Service ApolloService`) e restaurar os arquivos do backup por
   cima de `C:\Program Files\Apollo\config\` antes do primeiro start real.
4. Reimportar `SudoVDA.reg` (`reg import`).
5. Iniciar o serviço e conferir `root.uniqueid` em `sunshine_state.json` contra o
   anotado no passo 1 — **se mudou, atualizar `hosts.moonlight_host_id`** no Supabase
   pra essa máquina antes de considerar restaurado.
