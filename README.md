# QS_2026-2_DeepResearch_Atividade1

Este é o repositório de trabalho da nossa equipe para a **AV1 de Qualidade de Software (2026.2)**: especificação e avaliação inicial da qualidade de uma aplicação de IA generativa.

**Projeto analisado (item 20 da lista da atividade):** [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch)
**Recorte da equipe:** fontes, planejamento e confabulação no núcleo de inferência (`inference/react_agent.py`, `inference/tool_*.py`, `inference/prompt.py`).

**Relatório consolidado para PDF:** [docs/relatorio-atividade1-qs.md](docs/relatorio-atividade1-qs.md)

---

## Índice das partes

| Parte | Seção do enunciado | Tema | Responsável(is) | Documento |
|---|---|---|---|---|
| 1 | 4 | Contexto e expectativas de qualidade | Brenno | [docs/parte1-contexto-qualidade.md](docs/parte1-contexto-qualidade.md) |
| 2 | 5 | Requisitos de qualidade | Daniel + Filipe | [docs/parte2-requisitos-qualidade.md](docs/parte2-requisitos-qualidade.md) |
| 3 | 6 | Aplicação da ISO/IEC 25010:2023 | Thiago | [docs/parte3-iso25010.md](docs/parte3-iso25010.md) |
| 4 | 7 + 8 | Casos de teste, avaliação inicial, variabilidade e não determinismo | Emilly + _(colega a definir)_ | [docs/parte4-testes-variabilidade.md](docs/parte4-testes-variabilidade.md) |
| 5 | 9 + 10 | Diagnóstico, plano de melhoria e uso crítico de IA | Lais | [docs/parte5-melhoria-uso-ia.md](docs/parte5-melhoria-uso-ia.md) |

## Status e pendências antes da entrega (16/09/2026, 23h59)

- [x] Seções 4-10 redigidas com base em leitura direta do código-fonte do repositório oficial.
- [ ] **Pendência crítica:** os casos de teste (Parte 4, Seção 7) e a análise de variabilidade (Parte 4, Seção 8) foram feitos por **análise estática do código**, sem executar o modelo de verdade. Se possível, rodar ao menos os casos prioritários (CT-10, CT-03, CT-09) via OpenRouter + Serper + Jina antes da entrega, e substituir "previsto"/"pendente" por resultados reais.
- [ ] Completar a dupla da Parte 4 (segundo nome além de Emilly).
- [ ] Preencher a tabela de contribuição individual (item 14 dos entregáveis) com carga horária/participação real de cada integrante.
- [ ] Gravar o vídeo de até 10 minutos com participação de todos (Seção 12 do enunciado) e publicar a URL no README, em `VIDEO.md`/`video.txt` e no relatório em PDF.
- [ ] Converter [docs/relatorio-atividade1-qs.md](docs/relatorio-atividade1-qs.md) para PDF (entre 8 e 12 páginas, excluídos anexos).

## Estrutura do repositório

- `docs/` — relatório consolidado e documentos por parte.
- `docs/relatorio-atividade1-qs.md` — versão única para exportar a PDF.
