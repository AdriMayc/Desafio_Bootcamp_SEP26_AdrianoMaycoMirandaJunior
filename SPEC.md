# SPEC — Desafio Bootcamp SEP26
## Assistente de Perguntas e Respostas sobre Políticas Internas (RAG local, sem API key)

> Este documento é a **diretriz-mestra** do projeto. Qualquer implementação (notebook, scripts, evidências) deve seguir estritamente o que está aqui descrito. Em caso de dúvida durante a implementação, este SPEC prevalece sobre suposições, "boas práticas" genéricas ou tecnologias estudadas em outros contextos.
>
> Este SPEC foi escrito para ser consultado por uma IA de implementação (VS Code) que terá acesso ao código e à estrutura real do projeto. Ele não presume contexto além do que está descrito aqui e nos arquivos do repositório.

---

## 1. Objetivo

Construir um assistente de perguntas e respostas sobre políticas internas fictícias ("Jornada do Colaborador"), que:

- recebe uma pergunta em português;
- recupera localmente (sem chamadas externas, sem chave de API) os trechos mais relevantes do corpus de políticas;
- respeita a vigência dos documentos (documentos revogados não podem ser citados como resposta);
- retorna uma resposta **extrativa** (o texto literal do trecho recuperado, sem reescrita);
- cita o documento e a seção de origem da resposta;
- informa explicitamente quando a informação não está disponível no corpus;
- formata a saída segundo o modelo de 5 campos definido no desafio (seção 14).

O foco obrigatório do desafio é a **recuperação local com TF-IDF**. Geração com LLM é **opcional** e pertence exclusivamente ao Stretch (seção 19).

---

## 2. Fonte de verdade

1. O **PDF do desafio** é a fonte de verdade principal e prevalece sobre qualquer prática genérica de engenharia de dados/software.
2. Quando houver conflito entre uma boa prática genérica e uma exigência explícita do PDF, **a exigência do PDF vence**.
3. Nenhum requisito deve ser inventado. Se algo não estiver definido no PDF, ele deve ser tratado como uma **decisão de implementação**, explicitamente identificada como tal (ver seção 22 e a seção final "Pontos que precisam de decisão antes da implementação").
4. Este SPEC não substitui o PDF — ele organiza e torna operacional o que o PDF exige. Em caso de divergência entre este documento e o PDF original, **o PDF é soberano** e o SPEC deve ser corrigido.

---

## 3. Escopo

Escopo obrigatório = Partes **P0 a P5**, descritas na seção 9.

Tecnologias permitidas nas partes obrigatórias:
- `pandas`
- `scikit-learn` (`TfidfVectorizer`, similaridade do cosseno)
- biblioteca padrão do Python

Tecnologia permitida **apenas no Stretch B**:
- `rank_bm25`

Método de recuperação obrigatório na implementação principal: **TF-IDF + similaridade do cosseno**.

Saída obrigatória: modelo de 5 campos (seção 14), resposta extrativa (seção 13).

---

## 4. Fora de escopo

**Explicitamente proibido na implementação obrigatória** (e sem necessidade real, também fora do Stretch, salvo o BM25 do Stretch B):

- API externa de qualquer tipo;
- chave de API / uso de LLM via API;
- banco de dados;
- arquitetura distribuída;
- pipeline de produção;
- frontend / interface web;
- FastAPI ou qualquer outro framework web;
- LangChain, LlamaIndex ou qualquer framework de RAG;
- embeddings (implementação obrigatória);
- LLM (implementação obrigatória — permitido **somente** no Stretch A, com modelo local, sem API key);
- PySpark;
- SQL.

O fato de PySpark e SQL terem sido estudados anteriormente **não implica** que devam ser usados neste desafio. Eles ficam fora de escopo, exceto como "possibilidade futura" registrada textualmente (não implementada) caso haja uma aplicação legítima sem alterar os requisitos.

**Arquivos fora de escopo (não são documentação oficial do projeto):**

- `agenda/agenda.md` — anotações pessoais de organização;
- `duvidas.md` — anotações pessoais de dúvidas.

