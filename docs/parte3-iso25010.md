# Parte 3 — Aplicação da ISO/IEC 25010:2023

> **Disciplina:** Qualidade de Software — AV1 (2026.2) · **Responsável:** Thiago
> **Projeto analisado:** [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch)

---

Seis características do modelo de qualidade de produto da ISO/IEC 25010:2023 foram selecionadas por pertinência direta ao recorte da equipe (fontes, planejamento e confabulação) e aos requisitos da Parte 2.

## 1. Adequação funcional (*Functional suitability*)

| | |
|---|---|
| **Pertinência** | A função central do produto é "responder perguntas de pesquisa com base em evidência recuperada" — se a resposta não é rastreável a uma fonte, a característica central do produto falha, mesmo que o texto pareça correto. |
| **Risco** | Alto — resposta fluente e incorreta (confabulação) é mais perigosa que um erro óbvio, porque é mais difícil de detectar pelo usuário. |
| **Requisito relacionado** | RQ-01, RQ-04 |
| **Método de avaliação** | Inspeção do contrato de saída em `inference/prompt.py` (`SYSTEM_PROMPT`, `EXTRACTOR_PROMPT`) e rastreamento do fluxo de evidência do `visit`/`search` até o `<answer>` final em `react_agent.py`. |
| **Evidência** | `SYSTEM_PROMPT` (linhas 1-35) define apenas o contrato `<answer></answer>`; não exige lista de fontes na resposta final. O texto de evidência (`evidence`/`summary`) só existe internamente, por página, via `EXTRACTOR_PROMPT` — nada obriga o modelo a reaproveitá-lo citando a origem. |
| **Limitação** | Sem execução real do modelo, não medimos a taxa efetiva de confabulação — apenas o grau em que o *prompt* induz (ou deixa de induzir) esse comportamento. |

## 2. Confiabilidade (*Reliability*)

| | |
|---|---|
| **Pertinência** | O agente opera em múltiplos turnos, dependendo de sucesso repetido de chamadas de rede (LLM + 4 serviços externos); falha em qualquer elo pode interromper uma pesquisa longa. |
| **Risco** | Médio-alto — perguntas complexas podem levar dezenas de chamadas; uma falha não tratada no meio do processo desperdiça todo o trabalho já feito. |
| **Requisito relacionado** | RQ-06 |
| **Método de avaliação** | Leitura de `call_server` (retry com backoff exponencial) em `react_agent.py` e de `readpage_jina`/`html_readpage_jina` em `tool_visit.py`. |
| **Evidência** | `call_server`: até 10 tentativas com `sleep_time = base_sleep_time * (2**attempt) + random.uniform(0,1)`, limitado a 30s (linhas 70-109). `Visit.html_readpage_jina`: até 8 tentativas de leitura via Jina (linha 170). `Search.google_search_with_serp`: até 5 tentativas (linhas 63-72). |
| **Limitação** | Análise estática comprova a *existência* de retries, não seu comportamento sob condições reais de produção (latência, rate limit simultâneo de múltiplos usuários). |

## 3. Segurança (*Security*)

| | |
|---|---|
| **Pertinência** | O agente executa código Python **gerado pelo próprio LLM** (`PythonInterpreter`) e credenciais de múltiplos provedores externos convivem no mesmo processo. |
| **Risco** | Crítico se a execução de código escapasse do isolamento; médio para exposição de credenciais via variável de ambiente compartilhada entre tools. |
| **Requisito relacionado** | RQ-03, RQ-07 |
| **Método de avaliação** | Leitura de `inference/tool_python.py` e de `.env.example`. |
| **Evidência** | Nenhuma chamada local `exec()`/`eval()`: todo código roda via `run_code(RunCodeRequest(...))` contra um endpoint remoto do SandboxFusion, escolhido aleatoriamente entre `SANDBOX_FUSION_ENDPOINTS` (linhas 21-25, 79-93). Por outro lado, `.env.example` concentra 7+ segredos de provedores diferentes em texto simples, sem segregação por tool. |
| **Limitação** | A segurança do próprio serviço SandboxFusion (isolamento real do container remoto) está fora do escopo deste repositório e não foi avaliada. |

