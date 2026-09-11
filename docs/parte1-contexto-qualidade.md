# Parte 1 — Contexto e expectativas de qualidade

> **Disciplina:** Qualidade de Software — AV1 (2026.2) · **Responsável:** Brenno
> **Projeto analisado:** [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch) · **Recorte da equipe:** fontes, planejamento e confabulação (item 20 da lista de projetos da atividade)

---

## 1. Recorte escolhido e por quê

O DeepResearch é um agente ReAct multi-turno que responde perguntas de pesquisa combinando um LLM de 30B parâmetros (MoE, 3,3B ativados) com ferramentas de busca web (`search`), leitura de página (`visit`), busca acadêmica (`google_scholar`), execução de código (`PythonInterpreter`) e leitura de arquivos (`parse_file`). O núcleo do sistema está em `inference/react_agent.py`, `inference/tool_*.py` e `inference/prompt.py`.

Delimitamos a avaliação de qualidade a três fluxos interligados, que são exatamente onde a confiabilidade de um agente de "deep research" é mais frágil:

1. **Fontes** — como o agente recupera, resume e (não) cita evidências (`tool_visit.py`, `EXTRACTOR_PROMPT`).
2. **Planejamento** — como o loop ReAct decide entre pensar, chamar ferramenta e responder (`_run` em `react_agent.py`).
3. **Confabulação** — o quanto o contrato de prompt (`SYSTEM_PROMPT`) previne ou permite que o modelo afirme algo sem lastro em evidência recuperada.

Não avaliamos o processo de treinamento do modelo (RL, dados sintéticos) nem os 11 subprojetos irmãos em `WebAgent/`, que replicam padrões semelhantes com pequenas variações — o foco é o pipeline de inferência oficial documentado no README como caminho de uso (`inference/`).

## 2. Partes interessadas (stakeholders)

| # | Parte interessada | Objetivo | Expectativa de qualidade | Possível dano se a qualidade falhar | Evidência desejada | Responsabilidade |
|---|---|---|---|---|---|---|
| 1 | **Pesquisador/usuário final** que faz perguntas de pesquisa profunda (acadêmico, analista, jornalista) | Obter uma resposta correta, com fontes verificáveis, para uma pergunta complexa | Toda afirmação factual relevante vem acompanhada de uma fonte rastreável; incerteza é sinalizada quando a evidência é fraca | Tomar decisão (citar em trabalho, publicar, investir) com base em fato inventado (confabulação) apresentado com confiança | Resposta final (`<answer>`) com URLs/fontes associadas a cada afirmação | Verificar a fonte antes de usar a resposta em contexto crítico |
| 2 | **Mantenedores do projeto** (Tongyi Lab / Alibaba-NLP) | Manter reputação técnica e acadêmica do projeto (paper arXiv 2510.24701, benchmarks públicos) | O agente se comporta de forma consistente com o que o paper reivindica (SOTA em HLE, BrowseComp etc.) | Casos públicos de confabulação ou prompt injection viram crítica pública e desgastam a credibilidade do projeto | Taxa de acerto/citação correta em benchmarks e em auditorias externas como esta | Documentar limitações conhecidas no README/paper |
| 3 | **Desenvolvedor integrador / self-host** que baixa o repositório, configura `.env` com chaves próprias (Serper, Jina, OpenAI-compatible, SandboxFusion) e expõe o agente em um produto próprio | Reaproveitar o núcleo `inference/` como componente de um sistema maior | As ferramentas falham de forma previsível (mensagem controlada), sem vazar segredos nem travar o processo hospedeiro | Falha de rede em `tool_search`/`tool_visit` propaga exceção não tratada para o sistema hospedeiro; chave de API exposta em log | Testes de robustez das tools sob falha de rede (Seção 7, casos CT-13) | Isolar/tratar exceções antes de expor a terceiros |
| 4 | **Comunidade acadêmica / avaliadores de benchmark** que reproduzem HLE, BrowseComp, FRAMES etc. usando `evaluation/evaluate_*_official.py` | Reproduzir os números do paper de forma confiável | O critério de avaliação (LLM-como-juiz) é estável o suficiente para comparação entre modelos | Resultados de benchmark não reprodutíveis por não-determinismo do próprio avaliador (também um LLM) | Múltiplas execuções do mesmo benchmark com variância documentada | Relatar desvio padrão / variabilidade nos números publicados |

## 3. Contexto de uso