Esses dois arquivos **não devem** ser tratados como requisitos, código, documentação técnica ou fonte de verdade. A IA de implementação não deve derivar nenhuma exigência funcional a partir deles.

---

## 5. Estrutura atual do projeto

```
agenda/
└── agenda.md                          # pessoal, fora de escopo

corpus/
├── FAQ-001_perguntas_frequentes.md
├── POL-001_onboarding.md
├── POL-002_ferias.md
├── POL-003_beneficios.md
├── POL-004_home_office_v1_REVOGADA.md # revogada — armadilha intencional
├── POL-005_trabalho_hibrido_v2.md     # vigente — substitui a POL-004
├── POL-006_reembolso.md
├── POL-007_avaliacao_desempenho.md
├── POL-008_desligamento.md
├── POL-009_seguranca_informacao.md
├── POL-010_treinamento.md
└── POL-011_codigo_conduta.md

venv/

.gitignore
assistente_politicas_internas.ipynb                          # notebook principal de desenvolvimento
duvidas.md                             # pessoal, fora de escopo
metadados.csv                          # 1 linha por documento
perguntas_gabarito.csv                 # 10 perguntas de avaliação
README_insumos.md                      # contexto dos insumos
saidas.md                          # evidências P0–P5 (+ Stretch se houver)
reflexao.md

```

Papel de cada item:

| Item | Papel |
|---|---|
| `corpus/` | Fonte de dados oficial do projeto. Não deve ser alterado. |
| `metadados.csv` | Metadados por documento (usado no P0). |
| `perguntas_gabarito.csv` | 10 perguntas de avaliação, 9 com documento esperado, 1 (P10) sem resposta no corpus. Não deve ser alterado. |
| `saidas.md` | Registro das evidências textuais exigidas pelo desafio (P0–P5, + Stretch se houver). |
| `assistente_politicas_internas.ipynb` | Notebook principal de implementação (ver seção 17). |
| `README_insumos.md` | Contexto dos insumos fornecidos. |
| `agenda/agenda.md`, `duvidas.md` | Anotações pessoais — fora de escopo (seção 4). |

---

## 6. Requisitos funcionais

1. O assistente recebe perguntas em português.
2. O assistente recupera localmente (TF-IDF) os chunks mais relevantes do corpus.
3. Apenas documentos com `status == "vigente"` participam do ranqueamento (regra de vigência, seção 11).
4. A resposta é extrativa: texto literal do chunk de maior score, sem reescrita, resumo ou paráfrase.
5. Toda resposta cita `doc_id` e seção de origem.
6. Quando o score máximo estiver abaixo do threshold definido, o assistente retorna `STATUS = nao_encontrado` com a mensagem padrão do desafio.
7. A saída segue exatamente o modelo de 5 campos (seção 14).
8. **Armadilhas intencionais tratadas como requisitos funcionais:**
   - **POL-004 x POL-005**: POL-004 (home office v1) está revogada; POL-005 (trabalho híbrido v2) é a versão vigente sobre o mesmo tema. POL-004 deve ser excluída do ranqueamento por padrão (regra de vigência).
   - **FAQ-001**: pode recuperar informações resumidas, mas a origem real da informação (o documento de política correspondente) deve ser considerada — a implementação deve preservar o `doc_id` real do chunk retornado (que pode ser o FAQ-001 em si), sem se confundir com a política que ele resume.
   - **P10**: pergunta da avaliação sem resposta no corpus. Deve resultar em `STATUS = nao_encontrado`.

---

## 7. Requisitos técnicos

- **Vetorização**: `sklearn.feature_extraction.text.TfidfVectorizer`.
  - `lowercase=True` é **obrigatório**.
  - Remoção de acentos e de stopwords em português são **opcionais**, mas a decisão tomada deve ser documentada com justificativa (evidência do P2).
- **Similaridade**: cosseno (`sklearn.metrics.pairwise.cosine_similarity` ou equivalente).
- **Chunking**: por seção Markdown (`##`), ver seção 9 (P1).
- **Sem embeddings, sem LLM** na implementação obrigatória.
- **Execução**: o notebook (ou os scripts, se houver separação — seção 17) deve rodar do início ao fim **sem intervenção manual**.
- **Sem alterações no corpus, em `metadados.csv` ou em `perguntas_gabarito.csv`.**
- **Sem invenção de dados, scores ou resultados** — todas as evidências devem refletir a execução real do código.

