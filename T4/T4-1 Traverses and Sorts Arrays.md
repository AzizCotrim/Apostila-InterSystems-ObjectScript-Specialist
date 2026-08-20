# Apostila InterSystems ObjectScript Specialist
## Capítulo 13 — T4.1 Traverses and Sorts Arrays (Percorrer e ordenar arrays)

> Começa aqui o domínio **T4 — Functions & APIs**, o de **maior peso na prova**: 26 das 76 questões. Este primeiro tópico é a base de tudo: a estrutura de árvore que você já viu nas globais, agora usada como ferramenta de trabalho na memória — e a descoberta de que, no ObjectScript, **ordenar é gravar**.

---

## 1. O que você vai saber fazer ao terminar

1. Criar e usar **arrays locais** com subscritos, entendendo que são a mesma estrutura de árvore das globais.
2. Percorrer um nível com **`$ORDER`**, para frente e para trás, com e sem o terceiro argumento.
3. Percorrer uma árvore inteira com **`$QUERY`**.
4. Testar existência com **`$DATA`** e ler com segurança com **`$GET`**.
5. Dominar a **ordem de classificação** dos subscritos: vazio, números canônicos, texto.
6. Distinguir **número canônico** de texto que parece número — e prever a ordenação daí decorrente.
7. Copiar subárvores com **`MERGE`** e apagar com **`KILL`** e **`ZKILL`**.
8. Usar **`$NAME`**, **`$QLENGTH`** e **`$QSUBSCRIPT`** para manipular referências.
9. Reconhecer a **referência nua** e saber por que evitá-la.
10. Aplicar o idioma central do ObjectScript: **usar o valor como subscrito para ordenar**.
11. Ordenar por **vários critérios**, tratar **empates** e ordenar **texto sem diferenciar maiúsculas**.
12. Conhecer **`$SORTBEGIN`/`$SORTEND`** para ordenações de grande volume.
13. Passar arrays entre métodos com **`ByRef`** e **`Output`**.
14. Levar o projeto à versão **1.4**, com um utilitário de ordenação e agrupamento.

---

## 2. O conceito em linguagem de gente

### 2.1 Array local: a mesma árvore, sem o circunflexo

No Capítulo 8 você aprendeu que uma global é uma árvore de gavetas. A notícia deste capítulo é simples e poderosa:

> **Uma variável local com subscritos é exatamente a mesma coisa — só que na memória.**

```objectscript
set contagem("HGB") = 12
set contagem("GLU") = 30
set contagem("HGB", "normal") = 9
```

Sem circunflexo, mora na memória e morre com o processo. Com circunflexo, mora em disco. **Tudo o mais é idêntico**: subscritos, `$ORDER`, `$QUERY`, `$DATA`, `MERGE`, `KILL`.

Isso significa que aprender uma coisa é aprender as duas. E significa também que você pode prototipar em memória e depois trocar para global mudando um caractere.

Vocabulário, relembrando:

- **Nó** — cada gaveta: `contagem("HGB", "normal")`.
- **Subscritos** — o que vai entre parênteses.
- **Nível** — a profundidade.
- **Nó raiz** — a variável sem subscrito: `contagem`.

### 2.2 A grande revelação: no ObjectScript, ordenar é gravar

Em quase toda linguagem, ordenar é chamar uma função de ordenação sobre uma lista. No ObjectScript, **não existe** essa função — e ela não faz falta, porque existe algo melhor.

**Os subscritos já vêm ordenados.** Sempre. Automaticamente. Você nunca precisa pedir.

Então o idioma é este: **se você quer os dados ordenados por alguma coisa, use essa coisa como subscrito.**

Suponha que você tenha resultados de exames e queira listá-los do maior para o menor:

```objectscript
// Os dados como chegaram, na ordem que vierem
set exame(1) = 190
set exame(2) = 13.5
set exame(3) = 92

// Vira um índice: o VALOR passa a ser o subscrito
set porValor(190, 1) = ""
set porValor(13.5, 2) = ""
set porValor(92, 3) = ""
```

Pronto. Percorrer `porValor` com `$ORDER` já devolve `13.5`, `92`, `190` — em ordem. Percorrer com `$ORDER(..., -1)` devolve o contrário. **Você não ordenou nada; você guardou de um jeito que já nasce ordenado.**

A analogia é o fichário do Capítulo 2, e não é coincidência: **é exatamente assim que os índices do IRIS funcionam por dentro**. Ao aprender esse idioma, você está aprendendo o mecanismo do próprio banco.

E note o segundo subscrito, o `1`, `2`, `3`: ele existe para resolver empates. Se dois exames tivessem valor 92, precisariam ocupar gavetas diferentes. Voltaremos a isso.

### 2.3 A ordem exata dos subscritos

Se ordenar é gravar, então **saber a ordem exata** é essencial. A regra do IRIS:

1. Primeiro o **subscrito vazio** (`""`), se existir.
2. Depois os **números canônicos**, em ordem **numérica** — inclusive negativos.
3. Depois o **texto**, em ordem de **código de caractere**.

Um exemplo que vale mais que a regra:

```
LABSTUDY>KILL a
LABSTUDY>SET a(10)="", a(2)="", a(-5)="", a("B")="", a("a")="", a("10")="", a("007")=""

LABSTUDY>SET k="" FOR { SET k=$ORDER(a(k)) QUIT:k=""  WRITE "[",k,"] " }
[-5] [2] [10] [007] [B] [a]
```

Observe com atenção:

- **`-5` veio antes de `2`**, que veio antes de `10`. Ordem numérica de verdade, não alfabética.
- **`a("10")` e `a(10)` são o MESMO nó.** A string `"10"` é a forma canônica do número 10, então o IRIS os trata como idênticos. Por isso só apareceu um `10`.
- **`"007"` foi para o meio do texto**, e não junto dos números. Porque `"007"` **não é canônico** — números canônicos não têm zeros à esquerda.
- **`"B"` veio antes de `"a"`**, porque a ordenação de texto segue o código dos caracteres, e maiúsculas vêm antes de minúsculas.

### 2.4 O que é um número canônico

Um valor é tratado como **número** (e portanto ordenado numericamente) quando está escrito na forma canônica:

- sem zeros à esquerda: `7` é canônico, `007` não;
- sem zeros supérfluos à direita da vírgula decimal: `1.5` é canônico, `1.50` não;
- sem sinal de mais: `5` é canônico, `+5` não;
- sem ponto decimal desnecessário: `10` é canônico, `10.` e `10.0` não;
- `0` é canônico; `-0` não;
- números negativos canônicos: `-5` sim.

Tudo o que não for canônico é **texto**, e ordena como texto.

Isso tem uma consequência prática importante: se você usar códigos como `"001"`, `"002"`, `"010"` como subscritos, eles ordenarão **como texto** — o que, nesse caso específico, dá o resultado certo, porque todos têm o mesmo comprimento. Mas misturar `"7"` e `"007"` na mesma estrutura produz duas gavetas diferentes e uma ordenação confusa.

**Regra prática:** escolha um formato e seja consistente. Ou tudo numérico canônico, ou tudo texto com preenchimento de largura fixa.

### 2.5 Esparso por natureza

Um array do ObjectScript é **esparso**: só existem os nós que você criou.

```objectscript
set x(1) = "a"
set x(1000000) = "b"
```

Isso ocupa **dois nós**, não um milhão. Não há reserva de espaço, não há dimensionamento, não há índice fora de faixa.

E é por isso que percorrer com `$ORDER` é o único jeito correto: um `for i = 1:1:1000000` visitaria 999.998 posições inexistentes.

---

## 3. As ferramentas

### 3.1 `$DATA` — existe? tem filhos?

| Valor | Significado |
|---|---|
| **0** | não existe |
| **1** | tem valor, sem descendentes |
| **10** | sem valor, com descendentes |
| **11** | tem valor e descendentes |

**Dezenas = filhos. Unidades = valor.**

```
LABSTUDY>KILL a
LABSTUDY>SET a("x")=1, a("y","z")=2, a("w")=3, a("w","k")=4

LABSTUDY>WRITE $DATA(a("nada")), " ", $DATA(a("x")), " ", $DATA(a("y")), " ", $DATA(a("w")), !
0 1 10 11
```

Forma de dois argumentos, que também traz o valor:

```objectscript
if $DATA(a("x"), valor) {
    write valor, !
}
```

### 3.2 `$GET` — ler sem explodir

```objectscript
set v = $GET(a("talvez"))          // "" se não existir
set v = $GET(a("talvez"), 0)       // 0 se não existir
```

Ler um nó inexistente diretamente causa **`<UNDEFINED>`**. Em código de produção, **sempre `$GET`**.

### 3.3 `$ORDER` — andar num nível

A forma completa:

```
$ORDER(referência, direção, .valor)
```

- **`referência`** — o nó a partir do qual avançar. **Obrigatório.**
- **`direção`** — `1` (padrão, para frente) ou `-1` (para trás). **Opcional.**
- **`.valor`** — variável que recebe, por referência, o valor do nó encontrado. **Opcional.**

O padrão de laço, que você escreverá milhares de vezes:

```objectscript
set k = ""
for {
    set k = $ORDER(a(k))
    quit:k=""
    write k, " = ", $GET(a(k)), !
}
```

**Para trás:**

```objectscript
set k = ""
for {
    set k = $ORDER(a(k), -1)
    quit:k=""
    write k, " ", !
}
```

Note que **os dois começam com `""`**. O subscrito vazio significa "antes do primeiro" na direção crescente e "depois do último" na decrescente. É a mesma sentinela nos dois sentidos.

**Com o valor de brinde:**

```objectscript
set k = ""
for {
    set k = $ORDER(a(k), 1, v)
    quit:k=""
    write k, " = ", v, !
}
```

Isso economiza uma leitura por volta. Em laços de milhões de iterações, é uma otimização real — e você viu no Capítulo 12 por que isso importa.

**Um detalhe que cai na prova:** `$ORDER` **não exige que o subscrito atual exista**. Você pode perguntar "qual vem depois de `M`?" mesmo sem haver nada em `M`. É isso que permite buscas por faixa:

```objectscript
// Todos os nomes de "M" em diante
set k = "M"
for {
    set k = $ORDER(nomes(k))
    quit:k=""
    write k, !
}
```

**Pegando o primeiro e o último:**

```objectscript
set primeiro = $ORDER(a(""))
set ultimo = $ORDER(a(""), -1)
```

### 3.4 Percorrer níveis aninhados

Fixe os subscritos superiores e percorra o de baixo:

```objectscript
set codigo = ""
for {
    set codigo = $ORDER(dados(codigo))
    quit:codigo=""

    write codigo, ":", !

    set id = ""
    for {
        set id = $ORDER(dados(codigo, id))
        quit:id=""
        write "   ", id, " -> ", $GET(dados(codigo, id)), !
    }
}
```

Este é o padrão de dois níveis. Para três, aninhe mais um. Para profundidade **desconhecida**, use `$QUERY` ou recursão.

### 3.5 `$QUERY` — percorrer a árvore inteira

```objectscript
set ref = "a"
for {
    set ref = $QUERY(@ref)
    quit:ref=""
    write ref, " = ", @ref, !
}
```