- **Modo de operação:** processamento em lote sobre um arquivo `.jsonl`/`.json` de perguntas (`eval_data/`), não um chat interativo dentro do núcleo `inference/` — a interação turno-a-turno acontece *dentro* do loop ReAct entre o LLM e as ferramentas, não entre o LLM e um humano em tempo real.
- **Modelo e serving:** LLM local via vLLM (`http://127.0.0.1:{planning_port}/v1`, ver `inference/react_agent.py` linha 62) ou remoto via OpenRouter (README, seção "6. You can use OpenRouter's API").
- **Ferramentas externas:** Serper (busca), Jina AI (leitura de página via `r.jina.ai`), um segundo LLM OpenAI-compatível para sumarização (`API_KEY`/`API_BASE` em `.env.example`), DashScope (parsing de arquivo) e SandboxFusion (execução de Python).
- **Dados:** perguntas e (opcionalmente) arquivos anexados pelo usuário (`eval_data/file_corpus/`); nenhuma persistência de histórico de conversa é definida no núcleo — cada item do `.jsonl` é uma sessão isolada.
- **Segredos:** ficam em `.env` (listado em `.gitignore`), mas o **conteúdo de arquivos enviados pelo usuário sai para um serviço de terceiro** (DashScope) quando `parse_file` é usado — implicação de privacidade relevante para a Seção 5 (RQ-03).

## 4. Supervisão humana

Não há nenhuma etapa de aprovação humana **dentro** do loop (`_run` em `react_agent.py`): o agente decide sozinho quando buscar, visitar, executar código e finalizar com `<answer>`. As únicas travas existentes são técnicas, não humanas:

- `MAX_LLM_CALL_PER_RUN` = 100 chamadas de LLM por pergunta (variável de ambiente, linha 29).
- Limite de tempo de execução: 150 minutos por pergunta (linha 140).
- Limite de contexto: 110×1024 tokens, com corte forçado para resposta final ao ultrapassar (linhas 186–193).

Ou seja: a supervisão humana, quando existe, é **externa e posterior** — alguém lê a resposta final depois que o processo já terminou. Isso eleva a importância dos requisitos de rastreabilidade e sinalização de incerteza (Seção 5), já que não há "humano no loop" para interromper uma linha de raciocínio equivocada em andamento.

## 5. Decisões apoiadas pela resposta do agente

A saída é o conteúdo dentro de `<answer></answer>` — apresentado, pelo prompt de sistema, como "the definitive response" (`inference/prompt.py`, linha 1). Isso é usado para:

- Fundamentar afirmações em textos, relatórios ou artigos que o usuário está escrevendo.
- Servir de entrada para o "LLM-como-juiz" nos scripts de `evaluation/`, decidindo se a resposta está `"correct": "yes"/"no"`.
- Potencialmente alimentar decisões downstream em produtos que integrem o núcleo `inference/` como biblioteca.

## 6. Erros aceitáveis vs. inaceitáveis

| Categoria | Aceitável | Inaceitável |
|---|---|---|
| Precisão factual | Pequena imprecisão em resumo de conteúdo muito longo após truncamento sinalizado (`truncate_to_tokens`, `tool_visit.py`) | Afirmar um fato como certo sem qualquer evidência recuperada (confabulação silenciosa) |
| Fontes | Não encontrar página relevante e declarar isso explicitamente | Citar uma URL ou dado que não veio de nenhuma chamada real de `search`/`visit`/`google_scholar` |
| Conflito de evidências | Reportar duas visões quando as fontes divergem genuinamente | Escolher arbitrariamente uma fonte entre duas conflitantes sem mencionar a divergência |
| Robustez operacional | Retornar mensagem de erro controlada quando uma tool falha (ex.: `"[visit] Failed to read page."`) | Propagar exceção não tratada que interrompe o processo do usuário integrador |
| Manipulação externa | Ignorar formatação estranha em uma página lida | Seguir instruções escondidas dentro do HTML de uma página visitada (prompt injection via conteúdo — ver CT-10, Seção 7) |

## 7. Consequências de respostas incorretas

- **Para o pesquisador:** decisão de pesquisa mal fundamentada, citação inexistente em trabalho acadêmico, retrabalho ao descobrir o erro tardiamente.
- **Para o projeto:** dano reputacional se casos de confabulação/prompt-injection forem expostos publicamente (o projeto tem visibilidade alta — >14 mil "trendshift", cobertura em blog e paper arXiv).
- **Para o integrador:** falha em cascata no sistema hospedeiro se uma exceção de rede não tratada não for isolada (ver Seção 5, RQ-06).
- **Para a comunidade de benchmark:** conclusões de comparação entre modelos distorcidas se a variabilidade do "LLM-como-juiz" não for reportada (ver Seção 8).

---

**Referências:** `inference/react_agent.py`, `inference/prompt.py`, `inference/tool_visit.py`, `.env.example`, README.md — todos em https://github.com/Alibaba-NLP/DeepResearch (commit `f72f75d`).
