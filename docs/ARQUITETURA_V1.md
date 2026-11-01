# Jarvis Mobile — Arquitetura e Plano da V1

> Base: `livekit-examples/agent-starter-react-native` (Expo SDK 57, RN 0.86, LiveKit RN 3.x)
> Reuso: `open-jarvis/OpenJarvis` (Python, Apache-2.0, commit `45504ac`)
> Status: proposta para aprovação antes de codar.

---

## 1. Diagnóstico dos repositórios

### 1.1 agent-starter-react-native (este repo)

| Item | Estado |
|---|---|
| Stack | Expo 57 + expo-router, New Architecture, `@livekit/react-native`, `@livekit/components-react` (`useSession`, `useSessionMessages`) |
| Telas | `(start)` (botão conectar) e `assistant` (visualizador, chat log, chat bar, controle mic/câmera/tela) |
| Conexão | `hooks/useConnection.tsx` com `TokenSource` (dev server / literal / homepage agent) |
| Falta | auth, backend próprio, persistência, navegação real, notificações, permissões, testes |

Já vem pronto e é exatamente o que precisamos de mais difícil no cliente: áudio WebRTC, sessão de áudio iOS, transcrições e chat via text streams, envio de câmera/tela para o agente. **Interrupção (barge-in) é feita no servidor pelo LiveKit Agents**, o cliente não precisa fazer nada.

### 1.2 OpenJarvis

É um framework **local-first, mono-usuário** para desktop/servidor pessoal. Pontos importantes para nós:

- **Mono-usuário por processo**: todo estado fica em `get_config_dir()` (`$OPENJARVIS_HOME`/`~/.openjarvis`) em SQLite/JSONL; o servidor FastAPI tem uma única API key. **Não dá para usar `jarvis serve` como backend multi-tenant** de um app comercial.
- **Mas os módulos são bem desacoplados**: as stores aceitam caminho/backend (`ApprovalStore(db_path)`, `SchedulerStore(db_path)`, `create_fact_store(backend=...)` com `FactStoreRegistry` + ABC `FactStore`). Dá para usar como **biblioteca**, trocando a persistência por Postgres via adaptadores.
- Licença Apache-2.0: uso comercial permitido (manter NOTICE/atribuição).

Conclusão: **OpenJarvis entra como dependência Python (biblioteca) dentro dos nossos serviços**, não como servidor. Nada de fork profundo no início; adaptadores ficam no nosso código.

---

## 2. Arquitetura proposta

```
┌──────────────────────────── App (Expo / React Native) ────────────────────────────┐
│ expo-router: Chat · Voz · Projetos · Arquivos · Automações · Integrações · Config │
│ LiveKit RN (áudio/vídeo/text streams/RPC) · TanStack Query · Zustand · MMKV       │
│ expo-notifications · expo-calendar · expo-contacts · expo-location · image-picker │
│ "Device Action Bridge": handlers RPC com permissão + confirmação na UI            │
└──────────┬───────────────────────────────┬────────────────────────────────────────┘
           │ HTTPS (REST + SSE)            │ WebRTC (LiveKit Cloud)
           ▼                               ▼
┌──────────────────────┐        ┌──────────────────────────────────────┐
│ API Gateway (FastAPI)│        │ Voice Agent Worker (LiveKit Agents)  │
│ auth, token LiveKit, │        │ VAD Silero + turn detector + barge-in│
│ chat texto (SSE),    │        │ STT streaming → LLM → TTS streaming  │
│ projetos, arquivos,  │        │ visão (frames da câmera)             │
│ OAuth integrações,   │        │ RPC → celular (ações no device)      │
│ push tokens, billing │        └──────────────┬───────────────────────┘
└──────────┬───────────┘                       │
           └──────────────┬────────────────────┘
                          ▼
            ┌───────────────────────────────┐
            │ jarvis-core (pacote Python)   │  ← mesmo "cérebro" p/ voz e texto
            │ prompts/persona · tool registry│
            │ memória (OpenJarvis adaptado) │
            │ tools/conectores OpenJarvis   │
            │ políticas de aprovação/segur. │
            └──────────────┬────────────────┘
                           ▼
   Postgres + pgvector (dados, memória, embeddings, jobs) · Object storage (arquivos)
                           ▲
            ┌──────────────┴────────────────┐
            │ Worker de jobs (automações,   │
            │ sync de conectores, ingestão, │
            │ proativo/digest → push)       │
            └───────────────────────────────┘
```

### 2.1 Decisões-chave

