# Apostila InterSystems ObjectScript Specialist
## Capítulo 10 — T3.3 Handles Nulls (Lidando com valores vazios)

> Ainda em **T3 — IRIS Features**. Este capítulo é curto em sintaxe e longo em armadilhas. "Vazio" parece um assunto trivial até o dia em que um relatório mostra a média errada, um paciente aparece com idade zero e ninguém entende por quê.

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Distinguir os três estados que costumam ser confundidos: **indefinido**, **string vazia** e **zero**.
2. Testar existência com **`$DATA`** e ler com segurança usando **`$GET`** e valores padrão.
3. Entender como o ObjectScript avalia vazio em contexto **lógico**, **numérico** e **de texto**.
4. Explicar a diferença entre **`NULL`** e **string vazia** no SQL do IRIS, incluindo a representação interna com `$CHAR(0)`.
5. Escrever condições corretas com **`IS NULL`** e **`IS NOT NULL`**, e saber por que `= NULL` nunca funciona.
6. Usar **`IFNULL`** e **`COALESCE`** para substituir vazios em consultas.
7. Entender a **lógica de três valores** e por que `NULL` propaga.
8. Saber como as **funções de agregação** tratam vazios, e a diferença entre `COUNT(*)` e `COUNT(coluna)`.
9. Lidar com vazios em **listas** (`$LISTGET`, `$LISTDATA`) e em **coleções**.
10. Lidar com vazios em **JSON**: `null` versus campo inexistente, e o parâmetro `%JSONNULL`.
11. Modelar corretamente a diferença entre *"não informado"* e *"informado como zero"*.
12. Evoluir o projeto: exames com resultado pendente, relatórios que contam certo e exibição que não mente.

---

## 2. O conceito em linguagem de gente

### 2.1 Três coisas diferentes que parecem a mesma

Volte ao formulário de papel do laboratório. Olhe o campo **"Peso (kg)"** em três fichas:

**Ficha A — o campo está em branco.**
Ninguém pesou o paciente. Não sabemos o peso dele. O dado **não existe**.

**Ficha B — o campo tem um traço, e ao lado está escrito "não se aplica".**
Alguém olhou, decidiu que ali não cabe informação, e registrou essa decisão. O dado **existe e é vazio**.

**Ficha C — o campo tem o número 0.**
Alguém pesou e anotou zero. Isso seria absurdo para peso, mas para "número de alergias" faz todo o sentido: **zero é uma medida legítima**.

As três fichas são **três estados diferentes**, e tratá-las como iguais produz erros silenciosos:

- Se você somar os pesos e dividir por três, a média está errada, porque duas fichas não têm peso.
- Se você contar "quantos pacientes têm alergia registrada", a ficha C **conta** (zero alergias é uma informação) e as outras duas **não contam**.

No mundo do IRIS, esses três estados aparecem assim:

| Ficha | ObjectScript | SQL |
|---|---|---|
| A — em branco, desconhecido | variável **indefinida** | **`NULL`** |
| B — vazio registrado | **string vazia** `""` | **string vazia** (armazenada de forma especial) |
| C — zero medido | o número **`0`** | o número **`0`** |

### 2.2 O ObjectScript é mais simples — e por isso mais perigoso

O ObjectScript não tem uma palavra `null`. Ele tem apenas dois estados:

- A variável **existe** e tem algum conteúdo (que pode ser a string vazia).
- A variável **não existe**.

E aqui está o perigo: quando você lê uma variável que **não existe**, você não recebe vazio. Você recebe um **erro**:

```
LABSTUDY>KILL x
LABSTUDY>WRITE x

WRITE x
      ^
<UNDEFINED> *x
```

Por isso as duas ferramentas do Capítulo 8 são o coração deste capítulo também:

- **`$DATA(x)`** — pergunta se existe, sem tentar ler.
- **`$GET(x)`** — lê devolvendo vazio (ou um padrão) em vez de explodir.

### 2.3 O vazio é falso, e isso é uma faca de dois gumes

No ObjectScript, quando um valor é usado como condição, a regra é:

> **Zero é falso. String vazia é falsa. Qualquer outra coisa é verdadeira.**

Isso torna o código bonito:

```objectscript
if patientName {
    write "tem nome", !
}
```

E torna o código traiçoeiro, porque essa mesma linha trata `"0"` como **falso**. Se o nome do paciente fosse, por absurdo, `"0"`, a condição falharia.

O caso realista e comum é este:

```objectscript
if exam.ResultValue {
    write "resultado informado", !
}
```

Um resultado **legitimamente zero** — leucócitos zerados, por exemplo — cairia no `else`, e o sistema diria "não informado" para uma medição que existe e é clinicamente gravíssima.

A correção é sempre a mesma: **quando zero é um valor válido, teste a existência, não a veracidade.**

```objectscript
if exam.ResultValue '= "" {
    write "resultado informado: ", exam.ResultValue, !
}
```

### 2.4 O vazio vira zero em conta

Segundo comportamento que morde:

```
LABSTUDY>WRITE "" + 5, !
5

LABSTUDY>WRITE "" * 3, !
0
```

Numa expressão aritmética, a string vazia é convertida para `0`. Isso é conveniente e é uma bomba-relógio: uma soma de valores com alguns vazios **não dá erro** — ela simplesmente trata os vazios como zeros e produz uma média errada, sem avisar ninguém.

O SQL, como veremos, se comporta ao contrário: ele **ignora** os nulos nas agregações, que é o comportamento estatisticamente correto.

### 2.5 O `NULL` do SQL: a ausência que contamina

No SQL, `NULL` não é um valor. É a **marca de ausência de valor**. E ele tem uma propriedade que assusta quem vê pela primeira vez: **quase toda operação com `NULL` resulta em `NULL`**.

```
NULL + 10        → NULL
NULL = NULL      → não é verdadeiro
NULL = 5         → não é verdadeiro
NULL <> 5        → também não é verdadeiro
```

Por que `NULL = NULL` não é verdadeiro? Porque a pergunta, traduzida, é: *"o valor que eu não conheço é igual ao outro valor que eu não conheço?"*. A resposta honesta não é "sim" nem "não" — é **"não dá para saber"**.

Isso se chama **lógica de três valores**: além de verdadeiro e falso, existe **desconhecido**. E um `WHERE` só aceita linhas cujo resultado seja **verdadeiro** — linhas com resultado "desconhecido" ficam de fora, exatamente como as falsas.

A consequência prática é a mais cobrada do capítulo:

> **`WHERE coluna = NULL` nunca devolve nada.** Para testar ausência, use **`IS NULL`**.

### 2.6 A peculiaridade do IRIS: `NULL` e string vazia são coisas diferentes

Muitos bancos tratam a string vazia como `NULL`. **O IRIS não.** Ele mantém os dois estados separados, e isso corresponde exatamente às fichas A e B da nossa analogia.

Para conseguir essa distinção num sistema em que o armazenamento "vazio" já significa ausência, o IRIS usa um truque interno: a **string vazia do SQL é gravada como um caractere especial**, o `$CHAR(0)`.

O efeito colateral disso surpreende: se você inserir uma string vazia por SQL e depois ler aquele campo por objeto, encontrará `$CHAR(0)` em vez de `""`. Do lado do SQL, o campo se comporta como string vazia; do lado do ObjectScript, ele contém um caractere invisível de código zero.

É uma das poucas arestas do modelo unificado do IRIS, e ela cai na prova. O detalhe fino de como e quando essa conversão ocorre pode variar entre versões: **verificar na documentação oficial** quando o comportamento exato importar.

Recomendação prática para evitar o problema por completo: **escolha um dos dois estados e seja consistente no seu sistema.** Se "não informado" é o seu único conceito de vazio, grave sempre `NULL` (não atribua nada, ou atribua `""` pelo mundo dos objetos) e nunca insira `''` por SQL.

---

## 3. A sintaxe explicada

### 3.1 Do lado do ObjectScript

**Testar existência:**

```objectscript
if $DATA(x) {
    write "existe", !
}

if '$DATA(x) {
    write "não existe", !
}
```

Lembre dos valores do Capítulo 8: `0` inexistente, `1` só valor, `10` só filhos, `11` valor e filhos.

**Ler com segurança:**

```objectscript
set value = $GET(x)              // vazio se não existir
set value = $GET(x, 0)           // zero se não existir
set value = $GET(^Config("t"), 30)
```

**Distinguir "não existe" de "existe e é vazio":**

```objectscript
if '$DATA(x) {
    write "nunca informado", !
} elseif x = "" {
    write "informado como vazio", !
} else {
    write "tem valor: ", x, !
}
```

Essa cadeia de três ramos é a tradução literal das fichas A, B e C.

**Testar vazio corretamente quando zero é válido:**

```objectscript
// ERRADO quando 0 é um valor legítimo
if value { ... }

// CERTO
if value '= "" { ... }
```

**`$SELECT` com um ramo padrão:**

```objectscript
set label = $SELECT(value = "": "(não informado)", 1: value)
```

`$SELECT` avalia as condições em ordem e devolve o valor da **primeira** verdadeira. O `1:` final funciona como "senão", porque `1` é sempre verdadeiro.

**Atenção:** se **nenhuma** condição for verdadeira, `$SELECT` gera o erro `<ILLEGAL VALUE>`. Sempre inclua o ramo `1:`.

**Vazio em listas:**

```objectscript
set list = $LISTBUILD("a", , "c")

write $LISTLENGTH(list), !        // 3
write $LISTDATA(list, 2), !       // 0  -> o elemento 2 não tem valor
write "[", $LISTGET(list, 2), "]", !     // []  -> devolve vazio, sem erro
write "[", $LISTGET(list, 2, "?"), "]", ! // [?] -> com padrão
```

