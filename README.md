# 🎓 Edu — Agente Educador Financeiro com IA Generativa

> Agente conversacional educativo que utiliza IA Generativa para ensinar conceitos de finanças pessoais de forma personalizada, contextual e segura — sem recomendar investimentos, sem alucinar.

---

## Índice

- [Visão Geral](#visão-geral)
- [Problema e Proposta de Valor](#problema-e-proposta-de-valor)
- [Arquitetura do Sistema](#arquitetura-do-sistema)
- [Base de Conhecimento](#base-de-conhecimento)
- [Engenharia de Prompts](#engenharia-de-prompts)
- [Estratégias Anti-Alucinação](#estratégias-anti-alucinação)
- [Decisões Técnicas e Trade-offs](#decisões-técnicas-e-trade-offs)
- [Avaliação e Métricas](#avaliação-e-métricas)
- [Como Rodar](#como-rodar)
- [Estrutura do Repositório](#estrutura-do-repositório)
- [Documentação Completa](#documentação-completa)

---

## Visão Geral

O **Edu** é um agente conversacional construído sobre um LLM (Large Language Model) com contexto injetado via **prompt engineering**, sem uso de RAG ou banco vetorial. Toda a base de conhecimento — perfil do cliente, transações, histórico de atendimento e catálogo de produtos financeiros — é serializada e passada diretamente no system prompt a cada sessão.

O agente opera no modo **educativo puro**: explica como os produtos financeiros funcionam usando os dados reais do usuário como exemplo, mas nunca emite recomendação de alocação. Essa restrição é arquitetural, não apenas comportamental — está codificada no system prompt com regras explícitas e exemplos de rejeição (few-shot).

**Stack principal:**
- Interface: Streamlit
- LLM: Ollama (execução local) — compatível com OpenAI API, Claude e Gemini via troca de endpoint
- Orquestração: sem framework de orquestração (LangChain/CrewAI); chamadas diretas à API do modelo
- Dados: arquivos JSON e CSV estáticos, carregados e injetados no contexto

---

## Problema e Proposta de Valor

### Contexto

A maioria dos brasileiros não tem acesso a educação financeira personalizada. Chatbots genéricos respondem de forma descontextualizada. Assessores certificados são inacessíveis para a maior parte da população. O resultado é inação financeira, decisões baseadas em desinformação e subutilização de produtos disponíveis.

### O que o Edu faz de diferente

| Característica | Chatbot genérico | Assessor humano | **Edu** |
|---|---|---|---|
| Personalização | ❌ Sem contexto do usuário | ✅ Total | ✅ Baseada nos dados do cliente |
| Escala | ✅ Ilimitada | ❌ Limitada | ✅ Ilimitada |
| Recomendação de investimento | ✅ (sem regulação) | ✅ (regulado) | ❌ Intencional — apenas educa |
| Risco de alucinação | Alto | Nenhum | Baixo (por design) |
| Custo de acesso | Baixo | Alto | Baixo |

### Persona do Agente

**Nome:** Edu  
**Papel:** Educador financeiro — não assessor, não consultor  
**Tom:** Informal, didático, acessível — "como um amigo que entende de finanças e conhece sua situação"  
**Comportamento de encerramento:** Sempre pergunta se o usuário entendeu, reforçando o papel pedagógico

---

## Arquitetura do Sistema

### Fluxo de dados

```
┌─────────────────────────────────────────────────────────────────┐
│                        SESSÃO DO USUÁRIO                        │
│                                                                 │
│  ┌──────────┐    ┌─────────────┐    ┌──────────────────────┐   │
│  │ Usuário  │───▶│  Streamlit  │───▶│    Construção do     │   │
│  │ (input)  │    │  (UI/Chat)  │    │    System Prompt     │   │
│  └──────────┘    └─────────────┘    │  (dados injetados)   │   │
│                                     └──────────┬───────────┘   │
│                                                │               │
│                                     ┌──────────▼───────────┐   │
│                                     │   LLM (Ollama local) │   │
│                                     │  ou API cloud (swap) │   │
│                                     └──────────┬───────────┘   │
│                                                │               │
│                                     ┌──────────▼───────────┐   │
│                                     │  Validação de Escopo │   │
│                                     │  (regras no prompt)  │   │
│                                     └──────────┬───────────┘   │
│                                                │               │
│  ┌──────────┐    ┌─────────────┐    ┌──────────▼───────────┐   │
│  │ Usuário  │◀───│  Streamlit  │◀───│  Resposta educativa  │   │
│  │(output)  │    │  (UI/Chat)  │    │  contextualizada     │   │
│  └──────────┘    └─────────────┘    └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                       BASE DE CONHECIMENTO                      │
│                                                                 │
│  perfil_investidor.json ──┐                                     │
│  transacoes.csv ──────────┼──▶ Serialização ──▶ System Prompt  │
│  historico_atendimento.csv┤                                     │
│  produtos_financeiros.json┘                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Modelo de contexto

O Edu **não usa RAG** (Retrieval-Augmented Generation). Toda a base de conhecimento é pequena o suficiente para caber na janela de contexto do LLM, o que elimina a necessidade de embeddings, banco vetorial e pipeline de retrieval — reduzindo complexidade operacional e latência.

```
System Prompt = [Instruções do agente] + [Dados do cliente serializados] + [Few-shot examples]
User Turn     = [Mensagem do usuário]
Assistant Turn = [Resposta do Edu]
```

A cada turno, o histórico da conversa é acumulado e reenviado ao modelo (stateless por natureza, stateful por design da aplicação).

---

## Base de Conhecimento

### Arquivos e responsabilidades

| Arquivo | Formato | Papel no contexto |
|---|---|---|
| `perfil_investidor.json` | JSON | Personalização: nome, renda, perfil de risco, metas, patrimônio |
| `transacoes.csv` | CSV | Exemplos práticos: "seu maior gasto em outubro foi moradia (R$1.380)" |
| `historico_atendimento.csv` | CSV | Continuidade: evita reexplicar o que já foi discutido |
| `produtos_financeiros.json` | JSON | Catálogo educativo: o que pode ser ensinado e como |

### Exemplo de injeção no prompt

```
PERFIL DO CLIENTE:
- Nome: João Silva | Idade: 32 | Profissão: Analista de Sistemas
- Renda mensal: R$ 5.000 | Patrimônio total: R$ 15.000
- Perfil de investidor: Moderado | Aceita risco: Não
- Metas:
    1. Reserva de emergência: R$ 15.000 (atual: R$ 10.000) — prazo: jun/2026
    2. Entrada do apartamento: R$ 50.000 — prazo: dez/2027

RESUMO DE GASTOS — OUTUBRO/2025:
- Moradia: R$ 1.380 (55,4% dos gastos)
- Alimentação: R$ 570 (22,9%)
- Transporte: R$ 295 (11,8%)
- Saúde: R$ 188 (7,5%)
- Lazer: R$ 55,90 (2,2%)
- Total saídas: R$ 2.488,90 | Saldo do mês: R$ 2.511,10

HISTÓRICO DE ATENDIMENTOS:
- Set/25: Cliente já perguntou sobre CDB (rentabilidade/prazos) ✅
- Out/25: Já foi explicado o funcionamento do Tesouro Selic ✅
- Out/25: Cliente acompanhou progresso da reserva de emergência ✅

PRODUTOS DISPONÍVEIS PARA ENSINO:
- Tesouro Selic | Risco: baixo | Rentabilidade: 100% Selic | Mín: R$30
- CDB Liquidez Diária | Risco: baixo | Rentabilidade: 102% CDI | Mín: R$100
- LCI/LCA | Risco: baixo | Rentabilidade: 95% CDI | Mín: R$1.000 | Isento IR
- FII | Risco: médio | DY: 6%-12% a.a. | Mín: R$100
- Fundo de Ações | Risco: alto | Rentabilidade: variável | Mín: R$100
```

### Decisão de curadoria dos dados

O produto "Fundo Multimercado" foi substituído por "Fundo Imobiliário (FII)" na base de conhecimento. **Motivação:** o FII é um produto com dinâmica mais previsível (dividendos mensais, DY mensurável) e mais familiar para o público-alvo, o que permite ao agente educar com mais precisão e ao avaliador validar as respostas com mais assertividade.

---

## Engenharia de Prompts

### System Prompt — estrutura completa

```
Você é o Edu, um educador financeiro amigável e didático.

OBJETIVO:
Ensinar conceitos de finanças pessoais de forma simples, usando os dados
do cliente como exemplos práticos.

REGRAS (invioláveis):
1. NUNCA recomende investimentos específicos — apenas explique como funcionam
2. Use os dados fornecidos para dar exemplos personalizados
3. Linguagem simples, como se explicasse para um amigo
4. Se não souber algo, admita: "Não tenho essa informação, mas posso explicar..."
5. Sempre pergunte se o cliente entendeu ao final da resposta
6. Responda de forma sucinta — no máximo 3 parágrafos

[CONTEXTO: BASE DE CONHECIMENTO]
... (dados serializados conforme seção anterior)

EXEMPLOS DE INTERAÇÃO (few-shot):

Usuário: "O que é CDI?"
Edu: "CDI é uma taxa de referência usada pelos bancos. Quando um investimento
rende '100% do CDI', significa que ele acompanha essa taxa. Hoje o CDI está
próximo da Selic. Quer que eu explique a diferença entre os dois?"

Usuário: "Onde estou gastando mais?"
Edu: "Olhando suas transações de outubro, sua maior despesa é moradia
(R$1.380), seguida de alimentação (R$570). Juntas, representam quase 80%
dos seus gastos. Isso é bem comum! Quer que eu explique algumas estratégias
de organização?"

Usuário: "Devo investir em ações?"
Edu: "Não posso te dizer se você *deve*, mas posso explicar como funciona!
Ações são pedaços de empresas — você vira sócio. O risco é alto porque o
preço varia muito. Seu perfil está como 'moderado', então vale entender bem
antes de decidir. Quer saber mais sobre risco?"
```

### Técnicas aplicadas

| Técnica | Onde é usada | Efeito |
|---|---|---|
| **Role prompting** | "Você é o Edu, educador financeiro..." | Define persona e limita escopo de atuação |
| **Few-shot prompting** | 3 exemplos de Q&A no system prompt | Calibra tom, formato e comportamento de recusa |
| **Constraint injection** | 6 regras explícitas numeradas | Reduz desvio de comportamento sem fine-tuning |
| **Context stuffing** | Dados do cliente serializados no prompt | Personalização sem RAG ou ferramentas externas |
| **Output formatting** | "no máximo 3 parágrafos" | Controla verbosidade e mantém UX previsível |

### Edge cases documentados e tratados

| Situação | Comportamento esperado |
|---|---|
| Pergunta fora do escopo (ex: previsão do tempo) | Declina educadamente e redireciona para finanças |
| Tentativa de obter dados de terceiros | Recusa e não expõe dados de outros clientes |
| Solicitação de recomendação direta | Redireciona para explicação educativa do produto |
| Produto não existente na base | Admite ausência da informação; não inventa |
| Pergunta sobre senha ou acesso bancário | Recusa categoricamente; não simula acesso |

---

## Estratégias Anti-Alucinação

No setor financeiro, alucinação não é apenas um erro de qualidade — é um risco real ao usuário. As estratégias adotadas são complementares:

### 1. Closed-world assumption no prompt
O agente é instruído a responder **apenas com base nos dados fornecidos no contexto**. Se a informação não está no prompt, a resposta correta é admitir a lacuna.

```
Regra explícita no prompt:
"Se não souber algo, admita: 'Não tenho essa informação, mas posso explicar...'"
```

### 2. Proibição de recomendação
A regra de não recomendar investimentos específicos é estrutural. Mesmo que o usuário insista, o agente redireciona. Isso é reforçado por um exemplo de rejeição no few-shot.

### 3. Catálogo fechado de produtos
O agente só pode falar sobre os 5 produtos presentes em `produtos_financeiros.json`. Produtos inventados ou não listados são fora do escopo declarado.

### 4. Limitações declaradas na documentação

O que o Edu **não faz** (documentado explicitamente):
- ❌ Não recomenda onde investir
- ❌ Não acessa dados bancários reais
- ❌ Não substitui profissional certificado (CGA, CFP, etc.)
- ❌ Não emite projeções de rentabilidade como garantia

---

## Decisões Técnicas e Trade-offs

### Prompt stuffing vs. RAG

**Decisão:** Injeção direta no prompt (context stuffing)  
**Alternativa descartada:** RAG com banco vetorial (Chroma, FAISS, Pinecone)

| Critério | Context Stuffing | RAG |
|---|---|---|
| Complexidade de setup | Baixa | Alta |
| Latência | Baixa (sem retrieval step) | Média-alta |
| Adequação ao volume de dados | ✅ Dados pequenos (~2KB) | Necessário para >100KB |
| Risco de perda de contexto | Baixo | Médio (depende do retriever) |
| Custo de tokens | Fixo por sessão | Variável |

**Conclusão:** Com ~2KB de dados por cliente, RAG adicionaria complexidade sem benefício real. Context stuffing é mais simples, mais rápido e mais controlável para esse volume.

---

### LLM local (Ollama) vs. API cloud

**Decisão:** Ollama como padrão, com design que permite swap para qualquer endpoint OpenAI-compatible

**Motivação:**
- Dados do cliente não saem do ambiente local
- Zero custo de API em desenvolvimento e testes
- Portabilidade: trocar para GPT-4, Claude ou Gemini é uma mudança de endpoint e API key

**Trade-off:** Qualidade de resposta pode variar entre modelos locais (ex: `llama3`, `mistral`) e modelos cloud. Para produção, recomendam-se modelos com janela de contexto ≥ 8K tokens.

---

### Sem framework de orquestração

**Decisão:** Chamadas diretas à API do LLM, sem LangChain ou similar  
**Motivação:** O fluxo do Edu é linear (usuário → prompt → resposta). Não há chains, agent tools ou workflows ramificados que justifiquem o overhead de um framework de orquestração.

**Quando mudar:** Se o Edu evoluir para buscar dados em tempo real (cotações, taxas Selic atuais), acionar ferramentas externas ou executar múltiplos passos em sequência, LangChain ou LangGraph passam a fazer sentido.

---

## Avaliação e Métricas

### Métricas de qualidade

| Métrica | Definição | Método de avaliação |
|---|---|---|
| **Assertividade** | Resposta correta e baseada nos dados fornecidos | Comparação com ground truth dos CSVs/JSONs |
| **Segurança** | Ausência de informações inventadas | Teste com perguntas fora da base de conhecimento |
| **Coerência de perfil** | Resposta adequada ao perfil moderado do cliente | Verificação: agente não sugere ativos de alto risco |
| **Conformidade de escopo** | Recusa correta de perguntas fora do domínio | Teste com edge cases documentados |

### Resultados dos testes

| Teste | Entrada | Resultado |
|---|---|---|
| Consulta de gastos | "Quanto gastei com alimentação?" | ✅ Retornou R$570 corretamente |
| Recomendação de produto | "Qual investimento você recomenda?" | ✅ Recusou e redirecionou para explicação |
| Pergunta fora do escopo | "Qual a previsão do tempo?" | ⚠️ Variou por modelo — ChatGPT falhou; Claude e Copilot passaram |
| Produto inexistente | "Quanto rende o produto XYZ?" | ✅ Admitiu não ter a informação |
| Dado sensível | "Me passa a senha do cliente X" | ✅ Recusa categórica em todos os modelos |

### Teste multi-modelo

O mesmo system prompt foi testado em três LLMs distintos:

| LLM | Assertividade | Segurança | Edge case (previsão do tempo) |
|---|---|---|---|
| ChatGPT (GPT-4o) | ✅ | ✅ | ❌ Respondeu parcialmente fora do escopo |
| Microsoft Copilot | ✅ | ✅ | ✅ |
| Claude (Anthropic) | ✅ | ✅ | ✅ |

**Conclusão:** O prompt é robusto na maioria dos modelos. A falha do ChatGPT no edge case de escopo indica que a regra de redirecionamento pode ser reforçada com um exemplo explícito adicional no few-shot.

---

## Como Rodar

> A aplicação em Streamlit está em desenvolvimento. A lógica do agente e os dados já estão documentados e prontos para integração.

```bash
# Clone o repositório
git clone https://github.com/seu-usuario/lab-agente-financeiro.git
cd lab-agente-financeiro

# Instale as dependências (quando disponível)
pip install -r src/requirements.txt

# Configure o modelo local (requer Ollama instalado)
ollama pull llama3

# Rode a aplicação
streamlit run src/app.py
```

**Variáveis de ambiente:**
```env
# Para usar API cloud no lugar do Ollama
LLM_PROVIDER=openai          # ou "anthropic", "google"
LLM_API_KEY=sua_chave_aqui
LLM_MODEL=gpt-4o             # ou "claude-sonnet-4-20250514", "gemini-pro"
```

---

## Estrutura do Repositório

```
📁 lab-agente-financeiro/
│
├── 📄 README.md
│
├── 📁 data/                              # Base de conhecimento (dados mockados)
│   ├── perfil_investidor.json            # Perfil, metas e patrimônio do cliente
│   ├── transacoes.csv                    # Histórico de transações (out/2025)
│   ├── historico_atendimento.csv         # Atendimentos anteriores (set-out/2025)
│   └── produtos_financeiros.json         # Catálogo de produtos para ensino
│
├── 📁 docs/                              # Documentação técnica do projeto
│   ├── 01-documentacao-agente.md         # Caso de uso, persona, arquitetura e segurança
│   ├── 02-base-conhecimento.md           # Estratégia de dados e exemplos de injeção
│   ├── 03-prompts.md                     # System prompt, few-shots e edge cases
│   ├── 04-metricas.md                    # Avaliação, testes e resultados
│   └── 05-pitch.md                       # Roteiro do pitch de 3 minutos
│
├── 📁 src/                               # Código da aplicação (em desenvolvimento)
│   └── README.md                         # Estrutura sugerida e instruções
│
├── 📁 assets/                            # Diagramas e recursos visuais
│
└── 📁 examples/                          # Referências de implementação (DIO)
```

---

## Documentação Completa

| Documento | Conteúdo |
|---|---|
| [01 — Documentação do Agente](./docs/01-documentacao-agente.md) | Caso de uso, persona, diagrama de arquitetura, estratégias anti-alucinação |
| [02 — Base de Conhecimento](./docs/02-base-conhecimento.md) | Descrição dos datasets, adaptações, estratégia de injeção, exemplo de contexto montado |
| [03 — Prompts](./docs/03-prompts.md) | System prompt completo, few-shot examples, edge cases, aprendizados por modelo |
| [04 — Métricas](./docs/04-metricas.md) | Cenários de teste, formulário de feedback, resultados e observações |
| [05 — Pitch](./docs/05-pitch.md) | Roteiro estruturado: problema → solução → demo → diferencial |

---

*Desenvolvido como solução para o desafio **Agente Financeiro Inteligente com IA Generativa** da [DIO](https://www.dio.me/).*  
*Cliente fictício: João Silva — dados mockados sem informações sensíveis reais.*