1. **Um cérebro, dois transportes.** `jarvis-core` concentra persona, ferramentas, memória e políticas. O worker LiveKit usa-o na voz; o API Gateway usa-o no chat texto. Assim chat e voz compartilham histórico, memória e ferramentas.
2. **Chat texto fora do LiveKit.** Texto vai por REST/SSE (funciona sem abrir sala, mais barato, histórico offline). Durante uma chamada, o texto digitado vai pela própria sala (`useSessionMessages`, já no starter) e cai na mesma conversa.
3. **Pipeline STT→LLM→TTS (não modelo realtime) na V1.** Controle fino de ferramentas, memória e custo, troca de fornecedor fácil, PT-BR bom. Defaults sugeridos: STT Deepgram (streaming, pt-BR), LLM Claude (Sonnet para conversa, Haiku para extração de memória/classificação), TTS Cartesia ou ElevenLabs, `MultilingualModel` turn detector + Silero VAD, `allow_interruptions=True`, preemptive generation ligada. Fornecedores atrás de config → dá para avaliar realtime (OpenAI/Gemini) depois.
4. **Ações no celular via LiveKit RPC.** O agente chama `perform_rpc(destination=<user>, method="device.calendar.create", payload)`. O app registra handlers (`room.localParticipant.registerRpcMethod`) que: checam permissão do SO → checam política do usuário (sempre/perguntar/nunca, conceito reaproveitado do `ApprovalStore`) → mostram confirmação → executam → retornam resultado. Fora de chamada, a ação vira "pendente" + push.
5. **Multi-tenant por `user_id` em tudo**, Row-Level Security no Postgres. Nenhuma store do OpenJarvis gravando em disco local no servidor.
6. **Persistência gerenciada:** Supabase (Postgres+pgvector, Auth, Storage) acelera a V1; alternativa equivalente: Neon + Clerk + S3. Recomendo Supabase (o frontend do OpenJarvis já tem `lib/supabase.ts`).
7. **Pagamentos:** assinatura digital no app exige IAP → RevenueCat. Limites de uso (minutos de voz, mensagens) aplicados no Gateway e no worker.

---

## 3. O que reaproveitar do OpenJarvis

| Área | Módulo OpenJarvis | Como usar |
|---|---|---|
| Memória permanente | `memory/extractor.py` (FactExtractor), `memory/service.py` (fila assíncrona), `memory/store.py` (ABC `FactStore`, níveis de confiança) | **Reusar** extractor + service; **criar** `PostgresFactStore` registrado no `FactStoreRegistry` |
| Injeção de contexto | `tools/storage/context.py` (`inject_context`), `tools/storage/*` (chunking, híbrido BM25+denso) | Reusar lógica; backend de vetores → pgvector |
| Arquivos / RAG | `server/upload_router.py`, `connectors/chunker.py`, `embeddings.py`, `hybrid_search.py`, `tools/pdf_tool.py` | Reusar pipeline de ingestão; storage → S3/Supabase |
| Pesquisa web | `tools/web_search.py` (Tavily/ddgs), `tools/http_request.py`, `security/ssrf.py` | Reusar com **Tavily** (ddgs não é adequado comercialmente) |
| Pesquisa profunda | `agents/deep_research.py`, `research_loop.py` | Reusar como tool assíncrona (resultado chega por push) |
| Calendário/e-mail/tarefas | `connectors/gcalendar.py`, `gmail.py`, `google_tasks.py`, `outlook.py`, `google_auth.py`, `oauth.py` | Reusar clientes; **mover tokens OAuth** para tabela criptografada por usuário |
| Outras integrações | `connectors/notion.py`, `slack_connector.py`, `dropbox.py`, `gdrive.py`, `spotify.py`, `weather.py`, `news_rss.py`; `mcp/client.py` | V1: Google (Calendar/Gmail/Tasks) + clima/notícias; resto em V1.x; MCP client para integrações genéricas |
| Automações | `scheduler/scheduler.py` (cron/interval/once via croniter), `scheduler/tools.py` | Reusar modelo `ScheduledTask` e tools `schedule_task/list/pause/cancel`; execução no worker com store Postgres |
| Proativo | `agents/proactive_agent.py`, `agents/morning_digest.py`, `tools/digest_collect.py`, `tools/proactive_tools.py` | Reusar lógica de coleta → classificação → ações em tiers; saída = push notification |
| Aprovações | `tools/approval_store.py` (pending_actions, permission_memory, always/ask/never) | Reusar o **modelo** para ações no celular e em integrações; port para Postgres |
| Segurança | `security/injection_scanner.py`, `guardrails.py`, `credential_stripper.py`, `taint.py`, `audit.py`, `rate_limiter.py` | Reusar diretamente — essencial com e-mail/web como entrada não confiável |
| Sessões | `sessions/compression.py` | Reusar para resumir conversas longas |
| Skills | `skills/*` (padrão agentskills.io) | V2 (habilidades instaláveis) |
| Ferramentas simples | `calculator.py`, `think.py`, `image_tool.py`, `user_profile_manage.py`, `memory_manage.py` | Reusar |