- **`$LISTBUILD("a", , "c")`** — a vírgula dupla cria um elemento **sem valor**. Isso é diferente de um elemento contendo `""`.
- **`$LISTDATA(lista, n)`** — devolve `1` se o elemento tem valor, `0` se não tem.
- **`$LISTGET(lista, n)`** — lê sem erro, devolvendo vazio; com terceiro argumento, devolve o padrão.

Isso não é curiosidade acadêmica: como você viu no Capítulo 8, **os objetos são gravados exatamente assim**, como listas em que propriedades não informadas são elementos sem valor. Listas são o assunto principal do Capítulo 14; aqui interessa o comportamento diante do vazio.

### 3.2 Do lado do SQL

**Testar ausência:**

```sql
SELECT Name FROM LabStudy.PATIENT WHERE Address_City IS NULL
SELECT Name FROM LabStudy.PATIENT WHERE Address_City IS NOT NULL
```

Nunca `= NULL` nem `<> NULL`.

**Substituir o nulo por um valor:**

```sql
SELECT Name, IFNULL(Address_City, '(sem cidade)') AS City
FROM LabStudy.PATIENT
```

```sql
SELECT COALESCE(Address_City, Address_State, '(sem endereço)') AS Local
FROM LabStudy.PATIENT
```

- **`IFNULL(expr, substituto)`** — devolve o substituto quando a expressão é nula.
- **`COALESCE(a, b, c, ...)`** — devolve o **primeiro** argumento não nulo da lista. É o `IFNULL` generalizado.

O IRIS também aceita outras funções equivalentes por compatibilidade com outros bancos: **verificar na documentação oficial** para a lista completa.

**Agregações ignoram nulos:**

```sql
SELECT
    COUNT(*)              AS TodasAsLinhas,
    COUNT(ResultValue)    AS ComResultado,
    AVG(ResultValue)      AS MediaCorreta,
    SUM(ResultValue)      AS Soma
FROM LabStudy.EXAM
```

- **`COUNT(*)`** conta **linhas**, inclusive as que têm tudo nulo.
- **`COUNT(coluna)`** conta apenas as linhas em que aquela coluna **não é nula**.
- **`AVG`**, **`SUM`**, **`MIN`** e **`MAX`** ignoram nulos.

Aqui está a diferença de filosofia entre os dois mundos, e vale destacar:

> No **ObjectScript**, somar um vazio é somar **zero** — a média fica **errada**.
> No **SQL**, uma linha nula é **excluída** do cálculo — a média fica **certa**.

Se você calcular a média percorrendo objetos num laço, precisará ignorar os vazios **manualmente**. Se calcular com `AVG`, o banco já faz isso.

**Ordenação:**

```sql
SELECT Name, Address_City FROM LabStudy.PATIENT ORDER BY Address_City
```

Valores nulos são agrupados numa das pontas da ordenação. No IRIS, por serem representados internamente como o subscrito vazio, eles aparecem **primeiro** na ordem crescente. Se a posição importar para a apresentação, ordene explicitamente por uma expressão que force o que você quer:

```sql
ORDER BY CASE WHEN Address_City IS NULL THEN 1 ELSE 0 END, Address_City
```

**Junções externas produzem nulos:**

```sql
SELECT p.Name, e.TestCode
FROM LabStudy.PATIENT p
LEFT JOIN LabStudy.EXAM e ON e.Patient = p.%ID
```

Pacientes sem exame aparecem com `e.TestCode` nulo. Isso é o comportamento desejado — e é a fonte mais comum de nulos em relatórios, mesmo quando nenhuma coluna da tabela aceita nulo.

### 3.3 Do lado dos objetos

**Propriedade nunca preenchida:**

```
LABSTUDY>SET p = ##class(LabStudy.Patient).%New()
LABSTUDY>WRITE "[", p.Sex, "]", !
[]
```

Uma propriedade de objeto **nunca é indefinida**: ela nasce como string vazia. Isso é diferente de uma variável local, que pode não existir.

Portanto, dentro do mundo dos objetos, o teste é sempre `'= ""`, e `$DATA` sobre propriedade não faz sentido.

**Exigindo valor:**

```objectscript
Property Name As %String [ Required ];
```

`Required` recusa a gravação quando a propriedade está vazia. É a forma declarativa de dizer "esta ficha não pode ficar em branco".

**Fornecendo padrão:**

```objectscript
Property Active As %Boolean [ InitialExpression = 1 ];
```

Um `InitialExpression` **elimina** o problema do vazio para aquela propriedade: ela nunca nasce em branco.

**Coleções:**

```objectscript
write patient.Allergies.Count(), !          // 0 quando vazia
write "[", patient.Allergies.GetAt(5), "]", !  // [] para índice inexistente
```

`GetAt` de posição inexistente devolve vazio sem erro. Uma coleção vazia é uma coleção com zero itens — não é "nula".

**JSON:**

Recapitulando o Capítulo 4, porque é exatamente o mesmo problema:

```objectscript
set obj = {"a": null}

write obj.%GetTypeOf("a"), !      // null       -> existe, valor nulo
write obj.%GetTypeOf("zzz"), !    // unassigned -> não existe
write "[", obj.%Get("a"), "]", !  // []
write "[", obj.%Get("zzz"), "]", ! // []
```

`%Get` não distingue; `%GetTypeOf` e `%IsDefined` distinguem. Fichas A e B de novo.

E no `%JSON.Adaptor`:

```objectscript
Property Phone As %String(%JSONNULL = 1);
```

Com `%JSONNULL = 1`, uma propriedade vazia é exportada como `"Phone": null`. Sem isso, ela é **omitida** do JSON. A escolha entre omitir e mandar `null` é uma decisão de contrato com quem consome a API — e as duas dizem coisas diferentes.

---

## 4. Exemplo comentado

Uma classe que expõe todos os comportamentos:

Arquivo `src/LabStudy/Demo/Nulls.cls`:

```objectscript
/// Explores the difference between undefined, empty string and zero.
Class LabStudy.Demo.Nulls Extends %Persistent
{

Property Label As %String(MAXLEN = 50);

/// May legitimately be zero, or may be unknown.
Property Measure As %Numeric;

/// Optional free text.
Property Note As %String(MAXLEN = 200);

/// Shows the three states side by side, in ObjectScript.
ClassMethod ThreeStates() As %Status
{
    kill undefinedVar
    set emptyVar = ""
    set zeroVar = 0

    write "-- $DATA --", !
    write "undefined : ", $DATA(undefinedVar), !
    write "empty     : ", $DATA(emptyVar), !
    write "zero      : ", $DATA(zeroVar), !

    write !, "-- truthiness --", !
    write "undefined : ", $SELECT($GET(undefinedVar) '= "": "true", 1: "false"), !
    write "empty     : ", $SELECT(emptyVar: "true", 1: "false"), !
    write "zero      : ", $SELECT(zeroVar: "true", 1: "false"), !

    write !, "-- arithmetic --", !
    write "empty + 5 : ", emptyVar + 5, !
    write "zero + 5  : ", zeroVar + 5, !

    write !, "-- comparison --", !
    write "empty = 0     : ", (emptyVar = 0), !
    write "empty = ''    : ", (emptyVar = ""), !
    write "zero = ''     : ", (zeroVar = ""), !
    write "empty + 0 = 0 : ", ((emptyVar + 0) = 0), !

    quit $$$OK
}

/// The wrong way and the right way to test a value that may be zero.
ClassMethod TestMeasure(value As %Numeric) As %Status
{
    write "value = [", value, "]", !

    if value {
        write "  wrong test says: informed", !
    } else {
        write "  wrong test says: NOT informed", !
    }

    if value '= "" {
        write "  right test says: informed", !
    } else {
        write "  right test says: NOT informed", !
    }

    quit $$$OK
}

/// Creates sample rows: one with a value, one with zero, one unfilled.
ClassMethod Seed() As %Status
{
    do ..%KillExtent()

    set a = ..%New()
    set a.Label = "measured 12"
    set a.Measure = 12
    set a.Note = "normal"
    do a.%Save()

    set b = ..%New()
    set b.Label = "measured zero"
    set b.Measure = 0
    set b.Note = ""
    do b.%Save()

    set c = ..%New()
    set c.Label = "not measured"
    // Measure and Note are left untouched
    do c.%Save()

    write "3 rows created", !
    quit $$$OK
}

/// Aggregations: shows how SQL ignores nulls.
ClassMethod Aggregations() As %Status
{
    new SQLCODE, %msg, rows, withValue, average, total

    &sql(SELECT COUNT(*), COUNT(Measure), AVG(Measure), SUM(Measure)
         INTO :rows, :withValue, :average, :total
         FROM LabStudy.Demo.Nulls)

    if SQLCODE < 0 {
        write "SQL error: ", %msg, !
        quit $$$OK
    }

    write "COUNT(*)        : ", rows, !
    write "COUNT(Measure)  : ", withValue, !
    write "AVG(Measure)    : ", average, !
    write "SUM(Measure)    : ", total, !

    write !, "Average computed by hand over all rows: ", total / rows, !
    write "Average computed by SQL over known rows : ", average, !

    quit $$$OK
}

/// Shows IS NULL, IFNULL and COALESCE in action.
ClassMethod NullQueries() As %Status
{
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT Label, "
        _"IFNULL(Measure, -1) AS MeasureOrMinusOne, "
        _"CASE WHEN Measure IS NULL THEN 'unknown' ELSE 'known' END AS State "
        _"FROM LabStudy.Demo.Nulls ORDER BY Label")

    while rs.%Next() {
        write rs.%Get("Label"), " | ", rs.%Get("MeasureOrMinusOne"), " | ", rs.%Get("State"), !
    }

    write !, "-- rows where Measure IS NULL --", !
    set rs2 = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT Label FROM LabStudy.Demo.Nulls WHERE Measure IS NULL")
    while rs2.%Next() {
        write "  ", rs2.%Get("Label"), !
    }

    write !, "-- rows where Measure = NULL (never matches) --", !
    set rs3 = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT Label FROM LabStudy.Demo.Nulls WHERE Measure = NULL")
    set found = 0
    while rs3.%Next() {
        set found = found + 1
        write "  ", rs3.%Get("Label"), !
    }
    write "  (", found, " rows)", !

    quit $$$OK
}

/// Safe display: never shows a bare empty field.
Method Describe() As %String
{
    set measureText = $SELECT(..Measure = "": "(not measured)", 1: ..Measure)
    set noteText    = $SELECT(..Note = "": "(no note)", 1: ..Note)

    quit ..Label_" | measure: "_measureText_" | note: "_noteText
}

}
```

