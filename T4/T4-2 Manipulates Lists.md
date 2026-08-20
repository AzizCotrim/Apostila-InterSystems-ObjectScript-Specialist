# Apostila InterSystems ObjectScript Specialist
## Capítulo 14 — T4.2 Manipulates Lists (Manipulando listas)

> Ainda em **T4 — Functions & APIs**, o domínio de maior peso da prova. Este capítulo trata da estrutura mais característica do IRIS: a lista `$LIST`. Você já a viu sem saber — é ela que aparece como `$lb(...)` dentro das globais de toda classe persistente.

---

## 1. O que você vai saber fazer ao terminar

1. Explicar o que é uma **lista `$LIST`** e por que ela é diferente de um texto com separadores.
2. Criar listas com **`$LISTBUILD`**, inclusive com elementos **sem valor**.
3. Ler com **`$LIST`** e com segurança com **`$LISTGET`**.
4. Medir com **`$LISTLENGTH`** e testar elementos com **`$LISTDATA`**.
5. Procurar com **`$LISTFIND`** e comparar com **`$LISTSAME`**.
6. Validar com **`$LISTVALID`**.
7. Percorrer eficientemente com **`$LISTNEXT`**.
8. Extrair **sublistas** e concatenar listas com `_`.
9. **Alterar, inserir e remover** elementos.
10. Converter entre lista e texto com **`$LISTTOSTRING`** e **`$LISTFROMSTRING`**.
11. Explicar a diferença crucial entre **`$LIST` e `$PIECE`**, e quando cada um é a escolha certa.
12. Distinguir **`%List`** (o tipo de dado) de **`list Of`** (a coleção de objetos).
13. Reconhecer onde as listas aparecem no IRIS: armazenamento, `%INLIST`, `%BuildIndices`, retornos de método.
14. Levar o projeto à versão **1.5**, trocando os separadores improvisados por listas de verdade.

---

## 2. O conceito em linguagem de gente

### 2.1 O problema dos separadores

Você precisa guardar três informações num único valor: nome, cidade e observação.

A solução óbvia é juntar com um separador:

```
"Maria Silva|Potirendaba|Paciente pontual"
```

Funciona. Até o dia em que chega este dado:

```
"Maria Silva|Potirendaba|Trouxe exames de 2024|2025"
```

A observação continha o separador. Agora o valor tem quatro pedaços em vez de três, e o programa que lê acha que a cidade é `Potirendaba` e a observação é `Trouxe exames de 2024` — perdendo o resto silenciosamente.

A resposta tradicional é escolher um separador "que nunca aparece". Isso é uma aposta, e apostas perdem. Um dia alguém cola um texto de outro sistema, e o separador improvável aparece.

**A lista `$LIST` resolve esse problema de forma definitiva.**

### 2.2 A ideia: guardar o tamanho junto

Em vez de marcar onde um pedaço termina com um caractere especial, a lista **anota o comprimento de cada pedaço antes dele**.

A analogia é a de dois jeitos de empacotar encomendas:

**Jeito do separador:** você coloca todos os objetos numa caixa e põe uma folha de papel entre eles. Se um dos objetos *for* uma folha de papel, quem abre a caixa se confunde.

**Jeito da lista:** você coloca cada objeto num envelope, e escreve na frente do envelope quantos centímetros ele mede. Quem abre lê "12 cm", pega exatamente 12 cm, e sabe que o próximo envelope começa ali. **Não importa o que tenha dentro** — pode ser outra folha de papel, pode ser outro envelope inteiro, pode estar vazio. A contagem não mente.

Isso tem três consequências que valem o capítulo inteiro:

1. **Qualquer conteúdo é seguro.** Um elemento pode conter vírgulas, barras, quebras de linha, e até outra lista completa.
2. **Uma lista pode conter outra lista** sem ambiguidade. Você viu isso no Capítulo 8: o endereço embutido aparecia como uma lista dentro da lista do paciente.
3. **O acesso é direto.** Para chegar ao terceiro elemento, o IRIS lê o comprimento do primeiro, pula, lê o segundo, pula, e chega. Não precisa examinar caractere por caractere procurando separadores.

### 2.3 Onde você já viu listas sem saber

Volte à saída do Capítulo 8:

```
^LabStudy.PatientD(1)=$lb("","Maria Silva",54683,"REG-000001","F",1,"2026-08-19 15:02:11",$lb("","Potirendaba","SP",""))
```

Isso é uma lista. `$lb` é a forma abreviada de `$LISTBUILD` que o `ZWRITE` usa para exibir.

**Todo objeto persistente do IRIS é gravado como uma lista.** É por isso que:

- as propriedades ocupam **posições**, e não nomes — daí o mapa `Storage` do Capítulo 11;
- uma propriedade nova entra **no fim**, sem quebrar as anteriores;
- um valor pode conter qualquer caractere sem risco de corromper o registro;
- o objeto embutido aparece como **lista dentro de lista**.

Aprender `$LIST` é aprender o formato nativo do IRIS.

### 2.4 Elemento sem valor não é elemento vazio

Uma lista distingue duas coisas que parecem iguais:

- um elemento que **existe e contém a string vazia**;
- um elemento que **existe mas não tem valor atribuído**.

```objectscript
set a = $LISTBUILD("x", "", "z")     // três elementos, o do meio é string vazia
set b = $LISTBUILD("x", , "z")       // três elementos, o do meio SEM valor
```

Repare na **vírgula dupla** de `b`: ela cria um elemento sem valor.

Isso é a mesma distinção do Capítulo 10 — ficha B contra ficha A — agora dentro de uma lista. E é exatamente assim que o IRIS grava uma propriedade nunca preenchida: como elemento sem valor.

`$LISTDATA` distingue os dois; `$LISTGET` devolve vazio nos dois casos.

### 2.5 `%List` não é `list Of`

Confusão frequente, e a prova explora:

| | `%List` | `list Of %String` |
|---|---|---|
| O que é | um **tipo de dado**: a lista `$LIST` | uma **coleção de objetos** |
| Como se lê | `$LIST(valor, n)` | `obj.Prop.GetAt(n)` |
| Índice começa em | 1 | 1 |
| Como se cria | `$LISTBUILD(...)` | `obj.Prop.Insert(...)` |
| Onde vive | dentro de uma propriedade, como valor único | em tabela filha projetada no SQL |
| Quando usar | poucos itens, sempre lidos juntos | muitos itens, consultáveis separadamente |

E, para completar o quadro do Capítulo 4, existe ainda o **`%DynamicArray`** do JSON, que começa em **zero** e usa `%Get()`.

Três estruturas de sequência, três APIs, dois pontos de partida de índice. Vale fazer um cartão só disso.

---

## 3. As funções

### 3.1 Criando: `$LISTBUILD`

```objectscript
set lista = $LISTBUILD("HGB", 13.5, "g/dL")
set vazia = $LISTBUILD()
set comBuraco = $LISTBUILD("a", , "c")
set aninhada = $LISTBUILD("paciente", $LISTBUILD("rua", "cidade"), 2026)
```

- Cada argumento vira um elemento.
- A **vírgula dupla** cria elemento sem valor.
- Uma lista pode ser elemento de outra, sem escape nenhum.
- O resultado é uma **string binária**. Escrevê-la na tela com `WRITE` produz caracteres estranhos — use `ZWRITE` ou `$LISTTOSTRING` para inspecionar.

### 3.2 Lendo: `$LIST` e `$LISTGET`

```objectscript
write $LIST(lista, 1), !          // HGB
write $LIST(lista), !             // HGB  (sem posição = primeiro)
write $LIST(lista, 3), !          // g/dL
```

Formas do `$LIST`:

| Chamada | O que devolve |
|---|---|
| `$LIST(lista)` | o **primeiro** elemento |
| `$LIST(lista, n)` | o elemento na posição `n` |
| `$LIST(lista, de, ate)` | uma **sublista** com os elementos de `de` até `ate` |
| `$LIST(lista, de, *)` | sublista de `de` até o **último** |
| `$LIST(lista, -1)` | o **último** elemento (posição negativa conta do fim) |

**Atenção:** `$LIST` **gera erro** quando a posição não existe ou quando o elemento não tem valor. Em código de produção, use:

```objectscript
write $LISTGET(lista, 10), !            // "" — sem erro
write $LISTGET(lista, 10, "?"), !       // "?" — com padrão
```

**`$LISTGET(lista, posição, padrão)`** é a versão segura. É para listas o que `$GET` é para variáveis.

Um detalhe importante: `$LIST(lista, de, ate)` devolve uma **lista**, não um valor. Já `$LIST(lista, n)` devolve o **valor** do elemento. Confundir os dois é erro comum.

### 3.3 Medindo e testando

```objectscript
write $LISTLENGTH(lista), !               // quantos elementos
write $LISTDATA(lista, 2), !              // 1 se o elemento tem valor, 0 se não
write $LISTVALID(algumaCoisa), !          // 1 se for uma lista válida
write $LISTSAME(listaA, listaB), !        // 1 se forem iguais elemento a elemento
```

- **`$LISTLENGTH`** conta os elementos, incluindo os sem valor.
- **`$LISTDATA(lista, n)`** distingue "tem valor" de "não tem valor" — a distinção da seção 2.4.
- **`$LISTVALID(x)`** verifica se `x` é uma lista bem formada. Útil ao receber dados de fora.
- **`$LISTSAME(a, b)`** compara duas listas. Note que ela pode considerar iguais listas construídas de formas ligeiramente diferentes mas com o mesmo conteúdo lógico; para uma comparação byte a byte, compare com `=`.

### 3.4 Procurando: `$LISTFIND`

```objectscript
write $LISTFIND(lista, "g/dL"), !         // 3
write $LISTFIND(lista, "XXX"), !          // 0 — não encontrado
write $LISTFIND(lista, "a", 2), !         // procura DEPOIS da posição 2
```

- Devolve a **posição** do elemento, ou **`0`** se não achar.
- O terceiro argumento indica a partir de onde procurar (procura **depois** daquela posição).
- Para encontrar todas as ocorrências, chame em laço, avançando o ponto de partida.

### 3.5 Percorrendo: `$LISTNEXT`

O laço `for i = 1:1:$LISTLENGTH(lista)` funciona, mas obriga o IRIS a recontar posições a cada acesso. Para listas grandes, existe algo melhor:

```objectscript
set ptr = 0
while $LISTNEXT(lista, ptr, valor) {
    write valor, !
}
```

- **`ptr`** começa em **0** e é atualizado por referência a cada chamada.
- Devolve `1` enquanto houver elementos, `0` quando acabar.
- É mais eficiente porque mantém a posição no ponteiro, sem recomeçar do início.

**Cuidado:** quando o elemento não tem valor, `valor` fica **indefinido** após a chamada. Se a lista puder ter buracos, proteja:

```objectscript
set ptr = 0
while $LISTNEXT(lista, ptr, valor) {
    write $GET(valor), !
}
```

Sem o `$GET`, um elemento sem valor causa `<UNDEFINED>` no acesso seguinte.

### 3.6 Alterando elementos

```objectscript
set lista = $LISTBUILD("a", "b", "c")

set $LIST(lista, 2) = "B"                     // substitui o elemento 2
```

Sim: `$LIST` pode aparecer do **lado esquerdo** de um `SET`. Isso altera a lista no lugar.

Também funciona com faixa:

```objectscript
set $LIST(lista, 2, 3) = "X"                  // substitui os elementos 2 e 3 por um só
```

E é assim que se **remove** e se **insere**, embora a forma mais legível seja concatenar sublistas, como veremos a seguir.

### 3.7 Concatenando e fatiando

Esta é a propriedade mais elegante do formato: **listas se concatenam com `_`**, o mesmo operador de texto.