- Devolve a **referência completa** do próximo nó **com valor**, como texto: `a("y","z")`.
- Percorre entrando e saindo dos níveis, na ordem correta da árvore.
- **Nós que só têm filhos, sem valor próprio, não aparecem** — `$QUERY` visita apenas nós que têm valor.
- O `@` é **indireção**: trata o conteúdo da variável como referência.

Quando usar cada um:

- **`$ORDER`** — você conhece a estrutura, quer um nível. Mais rápido. É o caso normal.
- **`$QUERY`** — estrutura desconhecida ou despejo completo. É o que o `ZWRITE` faz por dentro.

### 3.6 `$NAME`, `$QLENGTH`, `$QSUBSCRIPT` — trabalhando com referências

Quando você tem uma referência como texto (vinda do `$QUERY`, por exemplo), estas funções a desmontam:

```
LABSTUDY>SET ref = $NAME(a("x", "y", 3))

LABSTUDY>WRITE ref, !
a("x","y",3)

LABSTUDY>WRITE $QLENGTH(ref), !
3

LABSTUDY>WRITE $QSUBSCRIPT(ref, 0), !
a
LABSTUDY>WRITE $QSUBSCRIPT(ref, 1), !
x
LABSTUDY>WRITE $QSUBSCRIPT(ref, 3), !
3
```

- **`$NAME(referência)`** — devolve a referência como **texto**, sem avaliar o valor. Serve para montar referências que você guardará ou passará adiante.
- **`$NAME(referência, n)`** — devolve a referência **truncada** em `n` subscritos: `$NAME(a("x","y",3), 1)` dá `a("x")`.
- **`$QLENGTH(ref)`** — quantos subscritos a referência tem.
- **`$QSUBSCRIPT(ref, n)`** — o n-ésimo subscrito. Com `n = 0`, devolve o **nome** da variável ou global.

Isso é o que permite escrever código genérico que trabalha sobre qualquer estrutura sem saber a forma dela de antemão.

### 3.7 A referência nua — reconheça e evite

O ObjectScript permite omitir os primeiros subscritos, reaproveitando os da última referência usada:

```objectscript
set ^X(1, 2, 3) = "a"
set ^(4) = "b"          // equivale a ^X(1, 2, 4)
```

Isso se chama **referência nua** (*naked reference*). Ela existe por razões históricas: economizava digitação e bytes.

**Não use em código novo.** Motivos:

- O significado depende de qual foi a **última** referência usada, o que pode ter sido decidido em outro método.
- Inserir uma linha no meio do código muda silenciosamente o alvo.
- Torna o código ilegível para quem chega depois.

Você precisa **reconhecê-la** para ler código antigo — isso cai na prova. Mas escrevê-la em 2026 é criar um problema para o seu sucessor.

### 3.8 `MERGE` — copiar subárvores

```objectscript
merge destino = origem
merge destino = origem("ramo")
merge ^Backup = local
merge local = ^Config
```

- Copia o nó **e toda a subárvore**, numa operação.
- **Não limpa o destino antes**: mescla e sobrescreve o que coincidir.
- Funciona entre qualquer combinação de local, PPG e global.

Para substituir de verdade:

```objectscript
kill destino
merge destino = origem
```

### 3.9 `KILL` e `ZKILL`

```objectscript
kill a("ramo")      // remove o nó E toda a subárvore
zkill a("ramo")     // remove só o valor, preserva os filhos
kill a              // remove a variável inteira
kill                // remove TODAS as variáveis locais
```

`KILL` sem argumento apaga todas as locais do contexto — dentro de um bloco de procedimento, isso é limitado ao próprio método, mas ainda assim é uma ferramenta afiada.

Existe também a forma **exclusiva**:

```objectscript
kill (manter1, manter2)     // apaga tudo EXCETO essas
```

### 3.10 Contando elementos

Não existe função de contagem. Ou você conta ao percorrer:

```objectscript
set total = 0, k = ""
for {
    set k = $ORDER(a(k))
    quit:k=""
    set total = total + 1
}
```

Ou você **mantém o contador enquanto insere**, que é muito melhor:

```objectscript
set a(chave) = valor
set a = $INCREMENT(a)      // contador na raiz
```

Repare: o contador fica no **nó raiz**, e os dados nos filhos — exatamente o padrão que o IRIS usa nas globais de classes persistentes, como você viu no Capítulo 8.

`$INCREMENT` funciona em variáveis locais também, não só em globais.

### 3.11 Passando arrays entre métodos

Arrays **não** são passados por valor. Você precisa de `ByRef` ou `Output`, e do **ponto** na chamada:

```objectscript
ClassMethod Preencher(Output dados) As %Status
{
    kill dados
    set dados("a") = 1
    set dados("b") = 2
    quit $$$OK
}

ClassMethod Usar() As %Status
{
    do ..Preencher(.resultado)      // o ponto é obrigatório
    write $GET(resultado("a")), !
    quit $$$OK
}
```

Sem o ponto, o método recebe uma cópia do **valor do nó raiz** — normalmente vazio — e nenhum dos subscritos. É o mesmo erro silencioso do Capítulo 3, agora com consequência maior, porque o array inteiro se perde.

**Convenção:** um método que preenche um array deve começar com `kill` sobre ele. Assim, quem chama não fica com restos de uma chamada anterior.

---

## 4. Ordenando na prática

### 4.1 O padrão do índice invertido

O problema: você tem dados indexados por ID e quer listá-los ordenados por um valor.

```objectscript
// Dados originais, na ordem de chegada
set exame(1) = 190
set exame(2) = 13.5
set exame(3) = 92
set exame(4) = 92

// Passo 1: construir o índice invertido
set id = ""
for {
    set id = $ORDER(exame(id))
    quit:id=""
    set porValor(exame(id), id) = ""
}

// Passo 2: percorrer o índice, já ordenado
set v = ""
for {
    set v = $ORDER(porValor(v))
    quit:v=""

    set id = ""
    for {
        set id = $ORDER(porValor(v, id))
        quit:id=""
        write v, " -> id ", id, !
    }
}
```

Saída:

```
13.5 -> id 2
92 -> id 3
92 -> id 4
190 -> id 1
```

Pontos essenciais:

- **O valor virou o primeiro subscrito.** É isso que produz a ordenação.
- **O ID virou o segundo subscrito.** É isso que permite **empates**: os dois exames com 92 ocupam gavetas diferentes.
- **O nó fica vazio (`""`).** Toda a informação está na posição. Exatamente como nos índices do IRIS.

Se você usasse apenas `porValor(exame(id)) = id`, o segundo exame com 92 sobrescreveria o primeiro, e um dado sumiria silenciosamente. **Esse é o erro número um deste capítulo.**

### 4.2 Ordem decrescente

Basta percorrer ao contrário:

```objectscript
set v = ""
for {
    set v = $ORDER(porValor(v), -1)
    quit:v=""
    // ...
}
```

Nenhuma estrutura extra é necessária. A mesma árvore serve para as duas direções.

### 4.3 Ordenar por vários critérios

Empilhe subscritos, na ordem de prioridade:

```objectscript
// Ordenar por cidade, depois por nome
set porCidadeNome(cidade, nome, id) = ""
```

Percorrer com três laços aninhados produz a ordenação por cidade e, dentro de cada cidade, por nome.

**A ordem dos subscritos é a ordem dos critérios.** Trocar `cidade` e `nome` de posição muda completamente o resultado.

E se um critério for decrescente e o outro crescente? Percorra o primeiro nível com `-1` e o segundo com `1`. Cada nível tem sua própria direção.

### 4.4 Ordenar texto sem diferenciar maiúsculas

O problema, como você viu na seção 2.3: `"B"` vem antes de `"a"`.

A solução é **normalizar a chave** e guardar o original no nó:

```objectscript
set porNome($ZCONVERT(nome, "U"), id) = nome
```

Agora a ordenação é insensível a maiúsculas, e o nome original continua disponível no valor do nó.

Repare que é exatamente o que o IRIS faz nos índices de texto — você viu `^LabStudy.PatientI("NameIdx", " MARIA SILVA", 1)` no Capítulo 8. Agora você sabe por quê.

Para ordenação correta de acentos, o IRIS oferece funções de classificação específicas por idioma. Os nomes e o comportamento variam conforme a configuração de localidade da instância: **verificar na documentação oficial**.

### 4.5 Ordenar números formatados como texto

Se as chaves são códigos como `"7"`, `"70"`, `"700"`, tudo bem — são canônicos e ordenam numericamente.

Mas se são `"A7"`, `"A70"`, `"A700"`, são texto, e a ordenação é por caractere: `"A7"`, `"A70"`, `"A700"` sai correto por acaso. Já `"A7"`, `"A10"` sai errado — `"A10"` vem **antes** de `"A7"`, porque `1` vem antes de `7`.

A solução clássica é **preencher com zeros à largura fixa**:

```objectscript
set chave = "A"_$TRANSLATE($JUSTIFY(numero, 6), " ", "0")
// A000007, A000010, A000700
```

Agora a ordenação de texto coincide com a numérica. É o mesmo truque que usamos no `NewRecordNumber` do Capítulo 5.

### 4.6 `$SORTBEGIN` e `$SORTEND` — ordenação de grande volume

Quando você vai inserir **muitos** nós numa global, numa ordem aleatória, o IRIS precisa reorganizar blocos de disco constantemente. Isso é caro.

As funções `$SORTBEGIN` e `$SORTEND` otimizam esse caso:

```objectscript
set sc = $SORTBEGIN(^TempOrdenado)

// ... muitas inserções em ^TempOrdenado, em qualquer ordem ...

set sc = $SORTEND(^TempOrdenado)
```

Entre as duas chamadas, o IRIS acumula as inserções e as grava de forma otimizada no `$SORTEND`.

Duas restrições importantes:

- **Não leia a global entre `$SORTBEGIN` e `$SORTEND`** — o conteúdo não está consolidado.
- Serve para **carga em massa**, não para uso interativo.

O comportamento exato e as condições de uso variam por versão: **verificar na documentação oficial** antes de usar em produção.

---

## 5. Exemplo comentado

Arquivo `src/LabStudy/Demo/Arrays.cls`:

```objectscript
/// Traversal and sorting patterns over local arrays.
Class LabStudy.Demo.Arrays Extends %RegisteredObject
{

/// Builds a sample array and returns it by reference.
ClassMethod Sample(Output data) As %Integer
{
    kill data

    // code, patient, value
    set data(1) = $LISTBUILD("HGB", "Maria",  13.5)
    set data(2) = $LISTBUILD("GLU", "Bruno",  92)
    set data(3) = $LISTBUILD("CHOL","Maria",  190)
    set data(4) = $LISTBUILD("GLU", "Carla",  92)
    set data(5) = $LISTBUILD("TRIG","Bruno",  150)
    set data(6) = $LISTBUILD("HGB", "Carla",  11.2)

    quit 6
}

/// Walks one level, forwards.
ClassMethod WalkForward(ByRef data) As %Status
{
    write "-- forwards --", !
    set k = ""
    for {
        set k = $ORDER(data(k), 1, row)
        quit:k=""
        write "  ", k, ": ", $LIST(row, 1), " ", $LIST(row, 2), " = ", $LIST(row, 3), !
    }
    quit $$$OK
}

/// Walks one level, backwards.
ClassMethod WalkBackward(ByRef data) As %Status
{
    write "-- backwards --", !
    set k = ""
    for {
        set k = $ORDER(data(k), -1, row)
        quit:k=""
        write "  ", k, ": ", $LIST(row, 1), !
    }
    quit $$$OK
}

/// Sorts by value, ascending, using the inverted index idiom.
ClassMethod SortByValue(ByRef data, descending As %Boolean = 0) As %Status
{
    kill byValue

    // build the inverted index: value first, id second (to allow ties)
    set k = ""
    for {
        set k = $ORDER(data(k), 1, row)
        quit:k=""
        set byValue($LIST(row, 3), k) = ""
    }

    write "-- by value, ", $SELECT(descending: "descending", 1: "ascending"), " --", !

    set direction = $SELECT(descending: -1, 1: 1)

    set v = ""
    for {
        set v = $ORDER(byValue(v), direction)
        quit:v=""

        set id = ""
        for {
            set id = $ORDER(byValue(v, id))
            quit:id=""
            set row = data(id)
            write "  ", $JUSTIFY(v, 8), "  ", $LIST(row, 1), " (", $LIST(row, 2), ")", !
        }
    }
    quit $$$OK
}

/// Groups by test code and totals, then prints ordered by code.
ClassMethod GroupByCode(ByRef data) As %Status
{
    kill group

    set k = ""
    for {
        set k = $ORDER(data(k), 1, row)
        quit:k=""

        set code = $LIST(row, 1)
        set group(code, "count") = $GET(group(code, "count"), 0) + 1
        set group(code, "sum")   = $GET(group(code, "sum"), 0) + $LIST(row, 3)
    }

    write "-- grouped by code --", !
    set code = ""
    for {
        set code = $ORDER(group(code))
        quit:code=""

        set n = group(code, "count")
        set s = group(code, "sum")
        write "  ", $JUSTIFY(code, 6), " : ", n, " exams, sum ", s,
              ", avg ", $FNUMBER(s / n, "", 2), !
    }
    quit $$$OK
}

/// Sorts by two keys: patient name, then value descending.
ClassMethod SortByPatientThenValue(ByRef data) As %Status
{
    kill idx

    set k = ""
    for {
        set k = $ORDER(data(k), 1, row)
        quit:k=""
        // key 1: normalised name; key 2: value; key 3: id (ties)
        set idx($ZCONVERT($LIST(row, 2), "U"), $LIST(row, 3), k) = ""
    }

    write "-- by patient, then value descending --", !

    set name = ""
    for {
        set name = $ORDER(idx(name))
        quit:name=""

        write "  ", name, ":", !

        set v = ""
        for {
            set v = $ORDER(idx(name, v), -1)      // descending inside each patient
            quit:v=""

            set id = ""
            for {
                set id = $ORDER(idx(name, v, id))
                quit:id=""
                write "      ", $JUSTIFY(v, 8), "  ", $LIST(data(id), 1), !
            }
        }
    }
    quit $$$OK
}

/// Dumps any array, whatever its shape, using $QUERY.
ClassMethod Dump(ByRef data, rootName As %String = "data") As %Integer
{
    write "-- full dump of ", rootName, " --", !

    set ref = rootName
    set count = 0

    for {
        set ref = $QUERY(@ref)
        quit:ref=""

        set count = count + 1
        write "  ", ref, " (", $QLENGTH(ref), " subscripts)", !
    }

    write "  ", count, " nodes with a value", !
    quit count
}

/// Runs everything.
ClassMethod Demo() As %Status
{
    do ..Sample(.data)

    do ..WalkForward(.data)
    write !
    do ..WalkBackward(.data)
    write !
    do ..SortByValue(.data)
    write !
    do ..SortByValue(.data, 1)
    write !
    do ..GroupByCode(.data)
    write !
    do ..SortByPatientThenValue(.data)

    quit $$$OK
}

}
```

Comentando as decisões:

- **`Sample` recebe `Output data` e começa com `kill`.** Quem chama passa `.data`, e a convenção de limpar garante que restos de chamadas anteriores não contaminem.
- **Cada linha é um `$LISTBUILD`** com três campos. Guardar uma linha inteira num nó, em vez de espalhar por subscritos, é comum e eficiente. Listas são o Capítulo 14; aqui bastam `$LIST(lista, n)` para ler o n-ésimo campo.
- **Todos os laços usam o terceiro argumento do `$ORDER`.** `set k = $ORDER(data(k), 1, row)` traz subscrito e valor numa operação, economizando uma leitura por volta.
- **`SortByValue` constrói o índice invertido com DOIS subscritos**: valor e ID. Faça o teste de tirar o ID e veja um dos exames com valor 92 desaparecer.
- **A direção vira uma variável** (`set direction = $SELECT(...)`), e o mesmo código serve para os dois sentidos. Escrever dois laços quase idênticos seria duplicação desnecessária.
- **`GroupByCode` usa subscritos nomeados** (`"count"`, `"sum"`) em vez de posições. Autoexplicativo ao inspecionar com `ZWRITE`.
- **`GroupByCode` não precisa ordenar nada.** O agrupamento já sai por código em ordem, porque o código é o primeiro subscrito. Agrupar e ordenar são a mesma operação aqui.
- **`SortByPatientThenValue` mistura direções**: primeiro nível crescente, segundo decrescente. Cada nível é independente.
- **A chave do nome é normalizada com `$ZCONVERT`**, garantindo ordenação insensível a maiúsculas.
- **`Dump` funciona sobre qualquer array**, porque usa `$QUERY` e indireção. Ele recebe o nome da raiz como texto — necessário porque a indireção trabalha sobre nomes, não sobre valores.

### 5.1 Executando

```
LABSTUDY>DO ##class(LabStudy.Demo.Arrays).Demo()
-- forwards --
  1: HGB Maria = 13.5
  2: GLU Bruno = 92
  3: CHOL Maria = 190
  4: GLU Carla = 92
  5: TRIG Bruno = 150
  6: HGB Carla = 11.2

-- backwards --
  6: HGB
  5: TRIG
  4: GLU
  3: CHOL
  2: GLU
  1: HGB

-- by value, ascending --
      11.2  HGB (Carla)
      13.5  HGB (Maria)
        92  GLU (Bruno)
        92  GLU (Carla)
       150  TRIG (Bruno)
       190  CHOL (Maria)

-- by value, descending --
       190  CHOL (Maria)
       150  TRIG (Bruno)
        92  GLU (Carla)
        92  GLU (Bruno)
      13.5  HGB (Maria)
      11.2  HGB (Carla)

-- grouped by code --
    CHOL : 1 exams, sum 190, avg 190.00
     GLU : 2 exams, sum 184, avg 92.00
     HGB : 2 exams, sum 24.7, avg 12.35
    TRIG : 1 exams, sum 150, avg 150.00

-- by patient, then value descending --
  BRUNO:
       150  TRIG
        92  GLU
  CARLA:
        92  GLU
      11.2  HGB
  MARIA:
       190  CHOL
      13.5  HGB
```

O que observar:

- **Os dois exames com valor 92 apareceram os dois**, na ordem crescente e na decrescente. O segundo subscrito fez o trabalho. Note que, na descendente, a ordem entre eles se inverteu (Carla antes de Bruno) — porque o percurso do primeiro nível é invertido, mas o do segundo continua crescente. Se você quisesse manter Bruno antes de Carla, inverteria também o segundo laço.
- **`11.2` veio antes de `13.5`, que veio antes de `92`.** Ordenação **numérica**, porque os valores são números canônicos. Se fossem texto (`"11.2"` com aspas seria canônico mesmo assim, mas `"011.2"` não), a ordem seria outra.
- **O agrupamento saiu em ordem alfabética de código** sem nenhum esforço adicional. `CHOL`, `GLU`, `HGB`, `TRIG` — ordem de texto.
- **Os nomes saíram em maiúsculas** porque a chave normalizada é que foi impressa. Num relatório real, você guardaria o nome original no valor do nó e o imprimiria.

---

## 6. Pegadinhas e erros comuns

**1) Esquecer o `QUIT:k=""` no laço de `$ORDER`.**
Laço infinito: `$ORDER("")` volta ao primeiro subscrito.

**2) Não incluir o ID (ou um contador) no índice invertido.**
Empates se sobrescrevem e dados somem silenciosamente. **É o erro mais comum do capítulo.**

**3) Ler nó inexistente diretamente.**
`<UNDEFINED>`. Use `$GET`.

**4) Achar que `a(10)` e `a("10")` são nós diferentes.**
São o mesmo nó: `"10"` é a forma canônica do número 10.

**5) Achar que `a(7)` e `a("007")` são o mesmo nó.**
São diferentes: `"007"` não é canônico, logo é texto.

**6) Esperar que texto ordene sem diferenciar maiúsculas.**
`"B"` vem antes de `"a"`. Normalize a chave com `$ZCONVERT(v, "U")`.

**7) Esperar que `"A10"` venha depois de `"A7"`.**
Como texto, `"A10"` vem antes. Preencha com zeros à largura fixa.

**8) Usar `for i = 1:1:n` para percorrer um array esparso.**
Visita posições inexistentes e perde as que estão fora da faixa. Use `$ORDER`.

**9) Passar array para um método sem o ponto.**
O array inteiro se perde. Sempre `.array` na chamada e `ByRef`/`Output` na assinatura.

**10) Esquecer o `kill` no início de um método que preenche array de saída.**
Restos de chamadas anteriores se misturam ao resultado.

**11) Confundir `$ORDER` com `$QUERY`.**
`$ORDER` anda num nível e devolve o **subscrito**. `$QUERY` percorre a árvore e devolve a **referência completa**.

**12) Esperar que `$QUERY` visite nós sem valor.**
Ele visita apenas nós que têm valor. Nós puramente estruturais não aparecem.

**13) Achar que `KILL a("x")` apaga só aquele nó.**
Apaga o nó **e toda a subárvore**. Para só o valor, `ZKILL`.

**14) Achar que `MERGE` limpa o destino.**
Não limpa: mescla. `KILL` antes se quiser substituir.

**15) Usar referência nua em código novo.**
O alvo depende da última referência usada. Ilegível e frágil.

**16) Ler a global entre `$SORTBEGIN` e `$SORTEND`.**
O conteúdo não está consolidado nesse intervalo.

**17) Ordenar em ObjectScript o que o SQL já ordenaria.**
Se os dados vêm de uma tabela, `ORDER BY` é mais simples e aproveita índices.

---

## 7. MÃO NA MASSA

---

### Exercício 13.1 — Percursos

**a) Enunciado:** Crie `LabStudy.Demo.Walk` com:

1. `ClassMethod Build(Output a)` — monta um array de dois níveis: cidades no primeiro, pacientes no segundo, com a idade como valor.
2. `ClassMethod Level1(ByRef a)` — imprime só as cidades.
3. `ClassMethod Level2(ByRef a)` — imprime cidades e pacientes, aninhado.
4. `ClassMethod Backwards(ByRef a)` — imprime as cidades ao contrário.
5. `ClassMethod FirstLast(ByRef a)` — imprime a primeira e a última cidade sem percorrer tudo.
6. `ClassMethod FromLetter(ByRef a, letra)` — imprime as cidades a partir de uma letra, inclusive quando essa letra não existe como chave.
7. `ClassMethod Tree(ByRef a)` — despeja tudo com `$QUERY`, mostrando também a quantidade de subscritos de cada nó.