---

## 8. Pipeline da solução

```
corpus/*.md + metadados.csv
        │
        ▼
   [P0] Setup e leitura do corpus
        │  (12 docs confirmados, 11 vigentes + 1 revogada,
        │   tabela doc_id / status / nº palavras)
        ▼
   [P1] Chunking por seção (##)
        │  (DataFrame de chunks: doc_id, título, seção, status, texto)
        ▼
   [P2] Indexação TF-IDF
        │  (TfidfVectorizer sobre os chunks, vocabulário registrado)
        ▼
   [P3] buscar(pergunta, k=3)
        │  (filtra vigentes → similaridade do cosseno → top-k)
        ▼
   [P4] responder(pergunta)
        │  (top-1 do buscar → threshold → resposta extrativa ou nao_encontrado)
        ▼
   [P5] Avaliação
        (perguntas_gabarito.csv → Hit@1, Hit@3, análise de erro)

   [Stretch A ou B] — opcional, isolado, não altera P0–P5
```

---

## 9. Especificação das Partes P0–P5

### P0 — Setup e leitura do corpus

**Deve:**
- descompactar/usar o corpus;
- confirmar os 12 documentos;
- ler `metadados.csv` com pandas;
- carregar o conteúdo Markdown de cada documento;
- armazenar o texto completo de cada documento;
- confirmar 11 documentos vigentes e 1 revogado.

**Evidência obrigatória:** tabela com `doc_id`, `status` e número de palavras de cada documento.

### P1 — Chunking por seção

**Deve:**
- dividir os documentos pelas seções Markdown marcadas por `##`;
- cada chunk deve conter: `doc_id`, título do documento, nome da seção, status, texto da seção;
- o cabeçalho que começa com `Empresa:` **não** deve virar chunk;
- armazenar os chunks em um DataFrame;
- calcular número total de chunks e número de chunks por documento.

**Evidência obrigatória:** número total de chunks; número por documento; um exemplo completo de chunk.

### P2 — Indexação com TF-IDF

**Deve:**
- usar `TfidfVectorizer` do scikit-learn;
- transformar todos os chunks em vetores;
- `lowercase=True` obrigatório; remoção de acentos/stopwords em português opcional, mas documentada com justificativa;
- registrar o tamanho do vocabulário.

**Evidência obrigatória:** forma (shape) da matriz TF-IDF; frase documentando a decisão de pré-processamento.

### P3 — Recuperação Top-k com regra de vigência

**Deve implementar** `buscar(pergunta, k=3)`, que:
- transforma a pergunta com o mesmo vetorizador do P2;
- calcula similaridade do cosseno entre a pergunta e os chunks;
- **filtra chunks com `status == "vigente"` antes do ranqueamento** (documentos revogados nunca participam);
- retorna os `k` maiores scores;
- retorna, no mínimo, `doc_id`, seção, status e score.

**Testar obrigatoriamente:** P01, P02, P10 (perguntas do gabarito). Em P02, confirmar explicitamente que POL-004 não aparece no resultado.

**Evidência obrigatória:** os 3 chunks retornados para P01, P02 e P10, com `doc_id`, seção e score (duas casas decimais).

### P4 — Resposta extrativa e threshold

**Deve implementar** `responder(pergunta)`, que:
- chama `buscar`;
- obtém o chunk de maior score;
- monta a saída no modelo de 5 campos (seção 14);
- usa como `RESPOSTA` o texto **exato** do chunk (sem reescrever, resumir ou parafrasear).

**Threshold:**
- deve ser definido um valor numérico de corte;
- se o maior score for menor que o threshold → `STATUS = nao_encontrado`, usando a mensagem padrão definida no desafio;
- o threshold deve ser justificado observando os scores das perguntas com resposta esperada versus os scores de perguntas sem resposta (P10).

**Evidência obrigatória:** valor do threshold; justificativa; saída completa (5 campos) para P02; saída completa (5 campos) para P10.

### P5 — Avaliação

