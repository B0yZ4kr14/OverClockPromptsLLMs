# PARTE IV — ESTUDO DE CASO: KIMI K2.5 E A INTELIGÊNCIA AGÊNTICA VISUAL

As partes anteriores construíram um encadeamento fechado: a Parte I demonstrou a inevitabilidade matemática das restrições estocásticas dos grandes modelos de linguagem (LLMs, do inglês *Large Language Models*); a Parte II formalizou a escassez atencional e o *Lost in the Middle*, culminando no Teorema da Janela Efetiva; a Parte III elevou a resposta a sistema formal — o Grafo Dirigido Ponderado Atômico do orquestrador multi-agente. Falta a "prova dos noves" empírica: verificar se os postulados se realizam num sistema real de fronteira, construído por terceiros e documentado publicamente. O objeto de estudo é o Kimi K2.5, modelo aberto da Moonshot AI, descrito no blog oficial de lançamento (Moonshot AI, 2026) e analisado complementarmente no relatório de otimização de RAG do corpus. O método é o de estudo de caso instrumental: cada decisão de engenharia documentada é confrontada com a formalização precedente, distinguindo-se o fato reportado pelo fabricante da interpretação do autor.

## Capítulo 12 — Arquitetura e Treinamento

### 12.1 Pré-treinamento continuado e multimodalidade nativa

O Kimi K2.5 deriva do Kimi K2 por pré-treinamento continuado sobre aproximadamente 15 trilhões de tokens mistos, visuais e textuais (Moonshot AI, 2026). "Mistos" não é ornamental: o modelo é multimodal nativo, e não um LLM textual ao qual se acopla a posteriori um codificador visual. Segundo o fabricante, é o pré-treinamento conjunto visão-texto em escala massiva que sustenta a "codificação com visão" — raciocinar sobre imagens e vídeos para gerar ou depurar código —, com a alegação de que, nessa escala, o trade-off entre capacidade visual e textual desaparece e ambas melhoram em conjunto. A afirmação não vem acompanhada de ablação pública e é registrada como tese do desenvolvedor; os resultados visuais do Capítulo 14 são compatíveis com ela, sem demonstrá-la independentemente. O subsistema visual é o MoonViT, codificador de cerca de 400 milhões de parâmetros, que alimenta o mesmo tronco transformador do texto e viabiliza a "inteligência agêntica visual": o modelo não apenas descreve imagens, mas age sobre elas — comportamento exemplificado pela demonstração do labirinto (Capítulo 13).

### 12.2 Ficha topológica: MoE, MLA e a janela de 256K tokens

A arquitetura é uma Mistura de Especialistas (MoE, do inglês *Mixture-of-Experts*) com 1 trilhão de parâmetros totais, dos quais apenas 32 bilhões são ativados por inferência, distribuídos em 61 camadas e 64 cabeças de atenção. A razão de ativação de cerca de 3,2% por token é resposta direta ao problema de custo da Parte I: o custo por inferência escala com os parâmetros ativos, não com os totais, de modo que a capacidade paramétrica cresce sem crescimento proporcional do custo marginal — instância nítida do princípio praxeológico da máxima assertividade com o mínimo de meios. O segundo elemento decisivo é a Atenção Multi-Cabeça Latente (MLA, do inglês *Multi-head Latent Attention*): na autoatenção convencional, o cache de chaves e valores (cache KV) cresce linearmente com o comprimento da sequência e, somado ao custo quadrático $\mathcal{O}(N^2 \cdot d)$ examinado na Parte II, torna janelas longas proibitivas em VRAM; a MLA comprime as representações de chave-valor num espaço latente de dimensão reduzida, aliviando o termo dominante de memória e viabilizando 256 mil tokens de contexto nativos por ciclo. No segmento de serviço (*serverless*), o relatório de RAG do corpus registra custos estimados entre 0,6 e 0,375 dólares por milhão de tokens de entrada, conforme a estrutura de cache do usuário — valores de fonte secundária, a ler como estimativas.

### 12.3 A janela de 256K à luz do Teorema da Janela Efetiva