```objectscript
set a = $LISTBUILD("x", "y")
set b = $LISTBUILD("z")
set c = a _ b                                  // lista de 3 elementos
```

Funciona porque o formato é autodescritivo: colar duas listas produz uma lista válida.

Combinando com sublistas, você monta as operações que faltam:

```objectscript
// inserir "novo" antes da posição n
set lista = $LIST(lista, 1, n - 1) _ $LISTBUILD("novo") _ $LIST(lista, n, *)

// remover a posição n
set lista = $LIST(lista, 1, n - 1) _ $LIST(lista, n + 1, *)

// acrescentar ao final
set lista = lista _ $LISTBUILD("ultimo")
```

**Cuidado com as bordas:** quando `n` é `1`, a expressão `$LIST(lista, 1, 0)` pede uma faixa vazia. Trate os extremos explicitamente, como faremos no exercício 14.3.

### 3.8 Convertendo para texto e de volta

```objectscript
set lista = $LISTBUILD("HGB", "GLU", "CHOL")

write $LISTTOSTRING(lista), !                  // HGB,GLU,CHOL  (vírgula é o padrão)
write $LISTTOSTRING(lista, ";"), !             // HGB;GLU;CHOL
write $LISTTOSTRING(lista, " | "), !           // HGB | GLU | CHOL

set devolta = $LISTFROMSTRING("A,B,C", ",")
write $LISTLENGTH(devolta), !                  // 3
```

- **`$LISTTOSTRING(lista, separador)`** junta os elementos num texto legível. O separador padrão é a vírgula.
- **`$LISTFROMSTRING(texto, separador)`** faz o caminho inverso, quebrando um texto em lista.

Estas duas são as **pontes** entre o mundo interno (lista) e o mundo externo (CSV, tela, arquivo).

E aqui está o ponto essencial: **converter para texto reintroduz o problema do separador.** Se um elemento contiver o separador escolhido, o `$LISTFROMSTRING` de volta produzirá uma lista diferente da original. A conversão é uma via de mão única segura apenas para exibição; para ida e volta confiável, mantenha a lista.

`$LISTTOSTRING` também aceita um terceiro argumento que controla o tratamento de elementos sem valor. O comportamento exato varia por versão: **verificar na documentação oficial**.

### 3.9 `$LIST` contra `$PIECE`

`$PIECE` é a função de separadores, e será tratada a fundo no Capítulo 15. A comparação, porém, pertence aqui:

| | `$LIST` | `$PIECE` |
|---|---|---|
| Como delimita | comprimento gravado antes de cada elemento | um caractere separador |
| Conteúdo com o separador | **seguro** | **quebra** |
| Estruturas aninhadas | naturais | impossíveis sem escape |
| Legível como texto | não (binário) | sim |
| Vindo de fora (CSV, arquivo) | precisa converter | direto |
| Acesso ao n-ésimo | direto | varre o texto |
| Distingue "sem valor" de "vazio" | sim | não |

**Regra de decisão:**

> **Dentro do seu sistema, use `$LIST`. Na fronteira com o mundo externo, use `$PIECE`** — porque o mundo externo fala em separadores.

Todo dado que entra em formato separado deve ser convertido para lista o quanto antes; todo dado que sai deve ser convertido no último momento.

### 3.10 Onde as listas aparecem no IRIS

- **Armazenamento de objetos** — cada nó de dados é uma lista.
- **Tipo `%List`** — uma propriedade pode ser declarada assim, guardando uma lista inteira num campo.
- **`%INLIST`** no SQL — `WHERE Coluna %INLIST ?` recebe uma lista como parâmetro, evitando montar `IN (...)` com valores colados.
- **`%BuildIndices($LISTBUILD("Idx1", "Idx2"))`** — vários métodos do sistema recebem listas de nomes.
- **Retornos de método** — devolver várias informações numa lista é comum e idiomático.
- **`$LISTBUILD` em subscritos** — evite: uma lista como subscrito ordena por bytes, não pelo conteúdo lógico.

---

## 4. Exemplo comentado

Arquivo `src/LabStudy/Demo/Lists.cls`:

```objectscript
/// Everything about $LIST, demonstrated.
Class LabStudy.Demo.Lists Extends %RegisteredObject
{

/// Building and reading.
ClassMethod Basics() As %Status
{
    set exam = $LISTBUILD("HGB", 13.5, "g/dL", "final")

    write "-- reading --", !
    write "  length      : ", $LISTLENGTH(exam), !
    write "  first       : ", $LIST(exam), !
    write "  element 2   : ", $LIST(exam, 2), !
    write "  last (-1)   : ", $LIST(exam, -1), !
    write "  as text     : ", $LISTTOSTRING(exam, " / "), !

    write !, "-- safe reading --", !
    write "  $LISTGET(10)         : [", $LISTGET(exam, 10), "]", !
    write "  $LISTGET(10,'none')  : [", $LISTGET(exam, 10, "none"), "]", !

    write !, "-- searching --", !
    write "  position of 'g/dL'   : ", $LISTFIND(exam, "g/dL"), !
    write "  position of 'XXX'    : ", $LISTFIND(exam, "XXX"), !

    write !, "-- sublists --", !
    set sub = $LIST(exam, 2, 3)
    write "  sublist length       : ", $LISTLENGTH(sub), !
    write "  sublist as text      : ", $LISTTOSTRING(sub), !

    quit $$$OK
}

/// Elements with and without a value.
ClassMethod Holes() As %Status
{
    set withEmpty = $LISTBUILD("a", "", "c")
    set withHole  = $LISTBUILD("a", , "c")

    write "-- both have length 3 --", !
    write "  withEmpty : ", $LISTLENGTH(withEmpty), !
    write "  withHole  : ", $LISTLENGTH(withHole), !

    write !, "-- but $LISTDATA tells them apart --", !
    write "  withEmpty position 2 : ", $LISTDATA(withEmpty, 2), !
    write "  withHole  position 2 : ", $LISTDATA(withHole, 2), !

    write !, "-- $LISTGET returns empty for both --", !
    write "  withEmpty : [", $LISTGET(withEmpty, 2), "]", !
    write "  withHole  : [", $LISTGET(withHole, 2), "]", !

    quit $$$OK
}

/// The reason $LIST exists: content that contains the separator.
ClassMethod WhyNotPiece() As %Status
{
    set note = "Trouxe exames de 2024|2025"

    write "-- with a delimited string --", !
    set text = "Maria Silva|Potirendaba|"_note
    write "  stored   : ", text, !
    write "  piece 1  : ", $PIECE(text, "|", 1), !
    write "  piece 2  : ", $PIECE(text, "|", 2), !
    write "  piece 3  : ", $PIECE(text, "|", 3), "   <-- truncated!", !
    write "  pieces   : ", $LENGTH(text, "|"), "   <-- expected 3", !

    write !, "-- with a list --", !
    set list = $LISTBUILD("Maria Silva", "Potirendaba", note)
    write "  length   : ", $LISTLENGTH(list), !
    write "  element 1: ", $LIST(list, 1), !
    write "  element 2: ", $LIST(list, 2), !
    write "  element 3: ", $LIST(list, 3), "   <-- intact", !

    quit $$$OK
}

/// Nesting: a list inside a list.
ClassMethod Nested() As %Status
{
    set address = $LISTBUILD("Rua das Flores, 100", "Potirendaba", "SP")
    set patient = $LISTBUILD("Maria Silva", 36, address)

    write "-- outer list --", !
    write "  length : ", $LISTLENGTH(patient), !
    write "  name   : ", $LIST(patient, 1), !
    write "  age    : ", $LIST(patient, 2), !

    write !, "-- inner list --", !
    set inner = $LIST(patient, 3)
    write "  is a valid list? ", $LISTVALID(inner), !
    write "  length         : ", $LISTLENGTH(inner), !
    write "  street         : ", $LIST(inner, 1), !
    write "  city           : ", $LIST(inner, 2), !

    write !, "  note the street contains a comma and nothing broke", !

    quit $$$OK
}

/// Efficient traversal with $LISTNEXT.
ClassMethod Traverse() As %Status
{
    set codes = $LISTBUILD("HGB", "GLU", , "CHOL", "TRIG")

    write "-- with $LISTNEXT --", !
    set ptr = 0
    set n = 0
    while $LISTNEXT(codes, ptr, value) {
        set n = n + 1
        write "  ", n, ": [", $GET(value), "]", !
    }

    write !, "-- with a counted loop --", !
    for i = 1:1:$LISTLENGTH(codes) {
        write "  ", i, ": [", $LISTGET(codes, i), "]  (has value: ", $LISTDATA(codes, i), ")", !
    }

    quit $$$OK
}

/// Modifying: replace, append, insert, remove.
ClassMethod Modify() As %Status
{
    set l = $LISTBUILD("a", "b", "c", "d")
    write "start   : ", $LISTTOSTRING(l), !

    set $LIST(l, 2) = "B"
    write "replace : ", $LISTTOSTRING(l), !

    set l = l _ $LISTBUILD("e")
    write "append  : ", $LISTTOSTRING(l), !

    // insert "X" before position 3
    set l = $LIST(l, 1, 2) _ $LISTBUILD("X") _ $LIST(l, 3, *)
    write "insert  : ", $LISTTOSTRING(l), !

    // remove position 1 (edge case: no left part)
    set l = $LIST(l, 2, *)
    write "remove 1: ", $LISTTOSTRING(l), !

    quit $$$OK
}

/// Round trip through text, and where it is unsafe.
ClassMethod RoundTrip() As %Status
{
    set safe = $LISTBUILD("HGB", "GLU", "CHOL")
    set text = $LISTTOSTRING(safe, ",")
    set back = $LISTFROMSTRING(text, ",")

    write "-- safe round trip --", !
    write "  original length : ", $LISTLENGTH(safe), !
    write "  as text         : ", text, !
    write "  back length     : ", $LISTLENGTH(back), !
    write "  same?           : ", $LISTSAME(safe, back), !

    set risky = $LISTBUILD("HGB", "valor, com virgula", "CHOL")
    set text2 = $LISTTOSTRING(risky, ",")
    set back2 = $LISTFROMSTRING(text2, ",")

    write !, "-- unsafe round trip --", !
    write "  original length : ", $LISTLENGTH(risky), !
    write "  as text         : ", text2, !
    write "  back length     : ", $LISTLENGTH(back2), "   <-- changed!", !
    write "  same?           : ", $LISTSAME(risky, back2), !

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Basics()      write !
    do ..Holes()       write !
    do ..WhyNotPiece() write !
    do ..Nested()      write !
    do ..Traverse()    write !
    do ..Modify()      write !
    do ..RoundTrip()
    quit $$$OK
}

}
```

### 4.1 Executando