**Deve:**
- ler `perguntas_gabarito.csv`;
- para as 10 perguntas, executar `buscar(pergunta, k=3)` e `responder(pergunta)`;
- calcular Hit@1 e Hit@3 **apenas sobre as 9 perguntas com documento esperado**;
- verificar P10 separadamente quanto ao `STATUS = nao_encontrado` (não entra nas métricas de Hit@k);
- escolher uma pergunta em que houve erro **ou** cujo resultado ficou próximo do threshold, e explicar por que ocorreu e o que seria tentado para corrigir.

**Evidência obrigatória:** tabela de avaliação (por pergunta); Hit@1; Hit@3; análise de erro (a pergunta escolhida + explicação + correção proposta).

---

## 10. Regras de recuperação

- A função `buscar(pergunta, k=3)` é a única via de recuperação da solução obrigatória.
- Pipeline interno: `vetorizar(pergunta)` → `cosine_similarity(pergunta_vetor, matriz_chunks_vigentes)` → ordenar decrescente → retornar top-`k`.
- O vetorizador usado na consulta deve ser **o mesmo** ajustado (`fit`) sobre os chunks no P2 — nunca um novo `fit` sobre a pergunta.
- Retorno mínimo por item: `doc_id`, seção, status, score.
- `k` é parametrizável, com valor padrão `3`, conforme o desafio.

---

## 11. Regra de vigência

- **Regra fundamental**: por padrão, apenas chunks cujo documento de origem tem `status == "vigente"` podem participar do ranqueamento em `buscar`.
- Chunks de documentos com `status == "revogada"` (ex.: POL-004) devem ser **excluídos antes** do cálculo de similaridade/ranqueamento — não apenas filtrados depois do top-k.
- Essa regra deve ser validada explicitamente com a pergunta P02 (o teste deve confirmar que POL-004 não aparece no resultado).
- Não existe, na implementação obrigatória, um modo de "incluir revogados" — isso não é exigido pelo PDF e não deve ser adicionado como funcionalidade extra.

---

## 12. Regra de threshold

- Um único valor numérico de threshold deve ser definido para decidir entre `encontrado` e `nao_encontrado`.
- Regra de decisão: se `score_top1 < threshold` → `STATUS = nao_encontrado`.
- O valor deve ser justificado empiricamente, comparando os scores obtidos para as 9 perguntas com documento esperado e o score obtido para P10 (que não tem resposta no corpus).
- O threshold **não deve ser inventado a priori** sem observar os scores reais da execução — a justificativa faz parte da evidência obrigatória do P4.

---

## 13. Regra de resposta extrativa

- O campo `RESPOSTA` deve ser **exatamente** o texto do chunk de maior score retornado por `buscar`.
- Proibido: reescrever, resumir ou parafrasear o conteúdo do chunk.
- Isso vale tanto para o fluxo principal (P4) quanto para a avaliação (P5) e para o Stretch A, caso implementado (no Stretch A, quem gera texto livre é o modelo local, mas a citação de `FONTE` deve ser preservada — ver seção 19).

---

## 14. Modelo de saída

Toda resposta do assistente deve usar exatamente estes cinco campos, nesta ordem, com rótulos em maiúsculas:

```
PERGUNTA:
RESPOSTA:
FONTE:
SCORE:
STATUS:
```

Regras:
- `STATUS` aceita somente dois valores: `encontrado` ou `nao_encontrado`.
- Não usar emojis, cores ou símbolos gráficos como indicação de sucesso/falha.
- A resposta deve ser textual e acessível (sem formatação visual decorativa).
- Quando `STATUS = nao_encontrado`, `RESPOSTA` deve conter a mensagem padrão definida pelo desafio (ver seção final de decisões — o texto exato deve ser confirmado no PDF).

---

## 15. Avaliação