**b) Dica:** No item 5, `$ORDER(a(""))` e `$ORDER(a(""), -1)`.

**c) Como testar:** No item 6, passar `"M"` deve funcionar mesmo não havendo cidade chamada exatamente `"M"`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Walk.cls`:

```objectscript
/// Traversal patterns over a two level local array.
Class LabStudy.Demo.Walk Extends %RegisteredObject
{

/// city -> patient -> age
ClassMethod Build(Output a) As %Status
{
    kill a

    set a("Rio Preto", "Bruno") = 47
    set a("Rio Preto", "Ana") = 32
    set a("Potirendaba", "Maria") = 36
    set a("Potirendaba", "Carla") = 28
    set a("Potirendaba", "Diego") = 51
    set a("Mirassol", "Elias") = 19
    set a("Bady Bassitt", "Fabio") = 63

    quit $$$OK
}

/// Prints only the first level.
ClassMethod Level1(ByRef a) As %Integer
{
    write "-- cities --", !
    set n = 0
    set city = ""
    for {
        set city = $ORDER(a(city))
        quit:city=""
        set n = n + 1
        write "  ", city, !
    }
    quit n
}

/// Prints both levels, nested.
ClassMethod Level2(ByRef a) As %Integer
{
    write "-- cities and patients --", !
    set total = 0
    set city = ""
    for {
        set city = $ORDER(a(city))
        quit:city=""

        write "  ", city, ":", !

        set name = ""
        for {
            set name = $ORDER(a(city, name), 1, age)
            quit:name=""
            set total = total + 1
            write "      ", $JUSTIFY(name, 10), " ", age, !
        }
    }
    quit total
}

/// Prints the first level in reverse order.
ClassMethod Backwards(ByRef a) As %Status
{
    write "-- cities, reversed --", !
    set city = ""
    for {
        set city = $ORDER(a(city), -1)
        quit:city=""
        write "  ", city, !
    }
    quit $$$OK
}

/// First and last key, without walking the whole thing.
ClassMethod FirstLast(ByRef a) As %Status
{
    write "first: ", $ORDER(a("")), !
    write "last : ", $ORDER(a(""), -1), !
    quit $$$OK
}

/// Cities from a given starting point, even if it does not exist as a key.
ClassMethod FromLetter(ByRef a, letter As %String = "M") As %Integer
{
    write "-- cities from '", letter, "' onwards --", !

    set n = 0
    set city = letter
    for {
        set city = $ORDER(a(city))
        quit:city=""
        set n = n + 1
        write "  ", city, !
    }
    quit n
}

/// Full dump, of any shape.
ClassMethod Tree(ByRef a) As %Integer
{
    write "-- full tree --", !

    set ref = "a"
    set n = 0
    for {
        set ref = $QUERY(@ref)
        quit:ref=""
        set n = n + 1
        write "  ", $JUSTIFY($QLENGTH(ref), 2), "  ", ref, " = ", @ref, !
    }
    quit n
}

ClassMethod Demo() As %Status
{
    do ..Build(.a)
    do ..Level1(.a)
    write !
    do ..Level2(.a)
    write !
    do ..Backwards(.a)
    write !
    do ..FirstLast(.a)
    write !
    do ..FromLetter(.a, "M")
    write !
    do ..Tree(.a)
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Walk).Demo()
-- cities --
  Bady Bassitt
  Mirassol
  Potirendaba
  Rio Preto

-- cities and patients --
  Bady Bassitt:
           Fabio 63
  Mirassol:
           Elias 19
  Potirendaba:
           Carla 28
           Diego 51
           Maria 36
  Rio Preto:
             Ana 32
           Bruno 47

-- cities, reversed --
  Rio Preto
  Potirendaba
  Mirassol
  Bady Bassitt

first: Bady Bassitt
last : Rio Preto

-- cities from 'M' onwards --
  Mirassol
  Potirendaba
  Rio Preto

-- full tree --
   2  a("Bady Bassitt","Fabio") = 63
   2  a("Mirassol","Elias") = 19
   2  a("Potirendaba","Carla") = 28
   2  a("Potirendaba","Diego") = 51
   2  a("Potirendaba","Maria") = 36
   2  a("Rio Preto","Ana") = 32
   2  a("Rio Preto","Bruno") = 47
```

**Por que cada resultado:**

- **As cidades saíram em ordem alfabética** sem que ninguém as ordenasse. Foram inseridas em ordem aleatória e saíram ordenadas. Este é o ponto central do capítulo, visível na primeira saída.
- **Dentro de cada cidade, os pacientes também saíram ordenados.** Cada nível ordena independentemente.
- **`FromLetter("M")` funcionou embora não exista cidade `"M"`.** `$ORDER` responde "qual vem depois de M?" sem exigir que M exista. É isso que permite busca por faixa — e é o mecanismo por trás de `WHERE Nome >= 'M'` num índice.
- **`FirstLast` obteve primeiro e último em duas operações**, sem percorrer nada. Num array de um milhão de nós, isso continua sendo duas operações.
- **`Tree` mostrou apenas nós de nível 2.** Os nós de nível 1 (`a("Mirassol")`, por exemplo) existem como estrutura, mas **não têm valor**, e `$QUERY` só visita nós com valor. Confirme com `WRITE $DATA(a("Mirassol"))` — devolve `10`.

---

### Exercício 13.2 — A ordem exata dos subscritos

**a) Enunciado:** Faça um experimento controlado. Crie um array com estes subscritos, nesta ordem de inserção:

`"banana"`, `10`, `-3`, `"Ana"`, `"10"`, `2.5`, `""`, `"007"`, `0`, `"2.50"`, `"+5"`, `100`, `"apple"`

Depois:
1. Percorra e imprima na ordem em que saem.
2. Conte quantos nós existem. É o mesmo número de inserções? Por quê?
3. Para cada subscrito, use `$DATA` para conferir se ele existe individualmente.
4. Explique, um a um, por que cada valor foi parar onde foi.

**b) Dica:** O subscrito vazio complica o laço padrão — pense em como percorrer incluindo-o.

**c) Como testar:** O número de nós deve ser **menor** que o número de inserções.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>KILL a

LABSTUDY>SET a("banana")="", a(10)="", a(-3)="", a("Ana")="", a("10")=""
LABSTUDY>SET a(2.5)="", a("")="", a("007")="", a(0)="", a("2.50")=""
LABSTUDY>SET a("+5")="", a(100)="", a("apple")=""

LABSTUDY>SET n=0, k="" FOR { SET k=$ORDER(a(k)) QUIT:k=""  SET n=n+1 }
LABSTUDY>WRITE "nós com subscrito não vazio: ", n, !
nós com subscrito não vazio: 11

LABSTUDY>WRITE "existe o subscrito vazio? ", $DATA(a("")), !
existe o subscrito vazio? 1

LABSTUDY>ZWRITE a
a("")=""
a(-3)=""
a(0)=""
a(2.5)=""
a(10)=""
a(100)=""
a("+5")=""
a("007")=""
a("2.50")=""
a("Ana")=""
a("apple")=""
a("banana")=""
```

**Por que cada valor foi parar onde foi:**

| Inserido | Onde foi | Por quê |
|---|---|---|
| `""` | primeiro de todos | o subscrito vazio precede tudo |
| `-3` | com os números | canônico e negativo |
| `0` | com os números | `0` é canônico |
| `2.5` | com os números | canônico |
| `10` e `"10"` | **mesmo nó** | `"10"` é a forma canônica do número 10 |
| `100` | com os números | canônico |
| `"+5"` | com o texto | o sinal de mais quebra a canonicidade |
| `"007"` | com o texto | zeros à esquerda quebram a canonicidade |
| `"2.50"` | com o texto | zero supérfluo à direita quebra a canonicidade |
| `"Ana"` | texto, antes de `"apple"` | `A` (65) vem antes de `a` (97) |
| `"apple"` | texto | ordem de caractere |
| `"banana"` | último | `b` depois de `a` |

**Por que cada decisão do exercício:**

- **Foram 13 inserções e restaram 12 nós.** `a(10)` e `a("10")` são o mesmo. A segunda atribuição sobrescreveu a primeira. Se você tivesse guardado valores diferentes nelas, um teria se perdido silenciosamente — e é exatamente esse tipo de perda que causa bugs difíceis num sistema real que mistura números e texto como chaves.
- **O laço padrão contou 11**, não 12, porque `""` é a sentinela do laço: começar com `""` significa "antes do primeiro", e o laço nunca visita o próprio subscrito vazio. Para incluí-lo, você precisa tratá-lo separadamente com `$DATA(a(""))` antes de iniciar o laço.
- **Os três "quase números"** — `"+5"`, `"007"`, `"2.50"` — foram para o meio do texto, cada um numa posição determinada pelo seu primeiro caractere: `+` (43), `0` (48), `2` (50). Todos antes das letras.
- **`ZWRITE` mostrou o formato com aspas para texto e sem aspas para números**, o que é uma forma rápida de conferir a canonicidade de um subscrito sem raciocinar sobre as regras.

---

### Exercício 13.3 — Ordenando com índice invertido

**a) Enunciado:** Crie `LabStudy.Demo.Sort1` com um conjunto de dados que **contenha empates** e:

1. `ClassMethod BuildWrong(ByRef data, Output idx)` — constrói o índice invertido **sem** o segundo subscrito, de propósito.
2. `ClassMethod BuildRight(ByRef data, Output idx)` — constrói corretamente, com o ID como segundo subscrito.
3. `ClassMethod Compare(ByRef data)` — mostra quantos registros cada abordagem recupera, provando a perda.
4. `ClassMethod Top(ByRef data, n)` — devolve os `n` maiores valores, **sem** percorrer tudo.
5. `ClassMethod Median(ByRef data)` — devolve a mediana, usando o índice ordenado.

**b) Dica:** No item 4, percorra de trás para frente e pare depois de `n` itens. No item 5, a mediana é o elemento do meio da sequência ordenada.