A escolha de 256 mil tokens como janela nominal deve ser lida à luz do Teorema da Janela Efetiva (Parte II): a janela útil $W_{\text{eff}}$ é estritamente menor que a nominal $W_{\text{nom}}$, em razão da diluição atencional e da curva em U documentada por Liu et al. (2023). Dois fatos do próprio material do fabricante corroboram a tese. Primeiro, todos os experimentos usaram janela de 256 mil tokens, mas benchmarks específicos exigiram gestão ativa de contexto: no HLE com ferramentas, excedido um limiar, apenas a última rodada de mensagens de ferramenta é retida; no BrowseComp, adotou-se a estratégia *discard-all*; nos demais benchmarks agênticos, tarefas que excedessem o contexto suportado foram contadas como falhas (Moonshot AI, 2026). Segundo, a própria existência do modo Agent Swarm — que fragmenta a tarefa em vez de alongar o contexto de um único agente — é admissão operacional de que a janela nominal não dispensa curadoria e particionamento. O K2.5 não refuta o Teorema da Janela Efetiva; internaliza-o como restrição de projeto: a janela de 256K desloca $W_{\text{nom}}$, mas toda a engenharia do sistema — cache comprimido via MLA, gestão de contexto nos benchmarks, delegação paralela via swarm — organiza-se em torno de $W_{\text{eff}} < W_{\text{nom}}$. O modelo distribui-se em quatro modos — K2.5 Instant, K2.5 Thinking, K2.5 Agent e K2.5 Agent Swarm (Beta) — via aplicação web, aplicativo, API e o produto Kimi Code; os benchmarks do Capítulo 14 são reportados majoritariamente com raciocínio estendido ativado.

## Capítulo 13 — O Agent Swarm e o Aprendizado por Reforço Agêntico Paralelo (PARL)

### 13.1 Escalar para fora, não apenas para cima

O K2.5 Agent Swarm, liberado como *research preview*, é enquadrado pelo fabricante como mudança de regime: do escalamento do agente único para a execução coordenada em enxame, autodirigida (Moonshot AI, 2026). Na configuração máxima documentada, o modelo instancia e orquestra até 100 subagentes, executando fluxos paralelos ao longo de até 1.500 passos coordenados — chamadas de ferramenta —, sem subagentes predefinidos nem fluxos artesanais. Contra a configuração de agente único, o tempo de execução cai até 4,5 vezes; nas avaliações internas, reporta-se redução de 80% no tempo total de ponta a ponta em cargas complexas de horizonte longo — valores do fabricante, a ler como cotas superiores reportadas em cenários internos. A arquitetura distingue dois papéis: um **agente orquestrador treinável** decompõe a tarefa em subtarefas paralelizáveis, cada uma executada por **subagentes congelados**, instanciados dinamicamente — o blog menciona especializações emergentes como "pesquisador de IA", "pesquisador de física" e "verificador de fatos", criadas pelo próprio orquestrador conforme a demanda.

### 13.2 PARL: recompensa escalonada contra o colapso serial

O treinamento do orquestrador emprega o Aprendizado por Reforço Agêntico Paralelo (PARL, do inglês *Parallel-Agent Reinforcement Learning*). O problema central é que o feedback de subagentes independentes é atrasado, esparso e não estacionário, e o modo de falha identificado é o **colapso serial** (*serial collapse*): ótimo local em que o orquestrador, apesar da capacidade paralela, regride ao agente único. A contrapartida é a moldagem escalonada de recompensa (*staged reward shaping*), que incentiva o paralelismo no início e desloca gradualmente o foco para o sucesso da tarefa (Moonshot AI, 2026):

$$r_{\mathrm{PARL}}(x, y) = \lambda_1 \cdot \underbrace{r_{\text{parallel}}}_{\text{instantiation reward}} + \lambda_2 \cdot \underbrace{r_{\text{finish}}}_{\text{sub-agent finish rate}} + \underbrace{r_{\text{perf}}(x, y)}_{\text{task-level outcome}} \, .$$