- Fonte: `perguntas_gabarito.csv` (10 perguntas — não alterar o arquivo).
- Para cada uma das 10 perguntas: rodar `buscar(pergunta, k=3)` e `responder(pergunta)`.
- **Hit@1**: proporção das 9 perguntas com documento esperado em que o `doc_id` do top-1 corresponde ao documento esperado.
- **Hit@3**: proporção das 9 perguntas com documento esperado em que o documento esperado aparece entre os 3 primeiros resultados.
- P10 é avaliada separadamente: sucesso = `STATUS = nao_encontrado`. Não entra no cálculo de Hit@1/Hit@3.
- Análise de erro: escolher 1 pergunta com erro (documento incorreto) ou com score próximo do threshold, explicar a causa provável e propor uma correção (ex.: ajuste de threshold, ajuste de pré-processamento, ajuste de chunking).

---

## 16. Evidências obrigatórias

Todas as evidências vivem em `saidas.md`, na ordem `Parte 0` → `Parte 5` (+ Stretch, se houver — ver seção 18).

| Parte | Evidência obrigatória |
|---|---|
| P0 | Tabela `doc_id`, `status`, nº de palavras (12 documentos). |
| P1 | Nº total de chunks; nº de chunks por documento; um exemplo completo de chunk. |
| P2 | Shape da matriz TF-IDF; frase justificando a decisão de pré-processamento. |
| P3 | Top-3 chunks para P01, P02 e P10 (`doc_id`, seção, score com 2 casas decimais); confirmação de que POL-004 não aparece em P02. |
| P4 | Valor do threshold + justificativa; saída completa (5 campos) para P02 e para P10. |
| P5 | Tabela de avaliação das 10 perguntas; Hit@1; Hit@3; análise de erro de 1 pergunta. |
| Stretch (se realizado) | Seção própria ao final, com evidências da opção escolhida (A ou B). |

Regra geral: **as evidências devem refletir resultados reais obtidos pela execução do código**. É proibido antecipar ou inventar números, scores ou textos de saída antes de rodar o notebook/script.

---

## 17. Organização do código

**Situação atual:** `assistente_politicas_internas.ipynb` como notebook único.

**Avaliação exigida por este SPEC (a decidir antes da implementação — ver seção 22):**

- Manter tudo em um único notebook é **suficiente** para o tamanho e escopo deste desafio (12 documentos, 10 perguntas, pipeline simples de 5 funções: leitura, chunking, indexação, busca, resposta).
- Uma separação em arquivos `.py` só deve ser recomendada se houver um motivo técnico claro, por exemplo:
  - **testabilidade**: permitir testar `buscar()`/`responder()` isoladamente;
  - **reutilização**: evitar redefinir as mesmas funções em múltiplas células;
  - **redução de repetição**: se o notebook crescer e repetir lógica de carregamento/chunking em vários pontos.
- **Recomendação por padrão deste SPEC**: manter o notebook único (`assistente_politicas_internas.ipynb`) como implementação principal, dado o tamanho pequeno do desafio, evitando uma arquitetura de múltiplos arquivos desnecessária para o escopo. Uma separação em `.py` (ex.: `retrieval.py` com `buscar`/`responder`) é aceitável **somente** se facilitar a execução ponta a ponta sem intervenção — nunca como decisão puramente estética.
- Em qualquer cenário, o requisito não negociável é: **o código deve rodar do início ao fim sem intervenção manual.**

Esta é uma decisão de implementação, não um requisito do PDF — está registrada formalmente na seção 22.

---

## 18. Organização dos arquivos

- `corpus/` é fonte de dados oficial — **não deve ser alterado**.
- `metadados.csv` e `perguntas_gabarito.csv` — **não devem ser alterados**.
- `saidas.md` concentra as evidências das Partes P0 a P5, na ordem:

```
# Parte 0
...
# Parte 1
...
# Parte 2
...
# Parte 3
...
# Parte 4
...
# Parte 5
...
# Stretch (apenas se realizado)
...
```

- `agenda/agenda.md` e `duvidas.md` **não** devem ser incorporados, citados ou transformados em documentação oficial do projeto. A IA de implementação não deve ler esses arquivos como se fossem requisitos.
- `README_insumos.md` pode ser consultado como contexto, mas não substitui o PDF como fonte de verdade.

---

## 19. Stretch (opcional)

O Stretch **não deve contaminar** a implementação obrigatória (P0–P5), que precisa funcionar de forma completa e independente, sem qualquer dependência do Stretch.

