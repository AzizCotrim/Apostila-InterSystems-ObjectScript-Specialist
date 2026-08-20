# Apostila InterSystems ObjectScript Specialist
## Capítulo 9 — T3.2 Leverages ObjectScript/SQL Features (ObjectScript e SQL juntos)

> Ainda em **T3 — IRIS Features**. Este é o capítulo em que as duas portas da casa se encontram: você vai escrever SQL de dentro do ObjectScript, chamar ObjectScript de dentro do SQL, e finalmente pagar as promessas feitas nos capítulos anteriores, substituindo os laços ingênuos por consultas de verdade.

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Escrever **SQL embutido** com `&sql(...)` e usar **variáveis de host** (`:variavel`).
2. Interpretar **`SQLCODE`** — em especial os valores `0`, `100` e negativos — e usar **`%msg`** e **`%ROWCOUNT`**.
3. Usar **cursores**: `DECLARE`, `OPEN`, `FETCH`, `CLOSE`.
4. Escrever **SQL dinâmico** com **`%SQL.Statement`**, `%Prepare`, `%Execute` e `%SQL.StatementResult`.
5. Percorrer resultados com **`%Next()`**, ler colunas com **`%Get()`**, e exibir com `%Display()` e `%Print()`.
6. Passar valores com **parâmetros (`?`)** e entender por que isso é obrigatório por segurança.
7. Declarar **class queries** (`Query ... As %SQLQuery`) com `ROWSPEC`, e chamá-las.
8. Expor métodos como **procedimentos armazenados** (`SqlProc`) e chamá-los de dentro do SQL.
9. Usar recursos exclusivos do IRIS: a **sintaxe de seta** (`->`) para junção implícita, `%ID`, `TOP`, `%STARTSWITH`, `%INLIST`.
10. Entender os **modos de seleção** (lógico, ODBC e exibição) e o efeito de cada um.
11. Escolher conscientemente entre **acesso por objeto** e **acesso por SQL**.
12. Levar o projeto à versão **1.0**, substituindo todos os laços ingênuos por SQL.

---

## 2. O conceito em linguagem de gente

### 2.1 A mesma casa, duas portas

No Capítulo 8 você abriu a caixa preta e viu: os objetos e as tabelas SQL são **a mesma árvore de globais**. Não há cópia, não há sincronização, não há atraso. É um dado só.

Isso significa que você pode escolher a porta pela conveniência da tarefa:

| A tarefa é... | Use |
|---|---|
| trabalhar com **um** registro por vez, com regras de negócio | **objetos** (`%OpenId`, `%Save`) |
| trabalhar com **conjuntos**: filtrar, agrupar, contar, ordenar, juntar | **SQL** |

Uma analogia: para atender **um** paciente, você pega a ficha dele e trabalha com ela na mão — abrir, alterar, guardar. Para responder *"quantos pacientes acima de 60 anos fizeram hemograma este mês?"*, você não pega ficha por ficha: você pergunta ao arquivo inteiro de uma vez.

Tentar responder a segunda pergunta com objetos é o que fizemos nos capítulos anteriores — o laço de 1 a 20 em `App.Status()`. Funcionava, e era lento e limitado. Este capítulo conserta isso.

### 2.2 Duas maneiras de escrever SQL no ObjectScript

Existem duas formas, e a diferença entre elas é o primeiro conceito importante do capítulo.

**SQL embutido — a receita gravada em placa de metal.**

Você escreve a consulta **dentro do código**, e ela é **compilada junto com a classe**. O texto da consulta é fixo, decidido no momento em que você compila.

Vantagem: é rápido, porque o trabalho de análise já foi feito na compilação. O IRIS já sabe qual índice usar antes mesmo de o programa rodar.

Limitação: você **não pode montar a consulta na hora**. A lista de colunas, as tabelas e a estrutura do `WHERE` são fixas. Só os **valores** podem variar.

**SQL dinâmico — a receita escrita à mão, na hora.**

Você monta o texto da consulta em tempo de execução, entrega ao IRIS, ele analisa e executa.

Vantagem: flexibilidade total. Uma tela de busca em que o usuário escolhe quais filtros aplicar só pode ser feita assim.

Custo: a análise acontece em tempo de execução. O IRIS guarda em cache as consultas já analisadas, então o custo se dilui na repetição — mas a primeira vez custa.

A regra de escolha:

> **Se a estrutura da consulta é conhecida quando você escreve o código, use SQL embutido. Se ela depende de escolhas do usuário, use SQL dinâmico.**

### 2.3 O problema de trazer muitas linhas de uma vez

Uma consulta pode devolver uma linha, ou dez milhões. Trazer dez milhões de linhas para a memória de uma vez seria desastroso.

A solução, em qualquer banco de dados, é o **cursor**: em vez de receber tudo, você recebe um **ponteiro** que caminha pelo resultado, uma linha por vez.

Analogia: você não pede ao arquivista as duzentas mil fichas. Você pede que ele fique ao lado do arquivo e vá lhe passando uma por vez, conforme você termina de olhar a anterior.

No SQL embutido, isso se faz com um **cursor explícito** (`DECLARE`, `OPEN`, `FETCH`, `CLOSE`). No SQL dinâmico, o objeto de resultado já **é** um cursor, e você caminha com `%Next()`.

### 2.4 O sinal de trânsito: `SQLCODE`

Toda operação de SQL embutido deixa um recado numa variável especial chamada **`SQLCODE`**. Ela é o semáforo, e ler esse semáforo **não é opcional**.

São três faixas de valor:

- **`0`** — deu certo. Verde.
- **`100`** — **não há (mais) linhas**. Amarelo. **Isso NÃO é erro.** É o fim natural de uma busca que não achou nada ou de um percurso que chegou ao fim.
- **negativo** — erro de verdade. Vermelho. A mensagem fica em **`%msg`**.

A confusão clássica, e favorita da prova, é tratar `100` como erro. Uma busca que não encontra ninguém não falhou: ela respondeu "não há". São coisas diferentes, e um sistema que confunde as duas assusta o usuário com mensagens de erro onde deveria dizer "nenhum resultado".

### 2.5 Chamando o ObjectScript de dentro do SQL

O caminho inverso também existe: um método ObjectScript pode ser exposto ao SQL como **procedimento armazenado** ou como **função**, e ser chamado dentro de uma consulta.

Analogia: você contrata um especialista e deixa o ramal dele na lista interna. Agora qualquer consulta pode "ligar para o especialista" no meio do trabalho.

Isso é especialmente útil quando o cálculo é complicado demais para expressar em SQL, ou quando ele já existe escrito em ObjectScript e você não quer duplicá-lo.

---

## 3. A sintaxe explicada

### 3.1 SQL embutido: a forma geral

```objectscript
&sql(comando SQL)
```

- **`&sql`** — o e-comercial seguido de `sql`, sem espaço. É a marca que diz ao compilador: *"o que vem entre parênteses é SQL, não ObjectScript"*.
- Os parênteses delimitam. O comando pode ocupar várias linhas.
- Depois da execução, **`SQLCODE`** carrega o resultado.

### 3.2 Variáveis de host

Uma **variável de host** é uma variável do ObjectScript usada dentro do SQL. Ela é marcada com **dois-pontos** na frente:

```objectscript
set id = 1

&sql(SELECT Name, RecordNumber
     INTO :name, :record
     FROM LabStudy.PATIENT
     WHERE ID = :id)

if SQLCODE = 0 {
    write name, " / ", record, !
}
```

Duas direções de uso:

- **Entrada** — `:id` no `WHERE` leva o valor da variável para dentro da consulta.
- **Saída** — a cláusula **`INTO :name, :record`** traz os valores das colunas para as variáveis.

Regras que evitam sofrimento:

- A ordem do `INTO` corresponde à ordem das colunas do `SELECT`.
- Use variáveis **locais simples**. Se o valor está numa propriedade (`..Name`), copie para uma local antes.
- Se `SQLCODE` não for `0`, **não confie no conteúdo das variáveis de saída**.

### 3.3 `SELECT` de uma linha só

O `SELECT ... INTO` sem cursor espera **exatamente uma linha**:

```objectscript
&sql(SELECT COUNT(*) INTO :total FROM LabStudy.PATIENT)
write "total: ", total, !
```

Se a consulta não devolver linha nenhuma, `SQLCODE` vale `100`. Se devolver mais de uma, o resultado não é confiável — para vários registros, use cursor.

Note que funções de agregação como `COUNT(*)` sempre devolvem uma linha, então esse padrão é perfeito para contagens.

### 3.4 `INSERT`, `UPDATE` e `DELETE` embutidos

```objectscript
&sql(INSERT INTO LabStudy.PATIENT (Name, BirthDate, RecordNumber)
     VALUES (:name, :birth, :record))

if SQLCODE '= 0 {
    write "erro: ", %msg, !
}

&sql(UPDATE LabStudy.PATIENT SET Active = 0 WHERE ID = :id)
write "linhas afetadas: ", %ROWCOUNT, !

&sql(DELETE FROM LabStudy.EXAM WHERE Patient = :id)
write "exames apagados: ", %ROWCOUNT, !
```

**`%ROWCOUNT`** contém quantas linhas foram afetadas. Depois de um `UPDATE` que não achou nada, `SQLCODE` vale `100` e `%ROWCOUNT` vale `0`.

Repare que essas operações **disparam os triggers** que você escreveu no Capítulo 6 — mas **não** disparam os callbacks (`%OnBeforeSave`), porque não passam pelo mundo dos objetos. É exatamente a assimetria daquele capítulo, agora vista do outro lado.

### 3.5 Cursores no SQL embutido

Para percorrer várias linhas:

```objectscript
&sql(DECLARE PatCur CURSOR FOR
     SELECT ID, Name, RecordNumber
     INTO :id, :name, :record
     FROM LabStudy.PATIENT
     WHERE Active = 1
     ORDER BY Name)

&sql(OPEN PatCur)

for {
    &sql(FETCH PatCur)
    quit:SQLCODE'=0

    write id, " | ", name, " | ", record, !
}

&sql(CLOSE PatCur)
```

Peça por peça:

- **`DECLARE nome CURSOR FOR ...`** — declara o cursor. **Não executa nada**; é uma declaração de compilação. O `INTO` fica aqui, na declaração.
- **`OPEN nome`** — abre o cursor e prepara o percurso.
- **`FETCH nome`** — traz a próxima linha para as variáveis do `INTO`. Quando acabam as linhas, `SQLCODE` vira `100`.
- **`CLOSE nome`** — fecha e libera. **Sempre feche.**

O nome do cursor é um identificador de compilação: ele não vai entre dois-pontos e não pode vir de uma variável.

Uma armadilha de estilo: `quit:SQLCODE'=0` sai do laço tanto no fim normal (`100`) quanto num erro (negativo). Se você quiser distinguir os dois, confira `SQLCODE` depois do laço.