Comentando as decisões:

- **`ThreeStates` usa `$SELECT` para imprimir "true"/"false"**, porque escrever o resultado de uma condição direto imprimiria `1` ou `0` e ficaria menos legível num experimento didático.
- **A primeira linha da seção de veracidade usa `$GET(undefinedVar)`**, e não `undefinedVar` direto. Sem isso, o método explodiria com `<UNDEFINED>` — o que, aliás, seria uma demonstração válida, mas interromperia o resto.
- **`TestMeasure` mostra o erro e o acerto lado a lado**, com a mesma entrada. É o formato mais convincente: você não precisa acreditar, você vê os dois resultados divergirem.
- **`Seed` cria deliberadamente três linhas diferentes**: uma com valor, uma com **zero explícito** e uma **sem tocar** na propriedade. Essas são as fichas C, C-zero e A.
- **`Aggregations` imprime a média de duas formas** — a "na mão" dividindo pela contagem total, e a do SQL. A divergência entre as duas é a lição do método.
- **`Describe` nunca devolve um campo em branco.** Uma tela que mostra `measure: ` sem nada faz o usuário pensar que houve erro de sistema. `(not measured)` comunica.

### 4.1 Usando no Terminal

```
LABSTUDY>DO ##class(LabStudy.Demo.Nulls).ThreeStates()
-- $DATA --
undefined : 0
empty     : 1
zero      : 1

-- truthiness --
undefined : false
empty     : false
zero      : false

-- arithmetic --
empty + 5 : 5
zero + 5  : 5

-- comparison --
empty = 0     : 0
empty = ''    : 1
zero = ''     : 0
empty + 0 = 0 : 1
```

Duas observações que valem muito:

- **`empty = 0` deu `0` (falso), mas `empty + 0 = 0` deu `1` (verdadeiro).** No primeiro caso a comparação foi feita como **texto** (`""` contra `"0"`, diferentes); no segundo, a soma forçou o contexto **numérico**, e aí `""` virou `0`. O mesmo valor, dois resultados, dependendo do contexto. Este é o ponto mais sutil do capítulo.
- **Os três estados são "falsos"** na avaliação lógica. Ou seja, `if valor` não distingue nenhum deles.

```
LABSTUDY>DO ##class(LabStudy.Demo.Nulls).TestMeasure(12)
value = [12]
  wrong test says: informed
  right test says: informed

LABSTUDY>DO ##class(LabStudy.Demo.Nulls).TestMeasure(0)
value = [0]
  wrong test says: NOT informed
  right test says: informed

LABSTUDY>DO ##class(LabStudy.Demo.Nulls).TestMeasure("")
value = []
  wrong test says: NOT informed
  right test says: NOT informed
```

**A linha do meio é o capítulo inteiro em três palavras.** Com o valor `0`, os dois testes discordam: o errado diz que não foi informado, o certo diz que foi. Num sistema clínico, essa divergência é a diferença entre "exame pendente" e "resultado crítico igual a zero".

```
LABSTUDY>DO ##class(LabStudy.Demo.Nulls).Seed()
3 rows created

LABSTUDY>DO ##class(LabStudy.Demo.Nulls).Aggregations()
COUNT(*)        : 3
COUNT(Measure)  : 2
AVG(Measure)    : 6
SUM(Measure)    : 12

Average computed by hand over all rows: 4
Average computed by SQL over known rows : 6
```

**Quatro contra seis.** A soma é 12 nos dois casos. A diferença está no divisor: dividir por 3 (todas as linhas) assume que a linha não medida vale zero; dividir por 2 (as linhas com valor) reconhece que ela é desconhecida.

Qual está certo? **Seis.** A média dos valores conhecidos é 6. Se você quisesse tratar "não medido" como zero, isso seria uma decisão de negócio explícita, e você a escreveria com `IFNULL(Measure, 0)`. O que não se pode é chegar em 4 **por acidente**, que é exatamente o que acontece quando se soma numa variável do ObjectScript dentro de um laço.

```
LABSTUDY>DO ##class(LabStudy.Demo.Nulls).NullQueries()
measured 12 | 12 | known
measured zero | 0 | known
not measured | -1 | unknown

-- rows where Measure IS NULL --
  not measured

-- rows where Measure = NULL (never matches) --
  (0 rows)
```

- **`IFNULL(Measure, -1)`** trocou o nulo por `-1`, e deixou o zero intacto. Repare: a linha "measured zero" mostra `0`, não `-1`. `IFNULL` age sobre **nulo**, não sobre falsidade.
- **`WHERE Measure = NULL` devolveu zero linhas**, apesar de existir uma linha nula. É a lógica de três valores: a comparação resultou em "desconhecido", e o `WHERE` descartou.

---

## 5. Variações e detalhes

### 5.1 Modelando "não informado" corretamente

Existe uma escolha de modelagem que evita boa parte dos problemas deste capítulo: **acrescentar um campo de estado** em vez de sobrecarregar o campo de valor.

Compare:

```objectscript
// Frágil: o vazio precisa carregar dois significados
Property ResultValue As %Numeric;
```

```objectscript
// Robusto: o estado é explícito
Property ResultValue As %Numeric;
Property ResultStatus As %String(VALUELIST = ",pending,final,cancelled")
                        [ InitialExpression = "pending" ];
```

Com o campo de estado:

- Não é preciso adivinhar o que o vazio significa.
- É possível distinguir "ainda não saiu", "saiu e é zero" e "foi cancelado" — três situações que o vazio sozinho jamais distinguiria.
- Consultas ficam legíveis: `WHERE ResultStatus = 'final'`.

Regra geral: **quando o vazio precisar significar mais de uma coisa, ele deixou de ser suficiente.** Crie o campo.

### 5.2 Vazios e índices

Valores vazios **são indexados** no IRIS. Um índice sobre uma coluna com muitas linhas vazias terá uma grande quantidade de entradas sob o mesmo subscrito.

Duas consequências:

- Uma consulta `WHERE Coluna IS NULL` **pode** usar o índice.
- Se 95% das linhas forem vazias, esse índice é pouco seletivo para aquele valor e o otimizador provavelmente preferirá outro caminho.

O detalhe de como o subscrito vazio é representado no índice e as opções para não indexar nulos variam por versão: **verificar na documentação oficial**.

### 5.3 Vazio em propriedades de referência

Uma propriedade que aponta para outro objeto pode estar vazia:

```objectscript
if $ISOBJECT(exam.Patient) {
    write exam.Patient.Name, !
} else {
    write "(exame sem paciente)", !
}
```

Sem essa proteção, `exam.Patient.Name` sobre uma referência vazia causa **`<INVALID OREF>`**. É a mesma proteção do Capítulo 1, aplicada agora ao caso do vazio.

E na consulta SQL com seta, o comportamento é diferente e mais gentil: `e.Patient->Name` sobre uma referência nula devolve **nulo**, sem erro. O SQL propaga; o ObjectScript explode.

### 5.4 `TRUNCATE` e conversões silenciosas

Vale relembrar do Capítulo 2, porque pertence à mesma família de armadilhas:

- `MAXLEN` estourado **recusa** — salvo com `TRUNCATE = 1`, quando **corta silenciosamente**.
- `SCALE` **arredonda** silenciosamente.

Ambos transformam um dado sem avisar. Vazio, truncamento e arredondamento formam o trio de mudanças invisíveis que produzem relatórios errados sem nenhuma mensagem de erro.

### 5.5 Vazios vindos de fora

Dados que chegam de arquivos, APIs e formulários trazem vazio de várias formas:

- campo ausente no JSON;
- campo presente com `null`;
- campo presente com `""`;
- campo presente com espaços em branco (`"   "`);
- campo presente com textos como `"N/A"`, `"-"`, `"null"`.

Os três últimos são especialmente traiçoeiros, porque **não são vazios** para o sistema: `"   "` tem comprimento 3 e passa em qualquer teste de `'= ""`.

Um método de normalização na entrada resolve de uma vez:

```objectscript
ClassMethod Normalize(value As %String) As %String
{
    set value = $ZSTRIP($GET(value), "<>W")

    if (value = "-") || ($ZCONVERT(value, "U") = "N/A") || ($ZCONVERT(value, "U") = "NULL") {
        quit ""
    }

    quit value
}
```

Aplicar isso na fronteira do sistema — e só na fronteira — é o padrão correto. Espalhar essa limpeza pelo código todo é o antipadrão.

### 5.6 Comparações no ObjectScript: texto ou número?

Recapitulando a regra que produziu o resultado surpreendente da seção 4.1:

O operador `=` do ObjectScript compara **numericamente** quando ambos os lados são números, e **como texto** caso contrário.

```
LABSTUDY>WRITE ("10" = 10), !
1
LABSTUDY>WRITE ("10.0" = 10), !
0
LABSTUDY>WRITE ("" = 0), !
0
LABSTUDY>WRITE (("" + 0) = 0), !
1
```

