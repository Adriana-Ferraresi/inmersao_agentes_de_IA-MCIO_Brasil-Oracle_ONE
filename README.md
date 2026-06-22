# HR Buddy — Agente de IA para RH via Telegram

**Documentação técnica do projeto**
Desenvolvido durante a Inmersão Agentes de IA do Programa MCIO Brasil + Alura + Oracle Next Education

![Banner ONE Imersão Agentes de IA](imagens/banner-imersao.png)

---

## 1. Visão geral

O **HR Buddy** é um assistente virtual de RH desenvolvido para a empresa fictícia **ChocolaTech**, integrado ao **Telegram**. O agente combina consulta a um banco de dados relacional (dados estruturados) com uma base de conhecimento vetorial (RAG) para responder dúvidas de colaboradores com precisão, sem inventar informações.

O projeto foi construído de ponta a ponta — desde a concepção do agente até a publicação em ambiente real — ao longo de uma masterclass introdutória e três aulas práticas.

### Objetivo
Permitir que colaboradores da ChocolaTech consultem, via chat, informações pessoais (saldo de férias, banco de horas) e dúvidas gerais sobre políticas de RH, com identificação individual e memória de conversa por usuário.

---

## 2. Estrutura da imersão

| Etapa | Conteúdo | Entregável |
|---|---|---|
| Masterclass | Introdução à IA Agêntica: diferença entre IA comum e Agente de IA; configuração de Cohere e Railway; criação do bot no Telegram | Bot inicial no Telegram |
| Aula 1 | RAG, embeddings e n8n: ingestão de documentos (Load Data Flow) e estruturação do agente | Base de conhecimento vetorial |
| Aula 2 | Memória e dados: integração com MySQL, diferença entre dados estruturados e não estruturados, uso de `$fromAI` | Tabela `funcionarios` + agente híbrido (SQL + RAG) |
| Aula 3 | Produto real: conexão definitiva ao Telegram, Webhook, guardrails e memória por `session ID` | HR Buddy publicado |

---

## 3. Arquitetura da solução

A automação roda inteiramente no **n8n**, organizada em dois fluxos principais:

### 3.1 Fluxo de ingestão de dados (Load Data Flow)
```
[Execute Manualmente] → [HTTP Request] → [Simple Vector Store]
                                              ↑
                                  [Embeddings] + [Default Data Loader]
```
Carrega o **Manual do Colaborador** e as **Políticas de RH** da ChocolaTech (hospedados no GitHub) e converte o conteúdo em embeddings, armazenados em um Vector Store.

![Fluxo de ingestão de dados no n8n - Load Data Flow](imagens/fluxo-ingestao-dados.png)
*Workflow "producao aula 1": execução manual → HTTP Request → Simple Vector Store, alimentado por Embeddings Cohere e Default Data Loader.*

### 3.2 Fluxo do agente em produção
```
[Telegram Trigger] → [AI Agent] → [Send a text message]
                          │
        ┌─────────────┬───┴───────────┬──────────────────┐
   [Chat Model]   [Simple Memory] [Simple Vector Store] [MySQL Tool]
   (Cohere/OpenAI)  (por sessão)   (RAG — políticas RH)  (tabela funcionarios)
                                          │
                                   [Embeddings]
```

**Componentes:**
- **Telegram Trigger** — recebe mensagens do colaborador via Webhook
- **AI Agent** — orquestra o raciocínio, decide quando consultar o banco ou a base de conhecimento
- **Chat Model** — Cohere Chat Model (ou OpenAI, conforme versão)
- **Simple Memory** — mantém o histórico da conversa por `session ID`, isolando o contexto de cada usuário
- **Simple Vector Store (RAG)** — busca semântica nas políticas de RH e no manual do colaborador
- **MySQL Tool (Select rows from a table)** — consulta a tabela `funcionarios`, com parâmetros preenchidos dinamicamente pela IA (`$fromAI`)
- **Send a text message** — envia a resposta de volta ao Telegram

![Fluxo do agente em desenvolvimento no n8n - Aula 2](imagens/fluxo-agente-aula2.png)
*Workflow "producao aula 2": montagem do AI Agent com Chat Model, Memory, Vector Store (RAG) e a tool de MySQL, testado via chat de desenvolvimento.*