**Opção A — Geração com modelo local:**
- geração com modelo local (ex.: Ollama), sem chave de API;
- o modelo recebe **apenas** os chunks recuperados por `buscar`;
- a resposta gerada deve ser baseada **somente** nesses chunks (sem conhecimento externo);
- a `FONTE` deve ser mantida na saída.

**Opção B — Comparação TF-IDF vs BM25:**
- usar `rank_bm25` para reimplementar a recuperação;
- rodar as mesmas 10 perguntas do gabarito;
- calcular Hit@1 e Hit@3 para BM25, da mesma forma que no P5;
- apresentar uma conclusão curta comparando TF-IDF e BM25.

Se o Stretch for realizado, suas evidências vão em uma seção própria ao final de `saidas.md`, claramente separada das Partes P0–P5.

---

## 20. Critérios de conclusão

O projeto é considerado concluído quando:

1. As Partes P0 a P5 estão implementadas exatamente conforme a seção 9;
2. Todas as evidências obrigatórias (seção 16) estão presentes em `saidas.md`, na ordem correta, e refletem execução real;
3. A regra de vigência (seção 11) está corretamente aplicada e validada com P02;
4. O threshold (seção 12) está definido e justificado com base nos scores reais;
5. A resposta é extrativa (seção 13) em 100% dos casos de `STATUS = encontrado`;
6. O modelo de saída de 5 campos (seção 14) é respeitado em toda resposta gerada;
7. Hit@1 e Hit@3 foram calculados corretamente sobre as 9 perguntas com gabarito, e P10 foi tratada separadamente;
8. Nenhuma tecnologia fora de escopo (seção 4) foi introduzida na implementação obrigatória;
9. O notebook/código roda do início ao fim sem intervenção manual;
10. `corpus/`, `metadados.csv` e `perguntas_gabarito.csv` permanecem inalterados;
11. `agenda/agenda.md` e `duvidas.md` não foram usados como fonte de requisitos ou documentação oficial;
12. (Se aplicável) O Stretch está isolado em seção própria e não interfere no funcionamento da solução obrigatória.

---

## 21. Checklist final

- [ ] P0–P5 implementados conforme a seção 9, sem desvio do PDF.
- [ ] Nenhuma tecnologia fora de escopo (PySpark, SQL, bancos de dados, APIs externas, LangChain, LlamaIndex, embeddings/LLM fora do Stretch A, frontend, FastAPI) foi usada na implementação obrigatória.
- [ ] `TfidfVectorizer` com `lowercase=True`; decisão sobre acentos/stopwords documentada com justificativa.
- [ ] Regra de vigência aplicada **antes** do ranqueamento; POL-004 confirmadamente ausente em P02.
- [ ] Threshold definido e justificado com base em scores reais (perguntas com resposta vs. P10).
- [ ] `RESPOSTA` sempre extrativa (texto literal do chunk), nunca reescrita/resumida/parafraseada.
- [ ] Saída sempre no modelo de 5 campos, rótulos em maiúsculas, `STATUS` restrito a `encontrado`/`nao_encontrado`, sem emojis/símbolos.
- [ ] Hit@1 e Hit@3 calculados sobre as 9 perguntas com gabarito; P10 avaliada separadamente por `STATUS`.
- [ ] Análise de erro de pelo menos 1 pergunta (erro ou score próximo do threshold), com causa e correção propostas.
- [ ] `saidas.md` organizado na ordem Parte 0 → Parte 5 (+ Stretch, se houver), com resultados reais (nada inventado/antecipado).
- [ ] `corpus/`, `metadados.csv`, `perguntas_gabarito.csv` inalterados.
- [ ] `agenda/agenda.md` e `duvidas.md` não tratados como documentação oficial.
- [ ] Código executa do início ao fim sem intervenção manual.
- [ ] Todas as ambiguidades da seção 22 / "Pontos que precisam de decisão" foram resolvidas antes (ou no início) da implementação, e as decisões tomadas foram registradas.

---

## 22. Decisões técnicas registradas

Cada item abaixo é uma decisão de implementação (não um requisito do PDF), com decisão sugerida, motivo, impacto e classificação.

