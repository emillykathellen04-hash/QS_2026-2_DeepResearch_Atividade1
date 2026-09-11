# Qualidade de Software — Atividade 1 (AV1)

## Especificação e avaliação inicial da qualidade de uma aplicação de IA generativa

**Disciplina:** Qualidade de Software | **Turma:** 01 — 2026.2 | **Modalidade:** trabalho em equipe (6-7 discentes)

**Projeto analisado (opção 20 da lista da atividade):** [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch)
**Organização responsável:** Alibaba-NLP / Tongyi Lab · **Licença:** Apache 2.0 · **Commit de referência:** `f72f75d` (branch `main`)
**Finalidade do projeto:** agente de LLM (30,5B parâmetros, MoE, 3,3B ativados por token) para tarefas de pesquisa profunda ("deep research") de longo horizonte, com paper associado ([arXiv:2510.24701](https://arxiv.org/pdf/2510.24701)).
**Recorte avaliado:** fontes, planejamento e confabulação no núcleo de inferência (`inference/react_agent.py`, `inference/tool_search.py`, `inference/tool_visit.py`, `inference/tool_python.py`, `inference/prompt.py`).
**Riscos e limitações gerais:** avaliação feita por leitura direta do código-fonte (análise estática); os casos de teste e a análise de variabilidade (Seções 7 e 8) **não** envolveram execução real do modelo — ver nota de método na Seção 7.

---

## 1. Contexto

A atividade propõe avaliar a qualidade inicial de uma aplicação de IA generativa a partir de requisitos, critérios de aceitação, casos de teste e evidências. A pergunta norteadora é: **como demonstrar que uma aplicação baseada em IA generativa é adequada, confiável e segura para seu contexto de uso, considerando que pode produzir respostas plausíveis, porém incorretas, variáveis ou não fundamentadas?**

O DeepResearch foi escolhido porque seu próprio propósito — pesquisa profunda multi-fonte — coloca a questão da fundamentação e da confabulação no centro do produto: uma resposta plausível, mas inventada, é o pior resultado possível para um agente cuja função declarada é "synthesize information from credible, diverse sources to deliver a comprehensive, accurate, and objective response" (`inference/prompt.py`, `SYSTEM_PROMPT`).

## 2. Identificação da equipe

| Integrante | Parte da atividade |
|---|---|
| Brenno | Parte 1 — Contexto e expectativas de qualidade |
| Daniel | Parte 2 — Requisitos de qualidade |
| Filipe | Parte 2 — Requisitos de qualidade |
| Thiago | Parte 3 — Aplicação da ISO/IEC 25010:2023 |
| Emilly | Parte 4 — Casos de teste, avaliação inicial e variabilidade |
| _(a definir)_ | Parte 4 — Casos de teste, avaliação inicial e variabilidade |
| Lais | Parte 5 — Diagnóstico, plano de melhoria e uso crítico de IA |

## 3. Metodologia

1. Leitura do enunciado oficial da atividade e mapeamento das 20 opções de projeto — o DeepResearch corresponde à opção 20, com recorte sugerido "fontes, planejamento e confabulação".
2. Inspeção direta do código-fonte do repositório oficial (não de documentação de terceiros): `inference/react_agent.py`, `inference/tool_search.py`, `inference/tool_visit.py`, `inference/tool_python.py`, `inference/prompt.py`, `.env.example`, `README.md`.
3. Elaboração de requisitos, características ISO/IEC 25010:2023 e casos de teste ancorados em trechos de código reais, citando arquivo e função/linha.
4. Para os casos de teste e a análise de variabilidade, adoção deliberada de **análise estática** (sem executar o modelo), por não haver, nesta fase, acesso às chaves de API e à infraestrutura de execução (GPU local ou créditos de OpenRouter/Serper/Jina/SandboxFusion) necessárias para rodar o agente de ponta a ponta.
5. Síntese em diagnóstico classificado por risco e plano de melhoria priorizado.

## 4. Contexto e expectativas de qualidade

*(conteúdo completo: [docs/parte1-contexto-qualidade.md](./parte1-contexto-qualidade.md))*

O DeepResearch opera em lote sobre arquivos `.jsonl`/`.json` de perguntas, orquestrando um LLM via ReAct com cinco ferramentas (`search`, `visit`, `google_scholar`, `PythonInterpreter`, `parse_file`) contra quatro serviços externos (Serper, Jina, um LLM sumarizador OpenAI-compatível e SandboxFusion). Não há supervisão humana dentro do loop — apenas travas técnicas (100 chamadas de LLM por pergunta, 150 minutos, 110k tokens de contexto). Identificamos quatro partes interessadas centrais:

| Parte interessada | Objetivo | Dano possível se a qualidade falhar |
|---|---|---|
| Pesquisador/usuário final | Resposta correta e rastreável a uma pergunta complexa | Decisão tomada sobre fato inventado apresentado com confiança |
| Mantenedores (Tongyi Lab) | Preservar credibilidade técnica/acadêmica | Casos públicos de confabulação desgastam a reputação do projeto |
| Desenvolvedor integrador/self-host | Reaproveitar `inference/` como componente | Exceção não tratada de uma tool derruba o sistema hospedeiro |
| Comunidade de benchmark | Reproduzir resultados do paper | Números não reprodutíveis por não-determinismo do avaliador (também um LLM) |

Erro aceitável: pequena imprecisão em resumo de conteúdo truncado. Erro inaceitável: afirmar um fato sem qualquer evidência recuperada, ou seguir instruções escondidas em uma página visitada.

## 5. Requisitos de qualidade

*(tabela completa com 12 requisitos: [docs/parte2-requisitos-qualidade.md](./parte2-requisitos-qualidade.md))*

Doze requisitos foram elaborados cobrindo as onze categorias exigidas. Os quatro de prioridade mais alta, diretamente ligados ao recorte da equipe:

| ID | Requisito | Categoria | Prioridade | Evidência |
|---|---|---|---|---|
| RQ-01 | Indicar fontes recuperáveis para afirmações factuais | Rastreabilidade | Alta | `SYSTEM_PROMPT` não exige lista de fontes no `<answer>` |
| RQ-02 | Sinalizar incerteza quando a evidência é insuficiente | Confiabilidade | Alta | Nenhuma instrução correspondente em `SYSTEM_PROMPT` |
| RQ-06 | Tratar falha de rede/tool sem interromper a sessão | Robustez | Alta | Retry com backoff em `call_server` (`react_agent.py`) e `readpage_jina` (`tool_visit.py`) |
| RQ-07 | Executar código gerado pelo modelo isolado do host | Segurança | Crítica | `tool_python.py` usa SandboxFusion remoto, nunca `exec()` local |

Os demais oito requisitos (adequação funcional, desempenho, manutenibilidade, flexibilidade, interação, proveniência de dados e supervisão humana) estão detalhados no documento da Parte 2.

## 6. Aplicação da ISO/IEC 25010:2023

*(análise completa das 6 características: [docs/parte3-iso25010.md](./parte3-iso25010.md))*

| Característica | Risco | Achado central |
|---|---|---|
| Adequação funcional | Alto | Nenhuma exigência de citação de fonte no contrato de saída (`<answer>`) |
| Confiabilidade | Médio-alto | Retries existem (10x no LLM, 8x na leitura de página, 5x na busca), mas não foram testados sob carga real |
| Segurança | Crítico/Médio | Execução de código isolada em sandbox remoto (bom sinal); segredos de 7+ provedores em texto simples no `.env` |
| Eficiência de desempenho | Médio | Limites de chamadas/tempo/tokens existem, mas não são adaptativos ao custo real da pergunta |
| Manutenibilidade | Alto | `TOOL_MAP` fixo + dispatcher `if/elif`; 26 arquivos `.py` instanciam `OpenAI(`/`AsyncOpenAI(` diretamente, sem interface comum |
| Capacidade de interação | Médio | Mensagens de erro das tools são legíveis e padronizadas; núcleo `inference/` não tem UI própria (é um script batch) |

## 7. Casos de teste e avaliação inicial

*(13 casos completos: [docs/parte4-testes-variabilidade.md](./parte4-testes-variabilidade.md))*

**Nota de método:** os casos abaixo foram desenhados e avaliados por **leitura de código**, não por execução real do agente — o modelo de 30B exige GPU local ou créditos de API (OpenRouter/Serper/Jina/SandboxFusion) que a equipe não tinha disponíveis nesta fase. Cada previsão cita o trecho de código que a sustenta e permanece com status **"Pendente de execução real"**.

Cobrimos os 12 tipos exigidos (esperado, ambiguidade, falta de informação, fora de domínio, dado sensível, reformulação, entrada curta/longa, fonte ausente/conflitante, fonte inventada, saída estruturada, conteúdo enviesado, indisponibilidade) em 13 casos de teste. O achado mais grave já visível por análise estática:

> **CT-10 (tentativa de fonte inventada / prompt injection):** `tool_visit.py` repassa o HTML bruto extraído de uma página, via Jina, diretamente ao prompt de sumarização (`EXTRACTOR_PROMPT`), sem nenhuma sanitização de instruções embutidas no conteúdo. Isso é uma porta de entrada plausível para um atacante induzir o agente, via conteúdo de uma página visitada, a citar uma fonte ou fato fabricado — o núcleo do risco de confabulação do recorte desta equipe.

Outros achados relevantes: ausência de reconciliação entre fontes conflitantes antes da resposta final (CT-09); ausência de filtro de dados sensíveis antes de disparar busca (CT-05); tratamento de indisponibilidade de serviço já implementado corretamente (CT-13, retries com mensagem de erro controlada).

## 8. Variabilidade e não determinismo

*(análise completa: [docs/parte4-testes-variabilidade.md](./parte4-testes-variabilidade.md), Seção 2)*

Cinco prompts foram selecionados para repetição (três vezes cada, quando executados de fato), variando de alta variabilidade esperada (tópico recente, dependente de busca ao vivo) a um controle negativo de baixa variabilidade (fato objetivo estável). A análise do código identificou três fontes estruturais de não-determinismo:

1. **Amostragem estocástica:** `temperature=0.85`, `top_p=0.95`, `presence_penalty=1.1` (`.env.example`/`react_agent.py`) — sem seed fixo documentado.
2. **Busca ao vivo sem cache:** `NLP_WEB_SEARCH_ONLY_CACHE=false` — o conjunto de fontes disponíveis pode mudar entre execuções da mesma pergunta, o que é uma variabilidade **de conteúdo**, não apenas de estilo.
3. **Truncamento adaptativo sob falha:** `readpage_jina` reduz progressivamente o tamanho do conteúdo sumarizado a cada tentativa mal sucedida — instabilidade de infraestrutura pode alterar a qualidade do resumo entre execuções idênticas.

Mecanismos de controle existentes (limite de chamadas, tempo e tokens) evitam loops descontrolados, mas não reduzem a variabilidade de conteúdo em si.

## 9. Diagnóstico e plano inicial de melhoria

*(detalhamento completo: [docs/parte5-melhoria-uso-ia.md](./parte5-melhoria-uso-ia.md))*

| # | Achado | Risco |
|---|---|---|
| A-01 | Conteúdo de página repassado sem sanitização ao sumarizador (prompt injection) | **Crítico** |
| A-02 | Contrato de saída não exige citação de fontes | **Alto** |
| A-03 | Nenhuma instrução de declarar incerteza | **Alto** |
| A-04 | Sem reconciliação entre fontes conflitantes | Médio |
| A-05 | Sem filtro de dados sensíveis antes de buscar/visitar | Médio |

**Plano priorizado (resumo):** (1) sanitizar conteúdo extraído contra instruções embutidas — prioridade alta, sem dependências; (2) exigir citação de fontes no contrato de saída — prioridade alta; (3) instruir o modelo a declarar incerteza — prioridade alta; (4) reconciliar fontes conflitantes — prioridade média, depende da ação 2; (5) filtrar dados sensíveis antes de buscar — prioridade média. Cada ação tem indicador de sucesso ligado a um caso de teste específico da Seção 7 (ex.: ação 1 é considerada concluída quando CT-10 é reexecutado com pontuação 2).

**Risco residual:** mesmo após as cinco ações, a variabilidade da busca ao vivo (Seção 8) e a dependência da capacidade do próprio LLM em seguir instruções mais rígidas permanecem como riscos estruturais que exigem monitoramento contínuo, não uma correção única.

## 10. Uso crítico de IA generativa

*(declaração completa: [docs/parte5-melhoria-uso-ia.md](./parte5-melhoria-uso-ia.md), Seção 10)*

Ferramenta usada: Claude (Anthropic), para apoiar leitura de código, estruturação das tabelas exigidas e redação inicial das cinco partes. A equipe verificou os trechos de código citados contra o repositório real antes de aceitar os achados, decidiu explicitamente pela metodologia de análise estática (em vez de execução real) e corrigiu um erro do rascunho inicial que misturava indevidamente referências a atividades de outra disciplina. IA generativa não foi a única autoridade de avaliação — a verificação de código e a decisão de escopo couberam à equipe.

## 11. Conclusão geral

O DeepResearch demonstra, no próprio código, o padrão clássico de agentes de pesquisa profunda baseados em LLM: forte capacidade funcional (busca, leitura, execução de código, tudo orquestrado por um loop ReAct robusto a falhas de rede), mas **nenhuma garantia estrutural contra confabulação** — o contrato de saída não exige fontes, o prompt de sistema não pede para declarar incerteza, e o conteúdo de páginas de terceiros entra sem sanitização no pipeline de sumarização. O recorte desta auditoria (fontes, planejamento, confabulação) mostra que os três problemas são interligados: sem exigência de citação (Seção 5, RQ-01), sem reconciliação entre fontes (Seção 7, CT-09) e sem barreira contra conteúdo malicioso de terceiros (Seção 7, CT-10), o sistema depende inteiramente da capacidade intrínseca do modelo para não confabular — não há rede de segurança no código. O plano de melhoria (Seção 9) ataca essa lacuna começando pela correção mais crítica e barata (sanitização de conteúdo) antes de investir em mudanças de contrato de prompt mais amplas.

**Limitação principal deste relatório:** por não termos executado o modelo, os resultados dos casos de teste e da análise de variabilidade são previsões fundamentadas em código, não observações empíricas — a equipe recomenda executar ao menos os casos CT-03, CT-09 e CT-10 antes de qualquer decisão de adoção do DeepResearch em produção.

---

## Referências

- Repositório analisado: https://github.com/Alibaba-NLP/DeepResearch (commit `f72f75d`)
- Paper técnico: https://arxiv.org/pdf/2510.24701
- Arquivos citados como evidência: `inference/react_agent.py`, `inference/tool_search.py`, `inference/tool_visit.py`, `inference/tool_python.py`, `inference/prompt.py`, `.env.example`, `README.md`
- Documentos de apoio desta equipe: [Parte 1](./parte1-contexto-qualidade.md) · [Parte 2](./parte2-requisitos-qualidade.md) · [Parte 3](./parte3-iso25010.md) · [Parte 4](./parte4-testes-variabilidade.md) · [Parte 5](./parte5-melhoria-uso-ia.md)
