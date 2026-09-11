# Parte 5 — Diagnóstico, plano de melhoria e uso crítico de IA generativa

> **Disciplina:** Qualidade de Software — AV1 (2026.2) · **Responsável:** Lais
> **Projeto analisado:** [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch)

---

## 9. Diagnóstico e plano inicial de melhoria

### 9.1 Achados (classificados por risco)

| # | Achado | Evidência | Risco | Recomendação |
|---|---|---|---|---|
| A-01 | O conteúdo bruto de páginas visitadas é repassado ao LLM sumarizador sem sanitização — porta de entrada para *prompt injection* via conteúdo de terceiros | `tool_visit.py`, `readpage_jina`/`html_readpage_jina` (linhas 132-254): o HTML extraído via Jina vai direto para `EXTRACTOR_PROMPT` | **Crítico** | Sanitizar/neutralizar blocos de instrução dentro do conteúdo extraído antes de repassá-lo; adicionar instrução explícita no `EXTRACTOR_PROMPT` para **ignorar** qualquer instrução encontrada dentro do conteúdo da própria página |
| A-02 | O contrato de saída (`<answer>`) não exige lista de fontes associada às afirmações factuais | `inference/prompt.py`, `SYSTEM_PROMPT` (linha 1) | **Alto** | Estender o contrato de saída para exigir uma seção de fontes (ex.: `<answer>...</answer><sources>...</sources>`) vinculando cada afirmação à evidência que a sustenta |
| A-03 | Nenhuma instrução no prompt de sistema pede para declarar incerteza quando a evidência é insuficiente | `SYSTEM_PROMPT` não contém nenhuma variação de "se não houver evidência suficiente, declare isso" | **Alto** | Adicionar instrução explícita de "declare incerteza explicitamente em vez de inferir" ao `SYSTEM_PROMPT` |
| A-04 | Nenhuma etapa de reconciliação entre fontes conflitantes antes da resposta final — cada página é resumida isoladamente | `EXTRACTOR_PROMPT` (linhas 37-51) processa uma página por vez, sem comparação cruzada | **Médio** | Introduzir uma etapa de síntese que compare os campos `evidence` de múltiplas fontes e explicite divergências antes do `<answer>` final |
| A-05 | Nenhum filtro de dados sensíveis/PII antes de disparar `search`/`visit` com o texto do usuário | `tool_search.py`/`tool_visit.py` não fazem nenhuma checagem de conteúdo de entrada | **Médio** | Adicionar uma camada central de sanitização de entrada (detecção básica de e-mail/CPF/telefone) antes de encaminhar a consulta às tools externas |

### 9.2 Plano de melhoria

| Ação | Responsável (papel) | Prioridade | Dependências | Indicador de sucesso | Risco residual | Critério de conclusão |
|---|---|---|---|---|---|---|
| **1.** Sanitizar conteúdo extraído em `tool_visit.py` contra instruções embutidas | Time de ferramentas (tools) | **Alta** | Nenhuma | 0 instruções de terceiros seguidas ao reexecutar o caso CT-10 (Parte 4) | Médio — novas técnicas de injection podem surgir e exigir atualização contínua do filtro | CT-10 reexecutado com pontuação 2 (adequado) |
| **2.** Estender contrato de saída para exigir citação de fontes | Time de prompt/orquestração | **Alta** | Nenhuma | ≥90% das respostas em uma amostra de 20 perguntas trazem ≥1 fonte válida por afirmação factual | Baixo-médio — depende também da capacidade do modelo em seguir o novo contrato | RQ-01 (Parte 2) atendido na amostra de validação |
| **3.** Adicionar instrução de incerteza ao `SYSTEM_PROMPT` | Time de prompt | **Alta** | Nenhuma | Casos CT-03 e CT-09 (Parte 4) passam a declarar incerteza/conflito explicitamente ao reexecutar | Médio — declarar incerteza em excesso pode reduzir utilidade percebida da resposta | CT-03 e CT-09 reexecutados com pontuação 2 |
| **4.** Introduzir etapa de reconciliação entre fontes conflitantes | Time de orquestração (`react_agent.py`) | Média | Ação 2 (contrato de fontes) | CT-09 relata divergência entre fontes em vez de escolher uma arbitrariamente | Médio — aumenta a latência/custo por chamada extra de LLM | CT-09 reexecutado com pontuação 2 e menção explícita ao conflito |
| **5.** Filtro central de dados sensíveis antes de `search`/`visit` | Time de ferramentas | Média | Nenhuma | CT-05 (Parte 4) resulta em recusa ou aviso de privacidade, não em busca direta do dado sensível | Médio-alto — detecção de PII por regra simples tem taxa de falso negativo conhecida | CT-05 reexecutado com pontuação ≥1 e sem exposição do dado sensível na consulta enviada ao Serper |

