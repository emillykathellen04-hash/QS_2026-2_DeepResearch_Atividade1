# Parte 2 — Requisitos de qualidade

> **Disciplina:** Qualidade de Software — AV1 (2026.2) · **Responsáveis:** Daniel + Filipe
> **Projeto analisado:** [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch)

---

## Método

Os requisitos foram derivados por leitura direta do código-fonte do núcleo de inferência (`inference/react_agent.py`, `inference/tool_search.py`, `inference/tool_visit.py`, `inference/tool_python.py`, `inference/prompt.py`, `.env.example`), cobrindo as categorias exigidas: adequação funcional, confiabilidade, desempenho, interação/usabilidade, segurança, privacidade, manutenibilidade, flexibilidade, rastreabilidade, supervisão humana, robustez e proveniência de dados.

## Tabela de requisitos

| ID | Requisito | Categoria | Prioridade | Critério de aceitação | Evidência |
|---|---|---|---|---|---|
| RQ-01 | Indicar fontes recuperáveis para afirmações factuais na resposta final | Rastreabilidade | Alta | Cada afirmação factual relevante em `<answer>` está associada a uma URL ou referência recuperada por `search`/`visit`/`google_scholar` | `inference/prompt.py` define `<answer>` sem exigir lista de fontes — contrato de saída atual não obriga citação |
| RQ-02 | Sinalizar incerteza quando a evidência recuperada é insuficiente | Confiabilidade | Alta | O agente declara explicitamente falta de evidência em vez de afirmar categoricamente | `SYSTEM_PROMPT` não contém nenhuma instrução de "declare incerteza se não houver evidência suficiente" |
| RQ-03 | Proteger dados/arquivos enviados pelo usuário | Privacidade | Alta | Arquivos processados por `parse_file` não são enviados a terceiros sem aviso; segredos não aparecem em logs | `parse_file` depende de `DASHSCOPE_API_KEY` (`.env.example`) — o conteúdo do arquivo do usuário sai para a API da DashScope |
| RQ-04 | Responder com base em pelo menos uma fonte por sub-afirmação factual verificável | Adequação funcional | Alta | Toda afirmação checável tem uma chamada de ferramenta correspondente no histórico de mensagens | `EXTRACTOR_PROMPT` (`inference/prompt.py`) gera `evidence`/`summary` por página, mas nada força o `<answer>` final a reaproveitá-los citando a fonte |
| RQ-05 | Respeitar orçamento de tempo e de chamadas por pergunta | Desempenho | Alta | Execução não ultrapassa 150 minutos nem 100 chamadas de LLM por pergunta | `inference/react_agent.py`: `MAX_LLM_CALL_PER_RUN=100` (linha 29); checagem de tempo `> 150*60` segundos (linha 140) |
| RQ-06 | Tratar falha de rede/tool sem interromper a sessão | Robustez | Alta | Timeout ou erro de uma ferramenta retorna mensagem de erro estruturada, nunca uma exceção não tratada | `Search.google_search_with_serp` tenta 5x e retorna string de erro (`tool_search.py`, linhas 63-71); `Visit.html_readpage_jina` tenta 8x (`tool_visit.py`, linha 170) |
| RQ-07 | Executar código gerado pelo modelo em ambiente isolado do host | Segurança | Crítica | Nenhuma chamada `exec`/`eval` local; toda execução Python passa por um serviço sandbox remoto | `inference/tool_python.py` usa `run_code(...)` da lib `sandbox_fusion` contra `SANDBOX_FUSION_ENDPOINTS` externos (linhas 21-25, 82) |
| RQ-08 | Permitir adicionar uma nova ferramenta sem alterar o parsing do loop principal | Manutenibilidade | Média | Uma nova tool é registrada e usada sem editar a árvore de decisão de `_run` | `TOOL_MAP` é um dicionário fixo (linhas 31-38) e `_run` usa `if "python" in tool_call.lower(): ... else: ...` como dispatcher (linhas 162-173) — hoje **não** atende ao critério |
| RQ-09 | Trocar o endpoint/provedor do LLM de planejamento sem editar múltiplos arquivos | Flexibilidade / portabilidade | Média | Um único ponto de configuração define o endpoint ativo (local vs. OpenRouter) | `call_server` cria `OpenAI(base_url=f"http://127.0.0.1:{planning_port}/v1")` direto no método (linhas 61-68); README pede editar manualmente `react_agent.py` para usar OpenRouter (seção 6) |
| RQ-10 | Retornar mensagens de erro legíveis ao próprio modelo para permitir replanejamento | Interação / usabilidade | Média | Erros de tool chegam como texto claro em `<tool_response>`, não como stack trace | `tool_python.py` retorna strings como `"[Python Interpreter Error]: ..."`; `react_agent.py` retorna `"Error: Tool call is not a valid JSON..."` (linha 176) |
| RQ-11 | Registrar data/hora de acesso e o texto original extraído de cada fonte | Proveniência de dados | Média | Cada evidência usada carrega timestamp e trecho original recuperável | `EXTRACTOR_PROMPT` retorna `evidence` (texto original) mas sem timestamp de acesso; nenhuma persistência do histórico é feita por padrão |
| RQ-12 | Prever um ponto de checagem humana antes de decisões de alto impacto baseadas na resposta | Supervisão humana | Baixa/Média (depende do caso de uso) | Existe ao menos um mecanismo (interno ou de integração) para revisão humana antes do uso da resposta em decisão crítica | Não há nenhuma etapa de aprovação dentro de `_run`; supervisão só é possível **depois** que o processo já terminou |

## Observação sobre priorização

As prioridades **Alta/Crítica** concentram-se nos requisitos ligados ao recorte da equipe (fontes, planejamento, confabulação — RQ-01, RQ-02, RQ-04) e aos riscos de segurança/robustez que já são parcialmente tratados no código (RQ-06, RQ-07). Os requisitos de manutenibilidade e flexibilidade (RQ-08, RQ-09) foram classificados como prioridade média porque, embora hoje não sejam atendidos, seu não cumprimento não gera dano direto ao usuário final — apenas eleva o custo de evolução do projeto.

---

**Referências:** `inference/react_agent.py`, `inference/tool_search.py`, `inference/tool_visit.py`, `inference/tool_python.py`, `inference/prompt.py`, `.env.example` — https://github.com/Alibaba-NLP/DeepResearch.
