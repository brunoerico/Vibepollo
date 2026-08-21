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
- [ ] Testar o encoder AMD/AMF na RX 7600 real via cliente nativo (LanHouse Native/
      moonlight-qt), comparando com o comportamento HEVC_AMF/H.264 já documentado no
      `host-apollo`. Em andamento (2026-08-21).
- [x] **WebRTC nativo — aprovado (2026-08-21).** Testado em LAN direto do navegador
      (`/stream`), conexão real com o host de teste:
      - H.264 — 60fps, 10-14ms de latência.
      - HEVC (H.265) — 60fps, 10-14ms de latência, 1080p.
      - AV1 — **quebrado**: tela rosa/magenta com FPS caindo de 60 pra ~21-25. Mesmo
        padrão de sintoma (corrupção de crominância) documentado num caso parecido do
        próprio Sunshine em outra plataforma/GPU
        ([LizardByte/Sunshine#5028](https://github.com/LizardByte/Sunshine/issues/5028),
        lá é NVIDIA/Linux/Vulkan, aqui é AMD/Windows) — mais provável ser imaturidade do
        caminho de encode AV1 nessa base de código inteira do que um problema específico
        de HDR/espaço de cor. H.264 e HEVC continuam sendo os dois codecs "de verdade"
        viáveis; não vale a pena oferecer AV1 como opção por enquanto.

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

## Decisão (2026-08-21): como cada cliente vai usar o WebRTC nativo do Vibepollo

Depois do teste aprovado (H.264/HEVC 60fps, ~9ms latência total, ver seção "WebRTC nativo —
aprovado" acima), decidimos usar o motor WebRTC do próprio Vibepollo em produção pros
clientes — em vez de terminar/manter nossa ponte própria (`dev/lanhouse-stream/client`,
fork do moonlight-web-stream, nunca chegou a ir pra produção de verdade). Investigação real
no código-fonte (não suposição) confirmou o caminho:

**Por que não dá pra simplesmente mandar o cliente pra `/stream` do painel:** essa página
usa a mesma sessão do painel administrativo (senha única, compartilhada entre todos os
hosts). Qualquer cliente que entrasse veria/controlaria o painel inteiro — outros jogos,
outras sessões. Inviável pra multi-tenant.

**Mecanismo real, confirmado lendo `src/http_auth_request_policy.cpp` e
`src/confighttp.cpp`:**

1. O HTML/JS/CSS da SPA **não exige autenticação nenhuma** (`RequestAuthPolicy::check` só
   protege rotas `/api/*` — o resto é sempre liberado). Só as chamadas de API por baixo
   precisam de credencial.
2. `/api/*` aceita `Authorization: Bearer <token>`, com token gerado via
   `POST /api/token` (autenticado com a credencial real do painel) recebendo um corpo tipo:
   ```json
   { "scopes": [{ "path": "^/api/webrtc/sessions$", "methods": ["GET", "POST"] }, ...] }
   ```
   O `path` de cada escopo é regex de verdade (`ApiTokenManager::authenticate_token` usa
   `std::regex_match`), então dá pra escopar exatamente pros endpoints que a página de
   stream usa e nada além disso — sem acesso a configurações, outros tokens, apagar apps,
   etc.
3. Endpoints reais que a página de stream/WebRTC usa (mapeados lendo
   `src_assets/common/assets/web/views/BrowserStreamView.vue` e
   `src_assets/common/assets/web/services/webrtc.ts` — token final deve cobrir só isso):
   `GET /api/webrtc/capabilities`, `GET /api/webrtc/cert`, `POST /api/webrtc/sessions`,
   `GET|DELETE /api/webrtc/sessions/{uuid}`, `POST /api/webrtc/sessions/{uuid}/offer`,
   `GET /api/webrtc/sessions/{uuid}/answer`, `GET|POST /api/webrtc/sessions/{uuid}/ice`,
   `GET /api/webrtc/sessions/{uuid}/ice/stream`, `GET /api/apps`,
   `GET /api/apps/{id}/cover`, `GET /api/apps/{id}/icon`, `GET /api/session/status`,
   `POST /api/apps/close`. Risco residual aceito pro MVP: o token é por-host, não
   por-sessão — um portador do token consegue, em teoria, mexer em qualquer sessão daquele
   host, não só a que ele criou (IDs são UUID, não enumeráveis, mas não é isolamento
   perfeito). Reavaliar se algum dia isso passar a incomodar de verdade.
4. Sinalização é HTTP puro (`fetch`/`apiGet`/`apiPost`), nada de WebSocket — mais simples
   de trabalhar. O vídeo/áudio em si (RTP/SRTP) continua indo direto do host pro navegador
   do cliente via WebRTC de verdade, nunca passa pela nossa infra — sem custo de banda
   nosso, só o overhead pequeno da sinalização.

**Certificado TLS — decidido resolver de verdade agora, não aceitar aviso de "não
seguro"** (opção descartada: aceitar o autoassinado só pro MVP). Como o cliente vai falar
**direto** com `https://<host>:47990` (sem proxy nosso no meio pra esconder isso), esse
endereço precisa de certificado confiável de verdade. Achado importante:
`confighttp.cpp:5700` monta o servidor HTTPS do painel (porta 47990, onde vive `/stream` e
toda a API WebRTC) com `config::nvhttp.cert`/`config::nvhttp.pkey` — os mesmos caminhos de
arquivo configuráveis em `sunshine.conf` (`cert`/`pkey`). Ou seja: **dá pra substituir por
um certificado real sem tocar em nada do código do Vibepollo**, só apontando esses dois
caminhos pra um cert/key de verdade.

Plano de emissão (DNS do domínio já está no Cloudflare, confirmado via `dig NS`):
1. Uma subdomínio fixo por host: `<hosts.id>.hosts.lanhousecloudgaming.com.br` (ex:
   `maquina-teste.hosts.lanhousecloudgaming.com.br`) → registro A apontando pro IP público
   da máquina.
2. Emitir via ACME **DNS-01** (não HTTP-01 — máquina pode estar atrás de CGNAT/sem porta 80
   exposta) usando [win-acme](https://www.win-acme.com/) com o plugin de validação
   Cloudflare — precisa de um **API token do Cloudflare escopado só pra edição de DNS**
   nessa zona (não a chave global da conta).
3. win-acme roda a renovação sozinho (cria a própria Scheduled Task) — só falta um script
   de pós-renovação (`--installation script`) que copie o cert/key novo pro caminho do
   `sunshine.conf` e reinicie `ApolloService`, senão o serviço continua servindo o
   certificado antigo até reiniciar manualmente.
4. Nova etapa no runbook do Maquinista (a inserir em `maquinista.md` quando executarmos
   isso de verdade): criar o registro DNS, instalar/configurar o win-acme, confirmar que
   `https://<subdomínio>/stream` carrega sem aviso nenhum de certificado.

**Bloqueado por**: falta IP fixo pra máquina (hoje atrás de CGNAT — confirmado via
`tracert`, segundo salto em `100.64.0.0/10`, IP público reportado é compartilhado e
inalcançável de fora). Cloudflare Tunnel resolveria só a camada de sinalização (47990),
não a mídia WebRTC em si (RTP/SRTP não passa por túnel — precisaria de um TURN nosso,
exatamente o custo de infra que queríamos evitar). IP fixo + redirecionamento de porta no
roteador resolve as duas camadas de uma vez, e também destrava a conexão nativa (que sofre
do mesmo problema). Decisão do usuário (2026-08-21): vai contratar IP fixo, mas testar sem
ele por enquanto — o que ficou provado abaixo já é suficiente pra continuar sem isso.

**Mecanismo validado de ponta a ponta em 2026-08-21 (só na LAN, sem certificado real
ainda)** — prova real, não suposição:
1. `POST /api/token` com Basic auth (credencial real do admin) e o corpo de escopos acima
   → devolveu um token.
2. Esse token, sozinho (sem cookie de sessão, sem Basic auth em mais nada): `GET /api/apps`
   → 200; `GET /api/webrtc/capabilities` → 200.
3. Mesmo token contra rotas fora do escopo: `POST /api/apps/delete` → 403 "Forbidden: Token
   does not have permission for this path/method"; `GET /api/config` → 403. Confirma que o
   escopo é respeitado de verdade, não é decorativo.
4. `POST /api/webrtc/sessions` com `app_id` (precisa ser **número**, não string — erro real
   encontrado testando) usando só o token → 200, sessão real criada com
   `cert_fingerprint`/`cert_pem` (handshake DTLS) devolvidos. **Prova que dá pra abrir uma
   sessão de streaming inteira usando só esse token, nunca a sessão do admin.**
5. `DELETE /api/webrtc/sessions/{id}` e `DELETE /api/token/{hash}` (via admin) funcionam —
   depois de revogado, o mesmo token volta a dar 403 em tudo. Ciclo de vida completo
   (emitir → usar → revogar) funciona.

Falta pra virar produto de verdade: (a) IP fixo + porta redirecionada (bloqueado, ver
acima) e o certificado real via win-acme+Cloudflare (plano já documentado); (b) o
`lanhouse-web` mintar um token desses por sessão de cliente (chamando `/api/token` com a
credencial do painel guardada do lado do servidor) em vez de eu gerar manualmente; (c) uma
página/cliente nosso que fale com essas rotas no lugar da SPA do Vibepollo (ou aceitar
carregar a SPA dele direto, já que ela não exige nenhuma autenticação pra servir o HTML).