**c) Como testar:** No item 3, a versão errada deve recuperar menos registros que o total.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Sort1.cls`:

```objectscript
/// Inverted index sorting, with and without tie handling.
Class LabStudy.Demo.Sort1 Extends %RegisteredObject
{

/// Sample data with deliberate ties.
ClassMethod Sample(Output data) As %Integer
{
    kill data

    set data(1) = 92
    set data(2) = 150
    set data(3) = 92
    set data(4) = 11
    set data(5) = 190
    set data(6) = 92
    set data(7) = 150

    quit 7
}

/// Wrong on purpose: value as the only subscript.
ClassMethod BuildWrong(ByRef data, Output idx) As %Status
{
    kill idx

    set id = ""
    for {
        set id = $ORDER(data(id), 1, v)
        quit:id=""
        set idx(v) = id           // ties overwrite each other
    }
    quit $$$OK
}

/// Right: value first, id second.
ClassMethod BuildRight(ByRef data, Output idx) As %Status
{
    kill idx

    set id = ""
    for {
        set id = $ORDER(data(id), 1, v)
        quit:id=""
        set idx(v, id) = ""
    }
    quit $$$OK
}

/// Counts what each approach can recover.
ClassMethod Compare(ByRef data) As %Status
{
    do ..BuildWrong(.data, .wrong)
    do ..BuildRight(.data, .right)

    // count original
    set total = 0, id = ""
    for { set id = $ORDER(data(id)) quit:id=""  set total = total + 1 }

    // count recoverable from the wrong index
    set nWrong = 0, v = ""
    for { set v = $ORDER(wrong(v)) quit:v=""  set nWrong = nWrong + 1 }

    // count recoverable from the right index
    set nRight = 0, v = ""
    for {
        set v = $ORDER(right(v))
        quit:v=""
        set id = ""
        for { set id = $ORDER(right(v, id)) quit:id=""  set nRight = nRight + 1 }
    }

    write "original records      : ", total, !
    write "recoverable (wrong)   : ", nWrong, "   <-- lost ", total - nWrong, !
    write "recoverable (right)   : ", nRight, !

    write !, "-- sorted, right version --", !
    set v = ""
    for {
        set v = $ORDER(right(v))
        quit:v=""
        set id = ""
        for {
            set id = $ORDER(right(v, id))
            quit:id=""
            write "  ", $JUSTIFY(v, 5), "  (id ", id, ")", !
        }
    }

    quit $$$OK
}

/// The n largest values, stopping as soon as n are found.
ClassMethod Top(ByRef data, n As %Integer = 3) As %Status
{
    do ..BuildRight(.data, .idx)

    write "-- top ", n, " --", !

    set found = 0
    set v = ""
    for {
        set v = $ORDER(idx(v), -1)
        quit:v=""

        set id = ""
        for {
            set id = $ORDER(idx(v, id))
            quit:id=""

            set found = found + 1
            write "  ", found, ". ", v, " (id ", id, ")", !
            quit:found>=n
        }
        quit:found>=n
    }

    quit $$$OK
}

/// Median, using the ordered index.
ClassMethod Median(ByRef data) As %Numeric
{
    do ..BuildRight(.data, .idx)

    // count first
    set total = 0, v = ""
    for {
        set v = $ORDER(idx(v))
        quit:v=""
        set id = ""
        for { set id = $ORDER(idx(v, id)) quit:id=""  set total = total + 1 }
    }

    if total = 0 { quit "" }

    set target = (total + 1) \ 2        // integer division
    set seen = 0, v = ""

    for {
        set v = $ORDER(idx(v))
        quit:v=""

        set id = ""
        for {
            set id = $ORDER(idx(v, id))
            quit:id=""
            set seen = seen + 1
            if seen = target {
                quit
            }
        }
        quit:seen=target
    }

    quit v
}

ClassMethod Demo() As %Status
{
    do ..Sample(.data)
    do ..Compare(.data)
    write !
    do ..Top(.data, 3)
    write !
    write "median: ", ..Median(.data), !
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Sort1).Demo()
original records      : 7
recoverable (wrong)   : 4   <-- lost 3
recoverable (right)   : 7

-- sorted, right version --
     11  (id 4)
     92  (id 1)
     92  (id 3)
     92  (id 6)
    150  (id 2)
    150  (id 7)
    190  (id 5)

-- top 3 --
  1. 190 (id 5)
  2. 150 (id 7)
  3. 150 (id 2)

median: 92
```

**Por que cada resultado:**

- **A versão errada perdeu 3 dos 7 registros.** Três exames com valor 92 viraram um; dois com 150 viraram um. E — este é o ponto — **nenhum erro foi levantado**. O índice ficou pequeno e silenciosamente incompleto. Num relatório de produção, você entregaria 4 linhas onde havia 7, e o usuário confiaria.
- **A versão certa recuperou os 7**, com os empates lado a lado e ordenados por ID dentro de cada valor.
- **`Top` parou assim que encontrou 3.** Ele não percorreu os 7 — percorreu 3. Num conjunto de um milhão, encontrar os 10 maiores custaria 10 passos, não um milhão. Este é o mesmo princípio do `TOP` do SQL, e mostra por que estruturas ordenadas são tão poderosas: **você paga só pelo que consome**.
- **Os dois `quit:found>=n`** são necessários: um sai do laço interno, outro do externo. Um `RETURN` do Capítulo 3 sairia dos dois de uma vez, mas o método precisa continuar para devolver o status — daí a escolha pelos dois `quit`.
- **`(total + 1) \ 2`** — a barra invertida é a divisão **inteira** do ObjectScript. Com 7 elementos, o alvo é o 4º, que é a mediana.
- **A mediana saiu correta (92)** percorrendo a estrutura já ordenada. Sem o índice invertido, calcular mediana exigiria ordenar primeiro — que é justamente o que a estrutura fez de graça.

---

### Exercício 13.4 — Multichave, normalização e referências

**a) Enunciado:** Crie `LabStudy.Demo.Sort2` com dados de pacientes (nome, cidade, idade) e:

1. `ClassMethod ByCityThenAge(ByRef data)` — ordena por cidade (crescente) e, dentro dela, por idade (**decrescente**).
2. `ClassMethod ByNameCaseInsensitive(ByRef data)` — ordena por nome ignorando maiúsculas, **exibindo o nome original**.
3. `ClassMethod ByPaddedCode(ByRef data)` — recebe códigos como `"A7"`, `"A10"`, `"A700"` e os ordena corretamente, usando preenchimento com zeros.
4. `ClassMethod Describe(ref)` — recebe uma referência como texto (vinda do `$QUERY`) e imprime nome da variável, quantidade de subscritos e cada subscrito separadamente.
5. `ClassMethod Copy(ByRef origem, Output destino)` — copia usando `MERGE` e prova que o destino foi limpo antes.

**b) Dica:** No item 2, guarde o nome original no **valor** do nó. No item 4, use `$QSUBSCRIPT` e `$QLENGTH`.

**c) Como testar:** No item 3, sem o preenchimento, `"A10"` viria antes de `"A7"`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Sort2.cls`:

```objectscript
/// Multi key sorting, key normalisation and reference handling.
Class LabStudy.Demo.Sort2 Extends %RegisteredObject
{

/// id -> $LB(name, city, age)
ClassMethod Sample(Output data) As %Status
{
    kill data

    set data(1) = $LISTBUILD("maria silva",  "Potirendaba", 36)
    set data(2) = $LISTBUILD("Bruno Lima",   "Rio Preto",   47)
    set data(3) = $LISTBUILD("ana souza",    "Potirendaba", 28)
    set data(4) = $LISTBUILD("Carla Nunes",  "Rio Preto",   51)
    set data(5) = $LISTBUILD("Diego Alves",  "Potirendaba", 63)
    set data(6) = $LISTBUILD("elias prado",  "Mirassol",    19)

    quit $$$OK
}

/// City ascending, age descending inside each city.
ClassMethod ByCityThenAge(ByRef data) As %Status
{
    kill idx

    set id = ""
    for {
        set id = $ORDER(data(id), 1, row)
        quit:id=""
        set idx($LIST(row, 2), $LIST(row, 3), id) = ""
    }

    write "-- city ascending, age descending --", !

    set city = ""
    for {
        set city = $ORDER(idx(city))
        quit:city=""

        write "  ", city, ":", !

        set age = ""
        for {
            set age = $ORDER(idx(city, age), -1)
            quit:age=""

            set id = ""
            for {
                set id = $ORDER(idx(city, age, id))
                quit:id=""
                write "      ", $JUSTIFY(age, 3), "  ", $LIST(data(id), 1), !
            }
        }
    }
    quit $$$OK
}

/// Name order ignoring case, printing the original spelling.
ClassMethod ByNameCaseInsensitive(ByRef data) As %Status
{
    kill idx

    set id = ""
    for {
        set id = $ORDER(data(id), 1, row)
        quit:id=""
        // normalised key, original kept in the node value
        set idx($ZCONVERT($LIST(row, 1), "U"), id) = $LIST(row, 1)
    }

    write "-- by name, case insensitive --", !

    set key = ""
    for {
        set key = $ORDER(idx(key))
        quit:key=""

        set id = ""
        for {
            set id = $ORDER(idx(key, id), 1, original)
            quit:id=""
            write "  ", original, !
        }
    }
    quit $$$OK
}

/// Codes like A7, A10, A700 sorted properly.
ClassMethod ByPaddedCode() As %Status
{
    set codes = $LISTBUILD("A7", "A10", "A700", "A2", "A70", "A1000")

    write "-- naive text order --", !
    kill naive
    for i = 1:1:$LISTLENGTH(codes) {
        set naive($LIST(codes, i)) = ""
    }
    set k = ""
    for { set k = $ORDER(naive(k)) quit:k=""  write "  ", k, ! }

    write !, "-- padded order --", !
    kill padded
    for i = 1:1:$LISTLENGTH(codes) {
        set code = $LIST(codes, i)
        set letter = $EXTRACT(code, 1)
        set number = $EXTRACT(code, 2, *)
        set key = letter_$TRANSLATE($JUSTIFY(number, 6), " ", "0")
        set padded(key) = code
    }
    set k = ""
    for {
        set k = $ORDER(padded(k), 1, original)
        quit:k=""
        write "  ", $JUSTIFY(original, 6), "   (key ", k, ")", !
    }

    quit $$$OK
}

/// Breaks a reference string into its parts.
ClassMethod Describe(ref As %String) As %Status
{
    write "reference : ", ref, !
    write "root name : ", $QSUBSCRIPT(ref, 0), !
    write "subscripts: ", $QLENGTH(ref), !

    for i = 1:1:$QLENGTH(ref) {
        write "   ", i, ": [", $QSUBSCRIPT(ref, i), "]", !
    }
    quit $$$OK
}

/// Copies an array, clearing the destination first.
ClassMethod Copy(ByRef source, Output target) As %Integer
{
    kill target
    merge target = source

    set n = 0
    set ref = "target"
    for {
        set ref = $QUERY(@ref)
        quit:ref=""
        set n = n + 1
    }
    quit n
}

ClassMethod Demo() As %Status
{
    do ..Sample(.data)

    do ..ByCityThenAge(.data)
    write !
    do ..ByNameCaseInsensitive(.data)
    write !
    do ..ByPaddedCode()

    write !
    set r = $NAME(data(3))
    do ..Describe(r)

    write !
    set r2 = $NAME(^LabStudy.PatientI("NameIdx", " MARIA SILVA", 1))
    do ..Describe(r2)

    write !
    set copied = ..Copy(.data, .backup)
    write "copied ", copied, " nodes", !

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Sort2).Demo()
-- city ascending, age descending --
  Mirassol:
       19  elias prado
  Potirendaba:
       63  Diego Alves
       36  maria silva
       28  ana souza
  Rio Preto:
       51  Carla Nunes
       47  Bruno Lima

-- by name, case insensitive --
  ana souza
  Bruno Lima
  Carla Nunes
  Diego Alves
  elias prado
  maria silva

-- naive text order --
  A10
  A1000
  A2
  A7
  A70
  A700

-- padded order --
      A2   (key A000002)
      A7   (key A000007)
     A10   (key A000010)
     A70   (key A000070)
    A700   (key A000700)
   A1000   (key A001000)

reference : data(3)
root name : data
subscripts: 1
   1: [3]

reference : ^LabStudy.PatientI("NameIdx"," MARIA SILVA",1)
root name : ^LabStudy.PatientI
subscripts: 3
   1: [NameIdx]
   2: [ MARIA SILVA]
   3: [1]

copied 6 nodes
```

**Por que cada resultado:**

- **Cidades crescentes, idades decrescentes dentro de cada uma.** Cada nível tem sua direção, e isso sai de uma única estrutura. Nenhuma reordenação foi feita.
- **A ordenação de nomes ignorou as maiúsculas**, e ainda assim os nomes originais foram impressos com a grafia certa. O truque é a divisão de trabalho: a **chave** normalizada ordena, o **valor** preserva o original. Sem normalizar, `"Bruno Lima"` e `"Carla Nunes"` viriam antes de todos os minúsculos.
- **A comparação dos códigos é a demonstração mais clara do capítulo.** Na ordem ingênua, `A1000` aparece em segundo lugar, entre `A10` e `A2` — porque a comparação é caractere a caractere e `0` vem antes de `2`. Com o preenchimento, a ordem de texto passa a coincidir com a numérica. Este é exatamente o problema que o `NewRecordNumber` do Capítulo 5 já resolvia, agora explicado.
- **`$EXTRACT(code, 2, *)`** pega do segundo caractere até o **fim** — o asterisco significa "último". Funções de texto são o Capítulo 15.
- **`Describe` funcionou tanto num array local quanto numa global de índice real.** `$QSUBSCRIPT(ref, 0)` devolveu `^LabStudy.PatientI` com o circunflexo, mostrando que a função distingue global de local. Isso é o que permite escrever utilitários genéricos.
- **`Copy` limpou o destino antes do `MERGE`.** Se não limpasse e o destino já tivesse conteúdo, o resultado seria uma mistura dos dois — que é o comportamento documentado do `MERGE` e a origem de bugs sutis.

---

### Exercício 13.5 — PROJETO CONTÍNUO: relatórios ordenados em memória

**a) Enunciado:** Nem sempre você pode pedir ao SQL. Às vezes os dados já estão na memória, vêm de várias fontes, ou o critério de ordenação é calculado. Crie a camada de ordenação do laboratório:

1. Crie `LabStudy.Sorter`, um utilitário genérico:
   - `ClassMethod Add(ByRef idx, key, id, value)` — acrescenta uma entrada ao índice, tratando empates automaticamente;
   - `ClassMethod AddNormalized(ByRef idx, key, id, value)` — o mesmo, normalizando a chave de texto;
   - `ClassMethod Walk(ByRef idx, descending, Output ordered)` — percorre e devolve um array sequencial `1, 2, 3...` na ordem correta;
   - `ClassMethod TopN(ByRef idx, n, Output ordered)` — os `n` primeiros ou últimos, parando cedo;
   - `ClassMethod Count(ByRef idx)` — conta as entradas.
2. Crie `LabStudy.RankReport` com:
   - `ClassMethod ByResult(testCode, n)` — os `n` maiores resultados de um código de exame, com nome do paciente;
   - `ClassMethod ByExamCount(n)` — os `n` pacientes com mais exames;
   - `ClassMethod ByCity()` — pacientes agrupados por cidade, ordenados por nome dentro de cada uma;
   - `ClassMethod AbnormalRanking()` — todos os exames finais, ordenados do maior desvio ao menor, sendo o desvio a distância do valor em relação à média daquele código.
3. Suba `LabStudy.App` para `"1.4"` e acrescente `ClassMethod Rankings()` ao `Run()`.

**b) Dica:** No item 2, use SQL para **buscar** e o `Sorter` para **ordenar por critério calculado** — que é o caso em que o SQL não ajuda tanto, porque a média precisa ser conhecida antes.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.RankReport).ByResult("GLU", 5)
LABSTUDY>DO ##class(LabStudy.RankReport).ByExamCount(5)
LABSTUDY>DO ##class(LabStudy.RankReport).AbnormalRanking()
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Sorter.cls`:

```objectscript
/// Generic in memory sorting helper, built on the inverted index idiom.
/// The index shape is always idx(key, id) = value.
Class LabStudy.Sorter Extends %RegisteredObject
{

/// Adds one entry. The id as second subscript is what allows ties.
ClassMethod Add(ByRef idx, key As %String, id As %String, value As %String = "") As %Status
{
    set idx(key, id) = value
    quit $$$OK
}

/// Same, but normalises a text key so ordering ignores case and outer spaces.
ClassMethod AddNormalized(ByRef idx, key As %String, id As %String, value As %String = "") As %Status
{
    set idx($ZCONVERT($ZSTRIP(key, "<>W"), "U"), id) = value
    quit $$$OK
}

/// Walks the index and returns a dense, ordered array:
///   ordered(n) = $LB(key, id, value)
/// Returns how many entries were produced.
ClassMethod Walk(ByRef idx, descending As %Boolean = 0, Output ordered) As %Integer
{
    kill ordered

    set direction = $SELECT(descending: -1, 1: 1)
    set n = 0
    set key = ""

    for {
        set key = $ORDER(idx(key), direction)
        quit:key=""

        set id = ""
        for {
            set id = $ORDER(idx(key, id), 1, value)
            quit:id=""

            set n = n + 1
            set ordered(n) = $LISTBUILD(key, id, value)
        }
    }

    set ordered = n
    quit n
}

/// Same as Walk, but stops after n entries.
ClassMethod TopN(ByRef idx, n As %Integer = 10, descending As %Boolean = 1, Output ordered) As %Integer
{
    kill ordered

    set direction = $SELECT(descending: -1, 1: 1)
    set found = 0
    set key = ""

    for {
        set key = $ORDER(idx(key), direction)
        quit:key=""

        set id = ""
        for {
            set id = $ORDER(idx(key, id), 1, value)
            quit:id=""

            set found = found + 1
            set ordered(found) = $LISTBUILD(key, id, value)
            quit:found>=n
        }
        quit:found>=n
    }

    set ordered = found
    quit found
}

/// Counts entries in the index.
ClassMethod Count(ByRef idx) As %Integer
{
    set n = 0, key = ""
    for {
        set key = $ORDER(idx(key))
        quit:key=""
        set id = ""
        for { set id = $ORDER(idx(key, id)) quit:id=""  set n = n + 1 }
    }
    quit n
}

}
```

`src/LabStudy/RankReport.cls`:

```objectscript
/// Rankings that combine SQL retrieval with in memory ordering.
Class LabStudy.RankReport Extends %RegisteredObject
{

/// The n highest results of one test code.
ClassMethod ByResult(testCode As %String, n As %Integer = 10) As %Status
{
    kill idx

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT e.%ID AS Id, e.ResultValue AS Val, e.Unit AS Unit, "
        _"e.Patient->Name AS PatientName "
        _"FROM LabStudy.EXAM e "
        _"WHERE e.TestCode = ? AND e.ResultStatus = 'final'", testCode)

    while rs.%Next() {
        do ##class(LabStudy.Sorter).Add(.idx, rs.%Get("Val"), rs.%Get("Id"),
                                        rs.%Get("PatientName")_"|"_rs.%Get("Unit"))
    }

    write "=== top ", n, " results for ", testCode, " ===", !

    set found = ##class(LabStudy.Sorter).TopN(.idx, n, 1, .ordered)

    if found = 0 {
        write "  (no final results for this code)", !
        quit $$$OK
    }

    for i = 1:1:found {
        set row = ordered(i)
        write "  ", $JUSTIFY(i, 2), ". ", $JUSTIFY($LIST(row, 1), 10), " ",
              $PIECE($LIST(row, 3), "|", 2), "   ",
              $PIECE($LIST(row, 3), "|", 1), !
    }

    quit $$$OK
}

/// The n patients with the most exams.
ClassMethod ByExamCount(n As %Integer = 10) As %Status
{
    kill idx

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT p.%ID AS Id, p.Name AS Name, p.RecordNumber AS Rec, "
        _"(SELECT COUNT(*) FROM LabStudy.EXAM e WHERE e.Patient = p.%ID) AS Total "
        _"FROM LabStudy.PATIENT p")

    while rs.%Next() {
        do ##class(LabStudy.Sorter).Add(.idx, rs.%Get("Total"), rs.%Get("Id"),
                                        rs.%Get("Rec")_"|"_rs.%Get("Name"))
    }

    write "=== top ", n, " patients by exam count ===", !

    set found = ##class(LabStudy.Sorter).TopN(.idx, n, 1, .ordered)

    for i = 1:1:found {
        set row = ordered(i)
        write "  ", $JUSTIFY(i, 2), ". ", $JUSTIFY($LIST(row, 1), 4), " exams   ",
              $PIECE($LIST(row, 3), "|", 1), "  ",
              $PIECE($LIST(row, 3), "|", 2), !
    }

    quit $$$OK
}

/// Patients grouped by city, ordered by name inside each city.
ClassMethod ByCity() As %Status
{
    kill idx

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id, Name, RecordNumber AS Rec, Address_City AS City "
        _"FROM LabStudy.PATIENT")

    while rs.%Next() {
        set city = rs.%Get("City")
        if city = "" { set city = "(sem cidade)" }

        // three levels: city, normalised name, id
        set idx(city, $ZCONVERT(rs.%Get("Name"), "U"), rs.%Get("Id")) =
            rs.%Get("Rec")_"|"_rs.%Get("Name")
    }

    write "=== patients by city ===", !

    set city = ""
    for {
        set city = $ORDER(idx(city))
        quit:city=""

        write "  ", city, ":", !

        set key = ""
        for {
            set key = $ORDER(idx(city, key))
            quit:key=""

            set id = ""
            for {
                set id = $ORDER(idx(city, key, id), 1, info)
                quit:id=""
                write "      ", $PIECE(info, "|", 1), "  ", $PIECE(info, "|", 2), !
            }
        }
    }

    quit $$$OK
}

/// Ranks final exams by how far they are from the average of their own code.
/// The criterion is computed, so SQL alone would not order it directly.
ClassMethod AbnormalRanking(n As %Integer = 15) As %Status
{
    // Step 1: averages per code
    kill avg
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT TestCode, AVG(ResultValue) AS Average "
        _"FROM LabStudy.EXAM WHERE ResultStatus = 'final' GROUP BY TestCode")

    while rs.%Next() {
        set avg(rs.%Get("TestCode")) = rs.%Get("Average")
    }

    // Step 2: build the index keyed by the computed deviation
    kill idx
    set rs2 = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT e.%ID AS Id, e.TestCode AS Code, e.ResultValue AS Val, "
        _"e.Patient->Name AS PatientName "
        _"FROM LabStudy.EXAM e WHERE e.ResultStatus = 'final'")

    while rs2.%Next() {
        set code = rs2.%Get("Code")
        set value = rs2.%Get("Val")
        set average = $GET(avg(code), 0)

        set deviation = $SELECT(value > average: value - average, 1: average - value)
        set deviation = $FNUMBER(deviation, "", 4)

        do ##class(LabStudy.Sorter).Add(.idx, deviation, rs2.%Get("Id"),
            code_"|"_value_"|"_$FNUMBER(average, "", 2)_"|"_rs2.%Get("PatientName"))
    }

    write "=== most deviant results (top ", n, ") ===", !
    write "  dev      code    value    avg      patient", !
    write "  ------------------------------------------------", !

    set found = ##class(LabStudy.Sorter).TopN(.idx, n, 1, .ordered)

    for i = 1:1:found {
        set info = $LIST(ordered(i), 3)
        write "  ", $JUSTIFY($LIST(ordered(i), 1), 8), " ",
              $JUSTIFY($PIECE(info, "|", 1), 6), " ",
              $JUSTIFY($PIECE(info, "|", 2), 8), " ",
              $JUSTIFY($PIECE(info, "|", 3), 8), "   ",
              $PIECE(info, "|", 4), !
    }

    write "  ------------------------------------------------", !
    write "  ", ##class(LabStudy.Sorter).Count(.idx), " final exams considered", !

    quit $$$OK
}

}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.4";