**Não reaproveitar na V1:** `frontend/` (React web/Tauri — só como referência de UX e tipos da API), `desktop/`, `rust/`, engines locais (`ollama`, `vllm`, `mlx`…), `learning/`, `bench/`, `evals/`, `mining/`, CLI, a maioria de `channels/` (Telegram/WhatsApp podem virar V2), ferramentas de shell/código (`shell_exec`, `code_interpreter`) — risco alto num app de consumo. Conectores Apple (`apple_calendar`, `imessage`, …) dependem de macOS → no mobile isso é feito **no próprio aparelho** via RPC.

Integração com OpenJarvis via LLM: o `/v1/chat/completions` dele é OpenAI-compatível, mas usá-lo como LLM do LiveKit adiciona salto de rede e é mono-usuário. **Não** recomendado; usamos os módulos diretamente.

---

## 4. O que precisa ser criado

**Servidor**
- `jarvis-core`: registro de ferramentas no formato `@function_tool` do LiveKit **e** schema JSON para o chat texto (um adaptador gera os dois a partir das tools OpenJarvis); persona/prompt; montagem de contexto (perfil + fatos + projeto + RAG); política de aprovação.
- Adaptadores Postgres: `PostgresFactStore`, store de aprovações, store de automações, store de tokens OAuth (criptografado), vetores pgvector.
- API Gateway: auth (JWT Supabase), `POST /livekit/token` (com `agentName` e metadados do usuário/projeto), conversas/mensagens, chat SSE, projetos, arquivos (upload assinado), integrações (OAuth start/callback com deep link de volta ao app), automações, ações pendentes, push tokens, limites/billing (webhook RevenueCat).
- Voice Agent Worker (Python, LiveKit Agents 1.x): sessão, ferramentas, visão por frames, RPC para o device, gravação da transcrição na conversa, extração de memória ao fim do turno.
- Worker de jobs: automações cron, sync de conectores, ingestão de arquivos, proativo/digest, envio de push (Expo Push Service).
- Observabilidade: Sentry (app+backend), logs de custo por usuário, audit log.

**App**
- Auth + onboarding (permissões explicadas uma a uma).
- Navegação por abas: Chat · Voz (tela atual do starter, refeita) · Projetos · Automações · Ajustes/Integrações.
- Chat texto com streaming, anexos (foto/arquivo), markdown, citações.
- Device Action Bridge (RPC) + tela de permissões por ação + histórico de ações.
- Notificações (registro de token, categorias com ações "Aprovar/Negar", deep links).
- Memória visível e editável ("o que o Jarvis sabe sobre mim") — requisito de confiança e LGPD.
- Exclusão de conta e export de dados (exigido pela App Store).

---

## 5. Ações no celular — o que é possível (V1)

| Ação | iOS | Android | Lib |
|---|---|---|---|
| Criar/ler eventos do calendário do aparelho | ✅ | ✅ | `expo-calendar` |
| Lembretes | ✅ (Reminders) | via calendário | `expo-calendar` |
| Contatos (buscar) | ✅ | ✅ | `expo-contacts` |
| Abrir ligação / SMS / e-mail / mapa / URL / app via deep link | ✅ (abre, usuário confirma) | ✅ | `expo-linking`, `expo-sms`, `expo-mail-composer` |
| Notificação/alarme local | ✅ | ✅ | `expo-notifications` |
| Localização atual | ✅ | ✅ | `expo-location` |
| Câmera/foto para análise | ✅ | ✅ | LiveKit vídeo, `expo-image-picker` |
| Clipboard, compartilhar, haptics | ✅ | ✅ | `expo-clipboard`, `expo-sharing` |
| Controlar outros apps | ❌ | ⚠️ AccessibilityService — **fora da V1** (política Google Play) | — |
| Siri / Atalhos / Assistente padrão | V2 (App Intents / Assist API, módulo nativo) | V2 | Expo Modules API |

Regra: toda ação com efeito externo (enviar, apagar, comprar, criar evento) passa por confirmação, a menos que o usuário tenha marcado "sempre permitir" para aquele padrão.

---

## 6. Modelo de dados (resumo)

`users`, `profiles` · `projects` (instruções, escopo) · `conversations` (project_id, canal voz/texto) · `messages` (role, content, attachments, tool_calls) · `facts` (texto, trust, source, embedding, project_id opcional) · `files` / `chunks` (embedding) · `integrations` (provider, tokens criptografados, escopos) · `automations` (cron/once, prompt, alvo, estado) · `automation_runs` · `pending_actions` / `permission_rules` · `push_tokens` · `usage_events` · `audit_log`. RLS por `user_id` em todas.

