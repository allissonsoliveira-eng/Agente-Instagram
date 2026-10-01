# Plataforma de Agentes — Arquitetura

Plataforma multi-tenant para criar **agentes de IA** que atendem leads em **canais de mensagem**: começa no **Instagram**, com várias contas por empresa, e depois vai para o **WhatsApp**. Ela reúne:

- **Automações por palavra-chave**, no estilo ManyChat: comentário, Direct, resposta a story, menção.
- **Agentes de IA** que conversam com o lead e o qualificam.
- **Dashboard** com indicadores de atendimento e de qualificação.
- **Integrações** que enviam os leads qualificados para ferramentas externas (CRM, webhook, planilha).

Mockup navegável das telas: <https://claude.ai/artifact/DisDbvn4APHTN7baTEDpqT>

---

## 1. Conceitos do domínio

| Conceito | O que é |
|---|---|
| **Workspace** | A empresa cliente (tenant). Tudo fica isolado por workspace. |
| **Canal (ChannelAccount)** | Uma conta conectada: um perfil profissional do Instagram ou um número de WhatsApp Business. Um workspace pode ter **várias contas de cada tipo**. |
| **Agente** | Uma "persona" de IA com instruções, tom de voz, base de conhecimento, critérios de qualificação e regras de transferência para um humano. Um workspace pode ter **um ou vários agentes**. |
| **Vínculo agente ↔ canal** | Cada conta tem um **agente padrão**, e um mesmo agente pode atender várias contas. Uma automação pode entregar a conversa a outro agente. |
| **Automação** | Gatilho (tipo + palavras-chave + post/story opcional) seguido de um **fluxo** de passos: mensagem, pergunta, coleta de dado, passar para agente, passar para humano, tag, enviar para integração. |
| **Contato** | A pessoa no canal (IGSID no Instagram, telefone no WhatsApp). |
| **Conversa** | Uma thread entre um contato e uma conta. Estado: `BOT` (automação ou agente), `HUMANO` (atendente assumiu) ou `ENCERRADA`. |
| **Lead** | O contato visto como oportunidade: dados coletados, score, status (`NOVO`, `ENGAJADO`, `QUALIFICADO`, `DESQUALIFICADO`, `ENVIADO`). |
| **Integração** | Um destino externo para os leads (webhook genérico, RD Station, HubSpot, Pipedrive, Google Sheets, Make/Zapier), com mapeamento de campos e novas tentativas. |

---

## 2. Visão de componentes

```
                ┌────────────────────────── Painel (Next.js) ──────────────────────────┐
                │ Dashboard · Conversas · Automações · Agentes · Leads · Integrações · │
                │ Canais                                                                │
                └───────────────▲──────────────────────────────────────────────────────┘
                                │ API interna (REST / Server Actions)
 Meta (Instagram / WhatsApp)    │
   ── webhook ──► [API Webhooks] ──► fila "inbound" ──► [Worker de Eventos]
                    verifica assinatura,                 │ 1. dedup + upsert contato/conversa
                    responde 200 rápido                  │ 2. opt-out? (SAIR/PARAR)
                                                         │ 3. casa automação (palavra-chave)
                                                         │ 4. segue o fluxo OU chama o agente
                                                         │ 5. atualiza lead / score
                                                         ▼
                                       [Guarda de Conformidade do Canal]
                                       janela 24h · 1 resposta privada/comentário ·
                                       limite/hora por conta · opt-out
                                                         │
                                       fila "outbound" ──► [Adapter do canal] ──► Graph API
                                                         │
                         lead qualificado ──► fila "integrations" ──► [Dispatcher] ──► CRM / webhook
                                                                   (HMAC, retry com backoff, log de entregas)

 Postgres (dados) · Redis (filas BullMQ, rate limit, deduplicação) · Claude API (agentes)
```

### Abstração de canal

Todo canal implementa a mesma interface, então o motor de automações e os agentes **não sabem** se a conversa é no Instagram ou no WhatsApp:

```ts
interface ChannelAdapter {
  kind: "INSTAGRAM" | "WHATSAPP";
  parseWebhook(body: unknown): InboundEvent[];          // normaliza eventos
  verifySignature(rawBody: string, header: string): boolean;
  sendText(account, recipientId, text, opts?): Promise<SendResult>;
  sendQuickReplies(account, recipientId, text, options): Promise<SendResult>;
  // Instagram
  sendPrivateReply?(account, commentId, text): Promise<SendResult>;
  replyToComment?(account, commentId, text): Promise<SendResult>;
  // WhatsApp
  sendTemplate?(account, recipientId, template, vars): Promise<SendResult>;
  policy: ChannelPolicy;                                // regras do canal (seção 4)
}
```

`InboundEvent` normalizado: `{ channel, accountExternalId, contactExternalId, type: "DM" | "COMMENT" | "STORY_REPLY" | "STORY_MENTION" | "POSTBACK", text, externalId, mediaId?, commentId?, timestamp }`.

O webhook identifica **qual conta** recebeu o evento (`entry.id` = ID da conta do Instagram, ou `phone_number_id` no WhatsApp) e roteia para o `ChannelAccount` certo do workspace certo. É isso que permite várias contas por empresa.

---

## 3. Stack proposta

| Camada | Escolha | Por quê |
|---|---|---|
| Linguagem | TypeScript | Um só idioma no painel, na API e nos workers |
| Painel + API | Next.js (App Router) | Dashboard e rotas de webhook no mesmo projeto |
| Banco | PostgreSQL + Prisma | Relacional, multi-tenant, JSON para fluxo e dados do lead |
| Filas | Redis + BullMQ | Webhook responde rápido; retries; rate limit por conta |
| IA | Harness próprio, multi-provedor com chave do cliente: Anthropic (`@anthropic-ai/sdk`, padrão `claude-opus-5-5`), OpenAI e Google | Mesmo desenho do agente do CRM; ferramentas só registram a intenção e o código valida |
| Auth do painel | Auth.js ou Supabase Auth | Usuários e papéis por workspace |
| Segredos | Tokens dos canais criptografados no banco (AES-GCM, chave em env/KMS) | Tokens de acesso da Meta são sensíveis |
| Deploy | Web (Vercel/Render/Fly) + worker separado + Postgres e Redis gerenciados | O worker precisa rodar continuamente |

---

## 4. Regras do Instagram (o que a plataforma obriga)

Implementadas na **Guarda de Conformidade**: toda mensagem de saída passa por ela, sem exceção. Os limites numéricos ficam em configuração, porque a Meta muda essas regras. **Antes de lançar, confira tudo na documentação atual da Meta.**

1. **Só responde quem iniciou o contato.** Os gatilhos válidos são DM recebida, comentário, resposta a story, menção em story, ice breaker e link `ig.me`. Nada de abordagem fria nem disparo em massa para quem nunca falou com a conta.
2. **Janela de 24 horas.** Mensagens automáticas só podem ser enviadas até 24h depois da última mensagem do contato. Fora disso, o envio é bloqueado e a conversa fica marcada como "janela encerrada". A tag `HUMAN_AGENT`, que estende a janela para 7 dias, vale **apenas para resposta de um humano**, nunca para a automação.
3. **Resposta privada a comentário.** É **uma** DM por comentário, enviada em até **7 dias** após o comentário. A resposta pública no comentário é opcional e usa variações de texto para não parecer spam.
4. **Limite de envios por conta.** Um contador por hora, por conta, em Redis. Ao chegar perto do limite, os envios entram em fila em vez de serem descartados. O valor é configurável (o padrão inicial é 200 por hora).
5. **Opt-out.** "SAIR", "PARAR" e equivalentes encerram o fluxo e marcam o contato como `optedOut`. Nada mais é enviado a ele.
6. **Sempre existe saída para um humano.** O lead pode pedir para falar com uma pessoa, e o agente transfere a conversa quando as regras dele mandarem.
7. **Ignorar ecos e duplicatas.** Mensagens `is_echo` (enviadas pela própria conta) e eventos repetidos, identificados pelo `mid`/ID do comentário, são descartados.
8. **Segurança do webhook.** O webhook valida `X-Hub-Signature-256` (HMAC SHA-256 com o App Secret), responde ao desafio de verificação (`hub.challenge`) e devolve `200` imediatamente, deixando o processamento para a fila.
9. **Permissões e App Review.** Pelo Login com Instagram, são necessárias `instagram_business_basic`, `instagram_business_manage_messages` e `instagram_business_manage_comments`. Para atender contas de terceiros, é preciso acesso avançado via App Review. A conta precisa ser **profissional** (Empresa ou Criador).
10. **Tokens.** O token de longa duração deve ser renovado antes de expirar. O painel mostra quantos dias faltam e alerta quando o prazo está perto.