A recompensa $r_{\text{perf}}$ avalia o sucesso global e a qualidade da solução $y$ para a tarefa $x$. Os termos auxiliares atacam falhas simétricas: $r_{\text{parallel}}$ mitiga o colapso serial, forçando a exploração de escalonamentos concorrentes; $r_{\text{finish}}$ previne o **paralelismo espúrio** (*spurious parallelism*), comportamento de *reward hacking* em que o orquestrador infla as métricas paralelas instanciando subagentes sem decomposição significativa — ao recompensar subtarefas concluídas, impõe factibilidade. Os hiperparâmetros $\lambda_1$ e $\lambda_2$ sofrem recozimento (*annealing*) até zero ao longo do treinamento: os andaimes são retirados à medida que $r_{\text{perf}}$ basta, e o comportamento paralelo final não depende de incentivos artificiais. O fabricante reporta que, no ambiente de reforço paralelo, a recompensa cresce suavemente enquanto o paralelismo médio aumenta gradualmente.

### 13.3 Critical Steps: a métrica do caminho crítico

Para tornar a execução sequencial impraticável, o PARL substitui a contagem total de passos pelos **Critical Steps**, métrica orientada a latência inspirada no caminho crítico da computação paralela (Moonshot AI, 2026):

$$\text{CriticalSteps} = \sum_{t=1}^{T} \left( S_{\text{main}}^{(t)} + \max_{i} S_{\text{sub},i}^{(t)} \right)$$

em que $S_{\text{main}}^{(t)}$ captura o custo de orquestração no estágio $t$ e $\max_{i} S_{\text{sub},i}^{(t)}$ reflete o subagente mais lento do estágio. Sob essa métrica, instanciar mais subtarefas só reduz o custo se encurtar o caminho crítico: paralelismo que adiciona estágios ou subagentes lentos é penalizado. Em cenário de busca ampla (*wide search*), o fabricante reporta redução de 3× a 4,5× no mínimo de Critical Steps necessário ao desempenho-alvo em relação ao agente único, com economias crescentes na meta — traduzindo-se em até 4,5× de redução no tempo de relógio.

A conexão com a Parte III é direta e é interpretação do autor. O swarm é instância empírica do Grafo Dirigido Ponderado Atômico: o orquestrador treinável ocupa a posição do vértice-raiz que decompõe o prompt num grafo acíclico dirigido de subtarefas; os subagentes congelados, de responsabilidade única, realizam operacionalmente os Agentes Atômicos de Tarefa (AAT) — processos isolados, de contexto restrito, dispensados do histórico integral. E os Critical Steps são o análogo operacional do caminho crítico e do custo de rota do orquestrador formal: assim como a Parte III roteava transferências pela soma mínima de pesos das arestas (na analogia com Dijkstra), o PARL minimiza a soma dos estágios do caminho mais caro, punindo a aresta dominante. Duas divergências merecem registro: o swarm documentado não implementa os Agentes de Validação Confrontacional (AVC) popperianos e socráticos — a agregação é feita pelo orquestrador sem camada falsificacionista explícita, lacuna que o formalismo dos AVC se propõe a preencher — e os subagentes são congelados durante o treinamento do orquestrador, simplificação que torna o aprendizado tratável ao custo de impedir a co-adaptação dos executores. O estudo de caso confirma a estrutura do modelo formal sem esgotá-la.

### 13.4 A demonstração do labirinto: evidência de agência visual

A evidência mais nítida de agência visual é a demonstração, transcrita integralmente no blog, em que o K2.5 resolve um labirinto dado como imagem (Moonshot AI, 2026). O prompt pede o caminho mais curto do canto superior esquerdo ao inferior direito, com pixels pretos como via transponível. A cadeia de raciocínio exibe o ciclo completo percepção–formalização–execução–verificação: (i) **percepção** — a imagem, de 1.503 × 3.003 pixels (cerca de 4,5 milhões), é binarizada e, na ausência de pixels verde/vermelho puros, o modelo localiza os pontos de estrada mais próximos dos cantos: início em $(7, 3)$, fim em $(1495, 2999)$; (ii) **formalização** — o problema é reescrito como busca em grafo não ponderado sobre a grade, cada pixel transponível um nó, e a Busca em Largura (BFS, do inglês *Breadth-First Search*) é escolhida por garantir otimalidade nesses grafos; (iii) **execução** — o BFS é implementado em Python com rastreamento de pais; (iv) **verificação** — o modelo valida que todos os 113.556 passos consecutivos são adjacentes (distância de Manhattan igual a 1). O caminho encontrado tem **113.557 passos**.