```
LABSTUDY>DO ##class(LabStudy.Demo.Lists).Demo()
-- reading --
  length      : 4
  first       : HGB
  element 2   : 13.5
  last (-1)   : final
  as text     : HGB / 13.5 / g/dL / final

-- safe reading --
  $LISTGET(10)         : []
  $LISTGET(10,'none')  : [none]

-- searching --
  position of 'g/dL'   : 3
  position of 'XXX'    : 0

-- sublists --
  sublist length       : 2
  sublist as text      : 13.5,g/dL

-- both have length 3 --
  withEmpty : 3
  withHole  : 3

-- but $LISTDATA tells them apart --
  withEmpty position 2 : 1
  withHole  position 2 : 0

-- $LISTGET returns empty for both --
  withEmpty : []
  withHole  : []

-- with a delimited string --
  stored   : Maria Silva|Potirendaba|Trouxe exames de 2024|2025
  piece 1  : Maria Silva
  piece 2  : Potirendaba
  piece 3  : Trouxe exames de 2024   <-- truncated!
  pieces   : 4   <-- expected 3

-- with a list --
  length   : 3
  element 1: Maria Silva
  element 2: Potirendaba
  element 3: Trouxe exames de 2024|2025   <-- intact

-- outer list --
  length : 3
  name   : Maria Silva
  age    : 36

-- inner list --
  is a valid list? 1
  length         : 3
  street         : Rua das Flores, 100
  city           : Potirendaba

  note the street contains a comma and nothing broke

-- with $LISTNEXT --
  1: [HGB]
  2: [GLU]
  3: []
  4: [CHOL]
  5: [TRIG]

-- with a counted loop --
  1: [HGB]  (has value: 1)
  2: [GLU]  (has value: 1)
  3: []  (has value: 0)
  4: [CHOL]  (has value: 1)
  5: [TRIG]  (has value: 1)

start   : a,b,c,d
replace : a,B,c,d
append  : a,B,c,d,e
insert  : a,B,X,c,d,e
remove 1: B,X,c,d,e

-- safe round trip --
  original length : 3
  as text         : HGB,GLU,CHOL
  back length     : 3
  same?           : 1

-- unsafe round trip --
  original length : 3
  as text         : HGB,valor, com virgula,CHOL
  back length     : 4   <-- changed!
  same?           : 0
```

O que observar, ponto por ponto:

- **`$LIST(exam, -1)` devolveu o último elemento.** Posições negativas contam do fim, como em muitas funções de texto do IRIS.
- **A sublista `$LIST(exam, 2, 3)` é uma lista de verdade**, com `$LISTLENGTH` igual a 2. Não é um texto.
- **`withEmpty` e `withHole` têm o mesmo comprimento e o mesmo `$LISTGET`**, e mesmo assim são diferentes. Só `$LISTDATA` revela. Esta é a distinção do Capítulo 10, agora dentro de uma lista — e é exatamente assim que o IRIS grava uma propriedade nunca preenchida.
- **`WhyNotPiece` é o coração do capítulo.** O texto com separador virou quatro pedaços em vez de três, e o terceiro foi truncado — **sem nenhum erro**. A lista guardou o mesmo conteúdo intacto. Se você só levar uma coisa deste capítulo, leve esta tela.
- **A rua com vírgula dentro da lista aninhada não causou problema nenhum.** Numa estrutura com separadores, seria necessário escapar a vírgula, e escape é a origem de metade dos bugs de importação do mundo.
- **`$LISTNEXT` visitou o elemento sem valor** e devolveu vazio graças ao `$GET`. Sem ele, haveria `<UNDEFINED>`.
- **`remove 1` usou apenas `$LIST(l, 2, *)`**, porque não existe parte à esquerda. Tratar a borda explicitamente evitou a expressão inválida `$LIST(l, 1, 0)`.
- **A ida e volta insegura mudou o comprimento de 3 para 4.** O elemento com vírgula virou dois. É a prova de que `$LISTTOSTRING` seguido de `$LISTFROMSTRING` **não é uma identidade** quando o conteúdo pode conter o separador.

---

## 5. Variações e detalhes

### 5.1 Listas como valor de retorno

Devolver várias informações numa lista é idiomático:

```objectscript
ClassMethod Analyse(values As %List) As %List
{
    set total = 0, count = 0, max = ""
    set ptr = 0

    while $LISTNEXT(values, ptr, v) {
        set v = $GET(v)
        continue:v=""

        set count = count + 1
        set total = total + v
        if (max = "") || (v > max) { set max = v }
    }

    quit $LISTBUILD(count, total, $SELECT(count: total / count, 1: ""), max)
}
```

Quem chama desmonta:

```objectscript
set r = ..Analyse($LISTBUILD(10, 20, , 30))
write "count=", $LIST(r, 1), " sum=", $LIST(r, 2), " avg=", $LIST(r, 3), !
```

Compare com a alternativa de parâmetros `Output` do Capítulo 3: a lista é mais compacta e pode ser guardada, passada adiante ou gravada; os parâmetros de saída são mais explícitos e autodocumentados. **A lista ganha quando o resultado viaja; os parâmetros ganham quando o resultado é consumido ali mesmo.**

### 5.2 O tipo `%List` numa propriedade

```objectscript
Property ReferenceRange As %List;
```

```objectscript
set exam.ReferenceRange = $LISTBUILD(12, 16, "g/dL")
```

Isso guarda a lista inteira **num único campo**. Vantagens: compacto, uma leitura só, sem tabela filha.

Desvantagens: o SQL vê o campo como um valor opaco — você não consegue filtrar por um elemento individual nem indexá-lo. É a mesma decisão do Capítulo 2 sobre cifrar campos consultáveis: **não guarde como lista aquilo que você precisa consultar elemento a elemento**.

Use `%List` para conjuntos pequenos e fixos que são sempre lidos juntos. Para o resto, use `list Of` (tabela filha) ou uma classe própria.

### 5.3 Listas no SQL

```objectscript
set codes = $LISTBUILD("HGB", "GLU", "CHOL")

set rs = ##class(%SQL.Statement).%ExecDirect(,
    "SELECT %ID, TestCode FROM LabStudy.EXAM WHERE TestCode %INLIST ?", codes)
```

Isso é muito melhor do que montar `IN ('HGB','GLU','CHOL')` colando valores, pelos motivos do Capítulo 9: segurança contra injeção e reaproveitamento do cache de consultas. Uma consulta preparada serve para qualquer quantidade de códigos.

O SQL também oferece funções para trabalhar com colunas de tipo lista. Os nomes e a disponibilidade variam por versão: **verificar na documentação oficial**.

### 5.4 Listas como subscrito: não faça

```objectscript
set a($LISTBUILD("x", 1)) = "valor"      // evite
```

Funciona, mas a ordenação passa a ser pela representação binária, não pelo conteúdo lógico. Você perde a previsibilidade do Capítulo 13 e ganha uma estrutura ilegível no `ZWRITE`.

Se você precisa de chave composta, use **subscritos múltiplos**:

```objectscript
set a("x", 1) = "valor"                   // faça
```

### 5.5 Validando listas vindas de fora

```objectscript
if '$LISTVALID(dado) {
    quit $$$ERROR($$$GeneralError, "Expected a list")
}
```

Um valor que não é lista, passado a `$LIST`, causa erro em execução. Ao receber dados de outro sistema, de um arquivo ou de um parâmetro público, valide antes.

Um alerta: `$LISTVALID` verifica a **estrutura**, não o conteúdo. Certas strings comuns podem, por coincidência, ter estrutura de lista válida. A validação é uma boa primeira barreira, não uma garantia absoluta.

### 5.6 Desempenho

Para acessar o n-ésimo elemento:

- **`$LIST`** salta diretamente, usando os comprimentos gravados.
- **`$PIECE`** varre o texto contando separadores.

Para listas longas, a diferença é significativa. E `$LISTNEXT` é mais rápido que `$LIST` num laço, porque mantém a posição.

Listas também são **mais compactas** que texto separado quando os elementos são curtos, porque o comprimento cabe em poucos bytes e não há caractere separador desperdiçado.

Isso explica, retroativamente, por que o IRIS grava objetos como listas: é o formato mais rápido e mais compacto para o acesso posicional que o armazenamento exige.

---

## 6. Pegadinhas e erros comuns

**1) Usar `WRITE` numa lista.**
Sai lixo binário. Use `ZWRITE` para inspecionar ou `$LISTTOSTRING` para exibir.

**2) Usar `$LIST` numa posição que pode não existir.**
Gera erro. Use `$LISTGET`, que devolve vazio ou um padrão.

**3) Confundir `$LIST(l, n)` com `$LIST(l, de, ate)`.**
O primeiro devolve o **valor** do elemento; o segundo devolve uma **sublista**.

**4) Achar que elemento sem valor e elemento vazio são a mesma coisa.**
`$LISTLENGTH` e `$LISTGET` não distinguem; `$LISTDATA` distingue.

**5) Esquecer o `$GET` no valor devolvido por `$LISTNEXT`.**
Elementos sem valor deixam a variável indefinida.

**6) Achar que `$LISTFIND` devolve verdadeiro/falso.**
Devolve a **posição**, ou `0` se não achar.

**7) Começar o ponteiro do `$LISTNEXT` em 1.**
Ele começa em **0**.

**8) Esperar que `$LISTTOSTRING` seguido de `$LISTFROMSTRING` devolva a lista original.**
Se algum elemento contiver o separador, não devolve. A ida e volta por texto não é segura.

**9) Usar lista onde o dado vem de fora com separadores.**
Na fronteira, converta: `$LISTFROMSTRING` na entrada, `$LISTTOSTRING` na saída.

**10) Guardar como `%List` um dado que precisa ser consultado por elemento.**
O SQL não enxerga dentro da lista. Use `list Of` ou uma classe própria.

**11) Confundir `%List` com `list Of`.**
O primeiro é um tipo de dado lido com `$LIST`; o segundo é uma coleção de objetos lida com `GetAt`.

**12) Confundir o índice inicial das três estruturas.**
`%List` e `list Of` começam em **1**; `%DynamicArray` do JSON começa em **0**.

**13) Usar lista como subscrito de array.**
A ordenação vira binária e ilegível. Use subscritos múltiplos.

**14) Esquecer as bordas ao inserir ou remover por concatenação.**
`$LIST(l, 1, 0)` é uma faixa inválida. Trate o primeiro e o último elemento explicitamente.

**15) Passar a `$LIST` um valor que não é lista.**
Erro em execução. Valide com `$LISTVALID` quando o dado vier de fora.

---

## 7. MÃO NA MASSA

---

### Exercício 14.1 — O básico

**a) Enunciado:** Crie `LabStudy.Demo.List1` com:

1. `ClassMethod Build()` — cria uma lista com o código, o valor, a unidade e o estado de um exame; imprime comprimento, cada elemento, o último por posição negativa, e a lista como texto.
2. `ClassMethod Safe()` — demonstra a diferença entre `$LIST` e `$LISTGET` numa posição inexistente. Provoque o erro do `$LIST` de propósito e observe a mensagem.
3. `ClassMethod Find()` — procura três valores: um que existe, um que não existe, e um que existe duas vezes (encontre **as duas** posições).
4. `ClassMethod Compare()` — cria duas listas iguais e duas diferentes, comparando com `$LISTSAME` e com `=`.

**b) Dica:** Para achar todas as ocorrências, chame `$LISTFIND` em laço, passando a última posição encontrada como terceiro argumento.