| # | Decisão | Motivo | Impacto | Tipo |
|---|---|---|---|---|
| 1 | Manter `assistente_politicas_internas.ipynb` como notebook único, sem separar em `.py` | Escopo pequeno (12 docs, 10 perguntas, 5 funções); separar arquivos sem necessidade técnica adiciona complexidade desnecessária | Baixo, se mantido simples; recomendação reversível se o notebook crescer muito | Escolha de implementação |
| 2 | Não remover acentos nem stopwords em português no TF-IDF (usar apenas `lowercase=True`) | decisão confirmada após teste comparativo | O PDF torna essa remoção opcional; manter o texto mais próximo do original reduz risco de perder termos relevantes em um corpus pequeno | Pode reduzir levemente a precisão em variações ortográficas (com/sem acento); deve ser testado e a decisão final documentada com base nos resultados reais | Escolha de implementação (decisão a validar empiricamente antes de finalizar) |
| 3 | Threshold definido empiricamente após observar os scores das 9 perguntas com gabarito e da P10, não fixado a priori | O PDF exige justificativa baseada em observação dos scores; um valor arbitrário sem essa análise violaria o requisito | Alto — decisão central do P4 e do P5 | Requisito do PDF (a **forma** de decidir é requisito; o **valor numérico** é decisão de implementação) |
| 4 | Regra de exclusão do cabeçalho `Empresa:` implementada via verificação de que a linha inicial do documento (antes da primeira `##`) não gera chunk | O PDF exige que esse cabeçalho não vire chunk, mas não especifica o mecanismo de parsing | Baixo — afeta apenas a lógica interna de chunking do P1 | Escolha de implementação |
| 5 | `k=3` como valor padrão de `buscar`, parametrizável | Explicitamente definido no PDF | — | Requisito do PDF |
| 6 | Scores exibidos com 2 casas decimais nas evidências do P3 | Explicitamente definido no PDF | — | Requisito do PDF |
| 7 | `rank_bm25` usado exclusivamente no Stretch B, nunca na implementação obrigatória | Explicitamente definido no PDF (biblioteca listada separadamente do TF-IDF obrigatório) | — | Requisito do PDF |

---

# Pontos que precisam de decisão antes da implementação

Estes pontos **não estão determinados** pelas informações fornecidas e precisam ser confirmados no PDF original (ou decididos explicitamente por você) antes de a IA de implementação começar:

1. **Texto exato da "mensagem padrão"** para `STATUS = nao_encontrado`. O PDF define uma mensagem específica que deve ser usada literalmente no campo `RESPOSTA` — o texto exato não foi transcrito neste briefing e precisa ser copiado do PDF antes da implementação.
2. **Formato exato do campo `FONTE`**: apenas `doc_id` + nome da seção, ou também o título do documento? O briefing pede "citar documento e seção", mas não define o formato textual exato (ex.: `POL-005 · Modalidades de trabalho` vs `POL-005 - Trabalho Híbrido, seção "Modalidades de trabalho"`).
3. **Valor numérico do threshold**: deve ser calculado a partir da execução real (scores obtidos). Não deve ser assumido nem fixado antes de rodar o P3/P4 com os dados reais.
4. **Critério de contagem de "número de palavras"** no P0 (evidência exigida): separação simples por espaço em branco (`str.split()`) é a abordagem mais simples e compatível com "biblioteca padrão do Python" — mas deve ser confirmada como suficiente, já que o PDF não detalha o método de tokenização.
5. **Decisão final sobre remoção de acentos/stopwords em português no TF-IDF** (opcional segundo o PDF): precisa ser testada com os dados reais e documentada com uma frase de justificativa, conforme exigido na evidência do P2.
6. **Notebook único vs. separação em `.py`**: este SPEC recomenda manter único (seção 17), mas essa é uma escolha e pode ser revista se, durante a implementação, ficar claro que há repetição excessiva de código.
7. **Tratamento do FAQ-001 na regra de vigência**: confirmar no PDF/metadados se o FAQ-001 tem `status == "vigente"` como qualquer outro documento, ou se possui tratamento especial. O briefing apenas menciona que "a origem da informação deve ser considerada" — o mecanismo exato de rastrear a origem não é especificado.
8. **Se rank_bm25 pode ser usado fora do Stretch B para algum fim exploratório** (ex.: comparação informal durante o desenvolvimento, sem entrar nas evidências obrigatórias) — este SPEC assume que **não**, para evitar qualquer contaminação do escopo obrigatório, mas essa é uma interpretação conservadora que vale confirmar.