## 4. Eficiência de desempenho (*Performance efficiency*)

| | |
|---|---|
| **Pertinência** | Cada chamada de LLM e cada busca/visita tem custo monetário (tokens, API) e de tempo; um agente sem controle de orçamento pode ficar preso em loops caros. |
| **Risco** | Médio — já existem limites, mas nenhum deles é adaptativo ao custo real da pergunta. |
| **Requisito relacionado** | RQ-05 |
| **Método de avaliação** | Leitura das constantes e checagens de limite em `react_agent.py`. |
| **Evidência** | `MAX_LLM_CALL_PER_RUN = int(os.getenv('MAX_LLM_CALL_PER_RUN', 100))` (linha 29); corte por tempo (`> 150*60` segundos, linha 140); corte por tokens (`max_tokens = 110*1024`, linhas 186-193), forçando resposta final quando excedido. |
| **Limitação** | Não medimos consumo real de tokens/latência por pergunta — apenas confirmamos que os limites existem no código. |

## 5. Manutenibilidade (*Maintainability*)

| | |
|---|---|
| **Pertinência** | Adicionar uma nova ferramenta ou trocar de provedor de LLM são as duas mudanças mais prováveis ao longo da vida do projeto. |
| **Risco** | Alto — o custo de mudança está concentrado num único arquivo que mistura rede, parsing e orquestração. |
| **Requisito relacionado** | RQ-08, RQ-09 |
| **Método de avaliação** | Leitura estrutural de `react_agent.py` (`TOOL_MAP`, `_run`, `custom_call_tool`) e contagem de instanciações diretas do SDK OpenAI no repositório (`git grep -l "OpenAI(\|AsyncOpenAI("`). |
| **Evidência** | `TOOL_MAP` é um dicionário global fixo (linhas 31-38); `_run` decide qual tool chamar com `if "python" in tool_call.lower(): ... else: ...` (linhas 162-173); `custom_call_tool` repete o padrão `if/elif` por nome de tool (linhas 228-247). Uma busca no repositório encontra **26 arquivos `.py`** instanciando `OpenAI(`/`AsyncOpenAI(` diretamente, sem uma interface comum. |
| **Limitação** | A métrica é estrutural (contagem estática de padrões); não mede o esforço real de manutenção em campo, apenas o indício de acoplamento. |

## 6. Capacidade de interação (*Interaction capability*)

| | |
|---|---|
| **Pertinência** | Como o núcleo `inference/` não tem interface de chat própria, a "interação" relevante é entre o agente e (a) o próprio LLM, via mensagens de erro das tools, e (b) o operador humano, via saída padrão (stdout) e arquivo de resultado. |
| **Risco** | Médio — mensagens pouco claras dificultam tanto o replanejamento automático do agente quanto a depuração humana. |
| **Requisito relacionado** | RQ-10 |
| **Método de avaliação** | Leitura das strings de erro retornadas por cada tool. |
| **Evidência** | Mensagens padronizadas e legíveis: `"[Python Interpreter Error]: ..."` (`tool_python.py`), `"[visit] Failed to read page."` (`tool_visit.py`), `"Error: Tool call is not a valid JSON. Tool call must contain a valid \"name\" and \"arguments\" field."` (`react_agent.py`, linha 176). |
| **Limitação** | Não há interface de usuário final formal no núcleo `inference/` (é um script batch sobre `.jsonl`) — a avaliação de usabilidade real (para um humano interagindo diretamente) não se aplica a este ponto de entrada; ela seria mais relevante nas demos Streamlit dos subprojetos `WebAgent/`, fora do recorte desta auditoria. |

---

**Referências:** `inference/react_agent.py`, `inference/tool_python.py`, `inference/tool_visit.py`, `inference/tool_search.py`, `.env.example` — https://github.com/Alibaba-NLP/DeepResearch.