### 3.6 SQL dinâmico com `%SQL.Statement`

```objectscript
set stmt = ##class(%SQL.Statement).%New()

set sc = stmt.%Prepare("SELECT ID, Name FROM LabStudy.PATIENT WHERE Sex = ?")
if $$$ISERR(sc) {
    do $SYSTEM.Status.DisplayError(sc)
    quit
}

set rs = stmt.%Execute("F")

while rs.%Next() {
    write rs.%Get("ID"), " - ", rs.%Get("Name"), !
}

if rs.%SQLCODE < 0 {
    write "erro: ", rs.%Message, !
}
```

Peça por peça:

- **`%SQL.Statement.%New()`** cria o objeto de comando.
- **`%Prepare(texto)`** analisa a consulta. Devolve `%Status`.
- **`?`** é um **parâmetro posicional**. Os valores vão em `%Execute`, na ordem.
- **`%Execute(valores...)`** executa e devolve um **`%SQL.StatementResult`**.
- **`%Next()`** avança uma linha e devolve `1` enquanto houver, `0` quando acabar.
- **`%Get("Coluna")`** lê o valor da coluna na linha atual, pelo nome.
- **`rs.%SQLCODE`** e **`rs.%Message`** trazem o resultado e a mensagem.
- **`rs.%ROWCOUNT`**, depois de percorrer tudo, traz a contagem de linhas.

Existe um atalho para quando você não vai reutilizar o comando:

```objectscript
set rs = ##class(%SQL.Statement).%ExecDirect(, "SELECT COUNT(*) AS Total FROM LabStudy.PATIENT")
if rs.%Next() {
    write rs.%Get("Total"), !
}
```

A vírgula solta no primeiro argumento é intencional: aquele parâmetro receberia, por referência, o objeto de comando criado — e aqui não nos interessa.

**Exibindo rapidamente:**

```objectscript
do rs.%Display()
```

`%Display()` imprime o resultado inteiro formatado, com cabeçalho. É excelente para testes no Terminal e **péssimo** para código de produção, onde você quer controlar a saída.

### 3.7 Por que os parâmetros são obrigatórios

Compare:

```objectscript
// NUNCA faça isto
set rs = stmt.%Execute()
set sc = stmt.%Prepare("SELECT * FROM LabStudy.PATIENT WHERE Name = '"_digitado_"'")

// Faça isto
set sc = stmt.%Prepare("SELECT * FROM LabStudy.PATIENT WHERE Name = ?")
set rs = stmt.%Execute(digitado)
```

Na primeira forma, o que o usuário digitar vira **parte do comando**. É a injeção de SQL do Capítulo 7.

Na segunda, o valor viaja por um caminho separado e **nunca** é interpretado como comando, não importa o que contenha.

Há um segundo benefício, prático: consultas com parâmetro são reaproveitadas do cache de consultas. Consultas montadas com valores colados geram uma entrada de cache diferente **para cada valor**, entupindo o cache e desperdiçando análise.

Ou seja: parâmetros são simultaneamente mais seguros e mais rápidos. Não há argumento do outro lado.

### 3.8 Class queries

Uma **class query** é uma consulta **nomeada**, guardada dentro da classe, como um método:

```objectscript
/// Patients of a given city, ordered by name.
Query ByCity(city As %String) As %SQLQuery(ROWSPEC = "ID:%Integer,Name:%String,RecordNumber:%String") [ SqlProc ]
{
SELECT ID, Name, RecordNumber
FROM LabStudy.PATIENT
WHERE Address_City = :city
ORDER BY Name
}
```

Peça por peça:

- **`Query`** — palavra fixa, no lugar de `Method`.
- **`ByCity(city As %String)`** — nome e parâmetros, como num método.
- **`As %SQLQuery`** — o tipo. Significa "o corpo é SQL puro".
- **`ROWSPEC`** — a descrição das colunas devolvidas, no formato `nome:tipo`, separadas por vírgula. **Obrigatório** para que quem chama saiba o que vem.
- **`[ SqlProc ]`** — opcional; expõe a consulta ao SQL, permitindo `CALL`.
- No corpo, os parâmetros são referenciados como **variáveis de host** (`:city`).

Para chamar, o compilador gera um método com o sufixo **`Func`**:

```objectscript
set rs = ##class(LabStudy.Patient).ByCityFunc("Potirendaba")
while rs.%Next() {
    write rs.%Get("Name"), !
}
```

Por que usar class query em vez de escrever a consulta onde ela é necessária?

- A consulta fica **num lugar só**, com nome de negócio.
- Ela pode ser reaproveitada por várias telas e relatórios.
- Ela pode ser exposta ao SQL e a interfaces externas.
- Ela documenta o modelo: olhando a classe, você vê as perguntas que o sistema sabe responder.

Existe também `As %Query`, para consultas cuja lógica você escreve inteiramente em ObjectScript, implementando os métodos `Execute`, `Fetch` e `Close`. É o recurso para quando o resultado não vem de uma tabela — por exemplo, ler um diretório de arquivos e devolver como se fosse uma consulta. Os detalhes dessa implementação: **verificar na documentação oficial**.

### 3.9 Procedimentos armazenados

Um método vira procedimento com a palavra-chave `SqlProc`:

```objectscript
/// Formats a readable label for a patient.
ClassMethod Label(id As %String) As %String [ SqlProc, SqlName = PATIENT_LABEL ]
{
    set p = ..%OpenId(id)
    if '$ISOBJECT(p) {
        quit ""
    }
    quit p.RecordNumber_" - "_p.Name
}
```

E, no SQL:

```sql
SELECT ID, LabStudy.PATIENT_LABEL(ID) AS Label FROM LabStudy.PATIENT
```

- O nome no SQL é `schema.SqlName`. Sem `SqlName`, usa-se o nome do método.
- Métodos que **devolvem um valor** são usados como **função**, dentro de um `SELECT`.
- Métodos que **não devolvem valor** são chamados com `CALL`.

Cuidado importante de desempenho: uma função assim é executada **uma vez por linha**. Numa consulta sobre um milhão de linhas, ela roda um milhão de vezes. Se o corpo abre um objeto do disco, como o do exemplo, o custo é enorme. Funções em `SELECT` devem ser leves.

### 3.10 Recursos do SQL do IRIS que caem na prova

**Sintaxe de seta — junção implícita:**

```sql
SELECT e.TestCode, e.ResultValue, e.Patient->Name, e.Patient->RecordNumber
FROM LabStudy.EXAM e
```

A seta **`->`** segue uma referência a outro objeto **sem escrever `JOIN`**. Como o exame guarda o ID do paciente, o IRIS sabe fazer a junção sozinho.

Isso é exclusivo do IRIS, é muito mais legível que o `JOIN` equivalente, e a prova gosta de perguntar. Você pode encadear: `e.Patient->Address_City`.

**`%ID` — a coluna do identificador:**

```sql
SELECT %ID, Name FROM LabStudy.PATIENT
```

`%ID` é o nome universal da coluna do identificador, funcione qual for o `SqlRowIdName` configurado.

**`TOP` — limitando linhas:**

```sql
SELECT TOP 10 Name FROM LabStudy.PATIENT ORDER BY Name
```

No IRIS, `TOP` vem logo depois do `SELECT`.

**`%STARTSWITH` — começa com:**

```sql
SELECT Name FROM LabStudy.PATIENT WHERE Name %STARTSWITH 'Mar'
```

Mais eficiente que `LIKE 'Mar%'`, porque o otimizador o reconhece diretamente como uma busca por faixa no índice.

**`%INLIST` — pertence a uma lista:**

```objectscript
set codes = $LISTBUILD("HGB", "GLU", "CHOL")
set rs = stmt.%Execute(codes)
```

```sql
SELECT TestCode FROM LabStudy.EXAM WHERE TestCode %INLIST ?
```

Aqui o parâmetro recebe uma lista `$LISTBUILD` inteira, o que evita montar um `IN (...)` com valores colados.

**`%MATCHES`** compara com um padrão. E há mais predicados específicos: **verificar na documentação oficial**.

### 3.11 Modos de seleção: lógico, ODBC e exibição

Lembre-se dos três rostos de um valor, do Capítulo 2. O SQL pode devolver qualquer um deles.

```objectscript
set stmt = ##class(%SQL.Statement).%New()
set stmt.%SelectMode = 2                     // 0 = lógico, 1 = ODBC, 2 = exibição
set sc = stmt.%Prepare("SELECT Name, BirthDate FROM LabStudy.PATIENT")
set rs = stmt.%Execute()
do rs.%Display()
```

- **`%SelectMode = 0`** (lógico) — devolve o valor interno. `BirthDate` sai como `54683`.
- **`%SelectMode = 1`** (ODBC) — devolve `1990-05-17`.
- **`%SelectMode = 2`** (exibição) — devolve a data formatada segundo a configuração local, e `Sex` sai como `Female` em vez de `F`.

Isso explica um susto comum: a mesma consulta devolve coisas diferentes no Portal e no seu código. O Portal costuma usar modo de exibição; o SQL embutido, por padrão, usa modo lógico.

Para SQL embutido, o modo é definido em tempo de compilação por uma diretiva do pré-processador:

```objectscript
#SQLCompile Select = Display
```

**Recomendação prática:** para lógica de programa, use **lógico**. Para trocar dados com outros sistemas, **ODBC**. Reserve **exibição** para o que vai direto à tela.

---

## 4. Exemplo comentado

Uma classe de relatórios que usa as três formas:

Arquivo `src/LabStudy/Demo/Reports.cls`:

```objectscript
/// Demonstrates embedded SQL, cursors, dynamic SQL and class queries.
Class LabStudy.Demo.Reports Extends %RegisteredObject
{

/// Single row query: how many patients exist.
ClassMethod CountPatients() As %Integer
{
    new SQLCODE, total

    &sql(SELECT COUNT(*) INTO :total FROM LabStudy.PATIENT)

    if SQLCODE < 0 {
        write "SQL error: ", %msg, !
        quit -1
    }

    quit +$GET(total)
}

/// Single row query with a parameter, returning several columns.
ClassMethod PatientSummary(id As %String) As %Status
{
    new SQLCODE, name, record, active

    &sql(SELECT Name, RecordNumber, Active
         INTO :name, :record, :active
         FROM LabStudy.PATIENT
         WHERE ID = :id)

    if SQLCODE = 100 {
        write "No patient with id ", id, !
        quit $$$OK
    }

    if SQLCODE < 0 {
        write "SQL error: ", %msg, !
        quit $$$ERROR($$$GeneralError, %msg)
    }

    write record, " | ", name, " | active=", active, !
    quit $$$OK
}

/// Cursor: walks every exam of one patient.
ClassMethod ExamsOfPatient(patientId As %String) As %Integer
{
    new SQLCODE, code, value, unit, collected

    set count = 0

    &sql(DECLARE ExamCur CURSOR FOR
         SELECT TestCode, ResultValue, Unit, CollectedOn
         INTO :code, :value, :unit, :collected
         FROM LabStudy.EXAM
         WHERE Patient = :patientId
         ORDER BY TestCode)

    &sql(OPEN ExamCur)

    for {
        &sql(FETCH ExamCur)
        quit:SQLCODE'=0

        set count = count + 1
        write "  ", code, " = ", value, " ", unit, "  (", collected, ")", !
    }

    if SQLCODE < 0 {
        write "SQL error while fetching: ", %msg, !
    }

    &sql(CLOSE ExamCur)

    quit count
}

/// Dynamic SQL: the filters are decided at run time.
ClassMethod Search(namePart As %String = "", sex As %String = "", onlyActive As %Boolean = 1) As %Integer
{
    set sql = "SELECT %ID AS Id, Name, RecordNumber, Sex FROM LabStudy.PATIENT WHERE 1 = 1"
    set args = 0

    if namePart '= "" {
        set sql = sql_" AND Name %STARTSWITH ?"
        set args = args + 1
        set arg(args) = namePart
    }

    if sex '= "" {
        set sql = sql_" AND Sex = ?"
        set args = args + 1
        set arg(args) = sex
    }

    if onlyActive {
        set sql = sql_" AND Active = 1"
    }

    set sql = sql_" ORDER BY Name"

    set stmt = ##class(%SQL.Statement).%New()
    set sc = stmt.%Prepare(sql)
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    set rs = stmt.%Execute(arg...)

    set found = 0
    while rs.%Next() {
        set found = found + 1
        write rs.%Get("Id"), " | ", rs.%Get("RecordNumber"), " | ", rs.%Get("Name"), " | ", rs.%Get("Sex"), !
    }

    if rs.%SQLCODE < 0 {
        write "SQL error: ", rs.%Message, !
        quit -1
    }

    quit found
}

/// Named query, reusable and callable from SQL.
Query TopExams(howMany As %Integer = 10) As %SQLQuery(ROWSPEC = "TestCode:%String,Total:%Integer") [ SqlProc ]
{
SELECT TOP :howMany TestCode, COUNT(*) AS Total
FROM LabStudy.EXAM
GROUP BY TestCode
ORDER BY COUNT(*) DESC
}

/// Runs the named query above.
ClassMethod ShowTopExams(howMany As %Integer = 10) As %Status
{
    set rs = ..TopExamsFunc(howMany)

    write "Most frequent exams:", !
    while rs.%Next() {
        write "  ", rs.%Get("TestCode"), ": ", rs.%Get("Total"), !
    }
    quit $$$OK
}

}
```

Comentando as decisões:

- **`new SQLCODE, ...` no início dos métodos com SQL embutido.** `SQLCODE` e `%msg` são variáveis compartilhadas pelo processo. Sem o `NEW`, um método pode sujar o `SQLCODE` de quem o chamou, ou ler um valor deixado por outra operação. Este é um hábito profissional que quase ninguém ensina e que evita bugs muito difíceis de encontrar.
- **`CountPatients` distingue erro de "sem linhas".** Com `COUNT(*)`, `SQLCODE = 100` não ocorre na prática, mas o `+$GET(total)` protege contra qualquer surpresa e força o resultado a numérico.
- **`PatientSummary` trata os três casos de `SQLCODE` separadamente**, e o caso `100` produz uma mensagem amigável, não um erro. Isso é o item 2.4 posto em prática.
- **No cursor, o `INTO` está na declaração, não no `FETCH`.** Erro comum é tentar pôr o `INTO` no `FETCH`.
- **O `CLOSE` vem depois da verificação de erro**, mas antes do `QUIT` do método. Cursor não fechado é vazamento de recurso.
- **`Search` monta o texto, mas nunca cola valores.** Repare bem: os **filtros** são montados dinamicamente (isso é estrutura, decidida pelo programa), mas os **valores** viajam sempre por `?`. Essa é exatamente a linha divisória entre SQL dinâmico legítimo e injeção de SQL.
- **`WHERE 1 = 1`** é um truque de montagem: garante que todo filtro adicional possa ser acrescentado com ` AND `, sem precisar saber se é o primeiro. Custo zero para o otimizador, e o código fica bem mais simples.
- **`stmt.%Execute(arg...)`** usa a passagem de argumentos variáveis do Capítulo 3, na direção contrária: o array `arg` é **desmontado** em argumentos separados. Isso permite passar um número variável de parâmetros sem escrever um `if` para cada quantidade possível.
- **`TopExams` usa `TOP :howMany`** — um parâmetro no `TOP`, o que nem todo banco permite.
- **`ShowTopExams` chama `TopExamsFunc`**, o método gerado pelo compilador a partir da query.

### 4.1 Usando no Terminal

```
LABSTUDY>WRITE ##class(LabStudy.Demo.Reports).CountPatients(), !
2

LABSTUDY>DO ##class(LabStudy.Demo.Reports).PatientSummary(2)
REG-000002 | Bruno Lima | active=1

LABSTUDY>DO ##class(LabStudy.Demo.Reports).PatientSummary(999)
No patient with id 999

LABSTUDY>WRITE ##class(LabStudy.Demo.Reports).ExamsOfPatient(2), " exams", !
  CHOL = 190 mg/dL  (2026-08-19 16:22:10)
  GLU = 92 mg/dL  (2026-08-19 16:10:03)
  TRIG = 150 mg/dL  (2026-08-19 16:22:10)
3 exams

LABSTUDY>WRITE ##class(LabStudy.Demo.Reports).Search("Bru"), " found", !
2 | REG-000002 | Bruno Lima | M
1 found

LABSTUDY>WRITE ##class(LabStudy.Demo.Reports).Search(, "M"), " found", !
2 | REG-000002 | Bruno Lima | M
1 found

LABSTUDY>WRITE ##class(LabStudy.Demo.Reports).Search(), " found", !
2 | REG-000002 | Bruno Lima | M
1 found

LABSTUDY>DO ##class(LabStudy.Demo.Reports).ShowTopExams(3)
Most frequent exams:
  CHOL: 1
  GLU: 1
  TRIG: 1
```

O que observar:

- **`PatientSummary(999)` não deu erro.** Deu a resposta correta: não existe. Compare com o que aconteceria se o código tratasse `SQLCODE '= 0` como erro.
- **`Search` funcionou com um filtro, com outro, e sem nenhum.** O mesmo método, três consultas diferentes montadas em tempo de execução — e nenhum valor colado no texto.
- **A chamada `Search(, "M")`** deixa o primeiro argumento vazio e passa o segundo. A vírgula solta é sintaxe válida do ObjectScript para pular argumentos.

### 4.2 A sintaxe de seta em ação

No Portal, em **System Explorer → SQL**:

```sql
SELECT
    e.TestCode,
    e.ResultValue,
    e.Unit,
    e.Patient->Name AS PatientName,
    e.Patient->RecordNumber AS Record
FROM LabStudy.EXAM e
ORDER BY e.Patient->Name, e.TestCode
```

Sem escrever `JOIN` nenhum, a consulta trouxe colunas das duas tabelas. O IRIS sabe seguir a referência porque o exame guarda o ID do paciente — exatamente o que você viu na global no Capítulo 8.

O equivalente em SQL padrão seria:

```sql
SELECT e.TestCode, e.ResultValue, e.Unit, p.Name, p.RecordNumber
FROM LabStudy.EXAM e
JOIN LabStudy.PATIENT p ON e.Patient = p.ID
ORDER BY p.Name, e.TestCode
```

As duas funcionam e produzem o mesmo resultado. A primeira é mais curta e mais legível quando o caminho é uma referência simples; a segunda é portável para outros bancos. Escolha conforme o contexto.

---

## 5. Variações e detalhes

### 5.1 Reutilizando um comando preparado

Preparar custa; executar é barato. Quando você vai rodar a mesma consulta muitas vezes com valores diferentes, prepare **uma vez** e execute várias:

```objectscript
set stmt = ##class(%SQL.Statement).%New()
set sc = stmt.%Prepare("SELECT Name FROM LabStudy.PATIENT WHERE Sex = ?")

for sex = "M", "F" {
    set rs = stmt.%Execute(sex)
    while rs.%Next() {
        write sex, ": ", rs.%Get("Name"), !
    }
}
```

O IRIS mantém um **cache de consultas** que já ameniza isso automaticamente, mas preparar uma vez explicitamente é sempre melhor dentro de um laço.

### 5.2 Lendo colunas por posição

Além de `%Get("Nome")`, existe o acesso por número:

```objectscript
while rs.%Next() {
    write rs.%GetData(1), " ", rs.%GetData(2), !
}
```

Acesso por nome é mais legível e sobrevive a mudanças na ordem das colunas. Prefira nome, salvo em código genérico que não conhece a consulta.

Para descobrir as colunas em tempo de execução, o objeto de resultado expõe metadados:

```objectscript
set count = rs.%ResultColumnCount
```

Os detalhes completos da API de metadados: **verificar na documentação oficial**.

### 5.3 Objetos e SQL na mesma operação

Um padrão muito comum e muito bom: **use SQL para achar, objetos para alterar.**

```objectscript
set rs = ##class(%SQL.Statement).%ExecDirect(,
    "SELECT %ID AS Id FROM LabStudy.PATIENT WHERE Active = 1 AND Sex = ?", "F")

while rs.%Next() {
    set patient = ##class(LabStudy.Patient).%OpenId(rs.%Get("Id"))
    continue:'$ISOBJECT(patient)

    do patient.AddAllergy("test")
    set sc = patient.%Save()
}
```

Por que isso é bom: a busca aproveita índices e o otimizador; a alteração passa pelos callbacks, pelas validações e pela auditoria que você construiu nos capítulos anteriores.

E o contraponto: se você fizesse `UPDATE` direto por SQL, seria muito mais rápido, mas **pularia os callbacks**. Para uma correção em massa de um milhão de linhas, essa pode ser exatamente a escolha certa — desde que consciente.

### 5.4 Transações e SQL

O SQL embutido participa das transações do ObjectScript normalmente:

```objectscript
tstart
&sql(UPDATE LabStudy.PATIENT SET Active = 0 WHERE ID = :id)
if SQLCODE < 0 {
    trollback
    quit
}
tcommit
```

Você pode misturar livremente `TSTART`/`TCOMMIT` com operações de SQL e de objetos na mesma transação. É a mesma transação, sobre os mesmos dados.

### 5.5 `%ROWCOUNT` em cursores

Depois de um laço de `FETCH`, `%ROWCOUNT` traz quantas linhas foram efetivamente lidas:

```objectscript
&sql(CLOSE ExamCur)
write "linhas percorridas: ", %ROWCOUNT, !
```