- `"10" = 10` é verdadeiro porque `"10"` é a forma **canônica** do número 10.
- `"10.0" = 10` é falso porque `"10.0"` **não** é canônica — comparou-se texto com texto.
- Para forçar comparação numérica, some zero ou use o sinal `+` como prefixo: `+valor = 0`.

Esse comportamento será tratado em profundidade no Capítulo 12, com os operadores. Aqui basta a consciência de que **o contexto decide**.

---

## 6. Pegadinhas e erros comuns

**1) Ler variável ou global inexistente diretamente.**
`<UNDEFINED>`. Use `$GET`.

**2) Usar `if valor` quando zero é um valor legítimo.**
Zero é falso. Use `if valor '= ""`.

**3) Somar valores possivelmente vazios e dividir pela contagem total.**
O vazio vira zero na soma e distorce a média. Conte só os informados, ou use `AVG` no SQL.

**4) Escrever `WHERE coluna = NULL`.**
Nunca devolve nada. Use `IS NULL`.

**5) Escrever `WHERE coluna <> 'x'` esperando incluir os nulos.**
Não inclui: a comparação com nulo resulta em "desconhecido", e o `WHERE` descarta. Se quiser os nulos, escreva `WHERE (coluna <> 'x') OR (coluna IS NULL)`.

**6) Confundir `COUNT(*)` com `COUNT(coluna)`.**
O primeiro conta linhas; o segundo conta valores não nulos.

**7) Achar que `NULL = NULL` é verdadeiro.**
Não é. Para comparar duas colunas que podem ser nulas, trate os nulos explicitamente.

**8) Achar que string vazia e `NULL` são a mesma coisa no IRIS.**
Não são. E a string vazia inserida por SQL é gravada como `$CHAR(0)`, o que aparece do lado dos objetos.

**9) Misturar as duas formas de vazio no mesmo campo.**
Escolha uma convenção e mantenha. Metade das linhas com `NULL` e metade com `$CHAR(0)` transforma qualquer consulta num campo minado.

**10) Achar que uma propriedade de objeto pode estar indefinida.**
Não pode: ela nasce como string vazia. `$DATA` sobre propriedade não é o teste certo; `'= ""` é.

**11) Acessar `obj.Referencia.Campo` sem checar.**
Referência vazia causa `<INVALID OREF>`. Proteja com `$ISOBJECT`.

**12) Tratar `"   "` como vazio.**
Não é. Normalize com `$ZSTRIP(valor, "<>W")` antes de comparar.

**13) Tratar `"N/A"`, `"-"` ou `"null"` como valores.**
São vazios disfarçados vindos de fora. Normalize na fronteira do sistema.

**14) Esquecer o ramo `1:` no `$SELECT`.**
Sem nenhuma condição verdadeira, ocorre `<ILLEGAL VALUE>`.

**15) Exibir campo vazio sem indicação na tela.**
O usuário não distingue "não informado" de "erro no sistema". Escreva `(não informado)`.

**16) Esquecer que `LEFT JOIN` produz nulos.**
Mesmo com todas as colunas obrigatórias, a junção externa gera nulos para as linhas sem correspondência.

**17) Confundir `null` de JSON com campo ausente.**
`%Get` não distingue; `%GetTypeOf` e `%IsDefined` distinguem. E `%JSONNULL` controla se a exportação omite ou manda `null`.

---

## 7. MÃO NA MASSA

---

### Exercício 10.1 — Os três estados

**a) Enunciado:** Crie `LabStudy.Demo.Null1` com um `ClassMethod Classify(name)` que, recebendo **o nome de uma variável** (não o valor), classifica seu estado em um de quatro:

- `NÃO EXISTE`
- `VAZIO`
- `ZERO`
- `TEM VALOR: <valor>`

Depois teste com uma variável inexistente, uma vazia, uma com `0`, uma com `"0"`, uma com `"   "` e uma com `"abc"`.

**b) Dica:** Para inspecionar uma variável cujo nome está numa string, use indireção: `@name`. Combine com `$DATA(@name)`.

**c) Como testar:** `"0"` e `0` devem cair no mesmo ramo. `"   "` deve cair em `TEM VALOR`, e é isso mesmo que deve acontecer.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Null1.cls`:

```objectscript
/// Classifies the state of a variable given by name.
Class LabStudy.Demo.Null1 Extends %RegisteredObject
{

/// Returns a readable classification of the variable named in "name".
ClassMethod Classify(name As %String) As %String
{
    if '$DATA(@name) {
        quit "NAO EXISTE"
    }

    set value = @name

    if value = "" {
        quit "VAZIO"
    }

    if (value + 0 = 0) && (value ? .N) {
        quit "ZERO"
    }

    quit "TEM VALOR: ["_value_"]"
}

/// Runs the classification over several sample variables.
ClassMethod Demo() As %Status
{
    kill notThere
    set isEmpty = ""
    set isZeroNum = 0
    set isZeroText = "0"
    set isSpaces = "   "
    set isText = "abc"

    write "notThere   : ", ..Classify("notThere"), !
    write "isEmpty    : ", ..Classify("isEmpty"), !
    write "isZeroNum  : ", ..Classify("isZeroNum"), !
    write "isZeroText : ", ..Classify("isZeroText"), !
    write "isSpaces   : ", ..Classify("isSpaces"), !
    write "isText     : ", ..Classify("isText"), !

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Null1).Demo()
notThere   : NAO EXISTE
isEmpty    : VAZIO
isZeroNum  : ZERO
isZeroText : ZERO
isSpaces   : TEM VALOR: [   ]
isText     : TEM VALOR: [abc]
```

**Por que cada decisão:**

- **`Classify` recebe o nome, não o valor.** Se recebesse o valor, a passagem por argumento já teria transformado a variável inexistente em erro antes mesmo de o método começar. Só a indireção permite perguntar "esta variável existe?" a respeito de outra.
- **`@name` é indireção**, o mesmo recurso do Capítulo 8. É o que permite ao método trabalhar sobre qualquer variável.
- **A ordem dos testes importa.** Primeiro existência, depois vazio, depois zero. Invertê-la produziria erro na variável inexistente.
- **`(value + 0 = 0) && (value ? .N)`** — a primeira parte força contexto numérico; a segunda usa **casamento de padrão** (`?`) para exigir que o conteúdo seja composto só de dígitos. Sem a segunda parte, `"abc"` também passaria, porque `"abc" + 0` é `0`. Padrões são o Capítulo 13; aqui, `.N` significa "qualquer quantidade de dígitos".
- **`"   "` foi classificado como `TEM VALOR`, e está correto.** Três espaços são três caracteres. O sistema não tem como adivinhar que quem digitou queria dizer "nada" — a menos que você normalize na entrada, como discutido na seção 5.5. Reconhecer que este resultado é **certo, mas indesejável**, é o aprendizado do exercício.
- **`isZeroNum` e `isZeroText` deram o mesmo resultado**, porque `"0"` é a forma canônica do número zero.

---

### Exercício 10.2 — A média errada

**a) Enunciado:** Crie `LabStudy.Demo.Null2` com dados de teste e **três** formas de calcular a média de uma coluna que tem valores vazios:

1. `ClassMethod AverageWrong()` — percorre com cursor, soma tudo numa variável e divide pela contagem total de linhas.
2. `ClassMethod AverageManual()` — percorre com cursor, mas **ignora** as linhas vazias, contando só as que têm valor.
3. `ClassMethod AverageSql()` — usa `AVG` diretamente.

Crie cinco linhas: valores `10`, `20`, `0`, e duas sem valor. Compare os três resultados e explique cada um.

**b) Dica:** No cursor, teste `if value '= ""` antes de somar.

