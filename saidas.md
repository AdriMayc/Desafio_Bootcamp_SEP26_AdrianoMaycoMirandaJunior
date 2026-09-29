# Saídas — Desafio Bootcamp SEP26

**Autor:** Adriano Mayco Miranda Junior  
**Como reproduzir:** Rodar o notebook do início ao fim na pasta com corpus/, metadados.csv e perguntas_gabarito.csv.


## Parte 0 — Setup e leitura do corpus

> Tabela:

| doc_id | status | número de palavras |
|---|---|---:|
| POL-001 | vigente | 159 |
| POL-002 | vigente | 130 |
| POL-003 | vigente | 122 |
| POL-004 | revogada | 87 |
| POL-005 | vigente | 114 |
| POL-006 | vigente | 111 |
| POL-007 | vigente | 108 |
| POL-008 | vigente | 113 |
| POL-009 | vigente | 136 |
| POL-010 | vigente | 87 |
| POL-011 | vigente | 98 |
| FAQ-001 | vigente | 137 |

> Verificação:  12 documentos; 11 vigentes; 1 revogada.

## Parte 1 — Chunking por seção

> Total de chunks: 46  

> Número de chunks por documento:

| doc_id | quantidade de chunks |
|---|---:|
| POL-001 | 6 |
| POL-002 | 3 |
| POL-003 | 4 |
| POL-004 | 4 |
| POL-005 | 3 |
| POL-006 | 3 |
| POL-007 | 4 |
| POL-008 | 4 |
| POL-009 | 4 |
| POL-010 | 4 |
| POL-011 | 3 |
| FAQ-001 | 4 |


> Exemplo de chunk completo (todos os campos): 

````text
doc_id: POL-001
titulo do documento: Política de Onboarding
nome da seção: Objetivo
status: vigente
texto da seção: Esta política define as etapas dos primeiros 30 dias de um novo colaborador na Horizonte Tech.
````
> Observação: 

- Quantidade total de chunks: 46  
- Menor chunk em palavras: 10  
- Maior chunk em palavras: 64  
- Média de palavras por chunk: 21  

## Parte 2 — Indexação com TF-IDF

- Forma da matriz: (46, 350); vocabulário: 350
- Pré-processamento: 
    - Decisão utilizada foi usar apenas: lowercase=True; 
- Motivo:    
    - O lowercase foi adotado para evitar que diferenças de capitalização gerassem termos distintos e fragmentassem desnecessariamente o vocabulário.


> Teste de remoção de Acentos e Stopwords:

| **Configuração** | **Hit@1** | **Hit@3** | **Menor score positivo** | **Score P10** | **Folga** |
|---|---:|---:|---:|---:|---:|
| 0 — BASE | 1.0000 | 1.0000 | 0.3080 | 0.2661 | 0.0419 |
| 1 — SEM STOPWORDS | 1.0000 | 1.0000 | 0.3776 | 0.2242 | 0.1534 |
| 2 — SEM ACENTOS | 1.0000 | 1.0000 | 0.3080 | 0.2661 | 0.0419 |

- BASE: a folga permaneceu igual para 0.0419; Hit@1/Hit@3 não mudaram em relacao a BASE.  
- SEM STOPWORDS: a folga aumentou para 0.1534; Hit@1/Hit@3 não mudaram em relacao a BASE.  
- SEM ACENTOS: a folga permaneceu igual para 0.0419; Hit@1/Hit@3 não mudaram em relacao a BASE.  


## Parte 3 — Recuperação top-k com regra de vigência


P01 — Com quantos dias de antecedência devo solicitar minhas férias?

| índice | doc_id | seção | status | score |
|---:|---|---|---|---:|
| 0 | POL-002 | Como solicitar | vigente | 0.41 |
| 1 | POL-002 | Venda de dias | vigente | 0.41 |
| 2 | POL-002 | Direito a férias | vigente | 0.25 |

P02 — Quantos dias por semana posso trabalhar de forma remota?

| índice | doc_id | seção | status | score |
|---:|---|---|---|---:|
| 0 | POL-005 | Regra de trabalho remoto | vigente | 0.53 |
| 1 | FAQ-001 | Posso trabalhar remoto todos os dias? | vigente | 0.20 |
| 2 | POL-008 | Entrevista de saída | vigente | 0.17 |