---

# Checklist de conformidade P0–P5

**P0**
- [ ] 12 documentos confirmados (11 vigentes + 1 revogado).
- [ ] `metadados.csv` lido com pandas.
- [ ] Conteúdo Markdown de cada documento carregado e armazenado.
- [ ] Tabela `doc_id` / `status` / nº de palavras gerada como evidência.

**P1**
- [ ] Chunking feito por seção Markdown (`##`).
- [ ] Cada chunk contém `doc_id`, título, seção, status e texto.
- [ ] Cabeçalho `Empresa:` não virou chunk.
- [ ] Chunks armazenados em DataFrame.
- [ ] Nº total de chunks e nº por documento calculados e reportados.
- [ ] Um exemplo completo de chunk apresentado como evidência.

**P2**
- [ ] `TfidfVectorizer` usado, `lowercase=True`.
- [ ] Decisão sobre acentos/stopwords documentada com justificativa.
- [ ] Tamanho do vocabulário registrado.
- [ ] Shape da matriz TF-IDF reportado como evidência.

**P3**
- [ ] `buscar(pergunta, k=3)` implementada.
- [ ] Mesmo vetorizador do P2 usado para transformar a pergunta.
- [ ] Similaridade do cosseno calculada corretamente.
- [ ] Apenas chunks `vigente` participam do ranqueamento.
- [ ] Retorno inclui `doc_id`, seção, status e score.
- [ ] Testado com P01, P02 e P10.
- [ ] Confirmado que POL-004 não aparece em P02.
- [ ] Evidência: top-3 de P01, P02, P10, scores com 2 casas decimais.

**P4**
- [ ] `responder(pergunta)` implementada, usando `buscar` internamente.
- [ ] `RESPOSTA` é o texto exato do chunk (sem reescrita).
- [ ] Threshold definido e justificado com scores reais.
- [ ] `STATUS = nao_encontrado` quando score < threshold, com mensagem padrão do desafio.
- [ ] Evidência: threshold + justificativa; saída completa (5 campos) para P02 e P10.

**P5**
- [ ] `perguntas_gabarito.csv` lido sem alterações.
- [ ] `buscar` e `responder` executados para as 10 perguntas.
- [ ] Hit@1 e Hit@3 calculados sobre as 9 perguntas com gabarito.
- [ ] P10 avaliada separadamente quanto ao `STATUS`.
- [ ] 1 pergunta com erro/score próximo do threshold analisada (causa + correção proposta).
- [ ] Evidência: tabela de avaliação, Hit@1, Hit@3, análise de erro.

---

# Checklist final (rubrica do desafio)

- [ ] Recuperação local via TF-IDF, sem chave de API, conforme exigido.
- [ ] Nenhuma tecnologia fora de escopo (seção 4) presente na implementação obrigatória.
- [ ] Regra de vigência corretamente implementada e validada.
- [ ] Resposta sempre extrativa, nunca reescrita.
- [ ] Modelo de saída de 5 campos respeitado em 100% das respostas.
- [ ] Threshold justificado com dados reais, não arbitrário.
- [ ] Métricas Hit@1 e Hit@3 calculadas corretamente, apenas sobre as 9 perguntas com gabarito.
- [ ] P10 tratada corretamente como caso de "não encontrado".
- [ ] Evidências completas e organizadas em `saidas.md`, na ordem Parte 0 → Parte 5.
- [ ] Nenhum dado, score ou resultado inventado — tudo reflete execução real.
- [ ] Corpus, metadados e gabarito preservados sem alterações.
- [ ] `agenda/` e `duvidas.md` excluídos da documentação oficial.
- [ ] Código executável do início ao fim sem intervenção manual.
- [ ] Stretch (se realizado) isolado, opcional, e não interfere na solução obrigatória.