**c) Como testar:** Os três números devem ser diferentes entre si — e você deve saber dizer qual é o correto e por quê.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Null2.cls`:

```objectscript
/// Three ways to average a column that contains nulls.
Class LabStudy.Demo.Null2 Extends %Persistent
{

Property Tag As %String(MAXLEN = 30);

Property Value As %Numeric;

/// Creates the sample data: 10, 20, 0, and two unfilled rows.
ClassMethod Seed() As %Status
{
    do ..%KillExtent()

    for pair = "a:10", "b:20", "c:0" {
        set row = ..%New()
        set row.Tag = $PIECE(pair, ":", 1)
        set row.Value = $PIECE(pair, ":", 2)
        do row.%Save()
    }

    for tag = "d", "e" {
        set row = ..%New()
        set row.Tag = tag
        // Value deliberately left untouched
        do row.%Save()
    }

    write "5 rows created", !
    quit $$$OK
}

/// Sums everything and divides by the total number of rows. Wrong.
ClassMethod AverageWrong() As %Numeric
{
    new SQLCODE, %msg, value

    set total = 0
    set rows = 0

    &sql(DECLARE C1 CURSOR FOR SELECT Value INTO :value FROM LabStudy.Demo.Null2)
    &sql(OPEN C1)
    for {
        &sql(FETCH C1)
        quit:SQLCODE'=0

        set rows = rows + 1
        set total = total + value          // "" becomes 0 here
    }
    &sql(CLOSE C1)

    if rows = 0 { quit "" }
    quit total / rows
}

/// Ignores rows without a value. Correct, done by hand.
ClassMethod AverageManual() As %Numeric
{
    new SQLCODE, %msg, value

    set total = 0
    set counted = 0

    &sql(DECLARE C2 CURSOR FOR SELECT Value INTO :value FROM LabStudy.Demo.Null2)
    &sql(OPEN C2)
    for {
        &sql(FETCH C2)
        quit:SQLCODE'=0

        continue:value=""                  // the whole difference is this line

        set counted = counted + 1
        set total = total + value
    }
    &sql(CLOSE C2)

    if counted = 0 { quit "" }
    quit total / counted
}

/// Lets SQL do it. Correct and shorter.
ClassMethod AverageSql() As %Numeric
{
    new SQLCODE, %msg, avg

    &sql(SELECT AVG(Value) INTO :avg FROM LabStudy.Demo.Null2)

    if SQLCODE '= 0 { quit "" }
    quit avg
}

/// Runs the three and shows the counters behind them.
ClassMethod Compare() As %Status
{
    new SQLCODE, %msg, rows, withValue, total

    &sql(SELECT COUNT(*), COUNT(Value), SUM(Value)
         INTO :rows, :withValue, :total
         FROM LabStudy.Demo.Null2)

    write "rows            : ", rows, !
    write "rows with value : ", withValue, !
    write "sum             : ", total, !
    write "------------------------------", !
    write "AverageWrong  : ", ..AverageWrong(), !
    write "AverageManual : ", ..AverageManual(), !
    write "AverageSql    : ", ..AverageSql(), !

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Null2).Seed()
5 rows created

LABSTUDY>DO ##class(LabStudy.Demo.Null2).Compare()
rows            : 5
rows with value : 3
sum             : 30
------------------------------
AverageWrong  : 6
AverageManual : 10
AverageSql    : 10
```

**Por que cada resultado:**

- **`AverageWrong` deu 6** porque dividiu 30 por 5. As duas linhas sem valor entraram na soma como zero e no divisor como linhas. O resultado é a média de um conjunto que **não existe**: um conjunto em que "não medido" vale zero.
- **`AverageManual` deu 10** porque dividiu 30 por 3. Repare que a linha `c`, com valor **zero legítimo**, **entrou** na conta — e deve entrar mesmo. Zero medido é uma medição.
- **`AverageSql` deu 10** também, e escreveu isso em uma linha, sem cursor, sem contador e sem chance de errar.
- **A linha que faz toda a diferença é `continue:value=""`.** Uma linha. Sem ela, o método parece correto, roda sem erro, devolve um número plausível — e mente.
- **`COUNT(*)` = 5 contra `COUNT(Value)` = 3** é o par de números que explica tudo. Sempre que uma média parecer estranha, compare esses dois.
- **Note que `AverageWrong` e `AverageManual` compartilham quase todo o código.** Bugs desse tipo não vêm de código feio; vêm de código que parece razoável.

---

### Exercício 10.3 — `NULL` no SQL

**a) Enunciado:** Usando a tabela do exercício anterior, escreva no Portal ou via SQL dinâmico consultas que respondam:

1. Todas as linhas, mostrando `Value` e um rótulo `known`/`unknown`.
2. Só as linhas sem valor.
3. Só as linhas com valor.
4. Uma consulta que tente `WHERE Value = NULL` — e conte quantas linhas voltam.
5. `Value` substituído por `-1` quando nulo, sem alterar o zero.
6. Todas as linhas cujo `Tag` seja diferente de `'a'`, **incluindo** as que teriam algum campo nulo.
7. Contagem por estado: quantas com valor, quantas sem.

**b) Dica:** Item 6: `WHERE (Tag <> 'a')` já basta aqui, porque `Tag` está sempre preenchido — mas escreva também a versão robusta com `OR Tag IS NULL` e reflita sobre quando ela seria necessária.

**c) Como testar:** O item 4 deve devolver zero linhas.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```sql
-- 1
SELECT Tag, Value,
       CASE WHEN Value IS NULL THEN 'unknown' ELSE 'known' END AS State
FROM LabStudy.Demo.Null2
ORDER BY Tag
```

```
Tag  Value  State
a    10     known
b    20     known
c    0      known
d           unknown
e           unknown
```

```sql
-- 2
SELECT Tag FROM LabStudy.Demo.Null2 WHERE Value IS NULL
-- d, e

-- 3
SELECT Tag FROM LabStudy.Demo.Null2 WHERE Value IS NOT NULL
-- a, b, c

-- 4
SELECT Tag FROM LabStudy.Demo.Null2 WHERE Value = NULL
-- (nenhuma linha)

-- 5
SELECT Tag, IFNULL(Value, -1) AS V FROM LabStudy.Demo.Null2 ORDER BY Tag
-- a:10  b:20  c:0  d:-1  e:-1

-- 6
SELECT Tag FROM LabStudy.Demo.Null2 WHERE Tag <> 'a'
-- b, c, d, e

SELECT Tag FROM LabStudy.Demo.Null2 WHERE (Tag <> 'a') OR (Tag IS NULL)
-- b, c, d, e   (mesmo resultado aqui, porque Tag nunca é nulo)

-- 7
SELECT
    COUNT(*) AS Todas,
    COUNT(Value) AS ComValor,
    COUNT(*) - COUNT(Value) AS SemValor
FROM LabStudy.Demo.Null2
-- 5, 3, 2
```

**Por que cada resultado:**

- **No item 1, a linha `c` mostrou `0` e `known`.** O `CASE ... IS NULL` distingue corretamente zero de ausência — coisa que um `CASE WHEN Value = 0` jamais faria.
- **O item 4 devolveu nada**, apesar de haver duas linhas nulas. `= NULL` produz "desconhecido", e o `WHERE` só aceita "verdadeiro". Se você quiser ver isso de outro ângulo, rode `SELECT NULL = NULL` e observe que o resultado não é 1.
- **O item 5 é a demonstração de que `IFNULL` age sobre nulo, não sobre falsidade.** A linha `c`, com zero, saiu como `0`, não como `-1`. Um `IFNULL` que trocasse zeros seria inútil.
- **O item 6 tem uma lição escondida.** Aqui as duas consultas deram o mesmo resultado porque `Tag` nunca é nulo. Mas suponha que fosse: `WHERE Tag <> 'a'` **excluiria** as linhas nulas, porque `NULL <> 'a'` é "desconhecido". O usuário que pediu "todos menos o A" ficaria sem os nulos, e nunca saberia. A versão com `OR Tag IS NULL` é a que traduz corretamente a intenção humana. Escreva a versão robusta sempre que a coluna puder ser nula.
- **O item 7 mostra o idioma padrão para contar nulos:** `COUNT(*) - COUNT(coluna)`.

---

### Exercício 10.4 — Vazios vindos de fora

**a) Enunciado:** Crie `LabStudy.Demo.Null4` com:

1. `ClassMethod Normalize(value) As %String` — remove espaços das pontas e converte os "vazios disfarçados" (`""`, `"-"`, `"N/A"`, `"NULL"`, `"null"`, `"--"`) em string vazia.
2. `ClassMethod ImportRow(json) As %Status` — recebe um `%DynamicObject` com `name`, `city` e `phone`, normaliza cada campo, e imprime um relatório dizendo, para cada campo, se ele veio ausente, veio nulo, veio vazio disfarçado ou veio com valor.

Teste com um JSON que contenha todos esses casos ao mesmo tempo.

**b) Dica:** `%IsDefined` distingue ausente; `%GetTypeOf` distingue `null`; a normalização distingue o disfarçado.

**c) Como testar:** Os quatro diagnósticos devem aparecer, cada um no seu campo.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Null4.cls`:

```objectscript
/// Normalises the many disguises of "empty" that arrive from outside.
Class LabStudy.Demo.Null4 Extends %RegisteredObject
{

/// Turns disguised empties into a real empty string.
ClassMethod Normalize(value As %String = "") As %String
{
    set value = $ZSTRIP($GET(value), "<>W")

    if value = "" {
        quit ""
    }

    set upper = $ZCONVERT(value, "U")

    if (upper = "-") || (upper = "--") || (upper = "N/A") || (upper = "NULL") || (upper = "NA") {
        quit ""
    }

    quit value
}

/// Diagnoses each field of an incoming payload.
ClassMethod ImportRow(json As %DynamicObject) As %Status
{
    if '$ISOBJECT(json) {
        write "no payload", !
        quit $$$OK
    }

    for field = "name", "city", "phone" {
        write $JUSTIFY(field, 8), " : "

        if 'json.%IsDefined(field) {
            write "AUSENTE no payload", !
            continue
        }

        if json.%GetTypeOf(field) = "null" {
            write "NULL explicito", !
            continue
        }

        set raw = json.%Get(field)
        set clean = ..Normalize(raw)

        if clean = "" {
            write "VAZIO (recebido como [", raw, "])", !
        } else {
            write "VALOR [", clean, "]", !
        }
    }

    quit $$$OK
}

}
```

```
LABSTUDY>SET payload = {"name":"  Maria Silva  ","city":"N/A","phone":null}

LABSTUDY>DO ##class(LabStudy.Demo.Null4).ImportRow(payload)
    name : VALOR [Maria Silva]
    city : VAZIO (recebido como [N/A])
   phone : NULL explicito

LABSTUDY>SET payload2 = {"name":"   ","city":"Potirendaba"}

LABSTUDY>DO ##class(LabStudy.Demo.Null4).ImportRow(payload2)
    name : VAZIO (recebido como [   ])
    city : VALOR [Potirendaba]
   phone : AUSENTE no payload
```

**Por que cada decisão:**

- **Quatro diagnósticos, quatro tratamentos diferentes.** Ausente, nulo explícito, vazio disfarçado e valor real são quatro situações que exigem quatro decisões de negócio possivelmente diferentes. Um sistema que só distingue "tem" e "não tem" perde essa informação para sempre.
- **`name` chegou com espaços e virou valor limpo.** `$ZSTRIP(valor, "<>W")` removeu os espaços das pontas — e note que ele **não** transformou o campo em vazio, porque havia conteúdo real.
- **`name` do segundo payload, com só espaços, virou vazio.** Mesma função, resultado diferente, porque não sobrou nada depois da limpeza. É exatamente o comportamento desejado, e é o que faltava no exercício 10.1.
- **`city` com `"N/A"` foi reconhecido como vazio.** Sem essa normalização, o sistema gravaria a cidade "N/A" e um dia alguém emitiria uma correspondência para a cidade de N/A.
- **A comparação é feita em maiúsculas** para pegar `"n/a"`, `"N/a"` e variantes.
- **A normalização acontece na fronteira**, num método só. É o padrão correto: o resto do sistema pode confiar que, dali para dentro, vazio é vazio.
- **`$JUSTIFY(field, 8)`** alinha à direita num campo de 8 caracteres, deixando a saída legível. Formatação será tratada no Capítulo 13.

---

### Exercício 10.5 — PROJETO CONTÍNUO: resultados pendentes

**a) Enunciado:** O laboratório tem um problema real: um exame é cadastrado **antes** de o resultado sair. Hoje, `ResultValue` vazio pode significar "ainda não saiu" ou "saiu e é zero" — e não há como distinguir.