### DM para novos seguidores ("Follow to DM"): situação em out/2026

- A Meta lançou em **22/10/2025** um gatilho de "novo seguidor" **em parceria exclusiva com o ManyChat**. É um beta fechado: só algumas contas entram (em geral com mais de ~1.000 seguidores e bom engajamento), há limite de **1 mensagem por pessoa por semana** e a própria comunidade do ManyChat relata falhas e atrasos.
- **A API pública não oferece esse evento.** Os campos de webhook do Instagram são `comments`, `live_comments`, `mentions`, `message_echoes`, `message_reactions`, `messages`, `messaging_handover`, `messaging_optins`, `messaging_policy_enforcement`, `messaging_postbacks`, `messaging_referral`, `messaging_seen`, `response_feedback`, `standby` e `story_insights`. Nenhum deles avisa sobre novos seguidores.
- **Não usar alternativas não oficiais** (scraping ou polling da lista de seguidores, automação de navegador). Elas violam os termos da Meta e podem levar ao bloqueio da conta do cliente.

**Como poderíamos ter acesso (o caminho do ManyChat):** o ManyChat não usa um truque técnico. O recurso é um **beta controlado pela Meta**, e o ManyChat recebe o evento por ser **parceiro oficial da Meta (Meta Business Partner)**. As contas também precisam ser conectadas pelo fluxo novo de conexão do Instagram ("Unified Instagram onboarding"). Para chegar lá:
1. Publicar o app com Login do Instagram e passar pelo **App Review** com acesso avançado às permissões de mensagens e comentários.
2. Ter clientes e volume de mensagens reais na plataforma.
3. Candidatar-se ao **Meta Business Partners** (especialidade em mensagens) ou ao programa de **Tech Provider**, e pedir ao gerente de parceria acesso ao beta de "Follow to DM" ou ao evento de seguidor.
4. Acompanhar o [changelog da Meta](https://developers.facebook.com/docs/instagram-platform/changelog): se o evento entrar na API pública, basta ligar o gatilho.

Não há garantia de aceite nem prazo; a decisão é só da Meta.

**Como a plataforma lida com isso:**
1. **Gatilho `NEW_FOLLOWER` já previsto no modelo**, mas desligado. Ele será ativado se a Meta abrir o evento na API pública ou se conseguirmos acesso como parceiro. Quando existir, aplicar o limite de 1 mensagem por pessoa por semana.
2. **"Exigir seguir" (follow gate)**, que funciona hoje pela API oficial. Quando um lead aciona uma automação (comentário, DM, story), a plataforma consulta o perfil dele (`is_user_follow_business` na User Profile API de mensagens). Se ele ainda não segue a conta, recebe "Siga a gente e toque em *Já segui* para receber o material"; o botão consulta o perfil de novo. É o que o ManyChat também faz para crescer seguidores.
3. **Gatilhos de entrada que trazem seguidores**: comentário com palavra-chave em post ou reels, resposta a story, link `ig.me` com `ref` (em bio e anúncios, via `messaging_referral`) e anúncios Click-to-Instagram-Direct.

Fontes: [webhooks do Instagram Platform](https://developers.facebook.com/docs/instagram-platform/webhooks/), [inDM: auto-DM para novos seguidores](https://indm.social/blog/auto-dm-new-followers-instagram), [comunidade ManyChat: gatilho de novo seguidor](https://community.manychat.com/general-q-a-43/instagram-new-follower-welcome-automation-is-live-but-not-triggering-tried-everything-8245), [SumGenius: Follow to DM do ManyChat](https://sumgenius.ai/blog/manychat-follow-to-dm-not-working/).

### WhatsApp (fase 2): diferenças que a abstração já prevê
- Fora da janela de 24h, só é possível enviar **modelos de mensagem (templates) aprovados**, via `sendTemplate`.
- É preciso ter **opt-in** do contato antes de qualquer mensagem ativa.
- Cada número (`phone_number_id`) é um `ChannelAccount`, então uma empresa pode ter vários.

---

## 5. Automações por palavra-chave (estilo ManyChat)

**Gatilhos:** comentário em post ou reels (todos ou específicos), mensagem no Direct, resposta a story, menção em story, primeira mensagem e link `ig.me` com `ref`. O gatilho **novo seguidor** fica previsto, mas desligado (ver seção 4).

**Correspondência:** `contém`, `exata` ou `começa com`. O texto é normalizado antes da comparação: minúsculas, sem acento, sem pontuação e com espaços colapsados. Se várias automações casarem, vence a de maior **prioridade**, seguida da mais específica (post definido ganha de "qualquer post").

**Passos do fluxo** (guardados como JSON na automação):

| Passo | Efeito |
|---|---|
| `public_reply` | Responde publicamente o comentário, sorteando uma das variações |
| `follow_gate` | Só continua se o lead segue a conta (`is_user_follow_business`); se não segue, pede para seguir e mostra o botão "Já segui" |
| `message` | Envia texto, com botões de resposta rápida opcionais |
| `question` | Pergunta e salva a resposta em `lead.data[campo]`, com validação (e-mail, telefone, texto) e número de tentativas |
| `agent` | Entrega a conversa a um agente de IA, com objetivo definido |
| `human` | Transfere para atendimento humano e notifica a equipe |
| `tag` | Adiciona uma tag ao lead |
| `integration` | Envia o lead para uma integração |

O estado do fluxo fica na conversa (`automationId`, `stepIndex`, `awaitingField`). Quando chega uma nova mensagem do lead, o worker continua o fluxo de onde parou.

---

## 6. Agentes de IA: cadastro e harness

O agente segue o mesmo desenho do agente do **CRM** (`allissonsoliveira-eng/crm`, pasta `lib/agente/`): mesmos campos de cadastro, mesmo harness próprio (sem framework de agentes) e as mesmas garantias de segurança. Por cima disso, acrescenta o que o Instagram e a qualificação de leads pedem. A ideia é que, com o tempo, o núcleo do harness vire um **pacote compartilhado** entre os dois projetos.

### 6.1 Cadastro do agente

| Campo | Limite | Como entra no prompt |
|---|---|---|
| **Nome do agente** | 60 | Abertura: "Você é {nome}, assistente da {empresa} no Instagram…" |
| **Instruções** | 20.000 | `## Quem você é e o que faz` |
| **Regras** (uma por linha) | 10.000 | `## Regras (obrigatórias)`, depois das regras fixas da plataforma e das regras do canal |
| **Conhecimento da empresa** | 60.000 inline | `## Conhecimento da empresa`. Arquivos e links vão para uma base de busca (6.4) |
| **Roteiro do funil** | 20.000 + etapas | `## Roteiro do funil` com a orientação geral e as **etapas estruturadas** (6.3) |
| **Campos do lead** | até 30 | Viram a ferramenta `preencher_campo` (enum dos campos e formato de cada tipo) |
| IA: provedor, chave, modelo, esforço | — | Chave própria do cliente (BYOK): Anthropic, OpenAI ou Google. Chave criptografada com AES-256-GCM e só os 4 últimos dígitos visíveis |
| Funcionamento | — | Contas atendidas; modo **Rascunho** ou **Automático** por conta; janela de agrupamento; divisão de mensagens; travas; quem avisar e quando |

O cadastro tem **versões**: cada vez que é salvo, gera uma nova versão. A execução registra qual versão respondeu, e é possível voltar para uma versão anterior. O CRM não tem isso.

Abas da tela: Configuração · Roteiro do funil · Campos e qualificação · Simulador · Respostas.

### 6.2 Uma rodada do harness, do começo ao fim

1. **Agrupamento das mensagens.** Cada mensagem recebida agenda um job `agente_responder` daqui a N segundos (15 por padrão). Se chegar outra mensagem antes, o job é adiado, e várias mensagens seguidas viram uma resposta só. É o mesmo mecanismo do CRM.
2. **Portões.** O job só segue se a conversa está com o agente, o agente está ligado, a chave existe, o contato não pediu para sair, a **janela de 24h está aberta** e a última mensagem é do contato.
3. **Contexto, em transação de leitura:**
   - **Histórico:** as últimas 100 mensagens, com teto de cerca de 40 mil caracteres. Quando passa disso, entra um **resumo** da parte antiga, gerado em segundo plano e guardado na conversa. O CRM só corta o histórico.
   - **Memória:** os dados do lead (campos preenchidos), a etapa atual do roteiro, as tags e a origem (que automação, que post, que palavra-chave).
4. **Prompt**, na ordem pensada para o cache. Primeiro a parte fixa (bloco com `cache_control`):
   1. Abertura com o nome do agente.
   2. Instruções.
   3. Regras: regras da plataforma, regras do canal Instagram e regras do cliente.
   4. Roteiro do funil.
   5. Conhecimento da empresa.

   Depois vem a parte que muda a cada rodada, `## Contexto desta conversa`:
   - data e hora;
   - origem do lead;
   - etapa atual, com o objetivo dela, o que falta descobrir e o critério para avançar;
   - "O que já sabemos do lead" (`- campo: valor`);
   - trechos da base de conhecimento que a busca encontrou;
   - o tempo que ainda resta da janela de 24h.
5. **Loop de ferramentas.** No máximo 4 rodadas; na última, `tool_choice: none`. Timeout de 45 s, abaixo do limite do job. As ferramentas **só registram a intenção**: são validadas e normalizadas, devolvem `ok` ou `erro: …` para o modelo se corrigir, e nada é gravado durante a chamada (como no CRM). Ferramentas disponíveis:
   - `preencher_campo {campo, valor}`
   - `avancar_etapa {etapa}`: só aceita a próxima etapa e só se o critério de saída for atendido. Os campos obrigatórios da etapa são checados no código, não deixados para o modelo.
   - `enviar_botoes {texto, opcoes[]}` e `enviar_link {link_id}`: só links cadastrados no conhecimento.
   - `marcar_qualificado {resumo}`: o código confere o score e os campos obrigatórios antes de aceitar.
   - `passar_para_humano {motivo, resumo}`: o resumo é interno; o lead não vê.
   - `criar_tarefa {titulo, quando}`
6. **Gravação, em transação de escrita:**
   - Trava a conversa. Se chegou ou saiu alguma mensagem enquanto o agente pensava, a resposta é **descartada** (concorrência otimista).
   - Passa pelas **travas**: frases proibidas e texto com cara de anotação interna.
   - Passa pela **guarda de conformidade do Instagram**: janela de 24h, opt-out e limite de envios por hora.
   - **Rascunho:** salva o texto com as ações; a equipe aprova e escolhe quais ações aplicar.
   - **Automático:** grava a mensagem e aplica as ações (cada uma num SAVEPOINT, a passagem para humano por último). Depois entrega na fila `outbound`.
7. **Envio no Instagram.** A resposta é dividida em até 3 mensagens curtas, com uma pequena pausa entre elas e o indicador de "digitando". Os botões viram respostas rápidas. O CRM não faz essa divisão.
8. **Registro.** Cada rodada grava uma linha em `agente_execucoes`: provedor, modelo, versão do agente, resultado (`enviada | rascunho | trava | descartada | humano | erro`), tokens de entrada, saída e cache, ações e etapa antes e depois. A aba **Respostas** mostra isso.
9. **Erros.** Seguem a mesma classificação do CRM:
   - `limite` e `fora_do_ar` voltam para a fila com backoff.
   - `chave_recusada`, `modelo_invalido` e `recusou` viram execução com erro e avisam a equipe.
   - Na Anthropic, os modelos novos usam `fallbacks: "default"`.

### 6.3 Roteiro do funil estruturado

O CRM guarda o roteiro como um texto livre por funil. Aqui ele tem **duas partes**:

- **Orientação geral** (texto livre), como no CRM.
- **Etapas**, cada uma com:
  - `nome`
  - `objetivo`
  - `descobrir`: os campos que a etapa deve preencher
  - `avança quando`: um critério verificável (campos obrigatórios + score mínimo opcional)
  - `ações ao avançar`: tag, marcar qualificado, enviar ao CRM, avisar a equipe, tarefa

O agente vê **todas** as etapas no prompt fixo, que fica em cache, e a **etapa atual em destaque** no contexto. O código, não o modelo, decide se ele pode avançar. As etapas também alimentam o funil do dashboard: quantos leads há em cada uma.

Exemplo SDR: Abertura → Descoberta → Qualificação → Próximo passo → Encerramento.

### 6.4 Conhecimento da empresa

- **Até cerca de 60 mil caracteres:** fica inline no prompt fixo, que vai para o cache. É igual ao CRM e é barato por causa do cache.
- **Arquivos (PDF), páginas do site e textos maiores:** são divididos em trechos e indexados (Postgres + `pgvector`). A cada rodada, os trechos mais relevantes para a última mensagem entram no contexto. Essa parte é a fase 2 do agente.

### 6.5 Regras fixas da plataforma

Vão sempre no prompt, antes das regras do cliente:

- Tudo o que você escreve vai para o lead; nunca escreva notas internas.
- Mensagens curtas no estilo Direct; uma pergunta por vez.
- Não invente preço, prazo ou condição. O que não estiver no conhecimento, diga que vai confirmar com a equipe.
- Se perguntarem, diga que é um assistente virtual.
- Se o lead pedir para sair, respeite. Se pedir uma pessoa, passe para humano.
- Não peça senha, dados de cartão nem documentos.

As garantias importantes ficam no **código**, não no prompt: opt-out, janela de 24h, limite por hora, critério de avanço de etapa, critério de qualificação e travas.

### 6.6 Simulador

Permite escolher uma origem (automação ou post) e uma etapa e conversar com o agente. A tela mostra o que o agente faria (as ações), o score, as travas acionadas, os tokens e o uso de cache. **Nada é aplicado.**

---

## 7. Integrações (envio do lead qualificado)

- **Eventos:** `lead.qualified` (padrão), `lead.captured` e `conversation.handoff`.
- **Webhook genérico:** POST com JSON, assinado no header `X-Signature` (HMAC SHA-256 com o segredo da integração) e com `Idempotency-Key` igual ao `leadId` + evento.
- **Retries:** até 5 tentativas com backoff exponencial (1 min → 1 h). Cada tentativa vira uma linha em `IntegrationDelivery`, que aparece no painel com a opção "Reenviar".
- **Conectores nativos:** cada um é uma implementação de `IntegrationAdapter.send(lead, config)`. A ordem sugerida é Webhook → RD Station CRM → HubSpot → Pipedrive → Google Sheets.
- **Mapeamento de campos** configurável: `lead.email → email`, e assim por diante.

---

## 8. Modelo de dados (resumo)

```
Workspace 1─N User (papel: owner | admin | atendente)
Workspace 1─N ChannelAccount (kind, externalId, username/phone, tokenCriptografado,
                              tokenExpiraEm, limitePorHora, status, defaultAgentId)
Workspace 1─N Agent (nome ≤60, instrucoes ≤20k, regras ≤10k, conhecimento ≤60k,
                     provedor, modelo, esforco, chaveCifrada, chaveFinal, ligado,
                     travas[], avisarQuando[], agruparSegundos, dividirMensagens, versao)
Agent     1─N AgentVersion (snapshot do cadastro a cada vez que é salvo)
Agent     1─N AgentFunnel (roteiroGeral ≤20k) 1─N FunnelStage (ordem, nome, objetivo,
                     camposAlvo[], criterioSaida JSON, acoesAoAvancar JSON)
Agent     1─N LeadField (rótulo, tipo, opções, obrigatório, pontos, etapa)
Agent     1─N KnowledgeItem (arquivo/link) 1─N KnowledgeChunk (texto, embedding)
Agent     1─N AgentExecution (conversa, versão, resultado, tokens, ações, etapaAntes/Depois)
Conversation 1─1 AgentDraft (texto, ações, alerta)   — modo rascunho
Agent     N─N ChannelAccount (contas que o agente atende + modo rascunho/automático por conta)
Workspace 1─N Automation (channelAccountId?, triggerType, keywords[], matchMode, mediaIds[],
                          flow JSON, prioridade, ativa, contadores)
ChannelAccount 1─N Contact (externalId, username, nome, optedOut, lastInboundAt)
Contact   1─N Conversation (channelAccountId, agentId?, status, flowState JSON)
Conversation 1─N Message (direction, externalId único, texto, autor: CONTATO|AUTOMACAO|AGENTE|HUMANO)
Contact   1─N CommentEvent (commentId único, mediaId, texto, privateReplyAt, publicReplyAt)
Contact   1─1 Lead (status, score, data JSON, tags[], sourceAutomationId, qualifiedAt)
Workspace 1─N Integration (tipo, config JSON, segredo, eventos[], ativa)
Integration 1─N IntegrationDelivery (leadId, status, tentativas, httpStatus, erro, próximaTentativa)
```

---

## 9. Dashboard (tela principal)

Filtros: **conta** (todas, ou uma conta específica de Instagram/WhatsApp) e **período** (hoje, 7 ou 30 dias).

- **KPIs:** conversas iniciadas, leads capturados, leads qualificados, taxa de qualificação e enviados ao CRM (com falhas).
- **Leads por dia:** capturados × qualificados.
- **Funil:** conversas → responderam ao fluxo → leads → qualificados → enviados.
- **Automações:** disparos, leads e qualificados por palavra-chave.
- **Agentes:** volume, taxa de qualificação e conversas aguardando humano.
- **Conformidade:** envios na última hora × limite, bloqueios por janela de 24h, opt-outs e validade do token de cada conta.

---

## 10. Estrutura de pastas proposta

```
src/
  app/                       # Next.js: painel + rotas
    (painel)/dashboard, conversas, automacoes, agentes, leads, integracoes, canais
    api/webhooks/instagram/route.ts
    api/webhooks/whatsapp/route.ts      # fase 2
  channels/
    types.ts                 # ChannelAdapter, InboundEvent, ChannelPolicy
    instagram/               # parse, assinatura, cliente Graph API, policy
    whatsapp/                # fase 2
  compliance/guard.ts        # aplica a policy do canal antes de todo envio
  automation/
    match.ts                 # normalização + casamento de palavra-chave
    flow.ts                  # máquina de passos (pura, testável)
  agente/                    # harness (mesmo desenho de crm/lib/agente)
    ia/                      # ClienteDeIa: anthropic.ts, openai.ts, google.ts
    contexto.ts              # prompt fixo (cacheável) + contexto da conversa
    ferramentas.ts           # definição das ferramentas + anotador (valida intenção)
    roteiro.ts               # etapas, critério de saída, avanço
    conhecimento.ts          # inline + busca em trechos (pgvector)
    trava.ts                 # travas de saída
    responder.ts             # job: portões → contexto → loop → gravação → envio
  leads/qualification.ts     # merge de dados + regra de qualificação
  integrations/
    types.ts, dispatcher.ts, webhook.ts, rdstation.ts, hubspot.ts ...
  queue/                     # filas BullMQ: inbound, outbound, integrations
  worker.ts                  # processo dos workers
prisma/schema.prisma
```

---

## 11. Roadmap

| Fase | Entregas |
|---|---|
| **0. Fundação** | Repositório, Prisma + Postgres, Redis/BullMQ, autenticação, workspaces, criptografia de tokens |
| **1. Instagram (MVP)** | Conectar **várias contas** do Instagram (OAuth), webhook, automações por comentário e DM, resposta privada e pública, guarda de conformidade, inbox com "assumir conversa" |
| **2. Agentes de IA** | Cadastro completo (nome, instruções, regras, conhecimento, roteiro por etapas, campos), harness no padrão do CRM (agrupamento, ferramentas que registram intenção, travas, rascunho/automático), simulador, aba Respostas, versões do cadastro |
| **3. Integrações + Dashboard** | Webhook genérico com HMAC e retries, primeiro CRM nativo, dashboard completo com filtro por conta |
| **4. WhatsApp** | Adapter da Cloud API, vários números, templates, opt-in |
| **5. Escala** | Gatilhos de story, A/B de mensagens, relatórios por agente, papéis e permissões finos, App Review da Meta |

---

## 12. Decisões em aberto

1. **Qual ferramenta externa recebe os leads primeiro?** RD Station, HubSpot, Pipedrive, planilha ou só webhook?
2. **Uso próprio ou SaaS?** Isso define se o app da Meta precisa passar pelo App Review com acesso avançado.
3. **Onde hospedar?** Vercel + Railway, Render, Fly ou AWS?
4. **Base de conhecimento:** só texto e FAQ no início, ou também PDFs e sites (o que exige busca semântica)?