**c) Como testar:** No item 2, o `$LIST` fora de faixa deve interromper a execução com erro.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/List1.cls`:

```objectscript
/// Basic $LIST operations.
Class LabStudy.Demo.List1 Extends %RegisteredObject
{

ClassMethod Build() As %Status
{
    set exam = $LISTBUILD("GLU", 92, "mg/dL", "final")

    write "length : ", $LISTLENGTH(exam), !
    for i = 1:1:$LISTLENGTH(exam) {
        write "  ", i, ": ", $LIST(exam, i), !
    }
    write "last   : ", $LIST(exam, -1), !
    write "as text: ", $LISTTOSTRING(exam, " | "), !

    quit $$$OK
}

ClassMethod Safe() As %Status
{
    set exam = $LISTBUILD("GLU", 92, "mg/dL")

    write "$LISTGET(exam, 9)        : [", $LISTGET(exam, 9), "]", !
    write "$LISTGET(exam, 9, 'n/a') : [", $LISTGET(exam, 9, "n/a"), "]", !

    write !, "now the unsafe version, which will fail:", !
    write "$LIST(exam, 9) : ", $LIST(exam, 9), !

    write "this line is never reached", !
    quit $$$OK
}

ClassMethod Find() As %Status
{
    set codes = $LISTBUILD("HGB", "GLU", "CHOL", "GLU", "TRIG")

    write "position of CHOL : ", $LISTFIND(codes, "CHOL"), !
    write "position of XXX  : ", $LISTFIND(codes, "XXX"), !

    write "all positions of GLU:", !
    set pos = 0
    for {
        set pos = $LISTFIND(codes, "GLU", pos)
        quit:pos=0
        write "  found at ", pos, !
    }

    quit $$$OK
}

ClassMethod Compare() As %Status
{
    set a = $LISTBUILD("x", "y", "z")
    set b = $LISTBUILD("x", "y", "z")
    set c = $LISTBUILD("x", "y")

    write "a vs b, $LISTSAME : ", $LISTSAME(a, b), !
    write "a vs b, =         : ", (a = b), !
    write "a vs c, $LISTSAME : ", $LISTSAME(a, c), !
    write "a vs c, =         : ", (a = c), !

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Build()   write !
    do ..Find()    write !
    do ..Compare() write !
    write "run Safe() separately: it ends with an error on purpose", !
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.List1).Demo()
length : 4
  1: GLU
  2: 92
  3: mg/dL
  4: final
last   : final
as text: GLU | 92 | mg/dL | final

position of CHOL : 3
position of XXX  : 0
all positions of GLU:
  found at 2
  found at 4

a vs b, $LISTSAME : 1
a vs b, =         : 1
a vs c, $LISTSAME : 0
a vs c, =         : 0

run Safe() separately: it ends with an error on purpose

LABSTUDY>DO ##class(LabStudy.Demo.List1).Safe()
$LISTGET(exam, 9)        : []
$LISTGET(exam, 9, 'n/a') : [n/a]

now the unsafe version, which will fail:
$LIST(exam, 9) :
DO ##class(LabStudy.Demo.List1).Safe()
^
<LIST>Safe+8^LabStudy.Demo.List1.1
```

**Por que cada decisão:**

- **O erro do `$LIST` fora de faixa foi provocado de propósito**, e a última linha do método nunca foi alcançada. Isso deixa claro que não se trata de um valor esquisito devolvido: é uma **interrupção**. Num método de produção sem tratamento de erro, isso derruba a operação inteira.
- **A busca de todas as ocorrências usa a própria posição encontrada como ponto de partida seguinte.** Como `$LISTFIND` procura **depois** da posição informada, passar `pos` avança naturalmente. Passar `pos + 1` seria um erro sutil que pularia ocorrências consecutivas — não aqui, mas em outros contextos.
- **`$LISTSAME` e `=` deram o mesmo resultado** neste caso, porque as listas foram construídas do mesmo jeito. `$LISTSAME` é a comparação correta a usar, porque compara logicamente; `=` compara bytes e pode divergir quando listas equivalentes foram montadas por caminhos diferentes.
- **O laço de `Build` usa `$LIST(exam, i)` sem `$LISTGET`** porque `i` vem de `1` até `$LISTLENGTH` — está garantidamente dentro da faixa. Usar `$LISTGET` ali seria defensivismo desnecessário. **Saber quando não precisa é tão importante quanto saber quando precisa.**

---

### Exercício 14.2 — Por que `$LIST` existe

**a) Enunciado:** Escreva `LabStudy.Demo.List2` que prova, com dados realistas, que separadores quebram e listas não:

1. `ClassMethod PipeProblem()` — monta um registro de paciente com nome, cidade e observação usando `|`; use uma observação que **contenha** `|`. Mostre a contagem de pedaços e o conteúdo recuperado.
2. `ClassMethod CommaProblem()` — o mesmo com vírgula e um endereço `"Rua das Flores, 100"`.
3. `ClassMethod ListSolution()` — os mesmos dois casos com `$LISTBUILD`, provando que nada quebra.
4. `ClassMethod AnySeparator()` — tente encontrar um separador "seguro" e mostre que qualquer um pode aparecer: teste com tabulação, com `~`, e com `$CHAR(1)`.
5. `ClassMethod NestedProof()` — guarde uma lista dentro de outra e recupere ambas, provando que aninhamento não precisa de escape.

**b) Dica:** `$CHAR(9)` é tabulação; `$CHAR(1)` é um caractere de controle.

**c) Como testar:** O item 4 deve mostrar que mesmo `$CHAR(1)` pode aparecer em dados vindos de fora.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/List2.cls`:

```objectscript
/// Proves why $LIST exists.
Class LabStudy.Demo.List2 Extends %RegisteredObject
{

ClassMethod PipeProblem() As %Status
{
    set note = "Trouxe exames de 2024|2025"
    set record = "Maria Silva|Potirendaba|"_note

    write "-- pipe separated --", !
    write "  expected 3 fields, got : ", $LENGTH(record, "|"), !
    write "  field 3 recovered      : ", $PIECE(record, "|", 3), !
    write "  original note          : ", note, !
    write "  lost?                  : ", $SELECT($PIECE(record,"|",3)=note: "no", 1: "YES"), !

    quit $$$OK
}

ClassMethod CommaProblem() As %Status
{
    set street = "Rua das Flores, 100"
    set record = street_","_"Potirendaba"_","_"SP"

    write "-- comma separated --", !
    write "  expected 3 fields, got : ", $LENGTH(record, ","), !
    write "  field 1 recovered      : ", $PIECE(record, ",", 1), !
    write "  field 2 recovered      : ", $PIECE(record, ",", 2), "   <-- wrong", !
    write "  original street        : ", street, !

    quit $$$OK
}

ClassMethod ListSolution() As %Status
{
    set note = "Trouxe exames de 2024|2025"
    set street = "Rua das Flores, 100"

    set r1 = $LISTBUILD("Maria Silva", "Potirendaba", note)
    set r2 = $LISTBUILD(street, "Potirendaba", "SP")

    write "-- as lists --", !
    write "  r1 length : ", $LISTLENGTH(r1), !
    write "  r1 note   : ", $LIST(r1, 3), !
    write "  r1 intact : ", $SELECT($LIST(r1,3)=note: "yes", 1: "no"), !
    write "  r2 length : ", $LISTLENGTH(r2), !
    write "  r2 street : ", $LIST(r2, 1), !
    write "  r2 intact : ", $SELECT($LIST(r2,1)=street: "yes", 1: "no"), !

    quit $$$OK
}

ClassMethod AnySeparator() As %Status
{
    // data that arrived from an external system, already containing control chars
    set nasty = "linha1"_$CHAR(9)_"linha2"_$CHAR(1)_"fim~agora"

    write "-- trying to find a 'safe' separator --", !

    for sep = "|", ",", ";", "~", $CHAR(9), $CHAR(1) {
        set label = $SELECT(sep=$CHAR(9): "TAB", sep=$CHAR(1): "CHAR(1)", 1: sep)
        write "  separator [", label, "] : "
        write $SELECT(nasty[sep: "APPEARS in the data", 1: "not present"), !
    }

    write !, "  with a list, the question does not arise", !
    set safe = $LISTBUILD("campo1", nasty, "campo3")
    write "  length : ", $LISTLENGTH(safe), !
    write "  intact : ", $SELECT($LIST(safe,2)=nasty: "yes", 1: "no"), !

    quit $$$OK
}

ClassMethod NestedProof() As %Status
{
    set inner = $LISTBUILD("a|b", "c,d", "e;f")
    set outer = $LISTBUILD("antes", inner, "depois")

    write "-- nesting with no escaping --", !
    write "  outer length     : ", $LISTLENGTH(outer), !
    write "  element 2 is list: ", $LISTVALID($LIST(outer, 2)), !

    set recovered = $LIST(outer, 2)
    write "  inner length     : ", $LISTLENGTH(recovered), !
    for i = 1:1:$LISTLENGTH(recovered) {
        write "    ", i, ": [", $LIST(recovered, i), "]", !
    }
    write "  identical?       : ", $LISTSAME(inner, recovered), !

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..PipeProblem()   write !
    do ..CommaProblem()  write !
    do ..ListSolution()  write !
    do ..AnySeparator()  write !
    do ..NestedProof()
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.List2).Demo()
-- pipe separated --
  expected 3 fields, got : 4
  field 3 recovered      : Trouxe exames de 2024
  original note          : Trouxe exames de 2024|2025
  lost?                  : YES

-- comma separated --
  expected 3 fields, got : 4
  field 1 recovered      : Rua das Flores
  field 2 recovered      :  100   <-- wrong
  original street        : Rua das Flores, 100

-- as lists --
  r1 length : 3
  r1 note   : Trouxe exames de 2024|2025
  r1 intact : yes
  r2 length : 3
  r2 street : Rua das Flores, 100
  r2 intact : yes

-- trying to find a 'safe' separator --
  separator [|] : not present
  separator [,] : not present
  separator [;] : not present
  separator [~] : APPEARS in the data
  separator [TAB] : APPEARS in the data
  separator [CHAR(1)] : APPEARS in the data

  with a list, the question does not arise
  length : 3
  intact : yes

-- nesting with no escaping --
  outer length     : 3
  element 2 is list: 1
  inner length     : 3
    1: [a|b]
    2: [c,d]
    3: [e;f]
  identical?       : 1
```

**Por que cada resultado:**

- **A observação com `|` produziu 4 campos onde deveriam ser 3, e o terceiro foi truncado.** Nenhum erro. O programa segue funcionando e entregando dado errado — o pior tipo de falha.
- **A vírgula no endereço quebrou o campo 1 e deslocou todos os outros.** Repare que o campo 2 virou `" 100"`, com espaço à frente. Se esse valor fosse gravado como cidade, ninguém notaria por meses.
- **`AnySeparator` mata a discussão sobre "escolher um separador seguro".** O til, a tabulação e até `$CHAR(1)` apareceram no dado. Você pode reduzir a probabilidade escolhendo caracteres exóticos, mas **não pode eliminá-la** — e sistemas que dependem de improbabilidade falham em produção, não em teste.
- **O aninhamento funcionou com elementos contendo `|`, `,` e `;` ao mesmo tempo.** Numa estrutura com separadores, seria necessário escapar todos eles, em dois níveis, e desescapar na leitura — código que ninguém acerta na primeira tentativa e que quebra quando alguém acrescenta um nível.
- **`$LISTSAME(inner, recovered)` devolveu 1**, provando que a lista interna atravessou o aninhamento sem perder um byte.

---

### Exercício 14.3 — Modificando listas

**a) Enunciado:** Crie `LabStudy.Demo.List3`, um pequeno conjunto de utilitários que trate corretamente **todas as bordas**:

1. `ClassMethod Insert(lista, posicao, valor) As %List` — insere antes da posição indicada; deve funcionar na posição 1, no meio e depois do fim.
2. `ClassMethod Remove(lista, posicao) As %List` — remove; deve funcionar no primeiro, no meio e no último.
3. `ClassMethod Replace(lista, posicao, valor) As %List` — substitui.
4. `ClassMethod Swap(lista, a, b) As %List` — troca dois elementos de posição.
5. `ClassMethod Reverse(lista) As %List` — inverte a ordem.
6. `ClassMethod Unique(lista) As %List` — remove duplicatas, preservando a primeira ocorrência.
7. `ClassMethod Sort(lista) As %List` — devolve a lista ordenada, usando o idioma do Capítulo 13.

Teste cada um nos casos de borda.

**b) Dica:** Para o item 7, jogue os elementos num array usando o valor como subscrito, e reconstrua percorrendo.