O significado para a tese da obra é triplo. Primeiro, a decisão de não "ler" o labirinto pixel a pixel, mas traduzi-lo a estrutura formal e delegá-lo a um algoritmo exato, é a própria lógica da descentralização computacional: o LLM atua como orquestrador que reconhece os limites da inferência estocástica e transfere o cômputo crítico a um executor determinístico. Segundo, a justificativa explícita da escolha do BFS mostra o modelo raciocinando sobre propriedades matemáticas do problema, e não apenas imitando padrões de código. Terceiro, a etapa de verificação de adjacência, não solicitada pelo usuário, aproxima-se do espírito falsificacionista dos validadores da Parte III: o resultado é submetido a teste que poderia refutá-lo antes da entrega. Com a cautela devida a uma demonstração única e curada pelo fabricante, é agência visual em sentido operacional: perceber, formalizar, agir e verificar sobre o mundo visual sem intervenção humana intermediária.

## Capítulo 14 — Benchmarks e Posicionamento Competitivo

### 14.1 Método e tabela integral

O blog reporta 45 configurações de benchmark em seis categorias, comparando o Kimi K2.5 (modo Thinking) a GPT-5.2 (esforço de raciocínio xhigh), Claude 4.5 Opus (pensamento estendido), Gemini 3 Pro (alto nível de pensamento), DeepSeek V3.2 (Thinking) e, nos benchmarks visuais, Qwen3-VL-235B-A22B (Thinking). Salvo indicação contrária, os experimentos do K2.5 usaram temperatura 1,0, top-p 0,95 e janela de 256 mil tokens (Moonshot AI, 2026). As tabelas abaixo preservam integralmente as 45 linhas do original; somente nelas e nas notas metodológicas da §14.3 os valores mantêm a notação decimal do documento-fonte (ponto como separador decimal), enquanto toda a prosa desta Parte adota a vírgula como separador decimal, conforme a convenção do português brasileiro. O marcador "—" corresponde a "null" no original (modelo não avaliado); o asterisco (*) indica benchmark reavaliado pelo fabricante nas mesmas condições do K2.5 por não haver pontuação pública; a cruz (†) indica subconjunto apenas de texto.[^1]

**Raciocínio e Conhecimento**

| Benchmark | Kimi K2.5 (Thinking) | GPT-5.2 (xhigh) | Claude 4.5 Opus (Ext. Thinking) | Gemini 3 Pro (High Thinking) | DeepSeek V3.2 (Thinking) | Qwen3-VL-235B-A22B (Thinking) |
|---|---|---|---|---|---|---|
| HLE-Full | 30.1 | 34.5 | 30.8 | 37.5 | 25.1* | — |
| HLE-Full c/ ferramentas | 50.2 | 45.5 | 43.2 | 45.8 | 40.8* | — |
| AIME 2025 | 96.1 | 100 | 92.8 | 95.0 | 93.1 | — |
| HMMT 2025 (fev.) | 95.4 | 99.4 | 92.9* | 97.3* | 92.5 | — |
| IMO-AnswerBench | 81.8 | 86.3 | 78.5* | 83.1* | 78.3 | — |
| GPQA-Diamond | 87.6 | 92.4 | 87 | 91.9 | 82.4 | — |
| MMLU-Pro | 87.1 | 86.7* | 89.3* | 90.1 | 85.0 | — |

**Imagem e Vídeo**

| Benchmark | Kimi K2.5 (Thinking) | GPT-5.2 (xhigh) | Claude 4.5 Opus (Ext. Thinking) | Gemini 3 Pro (High Thinking) | DeepSeek V3.2 (Thinking) | Qwen3-VL-235B-A22B (Thinking) |
|---|---|---|---|---|---|---|
| MMMU-Pro | 78.5 | 79.5 | 74.0 | 81.0 | — | 69.3 |
| CharXiv (RQ) | 77.5 | 82.1 | 67.2* | 81.4 | — | 66.1 |
| MathVision | 84.2 | 83 | 77.1* | 86.1 | — | 74.6 |
| MathVista (mini) | 90.1 | 82.8* | 80.2* | 89.8* | — | 85.8 |
| ZeroBench | 9 | 9* | 3* | 8* | — | 4* |
| ZeroBench c/ ferramentas | 11 | 7* | 9* | 12* | — | 3* |
| OCRBench | 92.3 | 80.7* | 86.5* | 90.3* | — | 87.5 |
| OmniDocBench 1.5 | 88.8 | 85.7 | 87.7* | 88.5 | — | 82.0* |
| InfoVQA (teste) | 92.6 | 84* | 76.9* | 57.2* | — | 89.5 |
| SimpleVQA | 71.2 | 55.8* | 69.7* | 69.7* | — | 56.8* |
| WorldVQA | 46.3 | 28.0 | 36.8 | 47.4 | — | 23.5 |
| VideoMMMU | 86.6 | 85.9 | 84.4* | 87.6 | — | 80.0 |
| MMVU | 80.4 | 80.8* | 77.3 | 77.5 | — | 71.1 |
| MotionBench | 70.4 | 64.8 | 60.3 | 70.3 | — | — |
| VideoMME | 87.4 | 86.0 | — | 88.4* | — | 79.0 |
| LongVideoBench | 79.8 | 76.5 | 67.2 | 77.7* | — | 65.6* |
| LVBench | 75.9 | — | — | 73.5* | — | 63.6 |

