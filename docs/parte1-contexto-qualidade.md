# Parte 1 — Contexto e expectativas de qualidade

> **Disciplina:** Qualidade de Software — AV1 (2026.2) · **Responsável:** Brenno
> **Projeto analisado:** [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch) · **Recorte da equipe:** fontes, planejamento e confabulação (item 20 da lista de projetos da atividade)

---

## 1. Recorte escolhido e por quê

O DeepResearch é um agente ReAct multi-turno: ele responde perguntas de pesquisa combinando um LLM de 30 bilhões de parâmetros (arquitetura MoE, com 3,3 bilhões ativados por token) a um conjunto de ferramentas — busca web (`search`), leitura de página (`visit`), busca acadêmica (`google_scholar`), execução de código (`PythonInterpreter`) e leitura de arquivos (`parse_file`). O núcleo do sistema mora em três arquivos principais: `inference/react_agent.py`, os módulos `inference/tool_*.py` e `inference/prompt.py`.

Como equipe, decidimos não tentar cobrir o projeto inteiro — ele é grande demais para isso em uma AV1. Escolhemos delimitar a avaliação a três fluxos que se conectam entre si e que são, na nossa leitura, onde a confiabilidade de um agente de "deep research" fica mais exposta:

1. **Fontes** — como o agente recupera, resume e (não) cita as evidências que encontra (`tool_visit.py`, `EXTRACTOR_PROMPT`).
2. **Planejamento** — como o loop ReAct decide entre pensar, chamar uma ferramenta ou já responder (`_run`, em `react_agent.py`).
3. **Confabulação** — até que ponto o contrato de prompt (`SYSTEM_PROMPT`) impede, ou permite, que o modelo afirme algo sem nenhum lastro em evidência recuperada.

Ficaram fora do nosso recorte o processo de treinamento do modelo (RL, geração de dados sintéticos) e os 13 subprojetos irmãos que vivem dentro de `WebAgent/` (WebDancer, WebSailor, WebWatcher, entre outros), que reaproveitam ideias parecidas com variações próprias. O motivo é simples: o caminho de uso oficial, documentado no README do repositório, é o pipeline de inferência em `inference/`, e foi nele que concentramos a leitura de código.

## 2. Partes interessadas (stakeholders)

| # | Parte interessada | Objetivo | Expectativa de qualidade | Possível dano se a qualidade falhar | Evidência desejada | Responsabilidade |
|---|---|---|---|---|---|---|
| 1 | **Pesquisador/usuário final** que faz perguntas de pesquisa profunda (acadêmico, analista, jornalista) | Obter uma resposta correta, com fontes verificáveis, para uma pergunta complexa | Toda afirmação factual relevante vem acompanhada de uma fonte rastreável; incerteza é sinalizada quando a evidência é fraca | Tomar decisão (citar em trabalho, publicar, investir) com base em fato inventado (confabulação) apresentado com confiança | Resposta final (`<answer>`) com URLs/fontes associadas a cada afirmação | Verificar a fonte antes de usar a resposta em contexto crítico |
| 2 | **Mantenedores do projeto** (Tongyi Lab / Alibaba-NLP) | Manter a reputação técnica e acadêmica do projeto (paper no arXiv, benchmarks públicos) | O agente se comporta de forma consistente com o que o paper reivindica (desempenho de ponta em HLE, BrowseComp etc.) | Casos públicos de confabulação ou prompt injection viram crítica pública e desgastam a credibilidade do projeto | Taxa de acerto/citação correta em benchmarks e em auditorias externas como esta | Documentar limitações conhecidas no README/paper |
| 3 | **Desenvolvedor integrador / self-host** que baixa o repositório, configura o `.env` com chaves próprias (Serper, Jina, provedor OpenAI-compatível, SandboxFusion) e expõe o agente em um produto próprio | Reaproveitar o núcleo `inference/` como componente de um sistema maior | As ferramentas falham de forma previsível, com mensagem controlada, sem vazar segredos nem travar o processo hospedeiro | Falha de rede em `tool_search`/`tool_visit` propaga uma exceção não tratada para o sistema hospedeiro; chave de API exposta em log | Testes de robustez das tools sob falha de rede (Seção 7, caso CT-13) | Isolar e tratar exceções antes de expor a terceiros |
| 4 | **Comunidade acadêmica / avaliadores de benchmark** que reproduzem HLE, BrowseComp, FRAMES etc. usando os scripts `evaluation/evaluate_*_official.py` | Reproduzir os números do paper de forma confiável | O critério de avaliação (um LLM atuando como juiz) é estável o suficiente para comparar modelos entre si | Resultados de benchmark que não se reproduzem, por causa do não determinismo do próprio avaliador (que também é um LLM) | Múltiplas execuções do mesmo benchmark, com a variância documentada | Relatar desvio padrão / variabilidade nos números publicados |

## 3. Contexto de uso

O DeepResearch roda em lote: ele processa um arquivo `.jsonl`/`.json` de perguntas (pasta `eval_data/`), não é um chat interativo. A "conversa" turno a turno acontece dentro do loop ReAct, entre o LLM e as ferramentas — não entre o LLM e uma pessoa em tempo real.