Corrija isso:

1. Em `LabStudy.Exam`, acrescente:
   - `Property ResultStatus As %String(VALUELIST = ",pending,final,cancelled") [ InitialExpression = "pending" ]`;
   - `Property ResultDate As %TimeStamp` — preenchida quando o resultado é lançado;
   - `Method SetResult(value, unit) As %Status` — lança o resultado, aceita **zero** como valor válido, recusa vazio, e muda o estado para `final`;
   - `Method Cancel(reason) As %Status` — muda o estado para `cancelled` e limpa o valor;
   - `Method HasResult() As %Boolean` — devolve verdadeiro **apenas** quando o estado é `final`;
   - `Method ResultText() As %String` — devolve o texto para exibição, nunca vazio: `(pendente)`, `(cancelado)` ou o valor com unidade;
   - `%OnValidateObject` que recuse estado `final` sem valor, e estado `pending` **com** valor.
2. Em `LabStudy.Reports`, acrescente:
   - `ClassMethod ResultStats()` — imprime, por consulta SQL: total de exames, quantos pendentes, quantos finais, quantos cancelados, a média **apenas dos finais**, e quantos finais têm resultado exatamente zero;
   - `Query PendingExams()` — lista os exames pendentes com nome do paciente.
3. Em `LabStudy.Patient`, ajuste `Show` para usar `ResultText()` em vez de imprimir o valor cru.
4. Suba `LabStudy.App` para `"1.1"`.

**b) Dica:** Para "média apenas dos finais", use `WHERE ResultStatus = 'final'` — e note que isso é diferente e melhor do que confiar em `AVG` ignorando nulos, porque um exame cancelado poderia ter valor residual.

**c) Como testar:**

```
LABSTUDY>SET e = ##class(LabStudy.Exam).%New("LEU")
LABSTUDY>SET e.Patient = ##class(LabStudy.Patient).%OpenId(1)
LABSTUDY>WRITE $$$ISOK(e.%Save()), !
LABSTUDY>WRITE e.ResultText(), !
LABSTUDY>WRITE $$$ISOK(e.SetResult(0, "10^9/L")), !
LABSTUDY>WRITE e.ResultText(), !
LABSTUDY>DO ##class(LabStudy.Reports).ResultStats()
```

Um exame com resultado **zero** deve aparecer como resultado, não como pendente.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Acrescente a `src/LabStudy/Exam.cls`:

```objectscript
/// Lifecycle state of the result. Empty ResultValue alone is ambiguous.
Property ResultStatus As %String(VALUELIST = ",pending,final,cancelled")
                        [ InitialExpression = "pending" ];

/// When the result was entered.
Property ResultDate As %TimeStamp;

/// Why the exam was cancelled, when applicable.
Property CancelReason As %String(MAXLEN = 200);

/// Enters the result. Zero is a perfectly valid result; empty is not.
Method SetResult(value As %Numeric, unit As %String = "") As %Status
{
    if value = "" {
        quit $$$ERROR($$$GeneralError, "A result value is required. Use Cancel() to discard the exam.")
    }

    set ..ResultValue = value
    if unit '= "" {
        set ..Unit = unit
    }
    set ..ResultStatus = "final"
    set ..ResultDate = $ZDATETIME($HOROLOG, 3)

    quit $$$OK
}

/// Cancels the exam, clearing any value.
Method Cancel(reason As %String = "") As %Status
{
    set ..ResultValue = ""
    set ..ResultStatus = "cancelled"
    set ..CancelReason = reason
    set ..ResultDate = ""

    quit $$$OK
}

/// True only when a final result exists. Zero counts as a result.
Method HasResult() As %Boolean [ CodeMode = expression ]
{
..ResultStatus = "final"
}

/// Text for display. Never returns an empty string.
Method ResultText() As %String
{
    if ..ResultStatus = "pending" {
        quit "(pendente)"
    }

    if ..ResultStatus = "cancelled" {
        quit "(cancelado"_$SELECT(..CancelReason'="": ": "_..CancelReason, 1: "")_")"
    }

    if ..ResultValue = "" {
        quit "(sem valor)"
    }

    quit ..ResultValue_$SELECT(..Unit'="": " "_..Unit, 1: "")
}

/// The state and the value must agree with each other.
Method %OnValidateObject() As %Status [ Private, ServerOnly = 1 ]
{
    if (..ResultStatus = "final") && (..ResultValue = "") {
        quit $$$ERROR($$$GeneralError, "A final result must have a value")
    }

    if (..ResultStatus = "pending") && (..ResultValue '= "") {
        quit $$$ERROR($$$GeneralError, "A pending exam must not have a value")
    }

    quit $$$OK
}
```

Acrescente a `src/LabStudy/Reports.cls`:

```objectscript
/// Exams still waiting for a result.
Query PendingExams() As %SQLQuery(ROWSPEC = "Id:%Integer,PatientName:%String,RecordNumber:%String,TestCode:%String,CollectedOn:%TimeStamp") [ SqlProc ]
{
SELECT e.%ID, e.Patient->Name, e.Patient->RecordNumber, e.TestCode, e.CollectedOn
FROM LabStudy.EXAM e
WHERE e.ResultStatus = 'pending'
ORDER BY e.CollectedOn
}

/// Counters that treat "pending", "zero" and "cancelled" as different things.
ClassMethod ResultStats() As %Status
{
    new SQLCODE, %msg, total, pending, final, cancelled, avgFinal, zeros

    &sql(SELECT
            COUNT(*),
            SUM(CASE WHEN ResultStatus = 'pending'   THEN 1 ELSE 0 END),
            SUM(CASE WHEN ResultStatus = 'final'     THEN 1 ELSE 0 END),
            SUM(CASE WHEN ResultStatus = 'cancelled' THEN 1 ELSE 0 END),
            AVG(CASE WHEN ResultStatus = 'final' THEN ResultValue ELSE NULL END),
            SUM(CASE WHEN ResultStatus = 'final' AND ResultValue = 0 THEN 1 ELSE 0 END)
         INTO :total, :pending, :final, :cancelled, :avgFinal, :zeros
         FROM LabStudy.EXAM)

    if SQLCODE < 0 {
        write "SQL error: ", %msg, !
        quit $$$OK
    }

    write "==============================", !
    write "Exams total     : ", +$GET(total), !
    write "  pending       : ", +$GET(pending), !
    write "  final         : ", +$GET(final), !
    write "  cancelled     : ", +$GET(cancelled), !
    write "Average (final) : ", $SELECT($GET(avgFinal)="": "(no data)", 1: avgFinal), !
    write "Final results that are exactly zero: ", +$GET(zeros), !
    write "==============================", !

    write !, "-- pending exams --", !
    set rs = ..PendingExamsFunc()
    set none = 1
    while rs.%Next() {
        set none = 0
        write "  ", rs.%Get("RecordNumber"), " ", rs.%Get("PatientName"),
              " -> ", rs.%Get("TestCode"), " (", rs.%Get("CollectedOn"), ")", !
    }
    if none {
        write "  (none)", !
    }

    quit $$$OK
}
```

Em `src/LabStudy/Patient.cls`, ajuste o trecho dos exames dentro de `Show`:

```objectscript
    write "Exams (", patient.Exams.Count(), "):", !
    for i = 1:1:patient.Exams.Count() {
        set exam = patient.Exams.GetAt(i)
        write "  - ", exam.TestCode, ": ", exam.ResultText(), !
    }
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.1";
```

Execução esperada:

```
LABSTUDY>SET e = ##class(LabStudy.Exam).%New("LEU")
LABSTUDY>SET e.Patient = ##class(LabStudy.Patient).%OpenId(1)
LABSTUDY>WRITE $$$ISOK(e.%Save()), !
1

LABSTUDY>WRITE e.ResultText(), !
(pendente)

LABSTUDY>WRITE e.HasResult(), !
0

LABSTUDY>SET e.ResultValue = 5
LABSTUDY>DO $SYSTEM.Status.DisplayError(e.%Save())
ERROR #5001: A pending exam must not have a value

LABSTUDY>SET e.ResultValue = ""
LABSTUDY>WRITE $$$ISOK(e.SetResult(0, "10^9/L")), !
1

LABSTUDY>WRITE $$$ISOK(e.%Save()), !
1

LABSTUDY>WRITE e.ResultText(), !
0 10^9/L

LABSTUDY>WRITE e.HasResult(), !
1

LABSTUDY>DO $SYSTEM.Status.DisplayError(e.SetResult("", ""))
ERROR #5001: A result value is required. Use Cancel() to discard the exam.

LABSTUDY>SET e2 = ##class(LabStudy.Exam).%New("VHS")
LABSTUDY>SET e2.Patient = ##class(LabStudy.Patient).%OpenId(1)
LABSTUDY>DO e2.%Save()
LABSTUDY>DO e2.Cancel("amostra insuficiente")
LABSTUDY>DO e2.%Save()
LABSTUDY>WRITE e2.ResultText(), !
(cancelado: amostra insuficiente)

LABSTUDY>DO ##class(LabStudy.Reports).ResultStats()
==============================
Exams total     : 6
  pending       : 0
  final         : 5
  cancelled     : 1
Average (final) : 76.5
Final results that are exactly zero: 1
==============================

-- pending exams --
  (none)

LABSTUDY>DO ##class(LabStudy.Patient).Show(1)
...
Exams (3):
  - HGB: 13.5 g/dL
  - LEU: 0 10^9/L
  - VHS: (cancelado: amostra insuficiente)
...
```