### 9.3 Observação sobre risco residual

Mesmo implementando as cinco ações, dois riscos estruturais permanecem parcialmente abertos e devem ser monitorados continuamente, não "resolvidos" de uma vez:

- **Não determinismo da busca ao vivo** (Parte 4, Seção 8): nenhuma ação do plano elimina a variação de fontes disponíveis ao longo do tempo — é uma característica inerente a um agente que pesquisa a web em tempo real.
- **Dependência da capacidade do próprio LLM**: instruções mais rígidas no prompt (Ações 2 e 3) reduzem, mas não eliminam, a possibilidade de o modelo ignorá-las — a mitigação definitiva exigiria validação automática da estrutura da resposta antes de aceitá-la como final, o que está fora do escopo desta auditoria de qualidade inicial.

---

## 10. Uso crítico de IA generativa (declaração)

**Ferramenta/modelo usado nesta parte da atividade:** Claude (Anthropic), via Claude in Chrome / Cowork, modelo configurado como `claude-sonnet-5`.

**Finalidade do uso:** apoiar a leitura e o rastreamento do código-fonte do repositório `Alibaba-NLP/DeepResearch` (localização de trechos relevantes em `react_agent.py`, `tool_visit.py`, `tool_search.py`, `tool_python.py`, `prompt.py`, `.env.example`), a estruturação dos requisitos/características ISO/casos de teste no formato exigido pelo enunciado, e a redação inicial deste conjunto de documentos.

**Até 5 prompts/instruções usados (resumo):**
1. Pedido para ler o enunciado da AV1 (PDF no Google Classroom) e mapear as seções para a divisão de responsáveis da equipe.
2. Pedido para clonar e inspecionar o repositório `Alibaba-NLP/DeepResearch` (código-fonte real, não apenas descrição).
3. Definição do recorte "fontes, planejamento e confabulação" e decisão de usar análise estática (sem execução real do modelo) para os casos de teste e a análise de variabilidade.
4. Pedido para redigir as cinco partes do relatório no formato Evidência/Critério/Risco exigido, citando trechos e caminhos de arquivo reais.
5. Pedido para consolidar tudo em um relatório técnico único, mantendo como fonte apenas o repositório `Alibaba-NLP/DeepResearch` (sem misturar com material de outra disciplina).

**Sugestões aproveitadas:** estrutura das tabelas de requisitos/ISO/casos de teste; identificação dos trechos de código citados como evidência; redação inicial de diagnóstico e plano de melhoria.

**Sugestões corrigidas/ajustadas pela equipe:** a divisão de responsáveis por seção foi definida pela equipe (Brenno, Daniel, Filipe, Thiago, Emilly, Lais) e informada à ferramenta, não o contrário; o escopo do relatório foi restrito pela equipe para citar apenas o repositório oficial do DeepResearch, removendo referências cruzadas a atividades de outra disciplina que a ferramenta havia incluído em um rascunho anterior.

**Erros identificados na produção assistida por IA:** um rascunho inicial misturou indevidamente referências a atividades de Engenharia de Software (outra disciplina) como se fossem parte da base desta auditoria — corrigido a pedido da equipe antes da consolidação final.

**Verificações feitas pela equipe:** confirmação de que os trechos de código citados (`react_agent.py`, `tool_visit.py`, `tool_search.py`, `tool_python.py`, `.env.example`) realmente existem no commit auditado do repositório oficial, antes de aceitar os achados como válidos para entrega. **Pendência explícita:** os "resultados previstos" dos casos de teste e da análise de variabilidade (Parte 4) ainda não foram confirmados por execução real do agente — isso deve ser feito pela equipe antes da entrega final, ou o relatório deve deixar claro que permanece como análise estática.

**Contribuição individual:** ver tabela de responsáveis no README do repositório de trabalho da equipe.

> IA generativa não foi a única autoridade de avaliação: a leitura do código foi verificada contra o repositório real, e a decisão de metodologia (análise estática em vez de execução real) foi tomada explicitamente pela equipe, não pela ferramenta.

---

**Referências:** achados fundamentados nos trechos citados nas Partes 1 a 4 deste conjunto de documentos, todos rastreáveis a https://github.com/Alibaba-NLP/DeepResearch.