---

## 7. Estrutura do repositório (monorepo)

```
apps/mobile/            ← este starter movido para cá
services/core/          ← jarvis-core (depende de openjarvis pinado)
services/api/           ← FastAPI gateway
services/agent/         ← LiveKit Agents worker
services/jobs/          ← worker de automações/proativo
packages/contracts/     ← OpenAPI + schemas RPC → tipos TS gerados
infra/                  ← supabase migrations, docker, deploy
```
Deploy sugerido: worker no **LiveKit Cloud Agents** (ou Fly.io), API/jobs em Fly.io/Render/Cloud Run, Supabase gerenciado. Builds mobile com **EAS Build/Submit**, OTA com EAS Update.

---

## 8. Ordem de implementação da V1

Cada fase termina com algo testável no aparelho.

| # | Fase | Entregas | Depende de |
|---|---|---|---|
| 0 | Fundação | monorepo, Supabase (schema + RLS), CI (lint/tsc/pytest), Sentry, EAS dev build, bundle id próprio, Expo Skills + Mobile MCP configurados | — |
| 1 | Auth + Voz própria | login (Supabase Auth: Apple/Google/e-mail), `POST /livekit/token`, worker LiveKit próprio com STT/LLM/TTS PT-BR, turn detection e interrupção; app conecta ao **nosso** agente | 0 |
| 2 | Conversas + chat texto | `jarvis-core` mínimo, chat SSE, histórico unificado voz+texto, transcrições salvas | 1 |
| 3 | Memória permanente | `PostgresFactStore` + FactExtractor/MemoryService do OpenJarvis, injeção de contexto, tela "Memória" editável, scanner de injeção | 2 |
| 4 | Projetos + arquivos | projetos com instruções, upload, ingestão (chunker/embeddings) → pgvector, RAG com citações | 3 |
| 5 | Pesquisa web + imagens | tool Tavily + fetch com SSRF guard; foto no chat e câmera na chamada (frames → LLM visão) | 2 |
| 6 | Device Action Bridge | RPC + permissões + confirmação; calendário/lembretes/contatos/links/localização do aparelho | 1 |
| 7 | Integrações | OAuth Google (Calendar, Gmail leitura, Tasks) com conectores OpenJarvis; tela de integrações | 2 |
| 8 | Automações + notificações | push (Expo), automações cron/once (scheduler OpenJarvis), ações pendentes com Aprovar/Negar na notificação | 6, 7 |
| 9 | Proativo | morning digest + proactive agent adaptados → push diário; regras de aprovação | 8 |
| 10 | Comercial | RevenueCat + limites, exclusão/export de conta, política de privacidade, revisão das stores, analytics de custo | todas |
| 11 | Hardening | testes E2E com Mobile MCP, carga do worker, latência de voz (< ~800 ms até 1º áudio), testes de segurança (prompt injection por e-mail/web) | todas |

Fases 4–7 podem andar em paralelo depois da 3 (5 e 6 só dependem de 1–2).

---

## 9. Ferramentas de desenvolvimento

- **Expo Skills** (Claude Code): `/plugin install expo@claude-plugins-official` (ou `/plugin marketplace add expo/skills` → `/plugin install expo`); alternativa: `npx skills@latest add expo/skills --skill '*'`. Usar para upgrades de SDK, EAS, config plugins, módulos nativos.
- **Mobile MCP**: `claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest`. Requer emulador Android / simulador iOS ou aparelho conectado (adb / Xcode). **Não funciona neste container de nuvem** (sem adb, emulador nem macOS): rodar numa máquina local (ex.: via Remote Control) ou em device farm (BrowserStack MCP, que precisa ser autorizado).
- Testes: Jest + RNTL no app; pytest no backend (inclusive testes de tools com `respx`); Maestro ou Mobile MCP para fluxos E2E.

---

## 10. Riscos e decisões em aberto

1. **Custo por minuto de voz** (STT+LLM+TTS+LiveKit) — define o preço do plano; medir na Fase 1.
2. **Revisão das lojas**: microfone/câmera/calendário/contatos exigem justificativas claras; acesso a Gmail exige verificação de segurança do Google (CASA) — começar cedo ou deixar Gmail para V1.1.
3. **LGPD**: consentimento, retenção configurável, memória editável, exclusão real (inclui embeddings e arquivos).
4. **Atualização do OpenJarvis** (alpha, muda rápido): pinar commit/versão e encapsular tudo em `jarvis-core`.
5. Background no iOS: o app não roda continuamente → todo proativo nasce no servidor e chega por push.