/// Prints the main rankings of the laboratory.
ClassMethod Rankings() As %Status
{
    do ##class(LabStudy.RankReport).ByExamCount(5)
    write !
    do ##class(LabStudy.RankReport).AbnormalRanking(10)
    quit $$$OK
}

ClassMethod Run() As %Status
{
    do ..About()
    write !
    do ##class(LabStudy.Schema).Report()
    write !
    do ##class(LabStudy.Reports).Dashboard()
    write !
    do ..Rankings()
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.RankReport).ByExamCount(5)
=== top 5 patients by exam count ===
   1.   10 exams   REG-000012  Paciente 00012
   2.   10 exams   REG-000007  Paciente 00007
   3.   10 exams   REG-000003  Paciente 00003
   4.   10 exams   REG-000198  Paciente 00198
   5.   10 exams   REG-000045  Paciente 00045

LABSTUDY>DO ##class(LabStudy.RankReport).ByResult("GLU", 5)
=== top 5 results for GLU ===
   1.        397 mg/dL   Paciente 00174
   2.        392 mg/dL   Paciente 00098
   3.        389 mg/dL   Paciente 00121
   4.        385 mg/dL   Paciente 00056
   5.        381 mg/dL   Paciente 00033

LABSTUDY>DO ##class(LabStudy.RankReport).AbnormalRanking(5)
=== most deviant results (top 5) ===
  dev      code    value    avg      patient
  ------------------------------------------------
   194.32    GLU      397   202.68   Paciente 00174
   192.14   CHOL      395   202.86   Paciente 00061
   189.32    GLU      392   202.68   Paciente 00098
   187.44   TRIG      390   202.56   Paciente 00061
   186.32    GLU      389   202.68   Paciente 00121
  ------------------------------------------------
  2000 final exams considered