**c) Como testar:** `Insert` na posição 1 e `Remove` da posição 1 são as bordas que quebram implementações ingênuas.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/List3.cls`:

```objectscript
/// List manipulation utilities, with all the edge cases handled.
Class LabStudy.Demo.List3 Extends %RegisteredObject
{

/// Inserts value before the given position.
ClassMethod Insert(list As %List, position As %Integer, value As %String) As %List
{
    set len = $LISTLENGTH(list)

    if position <= 1 {
        quit $LISTBUILD(value) _ list
    }
    if position > len {
        quit list _ $LISTBUILD(value)
    }

    quit $LIST(list, 1, position - 1) _ $LISTBUILD(value) _ $LIST(list, position, *)
}

/// Removes the element at the given position.
ClassMethod Remove(list As %List, position As %Integer) As %List
{
    set len = $LISTLENGTH(list)

    if (position < 1) || (position > len) {
        quit list
    }
    if len = 1 {
        quit ""
    }
    if position = 1 {
        quit $LIST(list, 2, *)
    }
    if position = len {
        quit $LIST(list, 1, len - 1)
    }

    quit $LIST(list, 1, position - 1) _ $LIST(list, position + 1, *)
}

/// Replaces the element at the given position.
ClassMethod Replace(list As %List, position As %Integer, value As %String) As %List
{
    if (position < 1) || (position > $LISTLENGTH(list)) {
        quit list
    }

    set $LIST(list, position) = value
    quit list
}

/// Swaps two positions.
ClassMethod Swap(list As %List, a As %Integer, b As %Integer) As %List
{
    set len = $LISTLENGTH(list)
    if (a < 1) || (b < 1) || (a > len) || (b > len) || (a = b) {
        quit list
    }

    set va = $LISTGET(list, a)
    set vb = $LISTGET(list, b)

    set $LIST(list, a) = vb
    set $LIST(list, b) = va
    quit list
}

/// Reverses the order of the elements.
ClassMethod Reverse(list As %List) As %List
{
    set result = ""
    for i = $LISTLENGTH(list):-1:1 {
        set result = result _ $LISTBUILD($LISTGET(list, i))
    }
    quit result
}

/// Removes duplicates, keeping the first occurrence.
ClassMethod Unique(list As %List) As %List
{
    kill seen
    set result = ""
    set ptr = 0

    while $LISTNEXT(list, ptr, value) {
        set value = $GET(value)
        continue:$DATA(seen(value))

        set seen(value) = ""
        set result = result _ $LISTBUILD(value)
    }

    quit result
}

/// Sorts using the inverted index idiom from chapter 13.
ClassMethod Sort(list As %List, descending As %Boolean = 0) As %List
{
    kill idx

    set ptr = 0, n = 0
    while $LISTNEXT(list, ptr, value) {
        set n = n + 1
        set idx($GET(value), n) = ""       // value first, order second (ties)
    }

    set direction = $SELECT(descending: -1, 1: 1)
    set result = ""
    set v = ""

    for {
        set v = $ORDER(idx(v), direction)
        quit:v=""

        set k = ""
        for {
            set k = $ORDER(idx(v, k))
            quit:k=""
            set result = result _ $LISTBUILD(v)
        }
    }

    quit result
}

/// Prints a list in a readable way.
ClassMethod Show(label As %String, list As %List) As %Status
{
    write $JUSTIFY(label, 14), " : [", $LISTTOSTRING(list, ","), "]  (",
          $LISTLENGTH(list), " elements)", !
    quit $$$OK
}

ClassMethod Demo() As %Status
{
    set base = $LISTBUILD("a", "b", "c", "d")
    do ..Show("original", base)

    write !, "-- insert --", !
    do ..Show("at 1", ..Insert(base, 1, "X"))
    do ..Show("at 3", ..Insert(base, 3, "X"))
    do ..Show("at 99", ..Insert(base, 99, "X"))

    write !, "-- remove --", !
    do ..Show("first", ..Remove(base, 1))
    do ..Show("middle", ..Remove(base, 3))
    do ..Show("last", ..Remove(base, 4))
    do ..Show("out of range", ..Remove(base, 99))
    do ..Show("single element", ..Remove($LISTBUILD("only"), 1))

    write !, "-- others --", !
    do ..Show("replace 2", ..Replace(base, 2, "B"))
    do ..Show("swap 1,4", ..Swap(base, 1, 4))
    do ..Show("reverse", ..Reverse(base))

    set dup = $LISTBUILD("b", "a", "c", "a", "b", "a")
    do ..Show("with dups", dup)
    do ..Show("unique", ..Unique(dup))
    do ..Show("sorted", ..Sort(dup))
    do ..Show("sorted desc", ..Sort(dup, 1))

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.List3).Demo()
      original : [a,b,c,d]  (4 elements)

-- insert --
          at 1 : [X,a,b,c,d]  (5 elements)
          at 3 : [a,b,X,c,d]  (5 elements)
         at 99 : [a,b,c,d,X]  (5 elements)

-- remove --
         first : [b,c,d]  (3 elements)
        middle : [a,b,d]  (3 elements)
          last : [a,b,c]  (3 elements)
  out of range : [a,b,c,d]  (4 elements)
single element : []  (0 elements)

-- others --
     replace 2 : [a,B,c,d]  (4 elements)
      swap 1,4 : [d,b,c,a]  (4 elements)
       reverse : [d,c,b,a]  (4 elements)
     with dups : [b,a,c,a,b,a]  (6 elements)
        unique : [b,a,c]  (3 elements)
        sorted : [a,a,a,b,b,c]  (6 elements)
   sorted desc : [c,b,b,a,a,a]  (6 elements)
```

**Por que cada decisão:**

- **`Insert` trata três casos antes de chegar à expressão geral.** A posição 1 não tem parte esquerda, e `$LIST(list, 1, 0)` seria inválido. A posição além do fim não tem parte direita. Só o caso do meio usa a concatenação tripla. **Implementações que só escrevem o caso do meio funcionam nos testes e quebram no primeiro elemento.**
- **`Remove` trata quatro casos**, incluindo lista de um elemento só (que vira lista vazia) e posição fora de faixa (que devolve a lista intacta em vez de falhar). Devolver a entrada inalterada quando a operação não faz sentido é mais defensável do que gerar erro num utilitário desse tipo.
- **`Swap` usa `$LISTGET`**, não `$LIST` — embora as posições já tenham sido validadas. Aqui é preferência por robustez num utilitário público; num laço interno controlado, `$LIST` seria a escolha por desempenho.
- **`Reverse` percorre com `for i = n:-1:1`** — a forma `início:passo:fim` com passo negativo. Note que não dá para usar `$LISTNEXT` aqui, porque ele só anda para frente.
- **`Unique` usa um array local como conjunto**, com `$DATA(seen(value))` para testar pertinência. É o idioma padrão para "já vi isto?" em ObjectScript, e é O(1) por consulta graças à árvore ordenada do Capítulo 13.
- **`Sort` é o Capítulo 13 aplicado a listas.** Repare no segundo subscrito `n`: sem ele, os três `"a"` duplicados colapsariam num só e a lista ordenada teria 3 elementos em vez de 6. A saída confirma que os seis sobreviveram.
- **`Show` centraliza a formatação**, e é o que torna a saída do `Demo` comparável linha a linha. Ferramentas de inspeção bem feitas economizam mais tempo do que parecem.

---

### Exercício 14.4 — Ida e volta com o mundo externo

**a) Enunciado:** Crie `LabStudy.Demo.List4` que lida com CSV, o formato externo mais comum:

1. `ClassMethod ParseLine(linha, separador) As %List` — converte uma linha de texto em lista.
2. `ClassMethod FormatLine(lista, separador) As %String` — converte de volta.
3. `ClassMethod RoundTripTest()` — testa a ida e volta com dados seguros e com dados que contêm o separador, mostrando onde ela falha.
4. `ClassMethod ParseQuoted(linha) As %List` — versão que respeita aspas, como CSV de verdade: `a,"b,c",d` deve produzir **três** elementos.
5. `ClassMethod ImportBlock(texto) As %Integer` — recebe um bloco de várias linhas separadas por quebra de linha, converte cada uma em lista, e devolve quantas foram processadas, imprimindo o resultado.

**b) Dica:** No item 4, percorra caractere a caractere controlando se está dentro ou fora de aspas. `$EXTRACT(texto, i)` pega o i-ésimo caractere.

**c) Como testar:** `ParseQuoted("a,""b,c"",d")` deve devolver 3 elementos, com o segundo sendo `b,c`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/List4.cls`:

```objectscript
/// Converting between lists and the delimited text of the outside world.
Class LabStudy.Demo.List4 Extends %RegisteredObject
{

/// Naive parse: splits on the separator.
ClassMethod ParseLine(line As %String, separator As %String = ",") As %List
{
    quit $LISTFROMSTRING(line, separator)
}

/// Naive format: joins with the separator.
ClassMethod FormatLine(list As %List, separator As %String = ",") As %String
{
    quit $LISTTOSTRING(list, separator)
}

/// Shows where the round trip is safe and where it is not.
ClassMethod RoundTripTest() As %Status
{
    write "-- safe data --", !
    set original = $LISTBUILD("HGB", "13.5", "g/dL")
    set text = ..FormatLine(original)
    set back = ..ParseLine(text)

    write "  original : ", $LISTLENGTH(original), " elements", !
    write "  text     : ", text, !
    write "  back     : ", $LISTLENGTH(back), " elements", !
    write "  same     : ", $LISTSAME(original, back), !

    write !, "-- unsafe data --", !
    set original2 = $LISTBUILD("HGB", "13,5", "g/dL")
    set text2 = ..FormatLine(original2)
    set back2 = ..ParseLine(text2)

    write "  original : ", $LISTLENGTH(original2), " elements", !
    write "  text     : ", text2, !
    write "  back     : ", $LISTLENGTH(back2), " elements   <-- broken", !
    write "  same     : ", $LISTSAME(original2, back2), !

    quit $$$OK
}

/// Real CSV parse: respects double quotes around fields.
ClassMethod ParseQuoted(line As %String, separator As %String = ",") As %List
{
    set result = ""
    set field = ""
    set inQuotes = 0
    set len = $LENGTH(line)

    for i = 1:1:len {
        set ch = $EXTRACT(line, i)

        if ch = """" {
            // a doubled quote inside a quoted field is a literal quote
            if inQuotes, $EXTRACT(line, i + 1) = """" {
                set field = field_""""
                set i = i + 1
                continue
            }
            set inQuotes = 'inQuotes
            continue
        }

        if (ch = separator) && 'inQuotes {
            set result = result_$LISTBUILD(field)
            set field = ""
            continue
        }

        set field = field_ch
    }

    set result = result_$LISTBUILD(field)
    quit result
}

/// Parses a block of lines.
ClassMethod ImportBlock(text As %String, separator As %String = ",") As %Integer
{
    set lineCount = $LENGTH(text, $CHAR(10))
    set imported = 0

    write "-- importing ", lineCount, " lines --", !

    for i = 1:1:lineCount {
        set line = $ZSTRIP($PIECE(text, $CHAR(10), i), "<>W")
        continue:line=""

        set row = ..ParseQuoted(line, separator)
        set imported = imported + 1

        write "  line ", i, " -> ", $LISTLENGTH(row), " fields:", !
        for f = 1:1:$LISTLENGTH(row) {
            write "      ", f, ": [", $LISTGET(row, f), "]", !
        }
    }

    quit imported
}

ClassMethod Demo() As %Status
{
    do ..RoundTripTest()

    write !, "-- quoted parse --", !
    set line = "a,""b,c"",d"
    set parsed = ..ParseQuoted(line)
    write "  input  : ", line, !
    write "  fields : ", $LISTLENGTH(parsed), !
    for i = 1:1:$LISTLENGTH(parsed) {
        write "      ", i, ": [", $LISTGET(parsed, i), "]", !
    }

    write !
    set block = "HGB,13.5,g/dL"_$CHAR(10)
                _"GLU,""92,0"",mg/dL"_$CHAR(10)
                _"CHOL,190,mg/dL"
    write ..ImportBlock(block), " lines imported", !

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.List4).Demo()
-- safe data --
  original : 3 elements
  text     : HGB,13.5,g/dL
  back     : 3 elements
  same     : 1

-- unsafe data --
  original : 3 elements
  text     : HGB,13,5,g/dL
  back     : 4 elements   <-- broken
  same     : 0

-- quoted parse --
  input  : a,"b,c",d
  fields : 3
      1: [a]
      2: [b,c]
      3: [d]

-- importing 3 lines --
  line 1 -> 3 fields:
      1: [HGB]
      2: [13.5]
      3: [g/dL]
  line 2 -> 3 fields:
      1: [GLU]
      2: [92,0]
      3: [mg/dL]
  line 3 -> 3 fields:
      1: [CHOL]
      2: [190]
      3: [mg/dL]
3 lines imported
```

**Por que cada decisão:**

- **O teste de ida e volta usa um dado realista brasileiro**: `"13,5"` com vírgula decimal. Esse é exatamente o caso que quebra importações de CSV no Brasil todo dia. A lista original tinha 3 elementos e voltou com 4.
- **`ParseQuoted` implementa a regra real do CSV**: aspas delimitam campos, e aspas duplicadas dentro de um campo entre aspas representam uma aspa literal. É por isso que o formato CSV "de verdade" é mais complicado do que `$LISTFROMSTRING` — e por isso vale a pena converter para lista **assim que possível** e nunca mais voltar.
- **A alteração de `i` dentro do laço `for`** (no tratamento da aspa duplicada) merece atenção: o ObjectScript permite, mas é uma prática que exige cuidado, porque o laço com passo definido pode se comportar de forma inesperada em algumas construções. Aqui funciona porque avançamos exatamente um caractere já consumido.
- **`set inQuotes = 'inQuotes`** alterna o valor com o operador de negação. É idiomático e compacto.
- **`ImportBlock` usa `$CHAR(10)` como separador de linhas** e `$ZSTRIP` para remover retorno de carro e espaços. Arquivos vindos de sistemas diferentes trazem terminadores diferentes, e limpar na entrada evita campos com lixo invisível no fim.
- **A linha 2 do bloco tinha `"92,0"` entre aspas e foi preservada corretamente.** É a prova de que `ParseQuoted` resolve o caso que `ParseLine` quebra.
- **O `continue:line=""`** ignora linhas em branco, comuns no fim de arquivos.
- **A lição estrutural:** toda a complexidade do CSV ficou confinada em **um método**, na fronteira. Do `ParseQuoted` para dentro, o sistema só vê listas, e o problema deixou de existir.

---

### Exercício 14.5 — PROJETO CONTÍNUO: listas no laboratório

**a) Enunciado:** No Capítulo 13 usamos o separador `|` para transportar vários campos num nó do índice — exatamente o antipadrão que este capítulo combate. Corrija isso e amplie:

