# Apostila InterSystems ObjectScript Specialist

Versão revisada — 23 capítulos, cobrindo os cinco domínios da certificação.

---

## Índice

| Arquivo | Capítulo | Tópico da prova |
|---|---|---|
| `cap00-ambiente` | 0 — Preparando o ambiente | — |
| `cap01-uses-classes` | 1 — Classes e objetos | T1.1 |
| `cap02-properties-indexes` | 2 — Propriedades e índices | T1.2 |
| `cap03-methods` | 3 — Métodos e callbacks | T1.3 |
| `cap04-json-streams` | 4 — JSON e streams | T1.4 |
| `cap05-transacoes-locking` | 5 — Transações e travas | T2.1 |
| `cap06-tracking-auditoria` | 6 — Auditoria e triggers | T2.2 |
| `cap07-seguranca` | 7 — Segurança | T2.3 |
| `cap08-armazenamento` | 8 — Globais e armazenamento | T3.1 |
| `cap09-objectscript-sql` | 9 — ObjectScript e SQL | T3.2 |
| `cap10-nulls` | 10 — Nulos e vazios | T3.3 |
| `cap11-schema-evolution` | 11 — Evolução de esquema | T3.4 |
| `cap12-performance` | 12 — Desempenho e escala | T3.5 |
| `cap13-arrays` | 13 — Arrays e ordenação | T4.1 |
| `cap14-listas` | 14 — Listas `$LIST` | T4.2 |
| `cap15-strings` | 15 — Manipulação de texto | T4.3 |
| `cap16-math-logic-datetime` | 16 — Matemática, lógica, data e hora | T4.4 |
| `cap17-controle-fluxo` | 17 — Decisão e controle de fluxo | T4.5 |
| `cap18-metodos-consultas` | 18 — Métodos e consulta a objetos | T4.6 |
| `cap19-apis` | 19 — APIs para operações comuns | T4.7 |
| `cap20-diagnostico` | 20 — Ferramentas de diagnóstico | T5.1 |
| `cap21-tratamento-erros` | 21 — Tratamento e registro de erros | T5.2 |
| `cap22-erros-comuns` | 22 — Erros comuns | T5.3 |

---

## Peso dos domínios na prova

| Domínio | Capítulos | Questões |
|---|---|---|
| T1 — Objects | 1 a 4 | 23 |
| T2 — Data Integrity | 5 a 7 | 15 |
| T3 — IRIS Features | 8 a 12 | 14 |
| **T4 — Functions & APIs** | **13 a 19** | **26** |
| T5 — Errors | 20 a 22 | 12 |
| | | **76** |

**T4 sozinho vale mais de um terço da prova.** Distribua o tempo de estudo proporcionalmente.

---

## Como cada capítulo é organizado

1. Objetivo
2. O conceito em linguagem de gente
3. A sintaxe explicada
4. Exemplo comentado
5. Variações e detalhes
6. Pegadinhas e erros comuns
7. **MÃO NA MASSA** — exercícios com solução comentada
8. Quiz com gabarito comentado
9. Resumo relâmpago
10. Cartões de memorização

Um projeto contínuo — **LabStudy**, um sistema de laboratório clínico — evolui da versão 0.1 (capítulo 0) até a 3.0 (capítulo 22), acumulando mais de trinta classes organizadas em camadas.

---

## Sobre esta revisão

Esta versão passou por uma auditoria completa, que corrigiu **59 problemas**:

| Grupo | Itens | O que era |
|---|---|---|
| A | 8 | bugs de código (o principal: cálculo de idade errado em 6 métodos) |
| B | 3 | trechos que contradiziam o que a própria apostila ensinava |
| C | 7 | dependências de comportamento não confirmado |
| D | 36 | referências cruzadas quebradas |
| E | 5 | fragilidades menores |

### Limite conhecido

A revisão verificou **coerência interna, lógica dos algoritmos e consistência das referências**. Ela **não** incluiu execução contra uma instância IRIS real.

Isso importa em dois pontos:

**1. Saídas de terminal são ilustrativas.** Números de tempo, contagens e distribuições vieram de execuções específicas. Os seus serão diferentes; o que deve se repetir são as proporções e as conclusões.

**2. Alguns nomes de API variam por versão.** Nesses pontos a apostila traz o **comando exato de verificação** em vez de uma afirmação. Rode-os antes de confiar nos trechos correspondentes:

- **Cap. 11** — `%BuildIndices`, `%PurgeIndices`, `%ValidateIndices`, `GatherTableStats`
- **Cap. 15 e 19** — a ordem dos argumentos de `$ZSTRIP`
- **Cap. 16** — a relação entre `$HOROLOG`, `$ZTIMESTAMP` e `$ZTIMEZONE`, e o dia da semana do dia zero
- **Cap. 19** — as colunas e o formato de data do `%File:FileSet`

### A postura que a apostila recomenda

> **Nunca confie numa saída que você não executou.**

Isso vale para as saídas deste material também. O capítulo 16 traz o relato de um erro que a própria apostila cometeu e que sobreviveu por vários capítulos — mantido de propósito, porque ilustra melhor do que qualquer exemplo inventado como um erro silencioso se instala.

---

## Sugestão de uso na reta final

1. Refaça os **quizzes** sem olhar o gabarito (mais de 350 questões no total).
2. Releia apenas os **resumos relâmpago**.
3. Use os **cartões** nas duas últimas semanas.
4. **Execute o projeto.** Ler código não é o mesmo que rodá-lo — e os erros que você cometer digitando são exatamente os que a prova cobra.
5. Revise as seções **"Pegadinhas e erros comuns"**, que listam o que os candidatos realmente erram.

Boa prova.