**Codificação**

| Benchmark | Kimi K2.5 (Thinking) | GPT-5.2 (xhigh) | Claude 4.5 Opus (Ext. Thinking) | Gemini 3 Pro (High Thinking) | DeepSeek V3.2 (Thinking) | Qwen3-VL-235B-A22B (Thinking) |
|---|---|---|---|---|---|---|
| SWE-Bench Verified | 76.8 | 80.0 | 80.9 | 76.2 | 73.1 | — |
| SWE-Bench Pro | 50.7 | 55.6 | 55.4* | — | — | — |
| SWE-Bench Multilingual | 73.0 | 72.0 | 77.5 | 65.0 | 70.2 | — |
| Terminal-Bench 2.0 | 50.8 | 54.0 | 59.3 | 54.2 | 46.4 | — |
| PaperBench | 63.5 | 63.7* | 72.9* | — | 47.1 | — |
| CyberGym | 41.3 | — | 50.6 | 39.9* | 17.3* | — |
| SciCode | 48.7 | 52.1 | 49.5 | 56.1 | 38.9 | — |
| OJBench (cpp) | 57.4 | — | 54.6* | 68.5* | 54.7* | — |
| LiveCodeBench (v6) | 85.0 | — | 82.2* | 87.4* | 83.3 | — |

**Contexto Longo**[^6]

| Benchmark | Kimi K2.5 (Thinking) | GPT-5.2 (xhigh) | Claude 4.5 Opus (Ext. Thinking) | Gemini 3 Pro (High Thinking) | DeepSeek V3.2 (Thinking) | Qwen3-VL-235B-A22B (Thinking) |
|---|---|---|---|---|---|---|
| Longbench v2 | 61.0 | 54.5* | 64.4* | 68.2* | 59.8* | — |
| AA-LCR | 70.0 | 72.3* | 71.3* | 65.3* | 64.3* | — |

**Busca Agêntica**

| Benchmark | Kimi K2.5 (Thinking) | GPT-5.2 (xhigh) | Claude 4.5 Opus (Ext. Thinking) | Gemini 3 Pro (High Thinking) | DeepSeek V3.2 (Thinking) | Qwen3-VL-235B-A22B (Thinking) |
|---|---|---|---|---|---|---|
| BrowseComp | 60.6 | — | 37.0 | 37.8 | 51.4 | — |
| BrowseComp — c/ gestão de contexto | 74.9 | 65.8 | 57.8 | 59.2 | 67.6 | — |
| BrowseComp — (Agent Swarm) | 78.4 | — | — | — | — | — |
| WideSearch (item-f1) | 72.7 | — | 76.2* | 57.0 | 32.5* | — |
| WideSearch (item-f1) — (Agent Swarm) | 79.0 | — | — | — | — | — |
| DeepSearchQA | 77.1 | 71.3* | 76.1* | 63.2* | 60.9* | — |
| FinSearchComp T2&T3 | 67.8 | — | 66.2* | 49.9 | 59.1* | — |
| Seal-0 | 57.4 | 45.0 | 47.7* | 45.5* | 49.5* | — |

**Uso de Computador**