1. Crie `LabStudy.ListUtil` com utilitários de uso geral do projeto:
   - `Insert`, `Remove`, `Replace`, `Unique`, `Sort` (reaproveite o exercício 14.3);
   - `ClassMethod ToDisplay(lista, separador)` — versão segura de exibição, que substitui o separador se ele aparecer no conteúdo;
   - `ClassMethod FromCsv(linha)` e `ClassMethod ToCsv(lista)` — com tratamento de aspas.
2. Em `LabStudy.Exam`, acrescente:
   - `Property ReferenceRange As %List` — faixa de referência como `$LB(minimo, maximo, unidade)`;
   - `Method SetReferenceRange(min, max, unit) As %Status`;
   - `Method IsAbnormal() As %Boolean` — agora **de verdade**, comparando o resultado com a faixa (o Capítulo 3 deixou um método que sempre devolvia zero);
   - `Method RangeText() As %String` — exibição da faixa, ou `(sem faixa)`.
3. Reescreva `LabStudy.RankReport` para transportar os campos em **listas** em vez de texto com `|`.
4. Em `LabStudy.Reports`, acrescente `ClassMethod AbnormalByRange()` — lista os exames fora da faixa de referência.
5. Crie `LabStudy.Demo.RangeSeed` que preenche faixas de referência para os códigos conhecidos.
6. Suba `LabStudy.App` para `"1.5"`.

**b) Dica:** No item 2, cuidado com o Capítulo 10: faixa não definida e resultado não informado são casos distintos, e nenhum dos dois significa "anormal".

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Demo.RangeSeed).Run()
LABSTUDY>DO ##class(LabStudy.Reports).AbnormalByRange()
LABSTUDY>DO ##class(LabStudy.RankReport).ByExamCount(5)
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/ListUtil.cls`:

```objectscript
/// General purpose list helpers for the LabStudy system.
/// Inside the system we carry fields in lists, never in delimited strings.
Class LabStudy.ListUtil Extends %RegisteredObject
{

/// Inserts value before the given position.
ClassMethod Insert(list As %List, position As %Integer, value As %String) As %List
{
    set len = $LISTLENGTH(list)
    if position <= 1 { quit $LISTBUILD(value) _ list }
    if position > len { quit list _ $LISTBUILD(value) }
    quit $LIST(list, 1, position - 1) _ $LISTBUILD(value) _ $LIST(list, position, *)
}

/// Removes the element at the given position.
ClassMethod Remove(list As %List, position As %Integer) As %List
{
    set len = $LISTLENGTH(list)
    if (position < 1) || (position > len) { quit list }
    if len = 1 { quit "" }
    if position = 1 { quit $LIST(list, 2, *) }
    if position = len { quit $LIST(list, 1, len - 1) }
    quit $LIST(list, 1, position - 1) _ $LIST(list, position + 1, *)
}

/// Replaces the element at the given position.
ClassMethod Replace(list As %List, position As %Integer, value As %String) As %List
{
    if (position < 1) || (position > $LISTLENGTH(list)) { quit list }
    set $LIST(list, position) = value
    quit list
}

/// Removes duplicates, keeping the first occurrence.
ClassMethod Unique(list As %List) As %List
{
    kill seen
    set result = "", ptr = 0

    while $LISTNEXT(list, ptr, value) {
        set value = $GET(value)
        continue:$DATA(seen(value))
        set seen(value) = ""
        set result = result _ $LISTBUILD(value)
    }
    quit result
}

/// Sorts the list, ascending or descending.
ClassMethod Sort(list As %List, descending As %Boolean = 0) As %List
{
    kill idx
    set ptr = 0, n = 0

    while $LISTNEXT(list, ptr, value) {
        set n = n + 1
        set idx($GET(value), n) = ""
    }

    set direction = $SELECT(descending: -1, 1: 1)
    set result = "", v = ""

    for {
        set v = $ORDER(idx(v), direction)
        quit:v=""
        set k = ""
        for {
            set k = $ORDER(idx(v, k))
            quit:k=""
            set result = result _ $LISTBUILD(v)
        }
    }
    quit result
}

/// Display only. Replaces the separator inside values so the output
/// never lies about how many fields there are.
ClassMethod ToDisplay(list As %List, separator As %String = " | ") As %String
{
    set out = "", ptr = 0, first = 1

    while $LISTNEXT(list, ptr, value) {
        set value = $GET(value)
        set value = $REPLACE(value, separator, " ")

        set out = out_$SELECT(first: "", 1: separator)_value
        set first = 0
    }
    quit out
}

/// Real CSV parse, respecting double quotes.
ClassMethod FromCsv(line As %String, separator As %String = ",") As %List
{
    set result = "", field = "", inQuotes = 0

    for i = 1:1:$LENGTH(line) {
        set ch = $EXTRACT(line, i)

        if ch = """" {
            if inQuotes, $EXTRACT(line, i + 1) = """" {
                set field = field_""""
                set i = i + 1
                continue
            }
            set inQuotes = 'inQuotes
            continue
        }

        if (ch = separator) && 'inQuotes {
            set result = result_$LISTBUILD(field)
            set field = ""
            continue
        }

        set field = field_ch
    }

    quit result_$LISTBUILD(field)
}

/// CSV output, quoting fields that need it.
ClassMethod ToCsv(list As %List, separator As %String = ",") As %String
{
    set out = "", ptr = 0, first = 1

    while $LISTNEXT(list, ptr, value) {
        set value = $GET(value)

        if (value [ separator) || (value [ """") || (value [ $CHAR(10)) {
            set value = """"_$REPLACE(value, """", """""")_""""
        }

        set out = out_$SELECT(first: "", 1: separator)_value
        set first = 0
    }
    quit out
}

}
```

Acrescente a `src/LabStudy/Exam.cls`:

```objectscript
/// Reference range as $LB(minimum, maximum, unit).
/// Stored as a single value: it is always read as a whole.
Property ReferenceRange As %List;

/// Defines the reference range of this exam.
Method SetReferenceRange(min As %Numeric, max As %Numeric, unit As %String = "") As %Status
{
    if (min = "") || (max = "") {
        quit $$$ERROR($$$GeneralError, "Both minimum and maximum are required")
    }
    if min > max {
        quit $$$ERROR($$$GeneralError, "Minimum cannot be greater than maximum")
    }

    set ..ReferenceRange = $LISTBUILD(min, max, unit)
    quit $$$OK
}

/// True only when there IS a final result AND a range AND the value is outside it.
/// Missing range or missing result are not abnormal: they are unknown.
Method IsAbnormal() As %Boolean
{
    quit:'..HasResult() 0
    quit:..ResultValue="" 0
    quit:'$LISTVALID(..ReferenceRange) 0
    quit:$LISTLENGTH(..ReferenceRange)<2 0

    set min = $LISTGET(..ReferenceRange, 1)
    set max = $LISTGET(..ReferenceRange, 2)
    quit:(min="")||(max="") 0

    quit (..ResultValue < min) || (..ResultValue > max)
}