![Fluxo do agente publicado no n8n - Aula 3](imagens/fluxo-agente-producao.png)
*Workflow "producao aula 3", já **Published**: Telegram Trigger → AI Agent → Send a text message, com todas as ferramentas conectadas e execução real registrada.*

---

## 4. Banco de dados

Tabela criada para armazenar os dados estruturados dos colaboradores:

Query SQL

```sql
CREATE TABLE funcionarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    departamento VARCHAR(100) NOT NULL,
    cargo VARCHAR(100) NOT NULL,
    data_admissao DATE NOT NULL,
    saldo_ferias INT NOT NULL DEFAULT 0,
    banco_horas DECIMAL(5,1) NOT NULL DEFAULT 0,
    regime VARCHAR(20) NOT NULL DEFAULT 'hibrido'
);

```

A tabela foi populada com dados fictícios de colaboradores de diferentes departamentos (Engenharia, RH, Financeiro, Marketing, Vendas, Operações, TI, Design, Atendimento, Produto), cada um com seu próprio saldo de férias e banco de horas.


Query SQL

```sql
INSERT INTO funcionarios (nome, email, departamento, cargo, data_admissao, saldo_ferias, banco_horas, regime) VALUES
('João Silva', 'joao.silva@empresa.com', 'Engenharia', 'Engenheiro de Software', '2022-03-10', 20, 0.0, 'hibrido'),
('Maria Souza', 'maria.souza@empresa.com', 'Recursos Humanos', 'Analista de RH', '2021-05-15', 5, 12.5, 'hibrido'),
('Carlos Oliveira', 'carlos.oliveira@empresa.com', 'Financeiro', 'Analista Financeiro', '2023-01-20', 0, 0.0, 'presencial'),
('Ana Lima', 'ana.lima@empresa.com', 'Marketing', 'Especialista em Marketing', '2020-11-05', 15, -4.0, 'remoto'),
('Pedro Santos', 'pedro.santos@empresa.com', 'Vendas', 'Executivo de Vendas', '2022-08-01', 10, 8.0, 'hibrido'),
('Fernanda Costa', 'fernanda.costa@empresa.com', 'Operações', 'Gerente de Operações', '2019-02-12', 30, 0.0, 'presencial'),
('Rafael Mendes', 'rafael.mendes@empresa.com', 'TI', 'Analista de Suporte', '2023-06-10', 0, 15.5, 'hibrido'),
('Juliana Rocha', 'juliana.rocha@empresa.com', 'Engenharia', 'Desenvolvedora Front-end', '2021-09-25', 12, 0.0, 'remoto'),
('Bruno Alves', 'bruno.alves@empresa.com', 'Design', 'Designer UX/UI', '2022-04-18', 8, 3.5, 'hibrido'),
('Camila Ferreira', 'camila.ferreira@empresa.com', 'Atendimento', 'Analista de Atendimento', '2024-01-05', 0, 0.0, 'hibrido'),
('Eric Monné', 'eric.monne@chocolatech.com', 'Produto', 'Instrutor de Cursos', '2024-01-15', 25, 8.0, 'hibrido');

```

## 5. System Prompt do agente

O comportamento do HR Buddy é controlado por um *system prompt* com regras claras de identificação e uso de fontes de dados:

````
Você é o HR Buddy, assistente virtual de RH da ChocolaTech.

"REGRAS:
- Sempre responda em português.
- Responda APENAS dúvidas relacionadas a RH.

IDENTIFICAÇÃO DO FUNCIONÁRIO:
- Se o usuário não disser quem é, pergunte o nome completo dele logo
  na primeira mensagem.
- Use a ferramenta MySQL para buscar na tabela funcionarios usando
  SEMPRE o NOME COMPLETO informado pelo usuário na conversa.
- Se encontrado: use os saldos de férias e banco de horas.
- Se não encontrado: não invente dados pessoais. Responda apenas com
  base nas políticas gerais de RH do Vector Store.
- Use a base de conhecimento para dúvidas gerais."

````
Esse desenho garante que o agente **nunca alucina dados pessoais**: ele só responde com números reais quando encontra o colaborador no banco; caso contrário, recorre exclusivamente à base de conhecimento (RAG) para informações genéricas.