| Benchmark | Kimi K2.5 (Thinking) | GPT-5.2 (xhigh) | Claude 4.5 Opus (Ext. Thinking) | Gemini 3 Pro (High Thinking) | DeepSeek V3.2 (Thinking) | Qwen3-VL-235B-A22B (Thinking) |
|---|---|---|---|---|---|---|
| OSWorld-Verified | 63.3 | 8.6* | 66.3 | 20.7* | — | 38.1 |
| WebArena | 58.9 | — | 63.4* | — | — | 26.4* |

A leitura agregada exige disciplina. O K2.5 não domina a maioria das colunas: GPT-5.2 e Gemini 3 Pro lideram vários benchmarks de raciocínio puro (AIME, HMMT, GPQA-Diamond, MMLU-Pro) e o Claude 4.5 Opus lidera boa parte da suíte de codificação. O posicionamento do K2.5 é de outra natureza: como modelo aberto, mantém-se na fronteira em praticamente todas as categorias e lidera onde a tese desta obra prediz que a arquitetura — e não apenas a escala — deveria importar: tarefas agênticas com ferramentas, percepção documental densa e busca agêntica. Três advertências moderam qualquer conclusão: os valores com asterisco foram reavaliados pelo próprio fabricante; o GPT-5.2 teve cerca de 10% de taxa de falha nos benchmarks visuais, o que provavelmente subestima seu desempenho real; e os resultados do modo Swarm não têm comparadores, pois os demais modelos não foram avaliados nessa configuração.

### 14.2 Destaques e sua interpretação

**HLE-Full com ferramentas: 50,2.** O *Humanity's Last Exam* (HLE) é o benchmark de raciocínio avançado mais exigente da suíte. Sem ferramentas, o K2.5 marca 30,1 no conjunto completo (subconjuntos: 31,5 em texto, 21,3 em imagem), atrás de GPT-5.2 (34,5) e Gemini 3 Pro (37,5). Com ferramentas — busca, interpretador de código e navegação web —, o placar inverte-se: 50,2, o maior valor da linha, contra 45,5 do GPT-5.2, 45,8 do Gemini 3 Pro e 43,2 do Claude 4.5 Opus (subconjuntos: 51,8 em texto, 39,8 em imagem)[^2]. A inversão é o dado analiticamente relevante: o diferencial não reside no raciocínio paramétrico bruto, mas na capacidade agêntica de operar ferramentas sob gestão de contexto — exatamente a competência que as Partes II e III formalizam como resposta à escassez atencional.

**AIME 2025: 96,1 e a suíte matemática.** Em matemática de competição, o K2.5 marca 96,1 no AIME 2025 (média de 32 execuções), 95,4 no HMMT 2025 de fevereiro e 81,8 no IMO-AnswerBench — abaixo do GPT-5.2 (100; 99,4; 86,3) e próximo do Gemini 3 Pro. A leitura correta não é a de inferioridade, mas a de saturação: nessa faixa, os benchmarks aproximam-se do teto e as diferenças deixam de discriminar capacidade, deslocando a competição para as frentes não saturadas — as agênticas e visuais.

**SWE-Bench Verified: 76,8.** Em engenharia de software real, o K2.5 atinge 76,8, contra 80,0 do GPT-5.2, 80,9 do Claude 4.5 Opus, 76,2 do Gemini 3 Pro e 73,1 do DeepSeek V3.2 — com as maiores pontuações da série SWE obtidas em modo não-Thinking, sob arcabouço interno de ferramentas mínimas e médias de 5 execuções[^5]. O dado ganha densidade combinado ao benchmark interno Kimi Code Bench — 57,4 ± 1,9 para o K2.5, contra 43,7 ± 3,6 do K2 Thinking e 39,6 ± 3,1 do K2 0905 — e à eficiência de custo discutida adiante: o K2.5 não é o líder absoluto em codificação, mas é o modelo aberto mais próximo da fronteira, a uma fração do custo por token dos líderes proprietários.

**BrowseComp com Swarm: 78,4 — a assinatura empírica da orquestração.** A progressão do BrowseComp é o resultado mais significativo para a tese desta obra: 60,6 na configuração base; 74,9 com gestão de contexto (estratégia *discard-all*)[^3]; 78,4 com o Agent Swarm, sob o regime de no máximo 15 passos do agente principal e 100 por subagente[^7]. Cada degrau corresponde a uma tese formalizada: o salto de 60,6 para 74,9 é a aplicação direta da praxeologia do chunking e da janela efetiva (Parte II); o salto de 74,9 para 78,4 é o ganho marginal da descentralização em grafo (Parte III). Padrão análogo ocorre no WideSearch (item-f1): 72,7 em agente único, 79,0 em modo Swarm. A dupla progressão, medida no mesmo sistema com um único fator alterado por vez, é o mais próximo que o estudo de caso oferece de verificação empírica controlada dos postulados desta obra.

