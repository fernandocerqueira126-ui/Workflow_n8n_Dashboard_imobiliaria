# 🏠 Imobiliária Neemias — CRM Inteligente com IA no WhatsApp

Sistema completo de atendimento automatizado via WhatsApp com CRM em tempo real, construído sobre **n8n**, **Supabase**, **Evolution API** e **Lovable**.

---

## 📋 Visão Geral

A **Camila** é a consultora virtual da Imobiliária Neemias. Ela atende clientes no WhatsApp 24/7, apresenta imóveis, agenda visitas com corretores, reagenda quando necessário e alimenta automaticamente o CRM com todas as informações coletadas durante o atendimento.

```
Cliente envia mensagem no WhatsApp
        ↓
Evolution API captura via Webhook
        ↓
n8n processa → Camila (IA) responde
        ↓
Supabase armazena tudo em tempo real
        ↓
Dashboard Lovable exibe no CRM
```

---

## ✨ Funcionalidades

### Atendimento Inteligente
- Qualificação automática do lead (nome completo + email) no primeiro contato
- Perguntas de qualificação antes de enviar imóveis (tipo, quartos, faixa de valor)
- Envio de cards de imóveis com imagem, descrição, valor e link para detalhes
- Call-to-Action após os cards para o cliente escolher o imóvel de interesse
- Transcrição de mensagens de áudio via Groq (Whisper)
- Suporte a texto e áudio

### Agendamento de Visitas
- Verificação de disponibilidade de horário antes de confirmar
- Cadastro automático do agendamento no Supabase
- Reagendamento de visitas (cancela o anterior, cria o novo)
- Lembrete automático 1 hora antes da visita via WhatsApp
- Salvamento do resumo da visita no card do lead (imóvel, data, hora, corretor)

### CRM e Pipeline
- Cadastro automático de leads no primeiro contato com `status: Novo Lead`
- Movimentação automática no pipeline Kanban:

| Etapa | Como é movido |
|---|---|
| Novo Lead | n8n — primeiro contato |
| Contato Inicial | IA (após qualificação) |
| Visita Marcada | Trigger automático no Supabase |
| Proposta Enviada | IA (quando enviada proposta) |
| Documentação/Análise | IA (quando cliente envia docs) |
| Fechado/Contrato | IA (contrato assinado) |
| Perdido | IA (cliente desistiu) |

- Proteção contra regressão de etapas (trigger no banco)
- Salvamento de email, nome completo e observações no card do lead
- Referência do imóvel de interesse salva automaticamente

### Segurança e Qualidade
- Anti-spam — bloqueia mensagens duplicadas do mesmo número em 4 segundos
- Horário de atendimento configurável (padrão: Seg–Sáb 8h–20h)
- Handoff para corretor humano quando solicitado ou necessário
- Relatório semanal automático toda segunda às 8h via WhatsApp
- Avaliação pós-visita enviada automaticamente 1h após o horário da visita

---

## 🛠️ Stack Tecnológica

