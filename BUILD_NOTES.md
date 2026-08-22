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

### Gotcha (2026-08-22): rota de login mudou de `/api/login` pra `/api/auth/login`

Vibepollo renomeou a rota de login do painel — `src/confighttp.cpp` registra
`"^/api/auth/login$"` via `register_api_route`, não mais `"^/api/login"` via
`server.resource[...]` como no Apollo antigo (`host-apollo/src/confighttp.cpp`, ainda
com o nome velho). Achado ao investigar por que `lanhouse-host-agent.ps1` (que faz
pareamento automático de clientes novos) nunca tinha sido revalidado contra o Vibepollo
depois da troca de host padrão em 2026-08-21 — a função `Connect-Apollo` do agente ainda
chamava `/api/login`, recebia 400 Bad Request, e todo pareamento de cliente novo numa
máquina Vibepollo falhava silenciosamente (PIN nunca era submetido, cliente via "Failed
to connect" sem log nenhum do lado do host explicando o motivo real).

Corrigido no próprio `lanhouse-host-agent.ps1`: tenta `/api/auth/login` primeiro, cai pra
`/api/login` se falhar — o script continua servindo as duas bases (Apollo e Vibepollo),
como o cabeçalho do arquivo já promete. Validado testando a sequência completa
(login + cookie + `/api/pin` + `/api/clients/update` pra conceder permissão de launch)
manualmente contra o Vibepollo real nesta máquina — sucesso ponta a ponta, inclusive
confirmado com uma sessão de stream real recebendo o HUD de telemetria novo.

**Se algum outro script/integração falar diretamente com o painel do Vibepollo (fora
deste agente), checar se também usa `/api/login` — mesmo bug se aplica.**

### Gotcha (2026-08-22): permissão de launch concedida ao cliente errado quando nomes colidem

Achado logo depois do fix acima, testando o pareamento automático de verdade contra o
Vibepollo real: o segundo dispositivo pareado (mesmo rótulo fixo "LanHouse Native" que
todo cliente reporta) recebeu 403 "lacks the Launch applications permission" ao tentar
jogar, mesmo com o log do `lanhouse-host-agent.ps1` mostrando "Pareado com sucesso" sem
nenhum aviso de falha.

Causa: quando já existe um pareamento com o mesmo nome, o Vibepollo desambigua sozinho
anexando `" (2)"`, `" (3)"` etc ao criar o novo — mas `Grant-FullPermissions` buscava por
nome **exato**, então acertava o cliente antigo (que já tinha permissão) em vez do que
tinha acabado de parear. Corrigido casando nome exato OU `"$ClientLabel (N)"`, pegando o
de `last_seen` mais recente entre os que baterem (o que acabou de parear é sempre o mais
recente).

---

# Histórico migrado de `host-apollo/BUILD_NOTES.md` (2026-08-22)

O fork `host-apollo` foi removido do projeto — nenhum host ativo roda Apollo hoje (só
`maquina-teste`, já em Vibepollo). Mas Vibepollo é um fork do Apollo e roda em cima da
mesma base (mesmo serviço `ApolloService`, mesmo caminho `C:\Program Files\Apollo`, mesma
config), então quase todo gotcha abaixo continua valendo — migrado aqui inteiro, sem
cortes, pra não perder conhecimento operacional real (vários desses incidentes custaram
horas pra diagnosticar). A seção de comandos de build/CMake é específica do Apollo em C++
puro (o `host-apollo` tinha um build próprio, separado do instalador vendorizado) e não se
aplica ao fluxo atual (que usa só o instalador do Vibepollo), mas foi mantida pro caso de
alguém precisar recompilar esse fork antigo a partir do zero um dia.

## Pré-requisitos de build do Apollo (histórico, não usado no fluxo atual)

- MSYS2 (`winget install MSYS2.MSYS2`)
- Dentro do MSYS2 UCRT64: boost, cmake, cppwinrt, curl-winssl, doxygen, graphviz, miniupnpc,
  onevpl, openssl, opus, toolchain, MinHook, nlohmann_json, nsis — **sem nodejs**, ver abaixo.
- Visual Studio Build Tools (workload C++)

### Node.js: NÃO instalar o pacote `mingw-w64-ucrt-x86_64-nodejs` do MSYS2
A própria documentação do Apollo (`docs/building.md:115-125`) avisa que o Node.js compilado
pelo MSYS2 usa gcc-16, que tem uma regressão em `std::bad_weak_ptr` que derruba o Node
durante a inicialização — quebra o target `web-ui` do CMake, que chama `npm install` via
`find_program(NPM npm)`. Instalar o Node.js oficial separadamente (nodejs.org, LTS ou
current, ou via nvm-windows) e garantir que `node.exe`/`npm` estejam no `PATH` **antes** de
rodar `cmake`.

## Comandos de build (histórico)

```
"C:\msys64\msys2_shell.cmd" -defterm -here -no-start -ucrt64 -c "cmake -B cmake-build-windows -G Ninja -S ."
"C:\msys64\msys2_shell.cmd" -defterm -here -no-start -ucrt64 -c "ninja -C cmake-build-windows"
```

**Atenção ao escapar aspas dentro de `.bat`**: `cmd.exe` não suporta `\"` pra aspas
aninhadas dentro de um argumento já entre aspas (isso quebra o parsing silenciosamente,
gera "system cannot find the path specified" ou "syntax of the command is incorrect").
Pra passar o `PATH` do Node.js (que tem espaço, "Program Files") pro `-c` do bash sem
precisar de aspas duplas aninhadas, usar aspas simples (que o bash aceita e o `cmd.exe`
ignora):
```
export PATH='/c/Program Files/nodejs':$PATH
```

**PATH do UCRT64 não herda o PATH do Windows por padrão** — `node`/`npm` (instalados
oficialmente, fora do MSYS2) não aparecem dentro do `msys2_shell.cmd -ucrt64` a menos que
sejam adicionados explicitamente ao `PATH` dentro do próprio comando `-c`.

**`msys2_shell.cmd` se desacopla da sessão SSH não-interativa em invocações
subsequentes** — na primeira vez que é chamado numa sessão SSH, roda de forma síncrona
normalmente; numa segunda invocação encadeada (ex: `cmake` e depois `ninja` no mesmo
`.bat`, via SSH), a sessão SSH costuma "voltar" (parecer ter terminado) assim que o
segundo `msys2_shell.cmd` é disparado, mesmo com o processo continuando a rodar (e
escrevendo no log) em segundo plano na máquina remota. Mitigação: rodar cada etapa
(`cmake` e `ninja`) como uma chamada SSH separada, e confirmar a conclusão checando o
arquivo de log / a existência do binário, não confiando no código de saída do SSH.

## Bug upstream encontrado e corrigido: `cfg.profile` não existe em `config_t`

Build parou em `src/video.cpp:830-831` com `'const struct video::config_t' has no member
named 'profile'`. Rastreado até o commit upstream `c71ea0a8bc` ("fix(video): enabled
profile and coder options for AMD AMF encoder", 2026-03-30, `ClassicOldSong/Apollo`) —
esse commit adicionou um lambda de opção `"profile"` (SDR, encoder AMD AMF/H.264) que lê
`cfg.profile`, mas nunca adicionou esse campo em `video::config_t` (que tem um comentário
explícito proibindo adicionar campos no meio da struct — é o layout do protocolo de
negociação RTSP do Moonlight).

**Correção aplicada** (`src/video.cpp`): revertido só o bloco `{"profile"s, [...]}` de
volta pra `{}` (era assim antes desse commit) — mantido o resto do commit (`"coder"s,
&config::video.amd.amd_coder`), que é válido e não depende de `cfg.profile`. **Nota:**
confirmado nesta mesma sessão (2026-08-21) que o Vibepollo já não tem esse bug — o código
dele já corresponde ao comportamento corrigido, zero patch necessário.

## SudoVDA — driver do monitor virtual

Vem empacotado junto no instalador oficial (Apollo e Vibepollo), não faz parte de nenhum
build via CMake. Repo separado: `SudoMaker/SudoVDA`.

**Assinatura do driver:** é assinado por atestação da Microsoft (Windows Indirect Display
Driver / IddCx oficial), não precisa de modo de teste nem desativar verificação de
assinatura.

**Causa real do bug conhecido `SudoVDA Driver status: Uninitialized` / Code 37** (issues
#1024, #1044, #1360 do Apollo): o certificado de assinatura de versões antigas do driver
**expira** com o tempo. Correção: sempre usar o instalador/release **mais recente** — não
é um problema de configuração nossa.

**Config do driver** (registro, não no `sunshine.conf`), em
`HKEY_LOCAL_MACHINE\SOFTWARE\SudoMaker\SudoVDA` (criar a chave se não existir):
- `gpuName` [STRING] — GPU que o adaptador virtual usa (default: escolhe a com mais VRAM)
- `maxMonitors` [DWORD] — máximo de monitores virtuais (default 10)
- `watchdog` [DWORD] — timeout em segundos (default 3, 0 desativa)
- `sdrBits`/`hdrBits` [DWORD] — profundidade de cor SDR/HDR

Mudança nesses valores exige recarregar o driver ou reiniciar — e se o host estiver com o
driver aberto, precisa fechar antes de recarregar.

## Credenciais do painel web via `--creds`

`sunshine.exe` tem um subcomando `creds` pra configurar usuário/senha sem passar pelo
assistente do navegador, mas **precisa do prefixo `--`** (`--creds <user> <pass>`, não
`creds <user> <pass>`) — sem o prefixo, o parser de argumentos (só reconhece flags que
começam com `--`) ignora o argumento e o binário cai no fluxo normal de inicialização do
servidor completo, que numa sessão SSH não-interativa falha repetidamente tentando sondar
displays (`ERROR_INVALID_PARAMETER`) sem nunca terminar. Com o prefixo certo, `--creds` é
despachado antes de qualquer inicialização de display/plataforma e retorna na hora, sem
precisar de sessão interativa.

## Incidente 2026-08-15: reboot não recuperava o serviço sozinho + reinstalação perdeu pareamentos

Dois problemas encadeados na Máquina de Teste, descobertos ao tentar cadastrar um jogo novo
(eFootball) e testar o fluxo completo pela primeira vez depois de vários reboots remotos.

### Sintoma 1 — depois de reiniciar a máquina, o painel web (porta 47990) nunca voltava

**Causa raiz (confirmada):** a máquina não tinha login automático do Windows configurado.
Um `Restart-Computer` remoto deixa a máquina parada na tela de login (mesmo sem senha, o
Windows ainda espera um clique/Enter) — sem sessão de desktop interativa, o host nunca
termina de inicializar a captura de tela, e a porta do painel nunca abre. `query session`
via SSH não-interativo dá falso negativo (sempre vazio) mesmo com sessão real ativa —
`Get-Process explorer` é o jeito confiável de checar isso remotamente.

**Correção:** `AutoAdminLogon=1` no registro
(`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`), com `DefaultUserName` e
`DefaultPassword` vazio (a conta Administrator não tem senha mesmo). Validado com um reboot
completo do zero — porta 47990 respondendo já na primeira checagem pós-boot.

### Sintoma 2 — depois de resetar as credenciais do painel (`--creds`), o serviço ficava travado

Ao tentar resetar a senha esquecida do painel web, o processo de `Stop-Service` + `--creds`
+ `Start-Service`, repetido algumas vezes com `Stop-Process -Force` no meio, deixou o
serviço num estado onde o processo ficava de pé mas a porta 47990 nunca abria — o log
mostrava negociação de encoder se repetindo indefinidamente a cada ~20s, sem nenhuma
conexão de rede real nas portas do GameStream.

**Causa raiz: não totalmente confirmada.** A suspeita mais forte (não provada) é que os
`Stop-Process -Force` no meio do reset de credenciais corromperam algum estado interno do
serviço. **Isso fica em aberto — se acontecer de novo, não repetir a mesma sequência de
force-kill.**

**Correção aplicada (contorno, não causa-raiz):** desinstalação completa + remoção manual
de `C:\Program Files\Apollo` (o instalador não limpa a config sozinho) + reinstalação do
instalador vendorizado (hash validado, copiado via `scp` da cópia local em `vendor/`, não
baixado de novo da internet). Resolveu de vez, sobreviveu a um reboot completo depois.

### O que se perde numa reinstalação, e por quê

Apagar `C:\Program Files\Apollo\config` inteiro (necessário pra garantir um estado
realmente limpo) remove tudo que não vive no Supabase:

1. **Os pareamentos de cliente** (`sunshine_state.json` → `root.named_devices`) — cada um é
   um certificado TLS confiado por essa instalação específica. Sem eles, qualquer cliente
   que tente conectar precisa parear de novo do zero.
2. **A identidade da própria máquina** (`root.uniqueid`) — muda a cada reinstalação. O
   campo `hosts.moonlight_host_id` no Supabase fica apontando pro valor antigo (obsoleto)
   até ser corrigido manualmente.
3. **O cadastro dos jogos** (`apps.json`) — precisa ser recriado. O catálogo do lado do
   `lanhouse-web` (tabela `games`) não é afetado — vive no Supabase, não na máquina.
4. **Usuário/senha do painel web** — resetados no processo.

### Plano de prevenção (checklist obrigatório em toda instalação nova de host)

1. `AutoAdminLogon` configurado e validado com um reboot completo real (não confiar que "o
   instalador resolve sozinho").
2. Credenciais do painel web definidas via `--creds` com um valor **padrão único do
   projeto** (não inventar uma nova a cada vez) — guardar esse valor junto com o resto dos
   segredos do projeto (mesmo lugar do `HOST_TICKET_SECRET`), não só na cabeça de quem
   configurou.
3. Scheduled Task do `lanhouse-host-agent.ps1` criada **e testada rodando uma vez na hora**
   (`schtasks /run`), não só criada e assumida como funcional.
4. Nunca usar `Stop-Process -Force` em `sunshine`/`sunshinesvc` como parte de rotina —
   preferir sempre `Stop-Service` → esperar → `Start-Service`, e se travar, reiniciar a
   máquina inteira em vez de forçar kill de processo.
5. Antes de qualquer reinstalação/limpeza de config, fazer backup de
   `sunshine_state.json` e `apps.json` — preserva referência rápida pra recriar os apps sem
   digitar tudo de novo, e permite comparar o `uniqueid` antigo vs. novo.
6. Depois de qualquer reinstalação: `hosts.moonlight_host_id` atualizado, `host_games`
   conferidos, reparear todos os clientes de teste, **`sunshine.conf` com a resolução fixa
   do monitor virtual restaurada** e a regra de firewall recriada (os dois se perdem numa
   reinstalação do zero e viraram parte fixa deste checklist justamente por já terem sido
   esquecidos uma vez).

## Firewall do Windows — regra nunca criada automaticamente

O instalador **não cria regra de firewall sozinho**. O painel web (47990) pode até
funcionar via acesso administrativo (Tailscale, que ignora o Firewall do Windows na
prática) — mas o app nativo conecta pelo IP da LAN direto (`hosts.local_ip`), e sem regra
de firewall o handshake RTSP falha (`Falha ao iniciar RTSP handshake: Erro -1`, portas TCP
48010 / UDP 48000 / UDP 48010) mesmo com a porta escutando do lado do servidor.

**Correção:** o próprio instalador já vem com um script pronto — `C:\Program
Files\Apollo\scripts\add-firewall-rule.bat` (tem também um `delete-firewall-rule.bat` pra
desfazer). Rodar como Administrator:

```powershell
Start-Process -FilePath "C:\Program Files\Apollo\scripts\add-firewall-rule.bat" -Verb RunAs -Wait -WindowStyle Hidden
```

Cria 4 regras (entrada, permitir, todos os perfis de rede). **Fazer isso em toda máquina
nova, logo depois de instalar** — antes ficava mascarado porque só testávamos o painel via
Tailscale, nunca o app nativo de fora do próprio host. **Nota:** o MSI do Vibepollo cria
essas regras automaticamente — só rodar este script manualmente se confirmar que faltam.

## Auto HDR do Windows derrubando a sessão ~15s depois de conectar

Sessão conectava, jogo abria, e caía sozinha ~15s depois, sempre com `HDR reverted for
display \\.\DISPLAY1` seguido de `Process terminated` no log — mesmo com a resolução do
monitor virtual já corretamente travada.

**Causa raiz real:** um **monitor físico de verdade conectado** na máquina de teste (com
EDID real e suporte a HDR nativo). O host reconfigura esse monitor físico a cada sessão
(não usa os adaptadores virtuais do SudoVDA, que existem mas ficam inativos) — e o **Auto
HDR do Windows** liga HDR sozinho no monitor real durante o jogo (feature nativa do
Windows 11, independente da config do host, que já pedia `hdr_state: Disabled`
explicitamente sem efeito). O host detecta a mudança de HDR ao vivo e derruba a sessão
como proteção.

**Correção:**
1. Desligar o Auto HDR do Windows via registro
   (`HKCU:\Software\Microsoft\DirectX\UserGpuPreferences`, valor
   `DirectXUserGlobalSettings` precisa conter `AutoHDREnable=0;`).
2. **Desconectar o monitor físico da máquina** — é a correção mais robusta e é como um
   servidor de cloud gaming de verdade deveria estar configurado (sem monitor físico
   nenhum, só o virtual do SudoVDA). Não afeta acesso remoto via SSH.

**Lição pro Maquinista:** todo host novo deveria já subir **sem monitor físico conectado**
por padrão — item do checklist de adequação (Etapa 1-2), não uma correção descoberta
depois que já deu problema.

## Sessão caindo sempre ~15s depois de conectar — causa real (não confundir com o Auto HDR acima)

Depois de descartar resolução fixa, monitor físico/Auto HDR, `wait-all` e o watchdog do
SudoVDA, a queda em ~15s **exatos** todo teste continuava idêntica. Root-caused ao vivo,
acompanhando `sunshine.log` em tempo real: o servidor manda pro cliente `Server notified
termination reason: 0x80030023` (`NVST_DISCONN_SERVER_TERMINATED_CLOSED` — encerramento
**gracioso**, não é erro/crash) e imediatamente depois `Process terminated` no lado do
host. Rastreado até `src/stream.cpp`, no loop principal de sessão (`if
(proc::proc.running() == 0 && !has_session_awaiting_peer) { break; }`).

**Causa raiz:** `apps.json` usava o campo `"cmd": "steam://rungameid/<appid>"` com
`"auto-detach": true`. Mas `auto-detach` só faz `proc_t::running()` tratar a sessão como
"rodando pra sempre" (`placebo = true`) **se o processo rastreado sair dentro de 5 segundos
do lançamento**. O processo que o Windows mantém vivo pra despachar a URI `steam://` pra
dentro da Steam já em execução leva, na prática, um pouco mais que 5 segundos até sair de
vez — então essa janela de tolerância nunca chegava a ser aplicada, e o host concluía
(errado) que o app tinha fechado, mesmo com o jogo rodando normalmente o tempo todo.

**Correção:** usar o campo `"detached": ["steam://rungameid/<appid>"]` em vez de `"cmd"` +
`"auto-detach"` — mesmo padrão do app nativo "Steam Big Picture" (`"detached":
["steam://open/bigpicture"]`). Com `cmd` vazio, `proc_t::running()` nunca chega a rastrear
processo nenhum — a sessão passa a durar enquanto o **cliente** estiver conectado, não
enquanto um processo específico do lançamento continuar vivo. Aplicado em
`vendor/apps-template.json` pros jogos do MVP.

**Becos sem saída investigados antes de achar isso (não repetir):**
- Resolução do monitor virtual ausente (real, corrigido, mas não era a causa desse bug).
- Monitor físico + Auto HDR do Windows (real, corrigido — mas o mesmo padrão de queda em
  ~15s persistiu mesmo com o monitor físico desconectado, provando que não era isso).
- `wait-all: true` interagindo mal com `auto-detach` — mudou pra `false`, não resolveu
  sozinho (mas manteve como `false` no fix final por ainda fazer sentido).
- Watchdog do driver SudoVDA ausente — restaurado, não resolveu.
- Timer de "segurar ESC 10s pra sair" — instrumentado com log de diagnóstico, confirmado
  que nunca disparava durante as quedas.
- Steam levando tempo pra abrir do zero — descartado, a Steam já estava rodando
  persistentemente o tempo todo.

## Investigação (2026-08-16/17): pipeline de captura reiniciando sozinho no meio da sessão — indício forte contra HEVC_AMF

Sintoma relatado pelo usuário: qualidade de stream boa num dia, "lagada" no dia seguinte,
sem nenhuma mudança de configuração entre um teste e outro.

**Achado concreto, confirmado 2x em `sunshine.log` no mesmo dia, mesmo padrão nas duas
vezes:** o pipeline de captura/encoder inteiro (`Creating encoder [hevc_amf]`, com todo o
bloco de negociação) é recriado do zero **no meio de sessões que continuam conectadas**
(uma vez 48s depois de conectar, outra vez 20min depois) — e nas duas vezes isso aconteceu
pouco antes da sessão cair de vez. Numa sessão saudável, a negociação de encoder acontece
uma vez só, no início.

**Descartado como causa direta:** versão do driver AMD, Windows Update, uso de CPU, tela
travando/protetor de tela (zero eventos 4800-4803 de bloqueio no momento exato,
confirmado via auditoria ligada de propósito), qualquer evento em System/Application log —
o Windows genuinamente não registra nada, a decisão de recriar acontece só dentro do
próprio host/AMF.

**Causa provável encontrada (2026-08-17):** o usuário trocou o codec de vídeo de
Automático (que sempre resolvia pra `hevc_amf`, o único codec presente em todo log
analisado até aqui, inclusive nos momentos do reinício) pra **H.264 manualmente** — sessão
rodou lisa, sem nenhuma travada. Forte indício de que o problema é específico do encoder
**HEVC via AMF** nesta GPU AMD (RX 7600, driver 31.0.21001.16014), não de tela/rede/Windows
como cogitado antes. Não confirmado com múltiplos testes (uma sessão só), mas bate
exatamente com o padrão: toda ocorrência do bug envolvia `hevc_amf` ativo. **Ver seção mais
acima sobre WebRTC testado em 2026-08-21 (H.264 e HEVC ambos 60fps/10-14ms sem esse
problema aparecer) e sobre o AV1 quebrado (tela rosa) — este bug do HEVC_AMF reiniciando
segue sem causa raiz 100% confirmada, mas é uma das razões concretas que motivou avaliar o
Vibepollo em paralelo (ver decisão de adoção mais acima neste arquivo).**