```

**Por que cada decisão:**

- **`LabStudy.Sorter` encapsula o idioma inteiro num lugar só.** O padrão `idx(chave, id)` com o valor no nó aparece em toda a apostila; concentrá-lo numa classe significa que a decisão sobre empates foi tomada **uma vez**, corretamente, e nenhum relatório futuro pode esquecê-la.
- **`Walk` e `TopN` devolvem um array denso `1, 2, 3...`.** Isso é deliberado: depois de ordenar, um array com posições sequenciais é muito mais fácil de consumir, permitindo `for i = 1:1:n` sem `$ORDER`. Converter uma estrutura esparsa em densa depois de ordenar é um padrão útil.
- **`set ordered = n` guarda a contagem na raiz**, o mesmo padrão do IRIS que você viu no Capítulo 8. Quem chama tem o total sem precisar contar.
- **`TopN` para cedo**, e isso importa: `ByResult` sobre uma tabela com 100 mil exames de um código constrói o índice inteiro (inevitável), mas **percorre apenas 5 entradas** para produzir o relatório.
- **`AbnormalRanking` é a justificativa de existência de todo o capítulo.** Ordenar por resultado bruto o SQL faz sozinho. Mas o critério aqui é o **desvio em relação à média do próprio código** — um valor que só existe depois de conhecidas as médias. Fazer isso em SQL exigiria subconsulta correlacionada ou função de janela; fazer com o índice invertido é direto e legível. **Quando o critério de ordenação é calculado a partir de dados de outra consulta, a ordenação em memória é a ferramenta certa.**
- **A chave do desvio passa por `$FNUMBER(deviation, "", 4)`.** Sem isso, valores como `194.3200000001` e `194.32` viram chaves diferentes por diferenças de arredondamento em ponto flutuante — criando duas gavetas onde deveria haver uma. Normalizar a chave numérica é a versão numérica do que `$ZCONVERT` faz com texto.
- **`ByCity` usa três níveis de subscrito** e não constrói índice invertido separado: a própria estrutura já é o índice. Agrupar por cidade e ordenar por nome dentro dela é uma única declaração de subscritos.
- **Cidades vazias viram `"(sem cidade)"`** antes de virarem chave. Sem isso, elas ocupariam o subscrito vazio e sairiam primeiro, sem rótulo — o problema de exibição do Capítulo 10, agora afetando também a ordenação.
- **Os campos compostos usam `|` como separador** e são desmontados com `$PIECE`. É um atalho legítimo para transportar poucos campos; para muitos, `$LISTBUILD` seria mais robusto, porque não quebra se um valor contiver o separador. Reconhecer essa limitação é parte da decisão.

---

## 8. Quiz do capítulo

**Q1.** Qual comando percorre **um nível** de um array, devolvendo o próximo subscrito?

- A) `$QUERY`
- B) `$ORDER`
- C) `$DATA`
- D) `$NAME`

---

**Q2.** Qual é a ordem correta de classificação dos subscritos no IRIS?

- A) Texto, depois números.
- B) Subscrito vazio, depois números canônicos em ordem numérica, depois texto em ordem de caractere.
- C) Tudo em ordem alfabética.
- D) Ordem de inserção.

---

**Q3.** `a(10)` e `a("10")` são o mesmo nó?

- A) Não, são sempre diferentes.
- B) Sim, porque `"10"` é a forma canônica do número 10.
- C) Depende do tipo declarado.
- D) Só em globais.

---

**Q4.** E `a(7)` e `a("007")`?

- A) São o mesmo nó.
- B) São nós diferentes, porque `"007"` não é canônico e é tratado como texto.
- C) Geram erro.
- D) `"007"` é convertido para 7 automaticamente.

---

**Q5.** Como percorrer um array do último subscrito para o primeiro?

- A) `$ORDER(a(k), 0)`
- B) `$ORDER(a(k), -1)`
- C) `$QUERY(a(k), -1)`
- D) Não é possível.

---

**Q6.** O que o terceiro argumento do `$ORDER` faz?

- A) Define a direção.
- B) Recebe, por referência, o valor do nó encontrado, evitando uma leitura extra.
- C) Limita a quantidade de iterações.
- D) Define o nível a percorrer.

---

**Q7.** No idioma de ordenação do ObjectScript, por que se usa o ID como **segundo** subscrito do índice invertido?

- A) Por convenção estética.
- B) Para permitir empates: dois registros com o mesmo valor ocupam nós diferentes.
- C) Para acelerar a busca.
- D) Para economizar espaço.

---

**Q8.** O que acontece se você construir o índice como `idx(valor) = id`, com valores repetidos?

- A) Gera erro de chave duplicada.
- B) Os registros com valor repetido se sobrescrevem, e dados se perdem silenciosamente.
- C) O IRIS cria automaticamente um subscrito extra.
- D) Nada; todos são preservados.

---

**Q9.** Qual é a diferença entre `$ORDER` e `$QUERY`?

- A) Nenhuma.
- B) `$ORDER` anda num nível e devolve o subscrito; `$QUERY` percorre a árvore inteira e devolve a referência completa dos nós **com valor**.
- C) `$QUERY` é mais rápido em todos os casos.
- D) `$ORDER` só funciona em globais.

---

**Q10.** `$DATA(a("x"))` devolveu `10`. O que significa?

- A) Existem 10 filhos.
- B) O nó não tem valor próprio, mas tem descendentes.
- C) O nó tem valor e não tem descendentes.
- D) O nó não existe.

---

**Q11.** Você chama `do ..Preencher(array)` sem o ponto. O que acontece?

- A) Funciona normalmente.
- B) O array não volta preenchido: sem o ponto, os subscritos não são compartilhados.
- C) Erro de compilação.
- D) O array é apagado.

---

**Q12.** O que faz `$NAME(a("x", "y"))`?

- A) Devolve o valor do nó.
- B) Devolve a referência como texto: `a("x","y")`.
- C) Cria o nó.
- D) Devolve o número de subscritos.

---

**Q13.** `$QSUBSCRIPT(ref, 0)` devolve o quê?

- A) O primeiro subscrito.
- B) O nome da variável ou global.
- C) A quantidade de subscritos.
- D) O valor do nó.

---

**Q14.** Como ordenar nomes ignorando maiúsculas e minúsculas, preservando a grafia original?

- A) Não é possível.
- B) Usar o nome normalizado com `$ZCONVERT(nome, "U")` como chave e guardar o original no valor do nó.
- C) Usar `$ORDER` com direção `0`.
- D) Ordenar duas vezes.

---

**Q15.** Os códigos `"A7"` e `"A10"` como subscritos. Qual vem primeiro?

- A) `"A7"`, porque 7 é menor que 10.
- B) `"A10"`, porque a comparação é de texto e `1` vem antes de `7`.
- C) Depende da inserção.
- D) Ficam no mesmo nó.

---

**Q16.** O que `MERGE destino = origem` faz com o conteúdo prévio do destino?

- A) Apaga antes de copiar.
- B) Preserva, mesclando e sobrescrevendo apenas o que coincidir.
- C) Gera erro se o destino não estiver vazio.
- D) Move o destino para outro lugar.

---

**Q17.** Para que servem `$SORTBEGIN` e `$SORTEND`?

- A) Ordenar um array local em memória.
- B) Otimizar a inserção em massa de muitos nós numa global, consolidando no `$SORTEND`.
- C) Definir a ordem de classificação dos subscritos.
- D) Reconstruir índices de classes.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** `$ORDER` anda num nível e devolve o próximo subscrito.
- **A está errada:** `$QUERY` percorre a árvore inteira.
- **C está errada:** `$DATA` testa existência.
- **D está errada:** `$NAME` monta a referência como texto.

**Q2 — Resposta: B.**
- **B está certa:** vazio, números canônicos em ordem numérica (inclusive negativos), depois texto por código de caractere.
- **A está errada:** inverte.
- **C está errada:** números não ordenam alfabeticamente.
- **D está errada:** a ordem de inserção é irrelevante.

**Q3 — Resposta: B.**
- **B está certa:** `"10"` é a forma canônica do número 10, então é o mesmo subscrito.
- **A está errada:** são o mesmo nó.
- **C está errada:** subscritos não têm tipo declarado.
- **D está errada:** vale igualmente em locais e globais.

**Q4 — Resposta: B.**
- **B está certa:** zeros à esquerda quebram a canonicidade, tornando `"007"` texto.
- **A está errada:** são nós distintos.
- **C está errada:** nenhum erro ocorre.
- **D está errada:** não há conversão automática de subscrito.

**Q5 — Resposta: B.**
- **B está certa:** o segundo argumento `-1` percorre na direção inversa.
- **A está errada:** `0` não é uma direção válida.
- **C está errada:** `$QUERY` não recebe direção dessa forma.
- **D está errada:** é perfeitamente possível.

**Q6 — Resposta: B.**
- **B está certa:** o terceiro argumento recebe o valor do nó, poupando uma leitura por iteração.
- **A está errada:** direção é o segundo argumento.
- **C e D estão erradas:** não existem essas funções no `$ORDER`.

**Q7 — Resposta: B.**
- **B está certa:** sem o segundo subscrito, valores repetidos ocupariam o mesmo nó e se sobrescreveriam.
- **A está errada:** é necessidade estrutural, não estética.
- **C está errada:** o ganho é de correção, não de velocidade.
- **D está errada:** ele consome espaço, não economiza.

**Q8 — Resposta: B.**
- **B está certa:** cada valor repetido sobrescreve o anterior, e o índice fica com menos entradas que os dados — sem nenhum erro.
- **A está errada:** arrays não têm restrição de unicidade.
- **C está errada:** o IRIS não acrescenta subscritos sozinho.
- **D está errada:** os dados se perdem.

**Q9 — Resposta: B.**
- **B está certa:** `$ORDER` é por nível e devolve subscrito; `$QUERY` é pela árvore e devolve a referência completa, visitando apenas nós com valor.
- **A está errada:** têm finalidades distintas.
- **C está errada:** `$ORDER` é mais barato quando serve.
- **D está errada:** ambos funcionam em locais e globais.

**Q10 — Resposta: B.**
- **B está certa:** dezenas indicam descendentes; unidades indicam valor. `10` = tem filhos, não tem valor próprio.
- **A está errada:** o `10` não é uma contagem.
- **C está errada:** isso seria `1`.
- **D está errada:** isso seria `0`.

**Q11 — Resposta: B.**
- **B está certa:** sem o ponto, o método recebe apenas uma cópia do valor do nó raiz; os subscritos não voltam.
- **A está errada:** é justamente o que falha.
- **C está errada:** compila normalmente — por isso o erro é silencioso.
- **D está errada:** o array de quem chamou fica intacto e vazio do que se esperava.

**Q12 — Resposta: B.**
- **B está certa:** `$NAME` devolve a referência como texto, sem avaliar o valor.
- **A está errada:** para o valor, basta usar a referência diretamente.
- **C está errada:** ele não cria nada.
- **D está errada:** contar subscritos é `$QLENGTH`.

**Q13 — Resposta: B.**
- **B está certa:** com `n = 0`, `$QSUBSCRIPT` devolve o nome da variável ou global (com o circunflexo, no caso de global).
- **A está errada:** o primeiro subscrito é `n = 1`.
- **C está errada:** isso é `$QLENGTH`.
- **D está errada:** ele trabalha sobre a referência, não sobre o conteúdo.

**Q14 — Resposta: B.**
- **B está certa:** a chave normalizada ordena; o valor do nó preserva o original.
- **A está errada:** é o padrão consagrado, inclusive usado internamente pelo IRIS.
- **C está errada:** não existe direção `0`.
- **D está errada:** uma passagem basta.

**Q15 — Resposta: B.**
- **B está certa:** como texto, compara-se caractere a caractere: `1` antes de `7`.
- **A está errada:** só seria assim se fossem números canônicos puros.
- **C está errada:** a ordem independe da inserção.
- **D está errada:** são chaves distintas.

**Q16 — Resposta: B.**
- **B está certa:** `MERGE` mescla; o que já existia e não coincide permanece.
- **A está errada:** para substituir, é preciso `KILL` antes.
- **C está errada:** não há erro.
- **D está errada:** nada é movido.

**Q17 — Resposta: B.**
- **B está certa:** otimizam carga em massa numa global, acumulando e consolidando no `$SORTEND`.
- **A está errada:** arrays locais já vêm ordenados por construção.
- **C está errada:** a ordem de classificação é fixa do produto.
- **D está errada:** isso é `%BuildIndices()`.

---

## 9. Resumo relâmpago

1. **Array local é a mesma árvore de uma global**, só que na memória. Sem circunflexo, morre com o processo.
2. **Os subscritos já vêm ordenados, sempre.** Não existe função de ordenação — e não faz falta.
3. **O idioma central: para ordenar por algo, use esse algo como subscrito.** É como os índices do IRIS funcionam.
4. **Ordem exata:** subscrito vazio → números canônicos em ordem numérica (negativos incluídos) → texto em ordem de código de caractere.
5. **Número canônico** = sem zeros à esquerda, sem zeros supérfluos à direita, sem `+`, sem ponto decimal desnecessário. `"10"` é canônico; `"007"`, `"1.50"` e `"+5"` não são.
6. `a(10)` e `a("10")` são **o mesmo nó**. `a(7)` e `a("007")` são **nós diferentes**.
7. Arrays são **esparsos**: só existem os nós criados. Percorra com `$ORDER`, nunca com `for i = 1:1:n`.
8. **`$DATA`**: `0` inexistente, `1` só valor, `10` só filhos, `11` valor e filhos.
9. **`$GET`** lê sem `<UNDEFINED>`, com valor padrão opcional.
10. **`$ORDER(ref, direção, .valor)`** — `-1` percorre ao contrário; o terceiro argumento traz o valor de brinde.
11. O laço padrão começa e termina com `""`. **Sempre `QUIT:k=""`.**
12. `$ORDER` **não exige que o subscrito atual exista** — é isso que permite busca por faixa.
13. `$ORDER(a(""))` dá o primeiro; `$ORDER(a(""), -1)` dá o último, em duas operações.
14. **`$QUERY(@ref)`** percorre a árvore inteira e devolve a referência completa, visitando **apenas nós com valor**.
15. **`$NAME`** monta a referência como texto; **`$QLENGTH`** conta subscritos; **`$QSUBSCRIPT(ref, n)`** extrai o n-ésimo, com `n = 0` dando o nome da raiz.
16. **Referência nua** (`^(4)`) existe, deve ser reconhecida em código legado e **nunca escrita** em código novo.
17. **`MERGE`** copia subárvores e **não limpa o destino**. **`KILL`** apaga nó e subárvore; **`ZKILL`** apaga só o valor.
18. **No índice invertido, o ID vai como segundo subscrito** — sem isso, empates se sobrescrevem e dados somem em silêncio.
19. **Ordem decrescente**: percorra com `-1`. Nenhuma estrutura extra.
20. **Vários critérios**: empilhe subscritos na ordem de prioridade; cada nível tem sua própria direção.
21. **Texto sem diferenciar maiúsculas**: chave com `$ZCONVERT(v, "U")`, original guardado no valor do nó.
22. **Códigos alfanuméricos**: preencha com zeros à largura fixa para que a ordem de texto coincida com a numérica.
23. **Chaves numéricas calculadas**: normalize com `$FNUMBER` para evitar chaves duplicadas por arredondamento.
24. **`$SORTBEGIN`/`$SORTEND`** otimizam carga em massa numa global; não leia entre as duas chamadas.
25. Arrays entre métodos exigem **`ByRef`/`Output` e o ponto na chamada**; limpe com `kill` no início do método que preenche.
26. **Se os dados vêm de uma tabela, prefira `ORDER BY`.** Ordene em memória quando o critério é calculado ou os dados vêm de várias fontes.

---

## 10. Cartões de memorização

**Frente:** Como se ordena no ObjectScript?
**Verso:** Usando o valor como subscrito. Os subscritos já vêm ordenados automaticamente.

**Frente:** Qual a ordem de classificação dos subscritos?
**Verso:** Vazio → números canônicos em ordem numérica → texto em ordem de código de caractere.

**Frente:** O que é um número canônico?
**Verso:** Sem zeros à esquerda, sem zeros supérfluos à direita, sem `+`, sem ponto decimal desnecessário. `7` sim; `007`, `7.0`, `+7` não.

**Frente:** `a(10)` e `a("10")` são o mesmo nó?
**Verso:** Sim — `"10"` é a forma canônica do número 10.

**Frente:** `a(7)` e `a("007")` são o mesmo nó?
**Verso:** Não — `"007"` não é canônico, logo é texto e vai para outro lugar da ordenação.

**Frente:** Qual o laço padrão do `$ORDER`?
**Verso:** `set k="" for { set k=$ORDER(a(k)) quit:k="" ... }`.

**Frente:** Como percorrer ao contrário?
**Verso:** `$ORDER(a(k), -1)`. Também se começa com `""`.

**Frente:** O que faz o terceiro argumento do `$ORDER`?
**Verso:** Recebe o valor do nó por referência, poupando uma leitura por iteração.

**Frente:** Como pegar o primeiro e o último subscrito sem percorrer?
**Verso:** `$ORDER(a(""))` e `$ORDER(a(""), -1)`.

**Frente:** `$ORDER` exige que o subscrito atual exista?
**Verso:** Não. É isso que permite busca por faixa: "qual vem depois de M?".

**Frente:** Por que o ID vai como segundo subscrito no índice invertido?
**Verso:** Para permitir empates. Sem ele, registros com o mesmo valor se sobrescrevem silenciosamente.

**Frente:** Diferença entre `$ORDER` e `$QUERY`.
**Verso:** `$ORDER` anda num nível e devolve o subscrito. `$QUERY` percorre a árvore e devolve a referência completa, só de nós com valor.

**Frente:** O que faz `$NAME(a("x","y"))`?
**Verso:** Devolve a referência como texto: `a("x","y")`, sem avaliar o valor.

**Frente:** O que fazem `$QLENGTH` e `$QSUBSCRIPT`?
**Verso:** `$QLENGTH(ref)` conta os subscritos; `$QSUBSCRIPT(ref, n)` extrai o n-ésimo — com `n = 0` devolvendo o nome da raiz.

**Frente:** O que é referência nua e o que fazer com ela?
**Verso:** `^(4)` reaproveita os subscritos da última referência. Reconheça em código legado; nunca escreva em código novo.

**Frente:** `MERGE destino = origem` limpa o destino?
**Verso:** Não. Mescla. Faça `KILL destino` antes se quiser substituir.

**Frente:** `KILL a("x")` apaga o quê?
**Verso:** O nó **e toda a subárvore**. Para apagar só o valor, `ZKILL`.

**Frente:** Como ordenar texto ignorando maiúsculas, mantendo a grafia original?
**Verso:** Chave `$ZCONVERT(nome, "U")`, original guardado no valor do nó.

**Frente:** Como fazer `"A7"` vir antes de `"A10"`?
**Verso:** Preenchendo com zeros à largura fixa: `A000007` e `A000010`.

**Frente:** Como passar um array para um método?
**Verso:** `ByRef` ou `Output` na assinatura, e **o ponto** na chamada: `..Metodo(.array)`.

**Frente:** Qual a primeira linha de um método que preenche um array de saída?
**Verso:** `kill array` — para não misturar restos de chamadas anteriores.

**Frente:** Para que servem `$SORTBEGIN` e `$SORTEND`?
**Verso:** Otimizar inserção em massa numa global. Não leia a global entre as duas chamadas.

**Frente:** Quando ordenar em memória em vez de usar `ORDER BY`?
**Verso:** Quando o critério é calculado a partir de outra consulta, ou quando os dados vêm de várias fontes.

---

Digite CONTINUAR para o próximo capítulo.