Isso poupa você de manter um contador manual — embora manter o seu próprio contador seja mais explícito e imune a surpresas de escopo.

### 5.6 Consultas que devolvem listas

Quando uma coluna guarda uma coleção (`list Of`), o SQL a projeta como uma tabela filha separada, como visto no Capítulo 2. Para consultá-la:

```sql
SELECT p.Name, a.Allergies
FROM LabStudy.PATIENT p, LabStudy.PATIENT_Allergies a
WHERE a.PATIENT = p.ID
```

O nome exato da tabela filha segue um padrão derivado do nome da classe e da propriedade: **verificar na documentação oficial** ou consultar a lista de tabelas no Portal, que é o caminho mais rápido.

### 5.7 Quando SQL é a escolha errada

Para equilibrar o capítulo: nem tudo deve virar SQL.

- **Abrir um registro pelo ID** — `%OpenId()` é mais direto e mais rápido que um `SELECT`.
- **Aplicar regras de negócio a um registro** — objetos, com validação e callbacks.
- **Percorrer uma estrutura de global que não é uma tabela** — `$ORDER`, como no Capítulo 8.
- **Contar algo trivial que você já tem na mão** — `patient.Exams.Count()` não precisa de consulta.

SQL brilha em **conjuntos**: filtro, ordenação, agrupamento, agregação, junção. Fora disso, frequentemente há um caminho melhor.

---

## 6. Pegadinhas e erros comuns

**1) Tratar `SQLCODE = 100` como erro.**
`100` significa "não há (mais) linhas". É resposta normal, não falha.

**2) Não conferir `SQLCODE` de jeito nenhum.**
As variáveis de saída ficam com lixo e o programa segue como se tivesse dado certo.

**3) Não fazer `NEW SQLCODE` num método com SQL embutido.**
`SQLCODE` é compartilhado no processo; sem o `NEW`, o método suja o valor de quem o chamou.

**4) Esquecer o `CLOSE` do cursor.**
Vazamento de recurso. Feche em todos os caminhos de saída.

**5) Pôr o `INTO` no `FETCH` em vez do `DECLARE`.**
No IRIS, o `INTO` do cursor vai na declaração.

**6) Usar `SELECT ... INTO` sem cursor esperando várias linhas.**
Essa forma serve para **uma** linha. Para várias, cursor ou SQL dinâmico.

**7) Concatenar valores no texto da consulta.**
Injeção de SQL, além de entupir o cache de consultas. Use `?`.

**8) Achar que o SQL dinâmico não pode ter estrutura variável.**
Pode: monte o texto da estrutura. O que nunca se cola são os **valores**.

**9) Confundir o método gerado por uma class query.**
Para a query `Foo`, o método gerado é **`FooFunc()`**.

**10) Esquecer o `ROWSPEC` numa class query.**
Sem ele, quem chama não sabe os nomes nem os tipos das colunas.

**11) Estranhar que a data saia como número.**
Modo lógico devolve o valor interno. Use `%SelectMode = 1` (ODBC) ou converta.

**12) Achar que `UPDATE` de SQL dispara `%OnBeforeSave`.**
Não dispara. Dispara os **triggers**, se tiverem `Foreach = row` ou `row/object`.

**13) Colocar uma função pesada dentro de um `SELECT`.**
Ela roda **uma vez por linha**. Funções em consulta devem ser leves.

**14) Usar `LIKE 'x%'` onde `%STARTSWITH` serve.**
`%STARTSWITH` é reconhecido pelo otimizador como busca por faixa em índice.

**15) Achar que `->` funciona em qualquer coluna.**
A seta segue **referências a outros objetos**. Numa coluna comum, não faz sentido.

**16) Preparar a mesma consulta dentro de um laço.**
Prepare fora, execute dentro.

---

## 7. MÃO NA MASSA

> Antes de começar, garanta que há dados: rode `DO ##class(LabStudy.App).Status()` e, se estiver vazio, recrie alguns pacientes e exames com `CreateWithExams`.

---

### Exercício 9.1 — SQL embutido de uma linha

**a) Enunciado:** Crie `LabStudy.Demo.Sql1` com:

1. `ClassMethod Total() As %Integer` — conta pacientes com `COUNT(*)`.
2. `ClassMethod NameOf(id) As %String` — devolve o nome de um paciente, ou `""` se não existir, distinguindo os três casos de `SQLCODE`.
3. `ClassMethod Stats(Output totalPatients, Output totalExams, Output avgResult)` — traz três agregações, cada uma numa consulta.
4. `ClassMethod Deactivate(id) As %Integer` — faz um `UPDATE` e devolve `%ROWCOUNT`.

Teste com um ID válido e um inválido.

**b) Dica:** Use `new SQLCODE` no início de cada método. Para a média, `AVG(ResultValue)`.

**c) Como testar:** `NameOf(999)` deve devolver vazio sem mensagem de erro. `Deactivate(999)` deve devolver `0`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Sql1.cls`:

```objectscript
/// Single row embedded SQL.
Class LabStudy.Demo.Sql1 Extends %RegisteredObject
{

/// Counts all patients.
ClassMethod Total() As %Integer
{
    new SQLCODE, %msg, total

    &sql(SELECT COUNT(*) INTO :total FROM LabStudy.PATIENT)

    if SQLCODE < 0 {
        write "SQL error: ", %msg, !
        quit -1
    }

    quit +$GET(total)
}

/// Name of one patient, or "" when not found.
ClassMethod NameOf(id As %String) As %String
{
    new SQLCODE, %msg, name

    &sql(SELECT Name INTO :name FROM LabStudy.PATIENT WHERE ID = :id)

    if SQLCODE = 100 {
        quit ""
    }

    if SQLCODE < 0 {
        write "SQL error: ", %msg, !
        quit ""
    }

    quit name
}

/// Three aggregations at once, returned through Output parameters.
ClassMethod Stats(Output totalPatients As %Integer, Output totalExams As %Integer, Output avgResult As %Numeric) As %Status
{
    new SQLCODE, %msg, p, e, a

    set totalPatients = 0
    set totalExams = 0
    set avgResult = 0

    &sql(SELECT COUNT(*) INTO :p FROM LabStudy.PATIENT)
    if SQLCODE = 0 { set totalPatients = +$GET(p) }

    &sql(SELECT COUNT(*) INTO :e FROM LabStudy.EXAM)
    if SQLCODE = 0 { set totalExams = +$GET(e) }

    &sql(SELECT AVG(ResultValue) INTO :a FROM LabStudy.EXAM)
    if SQLCODE = 0 { set avgResult = +$GET(a) }

    quit $$$OK
}

/// Deactivates one patient. Returns how many rows changed.
ClassMethod Deactivate(id As %String) As %Integer
{
    new SQLCODE, %msg, %ROWCOUNT

    &sql(UPDATE LabStudy.PATIENT SET Active = 0 WHERE ID = :id)

    if SQLCODE < 0 {
        write "SQL error: ", %msg, !
        quit -1
    }

    quit +$GET(%ROWCOUNT)
}

}
```

```
LABSTUDY>WRITE ##class(LabStudy.Demo.Sql1).Total(), !
2

LABSTUDY>WRITE "[", ##class(LabStudy.Demo.Sql1).NameOf(2), "]", !
[Bruno Lima]

LABSTUDY>WRITE "[", ##class(LabStudy.Demo.Sql1).NameOf(999), "]", !
[]

LABSTUDY>DO ##class(LabStudy.Demo.Sql1).Stats(.p, .e, .a)
LABSTUDY>WRITE p, " patients, ", e, " exams, avg ", a, !
2 patients, 3 exams, avg 110.833333

LABSTUDY>WRITE ##class(LabStudy.Demo.Sql1).Deactivate(2), !
1

LABSTUDY>WRITE ##class(LabStudy.Demo.Sql1).Deactivate(999), !
0
```

**Por que cada decisão:**

- **`new SQLCODE, %msg` em todos os métodos.** Repare que `Deactivate` também faz `new %ROWCOUNT`. Essas três variáveis são de processo; isolá-las é o que impede que um método interfira em outro. Faça o teste: tire o `NEW` de `NameOf`, chame `NameOf(999)` e depois consulte `SQLCODE` no Terminal — ele estará `100`, poluído pelo método.
- **`NameOf` devolve `""` tanto para "não existe" quanto para erro**, mas **imprime** no caso de erro. É uma decisão de projeto defensável para um método de conveniência; num método crítico, você devolveria `%Status` e o nome por `Output`.
- **`Stats` só atribui se `SQLCODE = 0`.** Como os `Output` foram zerados no início, uma consulta que falhe deixa zero em vez de lixo.
- **`+$GET(total)`** faz duas proteções numa expressão: `$GET` evita `<UNDEFINED>` se a variável nunca foi preenchida, e o `+` força interpretação numérica, garantindo que o método devolva número mesmo se vier vazio.
- **`Deactivate(999)` devolveu `0`, não erro.** `SQLCODE` valeu `100` ali — nenhuma linha correspondeu —, e isso é a resposta correta para "desative um paciente que não existe".

---

### Exercício 9.2 — Cursor

**a) Enunciado:** Crie `LabStudy.Demo.Sql2` com:

1. `ClassMethod ListPatients() As %Integer` — percorre todos os pacientes com um cursor, ordenados por nome, imprimindo ID, número de registro e nome. Devolve a contagem.
2. `ClassMethod ExamsAbove(threshold) As %Integer` — percorre exames com `ResultValue` acima de um limite, imprimindo o código do exame e **o nome do paciente**, usando a **sintaxe de seta**.
3. `ClassMethod Deactivate All Inactive...` — não; em vez disso: `ClassMethod ApplyToAll() As %Integer` — percorre todos os pacientes com cursor, abre cada um como **objeto** e chama `%Save()`, provando que dá para combinar as duas portas. Devolve quantos foram regravados.

**b) Dica:** No item 3, guarde o ID no cursor e use `%OpenId` dentro do laço.

**c) Como testar:** O item 2 deve mostrar o nome do paciente sem nenhum `JOIN` escrito.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Sql2.cls`:

```objectscript
/// Cursor based embedded SQL.
Class LabStudy.Demo.Sql2 Extends %RegisteredObject
{

/// Walks every patient, ordered by name.
ClassMethod ListPatients() As %Integer
{
    new SQLCODE, %msg, id, name, record

    set count = 0

    &sql(DECLARE PatCur CURSOR FOR
         SELECT %ID, Name, RecordNumber
         INTO :id, :name, :record
         FROM LabStudy.PATIENT
         ORDER BY Name)

    &sql(OPEN PatCur)

    for {
        &sql(FETCH PatCur)
        quit:SQLCODE'=0

        set count = count + 1
        write id, " | ", record, " | ", name, !
    }

    if SQLCODE < 0 {
        write "SQL error: ", %msg, !
    }

    &sql(CLOSE PatCur)

    quit count
}

/// Exams above a threshold, showing the patient name through the arrow syntax.
ClassMethod ExamsAbove(threshold As %Numeric = 100) As %Integer
{
    new SQLCODE, %msg, code, value, unit, patientName

    set count = 0

    &sql(DECLARE HighCur CURSOR FOR
         SELECT TestCode, ResultValue, Unit, Patient->Name
         INTO :code, :value, :unit, :patientName
         FROM LabStudy.EXAM
         WHERE ResultValue > :threshold
         ORDER BY ResultValue DESC)

    &sql(OPEN HighCur)

    for {
        &sql(FETCH HighCur)
        quit:SQLCODE'=0

        set count = count + 1
        write patientName, " -> ", code, " = ", value, " ", unit, !
    }

    &sql(CLOSE HighCur)

    quit count
}

/// Finds with SQL, updates with objects.
ClassMethod ApplyToAll() As %Integer
{
    new SQLCODE, %msg, id

    set updated = 0

    &sql(DECLARE AllCur CURSOR FOR
         SELECT %ID INTO :id FROM LabStudy.PATIENT)

    &sql(OPEN AllCur)

    for {
        &sql(FETCH AllCur)
        quit:SQLCODE'=0

        set patient = ##class(LabStudy.Patient).%OpenId(id)
        continue:'$ISOBJECT(patient)

        set sc = patient.%Save()
        if $$$ISOK(sc) {
            set updated = updated + 1
        }
    }

    &sql(CLOSE AllCur)

    quit updated
}

}
```

```
LABSTUDY>WRITE ##class(LabStudy.Demo.Sql2).ListPatients(), " patients", !
2 | REG-000002 | Bruno Lima
1 | REG-000001 | Maria Silva
2 patients

LABSTUDY>WRITE ##class(LabStudy.Demo.Sql2).ExamsAbove(100), " exams above 100", !
Bruno Lima -> CHOL = 190 mg/dL
Bruno Lima -> TRIG = 150 mg/dL
2 exams above 100

LABSTUDY>WRITE ##class(LabStudy.Demo.Sql2).ApplyToAll(), " patients resaved", !
2 patients resaved
```

**Por que cada decisão:**

- **`ORDER BY Name` mudou a ordem**: Bruno apareceu antes de Maria, embora os IDs sejam 2 e 1. Ordenação é trabalho do banco, e ele o faz aproveitando o índice `NameIdx` que criamos no Capítulo 2. Sem esse índice, seria uma ordenação em memória.
- **`Patient->Name` no `SELECT` do segundo método** trouxe uma coluna da outra tabela sem uma linha de `JOIN`. Compare mentalmente com o `JOIN` equivalente e veja quanto código foi economizado.
- **O `INTO` está na declaração dos três cursores.** Insista nisso até virar automático.
- **`ApplyToAll` é o padrão híbrido mais importante do capítulo.** O cursor faz o que o SQL faz bem — varrer o conjunto —, e o objeto faz o que o objeto faz bem — passar pelos callbacks, validações e triggers. Rode `##class(LabStudy.AuditEntry).PrintFor("PATIENT", 1)` depois e veja: a auditoria do Capítulo 6 registrou as regravações. Um `UPDATE` puro de SQL teria disparado os triggers, mas não os callbacks. Saber qual dos dois você quer é a decisão.
- **`continue:'$ISOBJECT(patient)`** protege contra uma corrida: entre o `FETCH` e o `%OpenId`, outro processo pode ter apagado o registro.

---

### Exercício 9.3 — SQL dinâmico com parâmetros

**a) Enunciado:** Crie `LabStudy.Demo.Sql3` com um `ClassMethod Find(filters)` que recebe um **`%DynamicObject`** com filtros opcionais — `namePart`, `sex`, `minAge`, `city` — e monta a consulta dinamicamente, usando `?` para todos os valores.

Requisitos:
- Nenhum valor pode aparecer colado no texto da consulta.
- O método deve funcionar com zero, um ou todos os filtros.
- Deve imprimir a consulta gerada antes de executá-la, para você conferir.
- Deve devolver a quantidade de linhas encontradas.

Depois, tente **de propósito** passar `namePart` com o valor `' OR 1=1 --` e comprove que nada de errado acontece.

**b) Dica:** Para idade mínima, filtre por `BirthDate <= ?`, calculando a data limite em ObjectScript. Para cidade, o campo do endereço embutido chama-se `Address_City`.

**c) Como testar:** O teste de injeção deve devolver zero linhas, não a tabela inteira.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Sql3.cls`:

```objectscript
/// Dynamic SQL with a variable set of filters, always parameterised.
Class LabStudy.Demo.Sql3 Extends %RegisteredObject
{

/// Searches patients using whatever filters are present in the payload.
ClassMethod Find(filters As %DynamicObject = "") As %Integer
{
    if '$ISOBJECT(filters) {
        set filters = {}
    }

    set sql = "SELECT %ID AS Id, Name, RecordNumber, Sex, Address_City AS City"
              _" FROM LabStudy.PATIENT WHERE 1 = 1"
    set n = 0

    if filters.%IsDefined("namePart") && (filters.namePart '= "") {
        set sql = sql_" AND Name %STARTSWITH ?"
        set n = n + 1
        set arg(n) = filters.namePart
    }

    if filters.%IsDefined("sex") && (filters.sex '= "") {
        set sql = sql_" AND Sex = ?"
        set n = n + 1
        set arg(n) = filters.sex
    }

    if filters.%IsDefined("minAge") && (filters.minAge '= "") {
        // born on or before this date
        set limit = $ZDATE($HOROLOG - (filters.minAge * 365.25), 3)
        set sql = sql_" AND BirthDate <= ?"
        set n = n + 1
        set arg(n) = limit
    }

    if filters.%IsDefined("city") && (filters.city '= "") {
        set sql = sql_" AND Address_City = ?"
        set n = n + 1
        set arg(n) = filters.city
    }

    set sql = sql_" ORDER BY Name"

    write "SQL : ", sql, !
    write "ARGS: ", n, !

    set stmt = ##class(%SQL.Statement).%New()
    set stmt.%SelectMode = 1

    set sc = stmt.%Prepare(sql)
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    set rs = stmt.%Execute(arg...)

    set found = 0
    while rs.%Next() {
        set found = found + 1
        write "  ", rs.%Get("Id"), " | ", rs.%Get("RecordNumber"),
              " | ", rs.%Get("Name"), " | ", rs.%Get("Sex"),
              " | ", rs.%Get("City"), !
    }

    if rs.%SQLCODE < 0 {
        write "SQL error: ", rs.%Message, !
        quit -1
    }

    write "-> ", found, " rows", !
    quit found
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Sql3).Find()
SQL : SELECT %ID AS Id, Name, RecordNumber, Sex, Address_City AS City FROM LabStudy.PATIENT WHERE 1 = 1 ORDER BY Name
ARGS: 0
  2 | REG-000002 | Bruno Lima | M |
  1 | REG-000001 | Maria Silva | F | Potirendaba
-> 2 rows

LABSTUDY>DO ##class(LabStudy.Demo.Sql3).Find({"sex":"F"})
SQL : SELECT %ID AS Id, ... WHERE 1 = 1 AND Sex = ? ORDER BY Name
ARGS: 1
  1 | REG-000001 | Maria Silva | F | Potirendaba
-> 1 rows

LABSTUDY>DO ##class(LabStudy.Demo.Sql3).Find({"namePart":"Mar","city":"Potirendaba"})
SQL : SELECT %ID AS Id, ... WHERE 1 = 1 AND Name %STARTSWITH ? AND Address_City = ? ORDER BY Name
ARGS: 2
  1 | REG-000001 | Maria Silva | F | Potirendaba
-> 1 rows

LABSTUDY>DO ##class(LabStudy.Demo.Sql3).Find({"namePart":"' OR 1=1 --"})
SQL : SELECT %ID AS Id, ... WHERE 1 = 1 AND Name %STARTSWITH ? ORDER BY Name
ARGS: 1
-> 0 rows
```

**Por que cada decisão:**

- **O último teste é o ponto do exercício.** A tentativa de injeção devolveu **zero linhas**, porque o IRIS procurou, literalmente, um paciente cujo nome começa com os caracteres `' OR 1=1 --`. O texto malicioso foi tratado como **dado**, que é o que ele é. Repare que o texto da consulta impresso é exatamente o mesmo das outras chamadas: o valor nunca chegou perto dele.
- **A estrutura varia, os valores nunca.** Cada `if` acrescenta um pedaço **fixo** de texto e empurra o valor para o array de argumentos. Essa separação é a definição prática de SQL dinâmico seguro.
- **Imprimir a consulta gerada** é uma prática de desenvolvimento excelente. Você vê exatamente o que foi montado, sem adivinhar. Em produção, isso viraria uma entrada de log condicional, não um `write`.
- **`stmt.%SelectMode = 1`** faz as datas e valores saírem em formato ODBC. Sem isso, `BirthDate`, se estivesse no `SELECT`, sairia como número.
- **`%IsDefined` combinado com `'= ""`** cobre os dois casos: campo ausente e campo presente mas vazio. Um formulário de busca costuma enviar campos vazios em vez de omiti-los.
- **`filters.minAge * 365.25`** é uma aproximação deliberada e imprecisa em anos bissextos. Está aqui para não desviar o foco do exercício; um cálculo correto de idade viria com as funções de data do Capítulo 12.

---

### Exercício 9.4 — Class query e procedimento armazenado

**a) Enunciado:** Em `LabStudy.Demo.Sql4`:

1. Declare a query `ActivePatients(sex)` que devolve `ID`, `Name` e `RecordNumber` dos pacientes ativos de um sexo, ordenados por nome, com `ROWSPEC` correto e marcada como `SqlProc`.
2. Declare a query `ExamCountByPatient()` que devolve, por paciente, o nome e a contagem de exames, ordenada da maior para a menor.
3. Escreva `ClassMethod Run()` que chama as duas e imprime os resultados.
4. Declare `ClassMethod AgeOf(id) As %Integer [ SqlProc, SqlName = PATIENT_AGE ]` que calcula a idade de um paciente.
5. No Portal, escreva uma consulta SQL que use `LabStudy.PATIENT_AGE(%ID)` como coluna, e outra que use `CALL` na query exposta.

**b) Dica:** O método gerado pela query `Foo` é `FooFunc()`. Para a contagem por paciente, agrupe por `Patient->Name`.