---

## 6. Guardrails e memória por sessão

Implementados na Aula 3, como etapa de "produto real":

- **Guardrails**: filtros que mantêm o agente restrito ao escopo de RH, evitando respostas fora de contexto.
- **Memória por `session ID`**: cada conversa no Telegram gera uma sessão isolada, permitindo que múltiplos colaboradores conversem com o mesmo bot simultaneamente sem misturar contextos.
- **Webhook**: o Telegram Trigger expõe um Webhook que aciona o fluxo do n8n em tempo real a cada nova mensagem.

---

## 7. Exemplo de interação real

```
Usuário: Eric Monné, e quero saber quantos dias de férias eu tenho
HR Buddy: Olá, Eric! Você tem 25 dias de férias disponíveis.
          Posso te ajudar com mais alguma coisa?

Usuário: sou o Erik Silva
HR Buddy: Olá, Erik Silva! Como posso te ajudar hoje?

Usuário: quantos dias de férias eu tenho
HR Buddy: Olá, Erik Silva! Infelizmente, não consegui encontrar seu
          registro no sistema. Se precisar de informações gerais
          sobre férias, posso te ajudar com base nas políticas
          da empresa.

Usuário: Olá sou a Ana Lima
HR Buddy: Olá, Ana Lima! Você tem 15 dias de férias disponíveis.
```

Esse exemplo evidencia o comportamento esperado: quando o nome completo bate com um registro real, o agente responde com dados precisos; quando não encontra o colaborador, evita inventar e oferece apenas informação geral.

<table>
<tr>
<td><img src="imagens/chat-exemplo-1.png" alt="Conversa real com o HR Buddy no Telegram - parte 1" width="400"></td>
<td><img src="imagens/chat-exemplo-2.png" alt="Conversa real com o HR Buddy no Telegram - parte 2" width="400"></td>
</tr>
</table>

*Conversa real no Telegram: o HR Buddy pede o nome completo, encontra Eric e Ana Lima na base (respondendo com saldo real de férias) e, para "Erik Silva" (sem registro), evita inventar dados e oferece apenas informação geral de política.*

---

## 8. Stack tecnológica

| Camada | Ferramenta |
|---|---|
| Orquestração / automação | [n8n](https://n8n.io/) |
| Modelo de linguagem | Cohere Chat Model / OpenAI (https://cohere.com)|
| Embeddings | Cohere Embeddings / OpenAI Embeddings (https://cohere.com)|
| Banco vetorial (RAG) | Simple Vector Store (n8n) |
| Banco de dados relacional | MySQL (Railway)|
| Canal de comunicação | Telegram [Bot API + Webhook](https://web.telegram.org) |
| Hospedagem | [Railway](https://railway.com) |
| Fonte de documentos | GitHub ([Manual do Colaborador e Políticas de RH](https://raw.githubusercontent.com/ericmonne/chocolatech-imersao/refs/heads/main/Manual%20de%20RH%20ChocolaTech.txt)) | [GitHub com os arquivos de CHOCOLATECH](https://github.com/ericmonne/chocolatech-imersao)

---

## 9. Resultados

- ✅ Agente de IA funcional, publicado e testado em cenário real de RH
- ✅ Combinação de dados estruturados (SQL) e não estruturados (RAG) para respostas mais inteligentes
- ✅ Identificação individual de colaboradores e memória de conversa isolada por sessão
- ✅ Guardrails para manter o agente dentro do escopo de RH
---

## 10. Possíveis melhorias futuras

- Autenticação mais robusta do colaborador (ex.: e-mail corporativo + confirmação)
- Logs e métricas de uso do agente (perguntas mais frequentes, taxa de identificação)
- Expansão da base de conhecimento (benefícios, políticas de remoto, onboarding)
- Suporte a outros canais além do Telegram (WhatsApp, Slack, Microsoft Teams)
- Tratamento de erros e fallback mais detalhado em caso de falha na consulta ao MySQL

---

*Projeto desenvolvido durante a Inmersão Agentes de IA (Alura + Oracle Next Education - Junho/2026).*
*Instrutores: Eric Monné, Giovani Polonio Ticianelli e Guilherme Lima.*