**Por que cada decisão:**

- **O campo de estado resolve o problema na raiz.** Antes, `ResultValue = ""` tinha que carregar dois significados e não dava conta. Agora o vazio do valor é apenas uma consequência do estado, e o estado é explícito. Este é o conteúdo da seção 5.1 aplicado ao projeto.
- **`SetResult(0, ...)` funcionou.** Um resultado de leucócitos igual a zero é gravado como resultado final — e `HasResult()` devolve verdadeiro. Se o sistema usasse `if ..ResultValue` para decidir, esse paciente apareceria eternamente como "pendente", e o achado clínico mais grave do dia ficaria invisível.
- **`SetResult("")` foi recusado, com uma mensagem que diz o que fazer.** "Use `Cancel()`" transforma um erro numa instrução. Mensagens de erro que apenas negam são desperdício de uma oportunidade de ensinar.
- **`%OnValidateObject` impede os dois estados incoerentes**: final sem valor e pendente com valor. Repare que a segunda regra pegou a tentativa manual de `SET e.ResultValue = 5` sem passar por `SetResult`. É a defesa do Capítulo 3 protegendo o modelo contra o uso direto das propriedades.
- **`ResultText()` nunca devolve vazio.** Toda a interface passa a ser honesta: o usuário lê `(pendente)`, `(cancelado: motivo)` ou o valor. Nunca um espaço em branco que ele terá de interpretar sozinho.
- **`AVG(CASE WHEN ResultStatus = 'final' THEN ResultValue ELSE NULL END)`** é mais forte do que confiar apenas em `AVG(ResultValue)`. Ele produz `NULL` deliberadamente para tudo que não é final, e `AVG` ignora nulos. Assim, mesmo que um exame cancelado tivesse um valor residual esquecido, ele não entraria na média.
- **`SUM(CASE WHEN ... THEN 1 ELSE 0 END)`** é o idioma padrão para "contar linhas que satisfazem uma condição" dentro de uma agregação única. Poderia-se rodar quatro consultas com `COUNT(*)` e `WHERE`; uma consulta só é mais rápida e mais consistente, já que todas as contagens vêm do mesmo instante.
- **O contador "final results that are exactly zero" existe de propósito.** Ele é a métrica que prova, em produção, que o sistema está tratando zero corretamente. Se ele estivesse sempre em zero num laboratório real, seria sinal de que zeros estão sendo perdidos em algum lugar do caminho.
- **`Cancel` limpa o valor e a data.** Deixar um valor residual num exame cancelado seria criar exatamente a ambiguidade que o capítulo inteiro combate.

---

## 8. Quiz do capítulo

**Q1.** No ObjectScript, o que acontece ao ler uma variável que nunca recebeu valor?

- A) Devolve string vazia.
- B) Devolve zero.
- C) Gera o erro `<UNDEFINED>`.
- D) Devolve `null`.

---

**Q2.** Qual expressão lê uma variável possivelmente inexistente, devolvendo `0` quando ela não existe?

- A) `$DATA(x)`
- B) `$GET(x, 0)`
- C) `$SELECT(x)`
- D) `+x`

---

**Q3.** Considere `set v = 0`. O que `if v { ... }` faz?

- A) Executa o bloco, porque `v` existe.
- B) Não executa o bloco, porque zero é falso.
- C) Gera erro.
- D) Executa apenas se `v` for texto.

---

**Q4.** Qual é o teste correto para "o valor foi informado", num campo em que zero é legítimo?

- A) `if value`
- B) `if value '= ""`
- C) `if value > 0`
- D) `if $DATA(value)`

---

**Q5.** Quanto vale `"" + 5` no ObjectScript?

- A) Erro.
- B) `5`
- C) `""`
- D) `"5"` como texto, não número.

---

**Q6.** Qual é o resultado de `WRITE ("" = 0)` no ObjectScript?

- A) `1`, porque vazio equivale a zero.
- B) `0`, porque a comparação é feita como texto entre `""` e `"0"`.
- C) Erro de sintaxe.
- D) Vazio.

---

**Q7.** No SQL, o que devolve `WHERE Coluna = NULL`?

- A) As linhas em que a coluna é nula.
- B) Todas as linhas.
- C) Nenhuma linha, porque a comparação resulta em desconhecido.
- D) Erro de sintaxe.

---

**Q8.** Qual é a forma correta de filtrar linhas com valor ausente?

- A) `WHERE Coluna = ''`
- B) `WHERE Coluna IS NULL`
- C) `WHERE Coluna = NULL`
- D) `WHERE NOT Coluna`

---

**Q9.** Qual é a diferença entre `COUNT(*)` e `COUNT(Coluna)`?

- A) Nenhuma.
- B) `COUNT(*)` conta todas as linhas; `COUNT(Coluna)` conta apenas as linhas em que aquela coluna não é nula.
- C) `COUNT(*)` é mais lento e devolve o mesmo valor.
- D) `COUNT(Coluna)` conta valores distintos.

---

**Q10.** Uma coluna tem os valores `10`, `20`, `0` e dois nulos. Quanto devolve `AVG(coluna)`?

- A) `6`
- B) `10`
- C) `30`
- D) `0`

---

**Q11.** O que faz `IFNULL(Valor, -1)` numa linha em que `Valor` é `0`?

- A) Devolve `-1`.
- B) Devolve `0`, porque `IFNULL` age sobre nulo, não sobre zero.
- C) Devolve vazio.
- D) Gera erro.

---

**Q12.** No IRIS SQL, `NULL` e string vazia são a mesma coisa?

- A) Sim, sempre.
- B) Não: são estados distintos, e a string vazia é representada internamente por `$CHAR(0)`.
- C) Sim, exceto em colunas numéricas.
- D) Não, mas a distinção só existe em índices.

---

**Q13.** Uma propriedade de objeto nunca preenchida contém o quê?

- A) Está indefinida, e lê-la causa `<UNDEFINED>`.
- B) Contém string vazia.
- C) Contém zero.
- D) Contém `null`.

---

**Q14.** Você escreve `WHERE Cidade <> 'Rio'` numa coluna que pode ser nula. O que acontece com as linhas nulas?

- A) São incluídas no resultado.
- B) São excluídas, porque a comparação com nulo resulta em desconhecido.
- C) Geram erro.
- D) São convertidas para string vazia e incluídas.

---

**Q15.** No JSON dinâmico, como distinguir um campo que veio como `null` de um campo ausente?

- A) Com `%Get()`, que devolve valores diferentes.
- B) Com `%GetTypeOf()`, que devolve `null` num caso e `unassigned` no outro.
- C) Com `%Size()`.
- D) Não é possível distinguir.

---

**Q16.** O que acontece se nenhuma condição de um `$SELECT` for verdadeira?

- A) Devolve vazio.
- B) Devolve zero.
- C) Gera o erro `<ILLEGAL VALUE>`.
- D) Devolve o último valor.

---

### Gabarito comentado

**Q1 — Resposta: C.**
- **C está certa:** ler variável inexistente causa `<UNDEFINED>`.
- **A está errada:** isso é o que `$GET` faz, não a leitura direta.
- **B está errada:** não há conversão automática na leitura.
- **D está errada:** o ObjectScript não tem uma palavra `null`.

**Q2 — Resposta: B.**
- **B está certa:** `$GET(x, 0)` devolve o padrão quando a variável não existe.
- **A está errada:** `$DATA` informa existência, não o valor.
- **C está errada:** `$SELECT` avalia condições e não protege contra indefinido.
- **D está errada:** `+x` sobre variável inexistente também gera `<UNDEFINED>`.

**Q3 — Resposta: B.**
- **B está certa:** zero é falso em contexto lógico.
- **A está errada:** existência não é o critério de veracidade.
- **C está errada:** não há erro; a variável existe.
- **D está errada:** o comportamento independe de ser texto.

**Q4 — Resposta: B.**
- **B está certa:** comparar com string vazia distingue "não informado" de "informado como zero".
- **A está errada:** zero seria classificado como não informado.
- **C está errada:** exclui zeros e negativos legítimos.
- **D está errada:** propriedades de objeto sempre existem; e para variáveis, `$DATA` não distingue vazio de valor.

**Q5 — Resposta: B.**
- **B está certa:** em contexto aritmético, a string vazia é convertida para zero.
- **A está errada:** não há erro — e é justamente isso que torna o comportamento perigoso.
- **C e D estão erradas:** o resultado é o número 5.

**Q6 — Resposta: B.**
- **B está certa:** como `""` não é um número, a comparação é textual, e `""` difere de `"0"`.
- **A está errada:** a conversão para número só ocorre em contexto aritmético.
- **C está errada:** a expressão é válida.
- **D está errada:** o resultado de uma comparação é 1 ou 0.

**Q7 — Resposta: C.**
- **C está certa:** a comparação com `NULL` resulta em desconhecido, e o `WHERE` só aceita verdadeiro.
- **A está errada:** para isso existe `IS NULL`.
- **B está errada:** nenhuma linha passa.
- **D está errada:** a sintaxe é aceita; o resultado é que é vazio.