**c) Como testar:** As duas queries devem funcionar tanto pelo ObjectScript quanto pelo SQL do Portal.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Sql4.cls`:

```objectscript
/// Named queries and stored procedures.
Class LabStudy.Demo.Sql4 Extends %RegisteredObject
{

/// Active patients of one sex.
Query ActivePatients(sex As %String) As %SQLQuery(ROWSPEC = "ID:%Integer,Name:%String,RecordNumber:%String") [ SqlProc ]
{
SELECT %ID, Name, RecordNumber
FROM LabStudy.PATIENT
WHERE Active = 1 AND Sex = :sex
ORDER BY Name
}

/// How many exams each patient has.
Query ExamCountByPatient() As %SQLQuery(ROWSPEC = "PatientName:%String,Total:%Integer") [ SqlProc ]
{
SELECT Patient->Name AS PatientName, COUNT(*) AS Total
FROM LabStudy.EXAM
GROUP BY Patient->Name
ORDER BY COUNT(*) DESC
}

/// Age of one patient, exposed to SQL as LabStudy.PATIENT_AGE.
ClassMethod AgeOf(id As %String) As %Integer [ SqlProc, SqlName = PATIENT_AGE ]
{
    new SQLCODE, birth

    &sql(SELECT BirthDate INTO :birth FROM LabStudy.PATIENT WHERE ID = :id)

    if (SQLCODE '= 0) || (birth = "") {
        quit ""
    }

    quit $ZDATE($HOROLOG, 4) - $ZDATE(birth, 4)
}

/// Runs both queries from ObjectScript.
ClassMethod Run() As %Status
{
    write "-- active female patients --", !
    set rs = ..ActivePatientsFunc("F")
    while rs.%Next() {
        write "  ", rs.%Get("RecordNumber"), " ", rs.%Get("Name"), !
    }

    write !, "-- exams per patient --", !
    set rs2 = ..ExamCountByPatientFunc()
    while rs2.%Next() {
        write "  ", rs2.%Get("PatientName"), ": ", rs2.%Get("Total"), !
    }

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Sql4).Run()
-- active female patients --
  REG-000001 Maria Silva

-- exams per patient --
  Bruno Lima: 3
  Maria Silva: 1

LABSTUDY>WRITE ##class(LabStudy.Demo.Sql4).AgeOf(1), !
36
```

E no Portal, em **System Explorer → SQL**:

```sql
SELECT %ID, Name, LabStudy.Demo.PATIENT_AGE(%ID) AS Age
FROM LabStudy.PATIENT
ORDER BY Name
```

```sql
CALL LabStudy.Demo.Sql4_ActivePatients('F')
```

*(O nome exato do procedimento gerado a partir de uma query segue um padrão derivado do pacote, da classe e do nome da query. Confira a lista em **System Explorer → SQL → Procedures** no seu namespace, que é o caminho mais rápido e confiável.)*

**Por que cada decisão:**

- **`ROWSPEC` é o contrato da query.** Ele diz a quem chama quais colunas vêm e de que tipo. Sem ele, o `%Get("Name")` não teria como funcionar por nome.
- **`ExamCountByPatient` agrupa por `Patient->Name`.** A seta funciona também em `GROUP BY` e em `ORDER BY`, não só no `SELECT`.
- **`AgeOf` é `SqlProc` com `SqlName`.** O nome no ObjectScript segue a convenção da linguagem; o nome no SQL segue a convenção de banco. Cada mundo mantém o seu estilo.
- **Atenção ao custo de `AgeOf` na consulta do Portal.** Ela executa **uma consulta SQL própria para cada linha** da consulta externa. Com dois pacientes, é irrelevante; com dois milhões, seria catastrófico. Este é exatamente o problema da pegadinha 13, e o exercício o coloca à sua frente de propósito. A solução correta seria calcular a idade **na própria consulta**, a partir da coluna `BirthDate` que já está sendo lida, ou usar uma propriedade `SqlComputed` como você viu no Capítulo 2. Reconhecer esse padrão ruim vale mais do que memorizar a sintaxe do `SqlProc`.
- **As duas queries são `SqlProc`**, o que as torna utilizáveis por qualquer ferramenta que fale SQL com o IRIS — incluindo relatórios externos e painéis. Uma class query bem nomeada é, na prática, uma API do seu modelo de dados.

---

### Exercício 9.5 — PROJETO CONTÍNUO: versão 1.0

**a) Enunciado:** Chegou a hora de pagar as promessas. Substitua toda a varredura ingênua do projeto por SQL:

1. Em `LabStudy.Patient`, reescreva `Statistics` para usar **uma única consulta** em vez do laço com `$ORDER`.
2. Em `LabStudy.AuditEntry`, reescreva `PrintFor` para usar SQL com parâmetros, aproveitando o índice `TableRecordIdx` que declaramos no Capítulo 6 e nunca usamos.
3. Crie `LabStudy.Reports` com:
   - `Query PatientList(onlyActive)` — lista pacientes com contagem de exames;
   - `Query ExamsByCode(code)` — exames de um código, com nome do paciente, usando a seta;
   - `Query AbnormalResults(minValue)` — exames acima de um limite;
   - `ClassMethod Dashboard()` — imprime um painel com totais, os exames mais frequentes e os pacientes sem exame algum;
   - `ClassMethod SearchPatients(filters)` — busca dinâmica parametrizada, como no exercício 9.3.
4. Em `LabStudy.App`, suba para **`"1.0"`**, reescreva `Status()` para chamar o novo `Statistics`, e acrescente `ClassMethod Run()` que imprime `About()`, o `Dashboard()` e o `StorageReport()`.

**b) Dica:** Para "pacientes sem exame algum", um `LEFT JOIN` com `IS NULL` ou uma subconsulta com `NOT EXISTS`.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.App).Run()
LABSTUDY>DO ##class(LabStudy.Reports).SearchPatients({"sex":"F"})
LABSTUDY>DO ##class(LabStudy.AuditEntry).PrintFor("PATIENT", 1)
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Substitua o `Statistics` em `src/LabStudy/Patient.cls`:

```objectscript
/// Counts patients and exams with a single query.
/// Returns, through QUIT, how many patients have at least one exam.
ClassMethod Statistics(Output totalPatients As %Integer, Output totalExams As %Integer) As %Integer
{
    new SQLCODE, %msg, p, e, w

    set totalPatients = 0
    set totalExams = 0

    &sql(SELECT
            (SELECT COUNT(*) FROM LabStudy.PATIENT),
            (SELECT COUNT(*) FROM LabStudy.EXAM),
            (SELECT COUNT(DISTINCT Patient) FROM LabStudy.EXAM)
         INTO :p, :e, :w)

    if SQLCODE < 0 {
        write "SQL error: ", %msg, !
        quit 0
    }

    set totalPatients = +$GET(p)
    set totalExams = +$GET(e)
    quit +$GET(w)
}
```

Substitua o `PrintFor` em `src/LabStudy/AuditEntry.cls`:

```objectscript
/// Prints the trail of one record, using the TableRecordIdx index.
ClassMethod PrintFor(tableName As %String, recordId As %String) As %Status
{
    write "Audit trail for ", tableName, " #", recordId, !
    write "--------------------------------------------------", !

    set stmt = ##class(%SQL.Statement).%New()

    set sc = stmt.%Prepare(
        "SELECT EventTime, UserName, Operation, FieldName, OldValue, NewValue "
        _"FROM LabStudy.AUDIT_ENTRY "
        _"WHERE TableName = ? AND RecordId = ? "
        _"ORDER BY Sequence")

    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit sc
    }

    set rs = stmt.%Execute(tableName, recordId)

    set found = 0
    while rs.%Next() {
        set found = found + 1

        write rs.%Get("EventTime"), " | ", rs.%Get("UserName"), " | ", rs.%Get("Operation")

        if rs.%Get("FieldName") '= "" {
            write " ", rs.%Get("FieldName")
        }
        if (rs.%Get("OldValue") '= "") || (rs.%Get("NewValue") '= "") {
            write " | [", rs.%Get("OldValue"), "] -> [", rs.%Get("NewValue"), "]"
        }
        write !
    }

    write "--------------------------------------------------", !
    write found, " entries", !
    quit $$$OK
}
```

`src/LabStudy/Reports.cls`:

```objectscript
/// Reporting layer of the LabStudy system. Set based, SQL first.
Class LabStudy.Reports Extends %RegisteredObject
{

/// Patients with their exam count.
Query PatientList(onlyActive As %Boolean = 1) As %SQLQuery(ROWSPEC = "Id:%Integer,RecordNumber:%String,Name:%String,Sex:%String,City:%String,ExamCount:%Integer") [ SqlProc ]
{
SELECT
    p.%ID,
    p.RecordNumber,
    p.Name,
    p.Sex,
    p.Address_City,
    (SELECT COUNT(*) FROM LabStudy.EXAM e WHERE e.Patient = p.%ID)
FROM LabStudy.PATIENT p
WHERE (:onlyActive = 0) OR (p.Active = 1)
ORDER BY p.Name
}

/// All exams of one test code, with the patient name.
Query ExamsByCode(code As %String) As %SQLQuery(ROWSPEC = "Id:%Integer,PatientName:%String,RecordNumber:%String,ResultValue:%Numeric,Unit:%String") [ SqlProc ]
{
SELECT
    e.%ID,
    e.Patient->Name,
    e.Patient->RecordNumber,
    e.ResultValue,
    e.Unit
FROM LabStudy.EXAM e
WHERE e.TestCode = :code
ORDER BY e.ResultValue DESC
}

/// Exams above a numeric threshold.
Query AbnormalResults(minValue As %Numeric = 0) As %SQLQuery(ROWSPEC = "PatientName:%String,TestCode:%String,ResultValue:%Numeric,Unit:%String") [ SqlProc ]
{
SELECT
    e.Patient->Name,
    e.TestCode,
    e.ResultValue,
    e.Unit
FROM LabStudy.EXAM e
WHERE e.ResultValue > :minValue
ORDER BY e.ResultValue DESC
}

/// Prints an overview of the whole laboratory.
ClassMethod Dashboard() As %Status
{
    write "==============================", !
    write "LabStudy dashboard", !
    write "==============================", !

    set withExams = ##class(LabStudy.Patient).Statistics(.patients, .exams)
    write "Patients          : ", patients, !
    write "Exams             : ", exams, !
    write "Patients w/ exams : ", withExams, !
    write "Patients w/o exams: ", patients - withExams, !

    write !, "-- most frequent test codes --", !
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT TOP 5 TestCode, COUNT(*) AS Total "
        _"FROM LabStudy.EXAM GROUP BY TestCode ORDER BY COUNT(*) DESC")

    while rs.%Next() {
        write "  ", rs.%Get("TestCode"), ": ", rs.%Get("Total"), !
    }

    write !, "-- patients without any exam --", !
    set rs2 = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT p.RecordNumber, p.Name FROM LabStudy.PATIENT p "
        _"WHERE NOT EXISTS (SELECT 1 FROM LabStudy.EXAM e WHERE e.Patient = p.%ID) "
        _"ORDER BY p.Name")

    set none = 1
    while rs2.%Next() {
        set none = 0
        write "  ", rs2.%Get("RecordNumber"), " ", rs2.%Get("Name"), !
    }
    if none {
        write "  (none)", !
    }

    write "==============================", !
    quit $$$OK
}