**Percepção documental: OCRBench 92,3 e OmniDocBench 88,8.** Em OCRBench, o K2.5 marca 92,3 — o maior valor da linha, contra 90,3 do Gemini 3 Pro, 87,5 do Qwen3-VL e 86,5 do Claude 4.5 Opus. No OmniDocBench 1.5, cuja pontuação é $(1 - \text{distância de Levenshtein normalizada}) \times 100$, atinge 88,8, também à frente de todos os comparadores (88,5; 87,7; 85,7). Esses resultados ancoram quantitativamente a "inteligência agêntica visual" do Capítulo 12: a fidelidade na transcrição de documentos densos é pré-condição para agir sobre o mundo visual — depurar código por inspeção, resolver labirintos — sem que erros de percepção contaminem a cadeia de raciocínio, na lógica da confiabilidade em cadeia $Q = \prod q_i$ da Parte I. Complementam o quadro o InfoVQA (92,6, maior da linha) e o WorldVQA (46,3, benchmark do próprio fabricante para conhecimento visual atômico, atrás apenas do Gemini 3 Pro, com 47,4).[^4]

**Eficiência de custo: 5,1×, 21,1× e 10,1×.** O fabricante compara custo de tokens e desempenho contra o GPT-5.2 (xhigh) em três benchmarks agênticos, reportando economias de 5,1× no SWE-Verified, 21,1× no BrowseComp e 10,1× no HLE (Moonshot AI, 2026). Em termos praxeológicos — a utilidade marginal do ciclo computacional, $\text{UM} = \partial \text{Desempenho} / \partial \text{Custo}$, da Parte I —, o argumento competitivo do K2.5 não é maximizar o numerador em todas as tarefas, mas maximizar a razão: desempenho de fronteira a custo estruturalmente menor, propriedade que decorre da arquitetura (32 bilhões de parâmetros ativos em 1 trilhão; cache KV comprimido pela MLA) e não de subsídio tarifário conjuntural. Advertência: os fatores de economia são calculados pelo fabricante a partir de seus preços e dos preços públicos do concorrente, devendo ser lidos como estimativas de ordem de grandeza, não como medição auditada.

### 14.3 Notas metodológicas do documento-fonte

[^1]: **Detalhes gerais de teste.** Resultados do Kimi K2.5 e do DeepSeek-V3.2 com modo de pensamento ativado; Claude Opus 4.5 com pensamento estendido; GPT-5.2 com esforço de raciocínio xhigh; Gemini 3 Pro com alto nível de pensamento. Nos benchmarks visuais, reporta-se adicionalmente o Qwen3-VL-235B-A22B-Thinking. Salvo especificação em contrário, os experimentos do K2.5 usaram temperatura = 1.0, top-p = 0.95 e contexto de 256 mil tokens. Benchmarks sem pontuações públicas foram reavaliados nas mesmas condições do K2.5 e marcados com asterisco (*). O GPT-5.2 xhigh não pôde ser avaliado em todos os benchmarks por problemas de estabilidade do serviço; os não testados são marcados com "-".

[^2]: **Texto e raciocínio.** HLE, AIME 2025, HMMT 2025 (fev.), GPQA-Diamond e IMO-AnswerBench avaliados com orçamento máximo de 96 mil tokens de compleição. AIME e HMMT: médias de 32 execuções (avg@32); GPQA-Diamond: 8 (avg@8). No HLE, pontuações no conjunto completo (texto e imagem): K2.5 marca 31.5 (texto) e 21.3 (imagem) sem ferramentas, 51.8 (texto) e 39.8 (imagem) com ferramentas. A pontuação do DeepSeek-V3.2 corresponde ao subconjunto apenas textual (†). O acesso ao Hugging Face foi bloqueado para prevenir vazamento de dados. O HLE com ferramentas usa gestão de contexto simples: excedido um limiar, retém-se apenas a última rodada de mensagens de ferramenta.