/// Readable reference range.
Method RangeText() As %String
{
    quit:'$LISTVALID(..ReferenceRange) "(sem faixa)"
    quit:$LISTLENGTH(..ReferenceRange)<2 "(sem faixa)"

    set min = $LISTGET(..ReferenceRange, 1)
    set max = $LISTGET(..ReferenceRange, 2)
    set unit = $LISTGET(..ReferenceRange, 3)

    quit:(min="")||(max="") "(sem faixa)"

    quit min_" - "_max_$SELECT(unit'="": " "_unit, 1: "")
}
```

`src/LabStudy/Demo/RangeSeed.cls`:

```objectscript
/// Fills reference ranges for the known test codes.
Class LabStudy.Demo.RangeSeed Extends %RegisteredObject
{

/// code -> $LB(min, max, unit)
ClassMethod Table(Output ranges) As %Status
{
    kill ranges

    set ranges("HGB")  = $LISTBUILD(12, 16, "g/dL")
    set ranges("GLU")  = $LISTBUILD(70, 99, "mg/dL")
    set ranges("CHOL") = $LISTBUILD(0, 190, "mg/dL")
    set ranges("TRIG") = $LISTBUILD(0, 150, "mg/dL")
    set ranges("UREA") = $LISTBUILD(15, 45, "mg/dL")
    set ranges("CREA") = $LISTBUILD(0.6, 1.3, "mg/dL")

    quit $$$OK
}

/// Applies the table to every exam that has no range yet.
ClassMethod Run() As %Integer
{
    do ..Table(.ranges)

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id, TestCode AS Code FROM LabStudy.EXAM")

    set applied = 0, skipped = 0

    while rs.%Next() {
        set code = rs.%Get("Code")

        if '$DATA(ranges(code)) {
            set skipped = skipped + 1
            continue
        }

        set exam = ##class(LabStudy.Exam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)

        // idempotent: do not overwrite a range that is already there
        continue:$LISTVALID(exam.ReferenceRange)&&($LISTLENGTH(exam.ReferenceRange)>=2)

        set range = ranges(code)
        set sc = exam.SetReferenceRange($LIST(range, 1), $LIST(range, 2), $LIST(range, 3))
        continue:$$$ISERR(sc)

        set sc = exam.%Save()
        continue:$$$ISERR(sc)

        set applied = applied + 1
    }

    write applied, " ranges applied, ", skipped, " exams with unknown code", !
    quit applied
}

}
```

Reescreva o `ByExamCount` de `src/LabStudy/RankReport.cls` usando listas:

```objectscript
/// The n patients with the most exams. Fields carried in a list, not in text.
ClassMethod ByExamCount(n As %Integer = 10) As %Status
{
    kill idx

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT p.%ID AS Id, p.Name AS Name, p.RecordNumber AS Rec, "
        _"(SELECT COUNT(*) FROM LabStudy.EXAM e WHERE e.Patient = p.%ID) AS Total "
        _"FROM LabStudy.PATIENT p")

    while rs.%Next() {
        // the whole row travels as a list: no separator, no escaping, no risk
        do ##class(LabStudy.Sorter).Add(.idx, rs.%Get("Total"), rs.%Get("Id"),
            $LISTBUILD(rs.%Get("Rec"), rs.%Get("Name")))
    }

    write "=== top ", n, " patients by exam count ===", !

    set found = ##class(LabStudy.Sorter).TopN(.idx, n, 1, .ordered)

    for i = 1:1:found {
        set row = ordered(i)
        set info = $LIST(row, 3)                 // the inner list

        write "  ", $JUSTIFY(i, 2), ". ", $JUSTIFY($LIST(row, 1), 4), " exams   ",
              $LISTGET(info, 1), "  ", $LISTGET(info, 2), !
    }

    quit $$$OK
}
```

E acrescente a `src/LabStudy/Reports.cls`:

```objectscript
/// Exams whose final result falls outside the reference range.
ClassMethod AbnormalByRange() As %Integer
{
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT e.%ID AS Id FROM LabStudy.EXAM e WHERE e.ResultStatus = 'final'")

    kill idx
    set checked = 0

    while rs.%Next() {
        set exam = ##class(LabStudy.Exam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)

        set checked = checked + 1
        continue:'exam.IsAbnormal()

        set patientName = ""
        if $ISOBJECT(exam.Patient) {
            set patientName = exam.Patient.Name
        }

        // sort key: how far outside the range
        set min = $LISTGET(exam.ReferenceRange, 1)
        set max = $LISTGET(exam.ReferenceRange, 2)
        set distance = $SELECT(exam.ResultValue > max: exam.ResultValue - max,
                               1: min - exam.ResultValue)

        do ##class(LabStudy.Sorter).Add(.idx, $FNUMBER(distance, "", 4), exam.%Id(),
            $LISTBUILD(exam.TestCode, exam.ResultValue, exam.Unit,
                       exam.RangeText(), patientName))
    }

    write "=== results outside the reference range ===", !
    write "  dist    code   value     range              patient", !
    write "  ---------------------------------------------------------", !

    set found = ##class(LabStudy.Sorter).TopN(.idx, 20, 1, .ordered)

    for i = 1:1:found {
        set info = $LIST(ordered(i), 3)
        write "  ", $JUSTIFY($LIST(ordered(i), 1), 6), " ",
              $JUSTIFY($LISTGET(info, 1), 6), " ",
              $JUSTIFY($LISTGET(info, 2), 8), " ",
              $JUSTIFY($LISTGET(info, 3), 6), "  ",
              $JUSTIFY($LISTGET(info, 4), 16), "  ",
              $LISTGET(info, 5), !
    }

    write "  ---------------------------------------------------------", !
    write "  ", found, " abnormal out of ", checked, " final exams", !

    quit found
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.5";
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.Demo.RangeSeed).Run()
2000 ranges applied, 0 exams with unknown code

LABSTUDY>SET e = ##class(LabStudy.Exam).%OpenId(1)
LABSTUDY>WRITE e.TestCode, " = ", e.ResultValue, "  faixa: ", e.RangeText(), !
GLU = 305  faixa: 70 - 99 mg/dL

LABSTUDY>WRITE "anormal? ", e.IsAbnormal(), !
anormal? 1

LABSTUDY>DO ##class(LabStudy.Reports).AbnormalByRange()
=== results outside the reference range ===
  dist    code   value     range              patient
  ---------------------------------------------------------
    298    GLU      397  mg/dL     70 - 99 mg/dL     Paciente 00174
    296   CHOL      486  mg/dL      0 - 190 mg/dL    Paciente 00061
    293    GLU      392  mg/dL     70 - 99 mg/dL     Paciente 00098
  ...
  ---------------------------------------------------------
  20 abnormal out of 2000 final exams

LABSTUDY>DO ##class(LabStudy.RankReport).ByExamCount(3)
=== top 3 patients by exam count ===
   1.   10 exams   REG-000012  Paciente 00012
   2.   10 exams   REG-000007  Paciente 00007
   3.   10 exams   REG-000003  Paciente 00003