/// Parameterised dynamic search over patients.
ClassMethod SearchPatients(filters As %DynamicObject = "") As %Integer
{
    if '$ISOBJECT(filters) {
        set filters = {}
    }

    set sql = "SELECT %ID AS Id, RecordNumber, Name, Sex, Address_City AS City"
              _" FROM LabStudy.PATIENT WHERE 1 = 1"
    set n = 0

    if filters.%Get("namePart", "") '= "" {
        set sql = sql_" AND Name %STARTSWITH ?"
        set n = n + 1, arg(n) = filters.namePart
    }

    if filters.%Get("sex", "") '= "" {
        set sql = sql_" AND Sex = ?"
        set n = n + 1, arg(n) = filters.sex
    }

    if filters.%Get("city", "") '= "" {
        set sql = sql_" AND Address_City = ?"
        set n = n + 1, arg(n) = filters.city
    }

    if filters.%Get("onlyActive", 1) {
        set sql = sql_" AND Active = 1"
    }

    set sql = sql_" ORDER BY Name"

    set stmt = ##class(%SQL.Statement).%New()
    set stmt.%SelectMode = 1

    set sc = stmt.%Prepare(sql)
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    set rs = stmt.%Execute(arg...)

    set found = 0
    while rs.%Next() {
        set found = found + 1
        write "  ", rs.%Get("RecordNumber"), " | ", rs.%Get("Name"),
              " | ", rs.%Get("Sex"), " | ", rs.%Get("City"), !
    }

    write "-> ", found, " patients", !
    quit found
}

}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.0";

/// Overall counters, now backed by SQL.
ClassMethod Status() As %Status
{
    set withExams = ##class(LabStudy.Patient).Statistics(.patients, .exams)
    write "Patients:            ", patients, !
    write "Exams:               ", exams, !
    write "Patients with exams: ", withExams, !
    quit $$$OK
}

/// Full system overview.
ClassMethod Run() As %Status
{
    do ..About()
    write !
    do ##class(LabStudy.Reports).Dashboard()
    write !
    do ##class(LabStudy.StorageInfo).ClassGlobals("LabStudy.Patient")
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.App).Run()
==============================
LabStudy Laboratory System
Version: 1.0
Namespace: LABSTUDY
==============================

==============================
LabStudy dashboard
==============================
Patients          : 2
Exams             : 4
Patients w/ exams : 2
Patients w/o exams: 0

-- most frequent test codes --
  CHOL: 1
  GLU: 1
  HGB: 1
  TRIG: 1

-- patients without any exam --
  (none)
==============================

Storage of LabStudy.Patient
  data   ^LabStudy.PatientD  nodes: 2
  index  ^LabStudy.PatientI  nodes: 3
  stream ^LabStudy.PatientS  nodes: 0
  next id counter: 2

LABSTUDY>DO ##class(LabStudy.Reports).SearchPatients({"sex":"F"})
  REG-000001 | Maria Silva | F | Potirendaba
-> 1 patients
```

**Por que cada decisão:**

- **`Statistics` passou de um laço com `$ORDER` sobre a global e um `%OpenId` por paciente, para uma consulta só.** Repare no que foi eliminado: a abertura de cada objeto do disco, a contagem de relacionamento objeto por objeto, e a dependência do nome interno da global (`^LabStudy.PatientD`), que era frágil e quebraria se o armazenamento fosse customizado. Ganhou-se velocidade e robustez ao mesmo tempo.
- **`COUNT(DISTINCT Patient)`** resolve, numa expressão, o que antes exigia um contador manual e uma condição dentro do laço.
- **`PrintFor` agora usa `WHERE TableName = ? AND RecordId = ?`**, que corresponde exatamente ao índice `TableRecordIdx On (TableName, RecordId)`. Aquele índice, criado no Capítulo 6 e deliberadamente ignorado, finalmente trabalha. Numa trilha de auditoria com milhões de entradas, a diferença entre varrer tudo e usar o índice é a diferença entre segundos e horas.
- **`ORDER BY Sequence` em vez de `ORDER BY EventTime`** — porque, como você descobriu no Capítulo 6, o carimbo de tempo tem resolução de um segundo e não desempata. A sequência sim.
- **As três queries de `LabStudy.Reports` são `SqlProc`.** Elas formam uma API de consulta: qualquer ferramenta de relatório externa pode chamá-las, sem precisar conhecer o modelo interno nem escrever SQL próprio. É assim que se expõe um modelo de dados de forma controlada.
- **`WHERE (:onlyActive = 0) OR (p.Active = 1)`** é um filtro opcional expresso dentro de uma consulta **estática**. É a alternativa ao SQL dinâmico quando há apenas um ou dois filtros opcionais: mais simples, e a consulta já vem analisada da compilação. Com muitos filtros, porém, essa técnica degrada o plano de execução — e aí o dinâmico volta a ser a escolha certa.
- **`NOT EXISTS` para "pacientes sem exame"** é preferível a `LEFT JOIN ... IS NULL` porque expressa a intenção diretamente e o otimizador pode parar na primeira ocorrência.
- **`SearchPatients` usa `filters.%Get("campo", padrão)`**, que resolve num só passo o que no exercício 9.3 exigiu `%IsDefined` mais comparação. Menos código, mesma proteção.
- **`set n = n + 1, arg(n) = filters.sex`** — duas atribuições num `SET`, separadas por vírgula, executadas na ordem. Idiomático e enxuto.
- **A versão 1.0 marca o fim da fase de construção.** O sistema agora tem modelo, validação, métodos de negócio, transações, auditoria, segurança, armazenamento conhecido e uma camada de consulta. Os capítulos seguintes vão refiná-lo: nulos, evolução de esquema, desempenho, e depois todo o arsenal de funções do domínio T4.

---

## 8. Quiz do capítulo

**Q1.** O que significa `SQLCODE = 100` depois de um `SELECT`?

- A) Erro de sintaxe.
- B) Cem linhas foram retornadas.
- C) Não há (mais) linhas — resultado normal, não é erro.
- D) A consulta foi cancelada.

---

**Q2.** Onde fica a mensagem de erro depois de um `SQLCODE` negativo em SQL embutido?

- A) `%msg`
- B) `%ROWCOUNT`
- C) `$ZERROR`
- D) `SQLCODE`

---

**Q3.** Como se marca uma variável do ObjectScript dentro de uma consulta embutida?

- A) Com `%` na frente.
- B) Com dois-pontos na frente: `:variavel`.
- C) Com `?`.
- D) Com `@`.

---

**Q4.** Num cursor de SQL embutido, onde deve ficar a cláusula `INTO`?

- A) No `FETCH`.
- B) No `OPEN`.
- C) No `DECLARE`.
- D) No `CLOSE`.

---

**Q5.** Qual sequência de comandos usa corretamente um cursor?

- A) `OPEN`, `DECLARE`, `FETCH`, `CLOSE`
- B) `DECLARE`, `OPEN`, `FETCH` (em laço), `CLOSE`
- C) `DECLARE`, `FETCH`, `OPEN`, `CLOSE`
- D) `PREPARE`, `EXECUTE`, `NEXT`

---

**Q6.** Em SQL dinâmico, qual método analisa a consulta e qual a executa?

- A) `%Execute` analisa, `%Prepare` executa.
- B) `%Prepare` analisa, `%Execute` executa.
- C) `%New` analisa, `%Next` executa.
- D) `%Display` faz as duas coisas.

---

**Q7.** Como se percorre um `%SQL.StatementResult`?

- A) `while rs.%Next() { ... }`
- B) `for { &sql(FETCH rs) }`
- C) `while rs.%GetNext()`
- D) `set rs = $ORDER(rs)`

---

**Q8.** Por que se deve usar `?` em vez de concatenar valores no texto da consulta?

- A) Apenas por elegância.
- B) Para evitar injeção de SQL e para aproveitar o cache de consultas.
- C) Porque o IRIS não aceita concatenação.
- D) Para permitir mais colunas.

---

**Q9.** Para a class query chamada `ByCity`, qual é o método gerado pelo compilador?

- A) `ByCity()`
- B) `ByCityFunc()`
- C) `%ByCity()`
- D) `ExecuteByCity()`

---

**Q10.** Para que serve o `ROWSPEC` de uma class query?

- A) Limitar o número de linhas.
- B) Definir os nomes e tipos das colunas devolvidas.
- C) Definir a ordenação.
- D) Definir os parâmetros de entrada.

---

**Q11.** O que faz a sintaxe `e.Patient->Name` numa consulta?

- A) Concatena duas colunas.
- B) Segue a referência ao objeto relacionado, fazendo uma junção implícita.
- C) Cria um alias.
- D) Chama um método.

---

**Q12.** Qual palavra-chave torna um método chamável de dentro do SQL?

- A) `[ SqlProc ]`
- B) `[ Final ]`
- C) `[ Private ]`
- D) `[ Calculated ]`

---

**Q13.** Com `%SelectMode = 1`, como uma coluna `%Date` é devolvida?

- A) Como número de dias (valor lógico).
- B) No formato ODBC, `AAAA-MM-DD`.
- C) No formato local de exibição.
- D) Como texto vazio.

---

**Q14.** Um `UPDATE` executado por SQL dispara o quê?

- A) Os callbacks `%OnBeforeSave` e `%OnAfterSave`.
- B) Os triggers definidos na classe, mas não os callbacks.
- C) Nem triggers nem callbacks.
- D) Ambos, sempre.

---

**Q15.** Você precisa de uma tela de busca em que o usuário escolhe quais filtros aplicar. Qual abordagem usar?

- A) SQL embutido, montando o `WHERE` com concatenação de valores.
- B) SQL dinâmico, montando a estrutura do `WHERE` em texto e passando os valores por `?`.
- C) Um cursor com todas as linhas, filtrando em ObjectScript.
- D) Uma class query com `ROWSPEC` variável.

---

**Q16.** Qual é a forma mais eficiente de expressar "o nome começa com 'Mar'"?

- A) `WHERE Name LIKE 'Mar%'`
- B) `WHERE Name %STARTSWITH 'Mar'`
- C) `WHERE $EXTRACT(Name,1,3) = 'Mar'`
- D) `WHERE Name > 'Mar'`

---

### Gabarito comentado

**Q1 — Resposta: C.**
- **C está certa:** `100` indica ausência de linhas, e é um resultado normal.
- **A está errada:** erros são valores **negativos**.
- **B está errada:** a contagem fica em `%ROWCOUNT`.
- **D está errada:** não há cancelamento implícito.

**Q2 — Resposta: A.**
- **A está certa:** `%msg` traz a mensagem descritiva do erro de SQL.
- **B está errada:** `%ROWCOUNT` conta linhas afetadas.
- **C está errada:** `$ZERROR` guarda o último erro do ObjectScript, não do SQL embutido.
- **D está errada:** `SQLCODE` traz o código, não o texto.

**Q3 — Resposta: B.**
- **B está certa:** variáveis de host levam dois-pontos: `:id`, `:name`.
- **A está errada:** `%` marca itens do sistema.
- **C está errada:** `?` é parâmetro do SQL **dinâmico**.
- **D está errada:** `@` é indireção do ObjectScript.

**Q4 — Resposta: C.**
- **C está certa:** no IRIS, o `INTO` de um cursor vai na declaração.
- **A, B e D estão erradas:** o `FETCH` apenas traz a próxima linha para as variáveis já indicadas no `DECLARE`.

**Q5 — Resposta: B.**
- **B está certa:** declarar, abrir, buscar em laço até `SQLCODE '= 0`, fechar.
- **A e C estão erradas:** a ordem quebra o funcionamento.
- **D está errada:** essa é a sequência do SQL dinâmico, com outros nomes.