O modelo pode rodar localmente via vLLM, atrás de `http://127.0.0.1:{planning_port}/v1` (`inference/react_agent.py`, linha 62), ou remotamente via OpenRouter, seguindo as instruções da seção 6 do README. Ao longo da execução, o agente conversa com quatro serviços externos: Serper para busca, Jina AI para ler páginas (via `r.jina.ai`), um segundo LLM OpenAI-compatível que resume o conteúdo lido (configurado por `API_KEY`/`API_BASE` no `.env.example`) e o SandboxFusion, que executa o código Python gerado pelo modelo. Quando o usuário anexa arquivos, entra ainda um quinto caminho: para vídeo e áudio (`.mp4`, `.mp3`), o processamento passa pela API da DashScope (`DASHSCOPE_API_KEY`); para os demais formatos (PDF, DOCX, planilhas etc.), o repositório usa por padrão o serviço de IDP da Alibaba Cloud (`docmind-api`, chaves `IDP_KEY_ID`/`IDP_KEY_SECRET`, em `file_tools/idp.py`). Nenhuma dessas informações — pergunta do usuário, arquivo anexado, conteúdo de página visitada — fica persistida entre execuções: cada linha do `.jsonl` é tratada como uma sessão isolada.

As chaves de API ficam em `.env`, que está listado no `.gitignore` (então não vão parar no repositório por acidente). Mas isso não resolve o problema de privacidade de fundo: sempre que `parse_file` é usado, o conteúdo do arquivo do usuário sai da máquina local e vai para um serviço de terceiro — DashScope ou o IDP da Alibaba Cloud, dependendo do tipo de arquivo. É um ponto relevante para os requisitos de privacidade que aparecem na Seção 5 (RQ-03).

## 4. Supervisão humana

Dentro do loop principal (`_run`, em `react_agent.py`) não existe nenhuma etapa de aprovação humana: o agente decide sozinho quando buscar, quando visitar uma página, quando executar código e quando já pode finalizar com `<answer>`. As únicas travas que existem são técnicas, não humanas:

- `MAX_LLM_CALL_PER_RUN`, no valor padrão de 100 chamadas de LLM por pergunta (linha 29, configurável por variável de ambiente).
- Um limite de tempo de execução de 150 minutos por pergunta (linha 140).
- Um limite de contexto de 110×1024 tokens, que força o agente a fechar a resposta quando é ultrapassado (linhas 186–193).

Na prática, a supervisão humana — quando acontece — é externa e vem depois: alguém só vai ler a resposta final quando o processo já tiver terminado. Isso torna mais importantes os requisitos de rastreabilidade e de sinalização de incerteza que discutimos na Seção 5, já que não existe um "humano no loop" capaz de interromper uma linha de raciocínio equivocada enquanto ela ainda está em andamento.

## 5. Decisões apoiadas pela resposta do agente

A saída do agente é o texto dentro de `<answer></answer>`, que o próprio prompt de sistema descreve como "the definitive response" (`inference/prompt.py`, linha 1). Esse conteúdo é usado para pelo menos três coisas:

- Fundamentar afirmações em textos, relatórios ou artigos que a pessoa usuária está escrevendo.
- Servir de entrada para o "LLM-como-juiz" nos scripts de `evaluation/`, que decide se a resposta é `"correct": "yes"` ou `"no"`.
- Alimentar, potencialmente, decisões em produtos que integrem o núcleo `inference/` como biblioteca.

## 6. Erros aceitáveis vs. inaceitáveis

| Categoria | Aceitável | Inaceitável |
|---|---|---|
| Precisão factual | Pequena imprecisão em resumo de conteúdo muito longo, após um truncamento sinalizado (`truncate_to_tokens`, `tool_visit.py`) | Afirmar um fato como certo sem nenhuma evidência recuperada — confabulação silenciosa |
| Fontes | Não encontrar uma página relevante e dizer isso com todas as letras | Citar uma URL ou um dado que não veio de nenhuma chamada real de `search`, `visit` ou `google_scholar` |
| Conflito de evidências | Reportar duas visões quando as fontes realmente divergem | Escolher uma fonte entre duas conflitantes, arbitrariamente, sem mencionar que havia divergência |
| Robustez operacional | Retornar uma mensagem de erro controlada quando uma ferramenta falha (por exemplo, `"[visit] Failed to read page."`) | Deixar vazar uma exceção não tratada que interrompe o processo de quem integrou o agente |
| Manipulação externa | Ignorar uma formatação estranha em uma página lida | Seguir instruções escondidas dentro do HTML de uma página visitada — prompt injection via conteúdo, ver CT-10 na Seção 7 |

## 7. Consequências de respostas incorretas

- **Para quem pesquisa:** uma decisão mal fundamentada, uma citação que não existe em um trabalho acadêmico, retrabalho quando o erro é descoberto tarde demais.
- **Para o projeto:** dano reputacional se casos de confabulação ou prompt injection forem expostos publicamente — o DeepResearch tem visibilidade considerável (repositório em destaque no Trendshift, cobertura em blog técnico e paper no arXiv), o que amplia o alcance de qualquer falha pública.
- **Para quem integra o agente:** uma falha em cascata no sistema hospedeiro, se uma exceção de rede não tratada não for isolada (ver RQ-06, na Seção 5).
- **Para a comunidade de benchmark:** conclusões distorcidas ao comparar modelos, se a variabilidade do "LLM-como-juiz" não for reportada junto com os números (ver Seção 8).

---

**Referências:** `inference/react_agent.py`, `inference/prompt.py`, `inference/tool_visit.py`, `inference/tool_file.py`, `inference/file_tools/idp.py`, `inference/file_tools/video_analysis.py`, `.env.example`, `README.md` — todos em https://github.com/Alibaba-NLP/DeepResearch (commit `f72f75d`).