| Componente | Tecnologia |
|---|---|
| Automação / Workflow | [n8n](https://n8n.io) (self-hosted) |
| Banco de Dados | [Supabase](https://supabase.com) (PostgreSQL) |
| WhatsApp API | [Evolution API](https://evolution-api.com) |
| Dashboard / CRM | [Lovable](https://lovable.dev) |
| IA Principal | Google Gemini (via n8n AI Agent) |
| Transcrição de Áudio | Groq — Whisper Large v3 Turbo |
| Busca Semântica (RAG) | PGVector + Embeddings OpenAI |
| Memória de Conversa | Redis |
| Cache / Anti-spam | Redis |

---

## 🗂️ Estrutura do Banco de Dados (Supabase)

### Tabela `leads`
| Coluna | Tipo | Descrição |
|---|---|---|
| id | uuid | Chave primária |
| name | text | Nome do cliente |
| phone | text | Telefone (com DDI 55) |
| email | text | Email coletado pela Camila |
| status | text | Etapa do pipeline Kanban |
| description | text | Resumo da visita agendada |
| link_imovel_interesse | text | Código do imóvel de interesse |
| value | numeric | Valor estimado |
| source | text | Origem (WhatsApp, Instagram, etc.) |
| created_at | timestamptz | Data de criação |

### Tabela `agendamentos`
| Coluna | Tipo | Descrição |
|---|---|---|
| id | uuid | Chave primária |
| lead_id | uuid | FK → leads |
| colaborador_id | uuid | FK → colaboradores |
| cliente_nome | text | Nome do cliente |
| cliente_telefone | text | Telefone do cliente |
| data | text | Data da visita (YYYY-MM-DD) |
| horario | text | Horário da visita (HH:mm:ss) |
| servico | text | Código do imóvel |
| status | text | agendado / cancelado / lembrete_enviado / avaliacao_enviada |
| created_at | timestamptz | Data de criação |

### Tabela `colaboradores`
Corretores e colaboradores da imobiliária com nome, email, telefone e cargo.

### Triggers automáticos no banco
- `trigger_auto_move_lead_on_agendamento` — move lead para "Visita Marcada" ao criar agendamento
- `trigger_protect_lead_regression` — impede lead de voltar para etapa anterior no pipeline

---

## 📦 Workflows n8n

| Arquivo | Descrição |
|---|---|
| `IMOBILIARIA_NEEMIAS_V6_4_2.json` | Workflow principal — atendimento completo |
| `IMOBILIARIA_FOLLOWUP.json` | Lembrete 1h antes da visita |
| `IMOBILIARIA_POS_VISITA.json` | Pesquisa de satisfação pós-visita |
| `IMOBILIARIA_RELATORIO_SEMANAL.json` | Relatório semanal toda segunda às 8h |

---

## 🤖 Tools do AI Agent (Camila)

| Tool | Função |
|---|---|
| `Postgres PGVector Store1` | Busca semântica de imóveis no RAG |
| `Think1` | Raciocínio interno antes de responder |
| `Atualizar Pipeline Lead` | Move lead entre etapas do Kanban |
| `Salvar Qualificação Inicial` | Salva nome completo + email do lead |
| `Salvar Resumo da Visita` | Salva descrição da visita no card do lead |

---

## 🔄 Fluxo Completo de Atendimento

```
1. Cliente envia "oi" no WhatsApp
2. Lead cadastrado no Supabase (status: Novo Lead)
3. Camila verifica se tem email cadastrado
   → NÃO: "Olá! Para iniciar, me informe nome completo e email 📧"
   → SIM: inicia atendimento
4. Cliente informa nome + email
   → Camila salva via tool → move para "Contato Inicial"
5. Cliente pede imóveis
   → Camila qualifica: tipo? quartos? faixa de valor?
6. Camila envia cards dos imóveis filtrados
7. CTA: "Qual deles vamos conhecer primeiro?"
8. Cliente escolhe imóvel → Camila coleta data e hora
9. Camila verifica disponibilidade no Supabase
   → Ocupado: sugere outro horário
   → Livre: confirma agendamento
10. Agendamento salvo → trigger move lead para "Visita Marcada"
11. Resumo salvo no card: "Visita agendada para CA522 em 20/03/2026 às 17h com Carlos Mendes"
12. 1h antes da visita: lembrete automático via WhatsApp
13. 1h após a visita: pesquisa de satisfação (1 a 5 estrelas)
```

---

## ⚙️ Configuração

### Pré-requisitos
- n8n instalado (self-hosted ou cloud)
- Conta Supabase com projeto criado
- Evolution API configurada e conectada ao WhatsApp
- Credenciais Google Gemini (AI Agent)
- Credenciais OpenAI (Embeddings para RAG)
- Groq API Key (transcrição de áudio)
- Redis (memória de conversa + anti-spam)

### Variáveis a configurar nos workflows

Nos nós de credencial do n8n:
- `Supabase imobiliaria-pro-crm` — URL e chave do projeto Supabase
- `Evolution Alavancaai` — URL e API Key da Evolution API
- `Google Gemini` — API Key
- `OpenAI` — API Key (para embeddings)
- `Groq` — API Key

Nos nós configuráveis:
- `instanceName` nos nós Evolution API → nome da sua instância (ex: `fortuna`)
- Número do corretor para notificações de handoff e alertas de erro
- Horário de atendimento no nó `Horário de Atendimento?` (padrão: Seg–Sáb 8h–20h, fuso Fortaleza)

### SQL necessário no Supabase
```sql
-- Adicionar coluna status na tabela leads (se não existir)
ALTER TABLE leads ADD COLUMN IF NOT EXISTS status text DEFAULT 'Novo Lead';

-- Adicionar coluna email (se não existir)  
ALTER TABLE leads ADD COLUMN IF NOT EXISTS email text;

-- Função RPC para mover lead para Contato Inicial com segurança
-- (criada pelo Lovable com proteção de regressão)
```

---

## 📊 Dashboard (Lovable)

O CRM foi construído no Lovable com acesso direto ao Supabase e inclui:

- **Leads CRM** — Kanban com todas as etapas do funil
- **Visitas Agendadas** — Calendário com visitas do mês
- **Atendimentos** — Histórico de interações
- **Corretores** — Gestão da equipe
- **Financeiro** — Controle de valores
- **Relatórios** — Métricas e conversão

---

## 📈 Versões

| Versão | Principal mudança |
|---|---|
| V2.2.9 | Agendamento, reagendamento e follow-up funcionando |
| V3.1 | Handoff humano e pipeline como tools do agente |
| V4.2.3 | Cadastro de leads no primeiro contato |
| V5.2.0 | Integração com RPCs do Supabase para pipeline |
| V6.2.2 | Captura de email e resumo da visita |
| V6.4.2 | Gate de qualificação + pipeline completo automatizado |

---

## 👤 Autor

Desenvolvido por **Fernando Borges Cerqueira** para **Imobiliária Neemias**.

---

## 📄 Licença

Projeto proprietário — todos os direitos reservados.