**Q6 — Resposta: B.**
- **B está certa:** `%Prepare` analisa e devolve `%Status`; `%Execute` executa e devolve o objeto de resultado.
- **A está errada:** inverte os papéis.
- **C está errada:** `%New` cria o objeto de comando e `%Next` avança uma linha.
- **D está errada:** `%Display` apenas imprime um resultado já obtido.

**Q7 — Resposta: A.**
- **A está certa:** `%Next()` avança e devolve 1 enquanto houver linhas.
- **B está errada:** `FETCH` é do SQL embutido.
- **C está errada:** `%GetNext` é de estruturas dinâmicas JSON.
- **D está errada:** `$ORDER` percorre subscritos de globais.

**Q8 — Resposta: B.**
- **B está certa:** parâmetros impedem injeção e permitem que a mesma consulta preparada seja reaproveitada no cache.
- **A está errada:** os motivos são substantivos.
- **C está errada:** aceita — e é justamente por isso que o erro é possível.
- **D está errada:** não tem relação com número de colunas.

**Q9 — Resposta: B.**
- **B está certa:** o compilador gera `<NomeDaQuery>Func()`.
- **A está errada:** o nome puro é o da definição da query, não um método executável.
- **C e D estão erradas:** esses nomes não são gerados.

**Q10 — Resposta: B.**
- **B está certa:** `ROWSPEC` descreve nomes e tipos das colunas do resultado — o contrato da query.
- **A está errada:** limitar linhas é `TOP`.
- **C está errada:** ordenação é `ORDER BY`.
- **D está errada:** os parâmetros ficam na assinatura da query.

**Q11 — Resposta: B.**
- **B está certa:** a seta segue uma referência a outro objeto, produzindo a junção sem escrever `JOIN`.
- **A está errada:** concatenação é `||` ou, no ObjectScript, `_`.
- **C está errada:** alias é `AS`.
- **D está errada:** métodos são chamados pelo nome do procedimento.

**Q12 — Resposta: A.**
- **A está certa:** `SqlProc` expõe o método ao SQL; `SqlName` define o nome usado lá.
- **B, C e D estão erradas:** tratam de sobrescrita, visibilidade e cálculo de propriedade.

**Q13 — Resposta: B.**
- **B está certa:** modo `1` é ODBC, que devolve datas como `AAAA-MM-DD`.
- **A está errada:** esse é o modo `0`, lógico.
- **C está errada:** esse é o modo `2`, exibição.
- **D está errada:** o valor é devolvido normalmente.

**Q14 — Resposta: B.**
- **B está certa:** operações de SQL disparam triggers, mas não passam pelos callbacks do mundo dos objetos.
- **A está errada:** callbacks só são acionados por `%Save()`/`%DeleteId()`.
- **C está errada:** triggers são acionados.
- **D está errada:** os callbacks não são.

**Q15 — Resposta: B.**
- **B está certa:** a **estrutura** varia e é montada em texto; os **valores** vão sempre por `?`.
- **A está errada:** concatenar valores é injeção de SQL.
- **C está errada:** trazer tudo e filtrar em código desperdiça índices e memória.
- **D está errada:** `ROWSPEC` é fixo na definição da query.

**Q16 — Resposta: B.**
- **B está certa:** `%STARTSWITH` é reconhecido pelo otimizador como busca por faixa em índice.
- **A está errada:** funciona, mas o otimizador pode não aproveitar o índice tão bem.
- **C está errada:** aplicar função sobre a coluna impede o uso do índice.
- **D está errada:** a comparação simples não expressa "começa com".

---

## 9. Resumo relâmpago

1. Objetos e SQL são **duas portas para a mesma árvore de globais**. Objetos para **um** registro com regras; SQL para **conjuntos**.
2. **SQL embutido** (`&sql(...)`) é compilado junto com a classe: rápido, estrutura fixa. **SQL dinâmico** (`%SQL.Statement`) é montado em execução: flexível.
3. **Variáveis de host** levam dois-pontos: `:id`. Servem de entrada (no `WHERE`) e de saída (no `INTO`).
4. **`SQLCODE`**: `0` sucesso, `100` sem (mais) linhas — **não é erro** —, negativo é erro, com mensagem em **`%msg`**.
5. **`%ROWCOUNT`** traz o número de linhas afetadas ou percorridas.
6. Faça **`NEW SQLCODE, %msg, %ROWCOUNT`** em métodos com SQL embutido: são variáveis de processo.
7. `SELECT ... INTO` sem cursor serve para **uma** linha. Para várias, cursor.
8. Cursor: **`DECLARE`** (com o `INTO`), **`OPEN`**, **`FETCH`** em laço até `SQLCODE '= 0`, **`CLOSE`**. Sempre feche.
9. SQL dinâmico: `%New()` → `%Prepare(texto)` → `%Execute(valores...)` → `%Next()` → `%Get("Coluna")`.
10. `rs.%SQLCODE`, `rs.%Message`, `rs.%ROWCOUNT`, `rs.%Display()`.
11. **Parâmetros `?` são obrigatórios**: impedem injeção e aproveitam o cache de consultas. Monte a **estrutura**; nunca cole **valores**.
12. **Class query**: `Query Nome(args) As %SQLQuery(ROWSPEC="col:tipo,...") [ SqlProc ]`. O método gerado é **`NomeFunc()`**.
13. **`SqlProc`** expõe um método ao SQL; **`SqlName`** define o nome usado lá. Funções em `SELECT` rodam **uma vez por linha** — mantenha-as leves.
14. **Seta `->`** faz junção implícita seguindo uma referência: `e.Patient->Name`. Funciona em `SELECT`, `WHERE`, `GROUP BY` e `ORDER BY`.
15. Recursos do IRIS: **`%ID`**, **`TOP n`**, **`%STARTSWITH`**, **`%INLIST`** (com `$LISTBUILD`), **`%MATCHES`**.
16. **`%SelectMode`**: `0` lógico, `1` ODBC, `2` exibição. Lógico para lógica, ODBC para integração, exibição para tela.
17. Operações de SQL disparam **triggers**, mas **não** os callbacks (`%OnBeforeSave`).
18. Padrão híbrido recomendado: **ache com SQL, altere com objetos**.
19. Prepare **fora** do laço, execute **dentro**.

---

## 10. Cartões de memorização

**Frente:** Como se escreve SQL embutido no ObjectScript?
**Verso:** `&sql(comando)` — e-comercial, `sql`, parênteses.

**Frente:** O que significam `SQLCODE` igual a 0, 100 e negativo?
**Verso:** `0` sucesso; `100` sem (mais) linhas, que **não é erro**; negativo é erro, com a mensagem em `%msg`.

**Frente:** Como marcar uma variável do ObjectScript dentro de SQL embutido?
**Verso:** Com dois-pontos: `:variavel`. No `WHERE` é entrada; no `INTO` é saída.

**Frente:** Por que fazer `NEW SQLCODE` num método?
**Verso:** Porque `SQLCODE`, `%msg` e `%ROWCOUNT` são variáveis de processo e seriam sujadas para quem chamou.

**Frente:** Onde vai o `INTO` num cursor?
**Verso:** No `DECLARE`, não no `FETCH`.

**Frente:** Qual a sequência de um cursor?
**Verso:** `DECLARE` → `OPEN` → `FETCH` em laço até `SQLCODE '= 0` → `CLOSE`.

**Frente:** Qual a sequência do SQL dinâmico?
**Verso:** `%New()` → `%Prepare(sql)` → `%Execute(args...)` → `while rs.%Next()` → `rs.%Get("Coluna")`.

**Frente:** Como saber se um resultado dinâmico deu erro?
**Verso:** `rs.%SQLCODE` menor que zero, com a mensagem em `rs.%Message`.

**Frente:** Por que usar `?` em vez de concatenar valores?
**Verso:** Impede injeção de SQL e permite reaproveitar a consulta preparada no cache.

**Frente:** O que pode e o que não pode ser montado dinamicamente numa consulta?
**Verso:** A **estrutura** (colunas, tabelas, filtros) pode. Os **valores** nunca — eles vão por `?`.

**Frente:** Qual método é gerado a partir da class query `ByCity`?
**Verso:** `ByCityFunc()`.

**Frente:** Para que serve o `ROWSPEC`?
**Verso:** Define os nomes e tipos das colunas devolvidas pela query — é o contrato dela.

**Frente:** O que faz `e.Patient->Name`?
**Verso:** Junção implícita: segue a referência ao objeto relacionado sem escrever `JOIN`.

**Frente:** Como expor um método ObjectScript ao SQL?
**Verso:** `[ SqlProc ]`, opcionalmente com `SqlName` para o nome usado no SQL.

**Frente:** Qual o risco de usar uma função `SqlProc` dentro de um `SELECT`?
**Verso:** Ela executa **uma vez por linha**. Numa tabela grande, o custo explode.

**Frente:** O que fazem os modos 0, 1 e 2 de `%SelectMode`?
**Verso:** `0` lógico (valor interno), `1` ODBC (`AAAA-MM-DD`), `2` exibição (formato local).

**Frente:** Um `UPDATE` de SQL dispara callbacks?
**Verso:** Não. Dispara **triggers**, mas não `%OnBeforeSave` nem `%OnAfterSave`.

**Frente:** Qual é o padrão híbrido recomendado para alterações em lote?
**Verso:** Ache os IDs com SQL, abra e altere com objetos — assim as validações, callbacks e auditoria continuam valendo.

**Frente:** Qual predicado do IRIS expressa "começa com"?
**Verso:** `%STARTSWITH`, que o otimizador reconhece como busca por faixa em índice.

**Frente:** Como passar uma lista de valores para um `IN` sem colar texto?
**Verso:** `WHERE Coluna %INLIST ?`, passando um `$LISTBUILD` como parâmetro.

---

Digite CONTINUAR para o próximo capítulo.