```

**Por que cada decisão:**

- **A troca de `"Rec|Name"` por `$LISTBUILD(Rec, Name)` no `RankReport` elimina um bug latente.** Um paciente chamado `"Silva | Souza"` — grafia rara mas possível — teria quebrado o relatório antigo, deslocando as colunas. Com lista, isso deixou de ser uma preocupação. **Foi o próprio capítulo anterior que introduziu o problema; este o corrige.** Reconhecer e corrigir o próprio antipadrão é parte do trabalho.
- **`ReferenceRange` como `%List` é a escolha certa aqui**, e vale entender por quê: a faixa tem exatamente três campos, é sempre lida inteira, e ninguém precisa filtrar exames "cujo mínimo seja 12". Se algum dia surgisse essa necessidade, a modelagem correta passaria a ser propriedades separadas ou uma classe de faixas de referência.
- **`IsAbnormal` tem cinco guardas antes de comparar**, e cada uma corresponde a uma lição desta apostila: resultado não final (Capítulo 10), resultado vazio (Capítulo 10), faixa inválida (`$LISTVALID`, este capítulo), faixa incompleta (`$LISTLENGTH`, este capítulo) e limites vazios. **Nenhuma dessas situações significa "anormal" — todas significam "desconhecido".** Devolver `0` para todas é a resposta correta, e devolver `1` seria alarme falso.
- **O `quit:condição valor` com pós-condicional** deixa as guardas em cinco linhas legíveis em vez de um `if` aninhado.
- **`IsAbnormal` agora faz o que promete.** No Capítulo 3 ele devolvia sempre `0`, com um comentário dizendo que subclasses o sobrescreveriam. A promessa foi paga.
- **`RangeSeed.Run()` é idempotente**: não sobrescreve faixas já definidas. Rodar duas vezes não desfaz ajustes manuais — a lição do Capítulo 11.
- **`AbnormalByRange` ordena pela distância em relação à faixa**, um critério calculado — exatamente o caso em que a ordenação em memória do Capítulo 13 é a ferramenta certa, porque o SQL não conhece a faixa, que está guardada como lista opaca.
- **Cada linha do ranking viaja como lista de cinco campos**, incluindo `RangeText()`, que contém espaços e hífen. Com separador de texto, o hífen ou o espaço poderiam conflitar com o separador escolhido. Com lista, a questão não existe.
- **`ToCsv` e `FromCsv` ficaram em `ListUtil`, na fronteira.** Todo o resto do sistema trabalha com listas. Esta é a arquitetura recomendada: **separadores só na borda, listas por dentro.**

---

## 8. Quiz do capítulo

**Q1.** O que distingue uma lista `$LIST` de um texto com separadores?

- A) Nada; são a mesma coisa com sintaxe diferente.
- B) A lista grava o comprimento de cada elemento, então qualquer conteúdo é seguro.
- C) A lista só aceita números.
- D) A lista é sempre menor.

---

**Q2.** O que `$LISTBUILD("a", , "c")` cria?

- A) Uma lista de 2 elementos.
- B) Uma lista de 3 elementos, sendo o do meio **sem valor**.
- C) Uma lista de 3 elementos, sendo o do meio uma string vazia.
- D) Erro de sintaxe.

---

**Q3.** Qual função lê um elemento sem gerar erro quando a posição não existe?

- A) `$LIST`
- B) `$LISTGET`
- C) `$LISTDATA`
- D) `$LISTFIND`

---

**Q4.** `$LIST(lista, 2, 4)` devolve o quê?

- A) O elemento da posição 2.
- B) Uma **sublista** com os elementos de 2 a 4.
- C) Três valores separados por vírgula.
- D) Erro.

---

**Q5.** Qual função distingue "elemento sem valor" de "elemento com string vazia"?

- A) `$LISTGET`
- B) `$LISTLENGTH`
- C) `$LISTDATA`
- D) `$LISTVALID`

---

**Q6.** O que `$LISTFIND(lista, "x")` devolve quando `"x"` não está na lista?

- A) `""`
- B) `-1`
- C) `0`
- D) Erro.

---

**Q7.** Em `$LISTNEXT(lista, ptr, valor)`, com que valor `ptr` deve começar?

- A) `1`
- B) `0`
- C) `""`
- D) `$LISTLENGTH(lista)`

---

**Q8.** Como se concatenam duas listas?

- A) `$LISTCONCAT(a, b)`
- B) Com o operador `_`, o mesmo de texto.
- C) `$LISTBUILD(a, b)`
- D) Não é possível.

---

**Q9.** Como acrescentar um elemento ao final de uma lista?

- A) `set lista = lista _ $LISTBUILD(valor)`
- B) `set lista = lista + valor`
- C) `do lista.Insert(valor)`
- D) `set $LIST(lista) = valor`

---

**Q10.** `$LISTTOSTRING` seguido de `$LISTFROMSTRING` sempre devolve a lista original?

- A) Sim, sempre.
- B) Não: se algum elemento contiver o separador, a lista volta diferente.
- C) Sim, desde que se use vírgula.
- D) Não, porque as funções são incompatíveis.

---

**Q11.** Qual é a diferença entre `%List` e `list Of %String`?

- A) Nenhuma.
- B) `%List` é um tipo de dado lido com `$LIST`; `list Of` é uma coleção de objetos lida com `GetAt`.
- C) `%List` começa em 0 e `list Of` em 1.
- D) `list Of` não pode ser persistido.

---

**Q12.** Onde as listas aparecem no armazenamento de um objeto persistente?

- A) Em lugar nenhum.
- B) Cada nó de dados **é** uma lista, com uma posição por propriedade.
- C) Apenas em propriedades declaradas como `%List`.
- D) Apenas em índices.

---

**Q13.** Você recebe um valor de outro sistema e vai passá-lo a `$LIST`. O que fazer antes?

- A) Nada.
- B) Validar com `$LISTVALID`.
- C) Converter com `$LISTTOSTRING`.
- D) Contar com `$LISTLENGTH`.

---

**Q14.** Ao inserir um elemento na **posição 1** por concatenação de sublistas, qual é o problema?

- A) Nenhum.
- B) `$LIST(lista, 1, 0)` é uma faixa inválida; a borda precisa ser tratada separadamente.
- C) A lista fica invertida.
- D) O primeiro elemento é perdido.

---

**Q15.** Qual afirmação sobre desempenho é correta?

- A) `$PIECE` é mais rápido que `$LIST` para acessar o n-ésimo elemento.
- B) `$LIST` salta direto usando os comprimentos gravados; `$PIECE` varre o texto contando separadores.
- C) Os dois têm o mesmo custo.
- D) `$LISTNEXT` é mais lento que `$LIST` num laço.

---

**Q16.** Qual é a regra de decisão entre `$LIST` e `$PIECE`?

- A) Usar sempre `$PIECE`, que é mais legível.
- B) Usar `$LIST` dentro do sistema e `$PIECE` na fronteira com o mundo externo.
- C) Usar `$LIST` só para números.
- D) Usar `$PIECE` dentro e `$LIST` fora.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** o formato guarda o comprimento antes de cada elemento, então nenhum conteúdo pode ser confundido com delimitador.
- **A está errada:** o comportamento diante de conteúdo com separador é completamente diferente.
- **C está errada:** aceita qualquer conteúdo, inclusive binário.
- **D está errada:** costuma ser compacta, mas não é essa a característica definidora.

**Q2 — Resposta: B.**
- **B está certa:** a vírgula dupla cria um elemento **sem valor**, distinto de string vazia.
- **A está errada:** o comprimento é 3.
- **C está errada:** string vazia seria `$LISTBUILD("a", "", "c")`.
- **D está errada:** a sintaxe é válida e intencional.

**Q3 — Resposta: B.**
- **B está certa:** `$LISTGET` devolve vazio (ou um padrão) em vez de gerar erro.
- **A está errada:** `$LIST` gera erro fora de faixa.
- **C está errada:** `$LISTDATA` testa presença de valor, não lê.
- **D está errada:** `$LISTFIND` procura.

**Q4 — Resposta: B.**
- **B está certa:** com dois números, `$LIST` devolve uma **sublista**, que é uma lista válida.
- **A está errada:** isso seria `$LIST(lista, 2)`.
- **C está errada:** não há conversão para texto.
- **D está errada:** a forma é válida.

**Q5 — Resposta: C.**
- **C está certa:** `$LISTDATA` devolve `1` se o elemento tem valor e `0` se não tem.
- **A está errada:** `$LISTGET` devolve vazio nos dois casos.
- **B está errada:** `$LISTLENGTH` conta os dois igualmente.
- **D está errada:** `$LISTVALID` verifica a estrutura da lista.

**Q6 — Resposta: C.**
- **C está certa:** devolve `0` quando não encontra, e a posição quando encontra.
- **A e B estão erradas:** o valor de "não encontrado" é zero.
- **D está errada:** não há erro.

**Q7 — Resposta: B.**
- **B está certa:** o ponteiro começa em `0` e é atualizado por referência.
- **A, C e D estão erradas:** começar em outro valor pula elementos ou não funciona.

**Q8 — Resposta: B.**
- **B está certa:** o formato é autodescritivo, então colar duas listas com `_` produz uma lista válida.
- **A está errada:** essa função não existe.
- **C está errada:** isso criaria uma lista de duas listas aninhadas.
- **D está errada:** é justamente uma das características mais úteis do formato.

**Q9 — Resposta: A.**
- **A está certa:** concatenar a lista com uma lista de um elemento.
- **B está errada:** `+` é soma numérica.
- **C está errada:** `Insert` é método de coleção `list Of`, não de `%List`.
- **D está errada:** `$LIST(lista)` sem posição refere-se ao **primeiro** elemento.

**Q10 — Resposta: B.**
- **B está certa:** a conversão para texto reintroduz o problema do separador; a volta pode produzir mais elementos.
- **A e C estão erradas:** nenhum separador é imune.
- **D está errada:** as funções são complementares; o risco está no conteúdo.

**Q11 — Resposta: B.**
- **B está certa:** são estruturas diferentes com APIs diferentes.
- **A está errada:** confundi-las é erro comum.
- **C está errada:** ambas começam em 1; quem começa em 0 é o `%DynamicArray` do JSON.
- **D está errada:** `list Of` é persistida como tabela filha.

**Q12 — Resposta: B.**
- **B está certa:** o nó de dados de cada objeto é uma lista, com uma posição por propriedade — como visto no Capítulo 8.
- **A está errada:** é o formato nativo do armazenamento.
- **C está errada:** vale para todas as propriedades.
- **D está errada:** os índices usam subscritos, não listas.

**Q13 — Resposta: B.**
- **B está certa:** `$LISTVALID` é a primeira barreira contra um valor que não é lista.
- **A está errada:** passar um valor inválido a `$LIST` gera erro em execução.
- **C e D estão erradas:** ambas falhariam igualmente sobre um valor inválido.

**Q14 — Resposta: B.**
- **B está certa:** não existe parte esquerda, e a faixa `1, 0` é inválida. Trate a borda com um caso próprio.
- **A está errada:** é exatamente onde implementações ingênuas quebram.
- **C e D estão erradas:** o sintoma é erro, não reordenação nem perda silenciosa.

**Q15 — Resposta: B.**
- **B está certa:** o acesso posicional da lista é direto; o do texto separado exige varredura.
- **A está errada:** inverte.
- **C está errada:** a diferença cresce com o tamanho.
- **D está errada:** `$LISTNEXT` é mais rápido num laço, porque mantém a posição.

**Q16 — Resposta: B.**
- **B está certa:** listas por dentro, separadores só na borda, onde o mundo externo os exige.
- **A está errada:** legibilidade não compensa a fragilidade estrutural.
- **C está errada:** listas aceitam qualquer conteúdo.
- **D está errada:** inverte a recomendação.

---

## 9. Resumo relâmpago

1. Uma **lista `$LIST`** grava o **comprimento** de cada elemento, em vez de usar um caractere separador. Por isso qualquer conteúdo é seguro.
2. **Todo objeto persistente do IRIS é gravado como lista** — é o `$lb(...)` que você vê no `ZWRITE` das globais.
3. **`$LISTBUILD(a, b, c)`** cria. A **vírgula dupla** cria elemento **sem valor**.
4. **`$LIST(l)`** = primeiro; **`$LIST(l, n)`** = valor do n-ésimo; **`$LIST(l, de, ate)`** = **sublista**; **`$LIST(l, -1)`** = último.
5. **`$LIST` gera erro** fora de faixa ou em elemento sem valor. **`$LISTGET(l, n, padrão)`** é a versão segura.
6. **`$LISTLENGTH`** conta; **`$LISTDATA(l, n)`** distingue "tem valor" de "não tem"; **`$LISTVALID`** valida a estrutura; **`$LISTSAME`** compara.
7. **`$LISTFIND(l, v, apos)`** devolve a **posição** ou **`0`**.
8. **`$LISTNEXT(l, ptr, v)`** percorre eficientemente. **`ptr` começa em 0**, e `v` fica indefinida em elementos sem valor — proteja com `$GET`.
9. **Listas se concatenam com `_`.** É isso que permite inserir, remover e acrescentar por fatiamento.
10. `set $LIST(l, n) = valor` altera no lugar. `$LIST` pode aparecer à esquerda de um `SET`.
11. **Trate as bordas** ao inserir e remover: `$LIST(l, 1, 0)` é inválido.
12. **`$LISTTOSTRING(l, sep)`** e **`$LISTFROMSTRING(t, sep)`** são as pontes com o mundo externo — e **a ida e volta não é segura** se o conteúdo contiver o separador.
13. **`$LIST` por dentro, `$PIECE` na fronteira.** Converta na entrada e na saída, nunca no meio.
14. Uma lista pode conter outra lista **sem escape** — é assim que objetos embutidos são gravados.
15. **`%List`** (tipo de dado, `$LIST`) ≠ **`list Of`** (coleção, `GetAt`) ≠ **`%DynamicArray`** (JSON, `%Get`, começa em **0**).
16. Guarde como `%List` apenas o que é **sempre lido junto** e nunca consultado elemento a elemento — o SQL não enxerga dentro.
17. **`%INLIST ?`** no SQL recebe uma lista, evitando montar `IN (...)` com valores colados.
18. **Não use listas como subscrito** de array: a ordenação vira binária. Use subscritos múltiplos.
19. `$LIST` é **mais rápido** que `$PIECE` para acesso posicional; `$LISTNEXT` é mais rápido que `$LIST` num laço.
20. Ao receber dados de fora, **valide com `$LISTVALID`** antes de usar `$LIST`.

---

## 10. Cartões de memorização

**Frente:** Qual é a diferença fundamental entre `$LIST` e um texto com separadores?
**Verso:** A lista grava o comprimento de cada elemento. Qualquer conteúdo é seguro, inclusive o que contiver o "separador".

**Frente:** Onde você já viu listas no IRIS sem saber?
**Verso:** No armazenamento: cada nó de dados de um objeto persistente é uma lista (`$lb(...)` no `ZWRITE`).

**Frente:** O que `$LISTBUILD("a", , "c")` cria?
**Verso:** Uma lista de 3 elementos, com o do meio **sem valor** — diferente de string vazia.

**Frente:** Qual a versão segura de `$LIST`?
**Verso:** `$LISTGET(lista, posição, padrão)` — devolve vazio ou o padrão em vez de gerar erro.

**Frente:** `$LIST(l, 2)` e `$LIST(l, 2, 4)` devolvem o quê?
**Verso:** O primeiro devolve o **valor** do elemento 2; o segundo devolve uma **sublista** dos elementos 2 a 4.

**Frente:** Como distinguir elemento sem valor de elemento vazio?
**Verso:** `$LISTDATA(lista, n)` — `1` se tem valor, `0` se não tem. `$LISTGET` devolve vazio nos dois casos.

**Frente:** O que `$LISTFIND` devolve quando não encontra?
**Verso:** `0`. Quando encontra, devolve a posição.

**Frente:** Com que valor começa o ponteiro do `$LISTNEXT`?
**Verso:** `0`.

**Frente:** O que acontece com a variável de valor do `$LISTNEXT` num elemento sem valor?
**Verso:** Fica indefinida. Proteja o acesso com `$GET`.

**Frente:** Como se concatenam duas listas?
**Verso:** Com o operador `_`, o mesmo de texto. O formato autodescritivo permite isso.

**Frente:** Como inserir um elemento antes da posição n?
**Verso:** `$LIST(l, 1, n-1) _ $LISTBUILD(novo) _ $LIST(l, n, *)` — tratando as bordas separadamente.

**Frente:** Como remover o elemento da posição n?
**Verso:** `$LIST(l, 1, n-1) _ $LIST(l, n+1, *)` — tratando primeiro e último como casos próprios.

**Frente:** Como acrescentar ao final?
**Verso:** `set lista = lista _ $LISTBUILD(valor)`.

**Frente:** `$LISTTOSTRING` seguido de `$LISTFROMSTRING` devolve a lista original?
**Verso:** Não necessariamente. Se algum elemento contiver o separador, a lista volta com mais elementos.

**Frente:** Qual a regra de decisão entre `$LIST` e `$PIECE`?
**Verso:** `$LIST` por dentro do sistema; `$PIECE` só na fronteira com o mundo externo.

**Frente:** `%List`, `list Of` e `%DynamicArray`: qual a diferença?
**Verso:** `%List` é tipo de dado (`$LIST`, começa em 1); `list Of` é coleção de objetos (`GetAt`, começa em 1); `%DynamicArray` é JSON (`%Get`, começa em **0**).

**Frente:** Quando usar `%List` numa propriedade?
**Verso:** Quando o conjunto é pequeno, fixo e **sempre lido junto**. Nunca quando você precisa consultar elemento a elemento — o SQL não enxerga dentro.

**Frente:** Como passar vários valores a um `IN` do SQL sem colar texto?
**Verso:** `WHERE Coluna %INLIST ?`, passando um `$LISTBUILD` como parâmetro.

**Frente:** Uma lista pode conter outra lista?
**Verso:** Sim, sem escape nenhum. É assim que objetos embutidos são gravados.

**Frente:** O que fazer antes de passar a `$LIST` um valor vindo de fora?
**Verso:** Validar com `$LISTVALID`.

**Frente:** Por que não usar lista como subscrito de array?
**Verso:** A ordenação passa a ser pela representação binária, não pelo conteúdo lógico. Use subscritos múltiplos.

**Frente:** Por que o IRIS grava objetos como listas?
**Verso:** Porque é o formato mais rápido e compacto para acesso **posicional**, e aceita qualquer conteúdo sem risco de corromper o registro.

---

Digite CONTINUAR para o próximo capítulo.