P10 — Qual é a política de estacionamento da empresa?

| índice | doc_id | seção | status | score |
|---:|---|---|---|---:|
| 0 | POL-003 | Auxílio para trabalho remoto | vigente | 0.27 |
| 1 | POL-008 | Aviso | vigente | 0.21 |
| 2 | POL-001 | Objetivo | vigente | 0.18 |

**Validação:** POL-004 aparece nos resultados de P02: `False`


## Parte 4 — Resposta extrativa e regra de não encontrado

> Justificativa do Threshold  

THRESHOLD = 0.30, definido com base nos scores observados: a consulta não encontrada P10 obteve 0.27, enquanto a menor pontuação entre as consultas encontradas foi 0.32 (P09). Este valor permite separar esses casos no conjunto de avaliação utilizado.

Saída P02:
````
PERGUNTA: Quantos dias por semana posso trabalhar de forma remota?
RESPOSTA: O colaborador pode trabalhar de forma remota em até 3 dias por semana. Os dias presenciais obrigatórios são terça-feira e quinta-feira.
FONTE: POL-005 | Política de Trabalho Híbrido (versão 2) | Seção: Regra de trabalho remoto
SCORE: 0.53
STATUS: encontrado
````

Saída P10:
````
PERGUNTA: Qual é a política de estacionamento da empresa?
RESPOSTA: Não encontrei essa informação nas políticas vigentes. Procure a área de Pessoas e Cultura.
FONTE: nenhuma
SCORE: 0.27
STATUS: nao_encontrado
````

> Armadilha intencional — FAQ-001

O FAQ-001 pode reproduzir, de forma resumida, informações presentes em outras políticas.  
Quando o FAQ contém uma referência explícita à política de origem, a recuperação é considerada aceitável se a fonte retornada identificar tanto o FAQ-001 quanto essa política no campo `Origem:`.

> Evidência:

Validação do FAQ-001 concluída: FAQ-001 | Perguntas Frequentes de Novos Colaboradores | Seção: Quando recebo meus acessos? | Origem: POL-001 | Política de Onboarding


## Parte 5 — Avaliação com o gabarito

> Tabela:

| **pergunta_id** | **doc_esperado** | **doc_retornado_1** | **score** | **STATUS** | **acerto** |
|---:|---:|---|---:|---|---|
| 0 | P01 | POL-002 | 0.4107 | encontrado | sim |
| 1 | P02 | POL-005 | 0.5276 | encontrado | sim |
| 2 | P03 | POL-003 | 0.5589 | encontrado | sim |
| 3 | P04 | POL-006 | 0.3080 | encontrado | sim |
| 4 | P05 | POL-009 | 0.3306 | encontrado | sim |
| 5 | P06 | POL-007 | 0.4958 | encontrado | sim |
| 6 | P07 | POL-008 | 0.3883 | encontrado | sim |
| 7 | P08 | POL-009 | 0.4686 | encontrado | sim |
| 8 | P09 | POL-010 | 0.3228 | encontrado | sim |
| 9 | P10 | nao_encontrado | 0.2661 | nao_encontrado | sim |

> Verificação do STATUS "nao_encontrado" referente ao P10:
- Acerto: sim

> Valores de Hit@1 e Hit@3:

- Hit@1: 1.00  
- Hit@3: 1.00

> Análise de erro:

Threshold utilizado: 0.30

P04 é um caso near-threshold: a POL-006 foi recuperada corretamente em primeiro lugar, mas com score próximo de 0,30.  
As duas alternativas do TOP-3 apresentam scores bem menores, mantendo a recuperação correta.  
A causa provável é a baixa sobreposição lexical entre a pergunta e o texto da política na representação TF-IDF.  
Como tentativa de correção, seria avaliado o uso de n-gramas no TF-IDF para capturar combinações de termos presentes na pergunta. 

> Referencial do TOP-3 (complementar a análise de erro):
1. POL-006 | Prazo para envio | score: 0.3080
2. POL-001 | Primeiro mês | score: 0.1676
3. POL-006 | Pagamento | score: 0.1426
