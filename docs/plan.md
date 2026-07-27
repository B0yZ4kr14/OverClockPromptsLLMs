# Plano de Execução — Consolidação em Obra Única + Site Interativo

**Objetivo do usuário:** Consolidar os 9 documentos + 2 figuras (arquitetura de LLMs, orquestração multi-agente, RAG, "Lost in the Middle", chunking, inevitabilidade matemática, Kimi K2.5) em uma **obra única coesa** (eliminando redundâncias e ordenando a progressão lógica) e entregar como **site interativo** de relatório de pesquisa.

**Idioma da obra:** Português (unificar para pt-BR; o corpus mistura pt-PT e pt-BR).

---

## Estágio 1 — Auditoria do Corpus (pesquisa baseada apenas em arquivos — Rota C)
**Skill:** nenhuma (descoberta orientada pelo Orquestrador).
3 agentes `explore` em paralelo, cada um lê um subconjunto e grava um mapa detalhado em `/mnt/agents/output/stage1_*.md`:
- **A — Compilação Integral** (117 KB, 1163 linhas): mapear os 6 documentos + blog Kimi K2.5; resumo denso por documento; fórmulas, tabelas, benchmarks; redundâncias internas.
- **B — Documentos longos avulsos**: `Formalização e Implementação - Multi-Agent Orchestrator.md`, `Otimização de RAG e Contexto.md`, `Seção Expandida — Lost in the Middle e a Praxeologia do Chunking.md`. Resumo denso + sobreposição com a Compilação.
- **C — Documentos curtos + figuras**: `Arquitetura de LLMs - Da Estocasticidade...`, `Inevitabilidade Matemática...`, `Matemáticos da Arquitetura de LLMs.md`, `Matemáticos da LLMs.md`, `secao_lost_in_the_middle_chunking.md` + análise visual detalhada de `fig2_modelo_formal.png` e `fig4_evolucao_janelas.png`. Confirmar duplicatas exatas (diff) e subconjuntos.

**Gate:** os 3 mapas existem, cobrem 100% dos arquivos e marcam todas as duplicidades/sobreposições.

## Estágio 2 — Arquitetura da Obra (outline mestre)
**Responsável:** Orquestrador (eu), a partir dos 3 mapas.
- Definir estrutura de partes/capítulos sem redundância (provável: I. Fundamentos matemático-epistemológicos dos LLMs; II. Janela de contexto, Lost in the Middle, chunking e RAG; III. Orquestração Multi-Agente formalizada; IV. Estudo de caso Kimi K2.5).
- Alocar as 2 figuras aos capítulos.
- Gravar `/mnt/agents/output/stage2_outline.md` com briefing por capítulo (fontes, recortes, ângulos, o que descartar como redundante).

**Gate:** outline cobre todo o conteúdo não-redundante; cada capítulo aponta suas fontes.

## Estágio 3 — Escrita dos Capítulos
**Skill:** `report-writing` (carregar neste estágio; escopo: apenas redação de capítulos a partir de material-fonte fornecido — sem pesquisa externa).
3 redatores em paralelo (capítulos derivam dos documentos-fonte, não uns dos outros):
- Writer 1 → Parte I (fundamentos: estocasticidade, praxeologia, prova dos noves, matemática da atenção).
- Writer 2 → Parte II (janela de contexto, Lost in the Middle, chunking, RAG + fig4).
- Writer 3 → Partes III e IV (orquestrador multi-agente formal + caso Kimi K2.5 + fig2).
Cada um grava `/mnt/agents/output/stage3_parteN.md` em pt-BR, prosa acadêmica unificada, preservando fórmulas LaTeX.

**Gate:** capítulos completos, sem sobreposição de conteúdo entre si, fórmulas preservadas.

## Estágio 4 — Montagem e Revisão da Obra
- Agente editor: unir os capítulos em `/mnt/agents/output/obra_final.md`, harmonizar terminologia e variante pt-BR, numeração, transições, sumário.
- Agente `reviewer`: verificação binária (coerência, redundância residual, integridade das fórmulas, referências às figuras). Falha → correção delegada.

**Gate:** revisão PASS.

## Estágio 5 — Site Interativo
**Skills:** `interactive-research-report-en` (principal — site editorial de relatório de pesquisa); `musepool` (consulta de design, se necessário).
- Copiar figuras para o projeto; agente construtor executa o skill sobre `obra_final.md` (conteúdo em português).
- Entrega obrigatória via `mshtools-website_version_manager` (build_version).

**Gate:** site construído, figuras renderizando, versão salva e URL entregue.

---

**Notas de propagação (A2A):** mapas do Estágio 1 → outline (Estágio 2) → briefings + caminhos dos arquivos-fonte → Writers (Estágio 3) → capítulos → Editor/Revisor (Estágio 4) → obra_final.md + figuras → Construtor do site (Estágio 5).
**Decisões:** sem pesquisa web (consolidação estritamente do corpus); duplicatas exatas são descartadas, não mescladas; `.docx` não se aplica (usuário escolheu site interativo).