**Q8 — Resposta: B.**
- **B está certa:** `IS NULL` é o predicado correto.
- **A está errada:** no IRIS, string vazia e nulo são estados distintos.
- **C está errada:** nunca casa.
- **D está errada:** não é sintaxe válida para esse fim.

**Q9 — Resposta: B.**
- **B está certa:** `COUNT(*)` conta linhas; `COUNT(coluna)` conta valores não nulos.
- **A está errada:** os resultados divergem quando há nulos.
- **C está errada:** não são equivalentes.
- **D está errada:** valores distintos seria `COUNT(DISTINCT coluna)`.

**Q10 — Resposta: B.**
- **B está certa:** `AVG` ignora nulos: soma 30 dividida por 3 valores conhecidos.
- **A está errada:** `6` seria dividir por 5, tratando nulos como zero.
- **C está errada:** `30` é a soma, não a média.
- **D está errada:** o zero é um dos valores, não o resultado.

**Q11 — Resposta: B.**
- **B está certa:** `IFNULL` substitui apenas nulos. O zero é um valor e permanece.
- **A está errada:** confundiria zero com ausência.
- **C e D estão erradas:** a função devolve o valor original.

**Q12 — Resposta: B.**
- **B está certa:** o IRIS mantém os dois estados separados, e usa `$CHAR(0)` internamente para representar a string vazia do SQL.
- **A e C estão erradas:** a distinção existe e vale para todos os tipos.
- **D está errada:** a distinção não se limita a índices.

**Q13 — Resposta: B.**
- **B está certa:** propriedades de objeto nascem como string vazia; nunca ficam indefinidas.
- **A está errada:** esse é o comportamento de variáveis locais e globais.
- **C está errada:** só se houvesse `InitialExpression = 0`.
- **D está errada:** não existe `null` no ObjectScript.

**Q14 — Resposta: B.**
- **B está certa:** `NULL <> 'Rio'` resulta em desconhecido, e o `WHERE` descarta. Para incluí-las, acrescente `OR Cidade IS NULL`.
- **A está errada:** é justamente o que não acontece, e é a origem de relatórios incompletos.
- **C está errada:** não há erro.
- **D está errada:** não há conversão automática.

**Q15 — Resposta: B.**
- **B está certa:** `%GetTypeOf` devolve `null` para campo presente e nulo, e `unassigned` para campo ausente.
- **A está errada:** `%Get` devolve vazio nos dois casos.
- **C está errada:** `%Size` só conta elementos.
- **D está errada:** a distinção existe e é essencial em integrações.

**Q16 — Resposta: C.**
- **C está certa:** `$SELECT` exige que alguma condição seja verdadeira; senão, `<ILLEGAL VALUE>`.
- **A e B estão erradas:** não há valor padrão implícito.
- **D está errada:** ele não devolve o último valor por omissão.

---

## 9. Resumo relâmpago

1. Três estados distintos: **indefinido** (não existe), **vazio** (`""`) e **zero** (`0`). Tratá-los como iguais produz erros silenciosos.
2. Ler variável ou global inexistente causa **`<UNDEFINED>`**. Use **`$DATA`** para testar e **`$GET`** para ler.
3. Em contexto **lógico**, tanto `0` quanto `""` são **falsos**. `if valor` não distingue nenhum dos três estados.
4. Quando **zero é um valor legítimo**, teste com **`valor '= ""`**, nunca com `if valor`.
5. Em contexto **aritmético**, `""` vira **`0`** — sem erro e sem aviso. Somas com vazios produzem médias erradas.
6. O operador `=` compara **numericamente** se ambos os lados são números canônicos, e **como texto** caso contrário. Por isso `"" = 0` é falso mas `("" + 0) = 0` é verdadeiro.
7. No SQL, **`NULL` não é valor: é ausência**, e quase toda operação com ele resulta em `NULL`.
8. **`= NULL` nunca casa.** Use **`IS NULL`** e **`IS NOT NULL`**.
9. **Lógica de três valores**: verdadeiro, falso e desconhecido. O `WHERE` só aceita **verdadeiro**.
10. `WHERE coluna <> 'x'` **exclui** as linhas nulas. Para incluí-las: `OR coluna IS NULL`.
11. **`IFNULL(expr, subst)`** e **`COALESCE(a, b, c)`** substituem nulos — e não tocam nos zeros.
12. **`COUNT(*)`** conta linhas; **`COUNT(coluna)`** conta valores não nulos. `COUNT(*) - COUNT(coluna)` conta os nulos.
13. **`AVG`, `SUM`, `MIN` e `MAX` ignoram nulos** — comportamento estatisticamente correto, oposto ao do ObjectScript.
14. No IRIS, **`NULL` e string vazia são estados diferentes**; a string vazia do SQL é gravada internamente como **`$CHAR(0)`**.
15. **Propriedade de objeto nunca fica indefinida**: ela nasce como string vazia. Teste com `'= ""`.
16. Referência de objeto vazia causa **`<INVALID OREF>`** no ObjectScript, mas devolve **nulo** numa consulta com seta.
17. **`LEFT JOIN` produz nulos** mesmo em colunas obrigatórias.
18. Listas: **`$LISTDATA`** testa presença, **`$LISTGET`** lê sem erro e aceita padrão. `$LISTBUILD("a", , "c")` cria elemento **sem valor**.
19. JSON: **`%GetTypeOf`** distingue `null` de `unassigned`; **`%JSONNULL`** decide entre omitir e exportar `null`.
20. **`$SELECT` exige o ramo `1:`**, senão gera `<ILLEGAL VALUE>`.
21. **Quando o vazio precisa significar mais de uma coisa, crie um campo de estado.**
22. **Normalize os vazios disfarçados** (`"   "`, `"-"`, `"N/A"`) na fronteira do sistema, num lugar só.
23. **Nunca exiba um campo vazio sem indicação**: escreva `(não informado)`.

---

## 10. Cartões de memorização

**Frente:** Quais são os três estados que costumam ser confundidos?
**Verso:** Indefinido (não existe), vazio (`""`) e zero (`0`).

**Frente:** O que acontece ao ler uma variável inexistente?
**Verso:** Erro `<UNDEFINED>`. Use `$GET` para ler com segurança.

**Frente:** Zero é verdadeiro ou falso em contexto lógico?
**Verso:** Falso — assim como a string vazia.

**Frente:** Qual o teste correto para "foi informado", quando zero é válido?
**Verso:** `if valor '= ""`. Nunca `if valor`.

**Frente:** Quanto vale `"" + 5`?
**Verso:** `5`. Em contexto aritmético, a string vazia vira zero, sem erro nem aviso.

**Frente:** Por que `"" = 0` é falso mas `("" + 0) = 0` é verdadeiro?
**Verso:** O primeiro compara como texto (`""` contra `"0"`); a soma força contexto numérico no segundo.

**Frente:** O que `WHERE Coluna = NULL` devolve?
**Verso:** Nada. A comparação resulta em desconhecido. Use `IS NULL`.

**Frente:** O que é lógica de três valores?
**Verso:** Verdadeiro, falso e **desconhecido**. O `WHERE` só aceita verdadeiro.

**Frente:** `WHERE Cidade <> 'Rio'` inclui as linhas com cidade nula?
**Verso:** Não. Para incluí-las, escreva `OR Cidade IS NULL`.

**Frente:** Diferença entre `COUNT(*)` e `COUNT(Coluna)`.
**Verso:** O primeiro conta linhas; o segundo conta valores não nulos. A diferença entre eles é a quantidade de nulos.

**Frente:** `AVG` sobre uma coluna com nulos: o que acontece?
**Verso:** Os nulos são **ignorados**. A média é calculada só sobre os valores conhecidos.

**Frente:** O que `IFNULL(Valor, -1)` faz quando `Valor` é `0`?
**Verso:** Devolve `0`. `IFNULL` age sobre nulo, não sobre falsidade.

**Frente:** O que faz `COALESCE(a, b, c)`?
**Verso:** Devolve o primeiro argumento não nulo da lista.

**Frente:** No IRIS SQL, `NULL` e string vazia são iguais?
**Verso:** Não. São estados distintos, e a string vazia é gravada internamente como `$CHAR(0)`.

**Frente:** Uma propriedade de objeto nunca preenchida contém o quê?
**Verso:** String vazia. Propriedades nunca ficam indefinidas.

**Frente:** O que acontece ao acessar `obj.Referencia.Campo` com a referência vazia?
**Verso:** `<INVALID OREF>`. Proteja com `$ISOBJECT`.

**Frente:** Como testar um elemento de lista que pode não ter valor?
**Verso:** `$LISTDATA(lista, n)` devolve 1 ou 0; `$LISTGET(lista, n, padrão)` lê sem erro.

**Frente:** Como distinguir `null` de campo ausente num JSON dinâmico?
**Verso:** `%GetTypeOf` devolve `null` num caso e `unassigned` no outro. `%Get` não distingue.

**Frente:** O que acontece se nenhuma condição do `$SELECT` for verdadeira?
**Verso:** Erro `<ILLEGAL VALUE>`. Sempre inclua o ramo `1:`.

**Frente:** Quando o vazio precisa significar duas coisas diferentes, o que fazer?
**Verso:** Criar um campo de **estado** explícito, em vez de sobrecarregar o campo de valor.

**Frente:** `"   "` (três espaços) é vazio?
**Verso:** Não, para o sistema. Normalize com `$ZSTRIP(valor, "<>W")` na fronteira.

**Frente:** Qual a regra de exibição para um campo vazio?
**Verso:** Nunca mostrar em branco. Escreva `(não informado)`, para o usuário não confundir com erro.

---

Digite CONTINUAR para o próximo capítulo.