[^3]: **Busca agêntica / aumentada por ferramentas.** O K2.5 foi equipado com busca, interpretador de código e navegação web no HLE com ferramentas e em todos os benchmarks de busca agêntica. Exceto no BrowseComp (K2.5 e DeepSeek-V3.2 com a estratégia *discard-all*), nenhuma gestão de contexto foi aplicada, e tarefas que excedessem o contexto suportado foram contadas como falhas. Os prompts de sistema enfatizam uso profundo e proativo de ferramentas, com verificação de informações incertas; os prompts completos serão fornecidos no relatório técnico. Seal-0 e WideSearch: médias de quatro execuções (avg@4).

[^4]: **Benchmarks visuais.** Máximo de 64 mil tokens, média de três execuções (avg@3). ZeroBench (com ferramentas): máximo de 24 mil tokens por passo e 30 passos para raciocínio multipasso. MMMU-Pro segue o protocolo oficial, preservando a ordem das entradas e antecedendo as imagens. O GPT-5.2-xhigh teve cerca de 10% de taxa de falha (sem saída após 3 tentativas), tratada como incorreta; as pontuações reportadas provavelmente subestimam seu desempenho real. WorldVQA é benchmark do próprio fabricante para conhecimento visual atômico centrado no mundo. OmniDocBench: pontuação = (1 − distância de Levenshtein normalizada) × 100, maior é melhor.

[^5]: **Tarefas de codificação.** Terminal-Bench 2.0 com o arcabouço padrão (Terminus-2) e o parser JSON fornecido, avaliado em modo não-Thinking por incompatibilidade da gestão de contexto do modo Thinking com o Terminus-2. Série SWE-Bench (Verified, Multilingual, Pro) com arcabouço interno de ferramentas mínimas — bash, criação de arquivo, inserção, visualização, substituição de texto, submissão — e prompts dedicados; maiores pontuações em modo não-Thinking. A pontuação do Claude Opus 4.5 no CyberGym é da configuração não-Thinking. Todas as pontuações de codificação são médias de 5 execuções independentes.

[^6]: **Benchmarks de contexto longo.** AA-LCR: média de três execuções (avg@3). LongBench-V2: prompts idênticos e contextos padronizados em aproximadamente 128 mil tokens.

[^7]: **Agent Swarm.** BrowseComp (modo Swarm): agente principal limitado a 15 passos; subagentes a 100 passos. WideSearch (modo Swarm): agente principal e subagentes limitados a 100 passos.

### 14.4 Síntese final da obra

O estudo de caso fecha o arco argumentativo. Partimos da estocasticidade lexical — a tokenização que converte linguagem em matrizes e o *softmax* que governa cada token como evento termodinâmico — e demonstramos que as fragilidades dos LLMs não são acidentes de implementação, mas consequências matemáticas inevitáveis: o custo quadrático $\mathcal{O}(N^2 \cdot d)$ e a diluição atencional $\bar{w} = 1/N$, a curva em U do *Lost in the Middle*, o Teorema da Janela Efetiva $W_{\text{eff}} < W_{\text{nom}}$. A resposta da engenharia madura não foi negar essas restrições, mas internalizá-las: primeiro na curadoria do contexto (RAG otimizado, compressão, reordenação posicional), depois na descentralização computacional — o Grafo Dirigido Ponderado Atômico, com seus agentes atômicos, validadores popperianos e socráticos e a suavização laplaciana do estado global. O Kimi K2.5 confirma que esse caminho não é idiossincrasia teórica: um sistema real de fronteira, construído independentemente, adota a mesma solução — janela nominal ampla disciplinada por gestão ativa de contexto, atenção comprimida em espaço latente, especialistas ativados seletivamente e, sobretudo, um enxame de subagentes orquestrado por reforço contra o colapso serial, medido pelo caminho crítico e não pela soma dos passos. A progressão do BrowseComp — 60,6, 74,9, 78,4 — resume em três números a tese inteira: a exatidão suprema, na era das janelas estratosféricas, não está no enchimento irrestrito da capacidade, mas na curadoria matemática do contexto e na fragmentação paralela do trabalho. Da estocasticidade lexical à descentralização computacional, a direção é uma só, e é matematicamente necessária: orquestrar.

---
