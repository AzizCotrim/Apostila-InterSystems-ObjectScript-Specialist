# Apostila InterSystems ObjectScript Specialist
## Capítulo 18 — T4.6 Executes Methods and Queries Objects (Executando métodos e consultando objetos)

> Ainda em **T4 — Functions & APIs**. Este capítulo junta duas coisas que na prática andam juntas: **as maneiras de chamar código** (incluindo polimorfismo, rotinas e processos em segundo plano) e **as maneiras de percorrer objetos** (extensão, class queries, `%ResultSet`).

> **Sobre os números deste capítulo.** As saídas de tempo, contagem e
> distribuição mostradas aqui vieram de uma execução específica, numa máquina
> específica, com um conjunto de dados específico. **Os seus números serão
> diferentes** — e isso é o esperado. O que deve se repetir são as
> **proporções** e as **conclusões**: se uma abordagem aparece aqui vinte vezes
> mais rápida que outra, essa relação deve se manter, ainda que os valores
> absolutos mudem. Se não se mantiver, vale investigar: ou o seu ambiente tem
> uma característica interessante, ou o teste não está medindo o que parece.

---

## 1. O que você vai saber fazer ao terminar

1. Escolher entre **`##class()`**, **`..`**, **`##super()`** e o despacho dinâmico.
2. Entender **polimorfismo**: o mesmo nome de método executando código diferente conforme a classe real.
3. Descobrir a classe de um objeto e testar herança com **`$CLASSNAME`**, **`%ClassName()`** e **`%Extends()`**.
4. Chamar código de **rotinas**: `DO ^Rotina`, `DO rótulo^Rotina` e funções extrínsecas **`$$rótulo^Rotina`**.
5. Disparar processamento em segundo plano com o comando **`JOB`** e comunicar resultados por global.
6. Percorrer **todos os objetos de uma classe** com a query **`Extent`**.
7. Executar class queries com **`%PrepareClassQuery`** e com a API antiga **`%ResultSet`**.
8. Escolher conscientemente entre percorrer **objetos** e percorrer **SQL**.
9. Combinar consulta com abertura de objeto sem cair no problema N+1.
10. Levar o projeto à versão **1.9**, com uma hierarquia polimórfica de exames e um processamento em segundo plano.

---

## 2. O conceito em linguagem de gente

### 2.1 Quatro maneiras de dizer "execute isto"

Você já usou todas nesta apostila. Vale colocá-las lado a lado, porque escolher errado produz erros sutis.

**`##class(Pacote.Classe).Metodo()` — chamar pelo nome da classe.**

Use quando você está **fora** da classe, ou quando quer explicitamente a implementação **daquela** classe.

**`..Metodo()` — chamar na própria classe/objeto.**

Use quando você está **dentro** da classe. E há uma vantagem que muita gente não percebe: `..Metodo()` respeita o **polimorfismo**. Se o objeto real for de uma subclasse que sobrescreveu o método, é a versão da subclasse que roda.

**`##super()` — chamar a implementação da superclasse.**

Use quando você sobrescreveu um método e quer **acrescentar** comportamento em vez de substituir.

**`$CLASSMETHOD(classe, metodo, args...)` — chamar por nome decidido em execução.**

Use quando o nome vem de configuração, de tabela de despacho ou de dados. Custa desempenho e verificação; use só quando necessário.

A regra rápida:

| Situação | Forma |
|---|---|
| estou fora da classe | `##class(...)` |
| estou dentro da classe | `..Metodo()` |
| sobrescrevi e quero estender | `##super()` |
| o nome só é conhecido em execução | `$CLASSMETHOD` / `$METHOD` |

### 2.2 Polimorfismo: o mesmo pedido, respostas diferentes

Imagine que o laboratório tem uma gaveta com fichas de vários tipos: hemograma, glicemia, exame urgente. Todas são "fichas de exame", mas cada tipo tem uma forma própria de ser **impressa**.

Você poderia escrever:

```objectscript
if tipo = "urgente" {
    do ..ImprimirUrgente(exame)
} elseif tipo = "hemograma" {
    do ..ImprimirHemograma(exame)
} elseif ...
```

E toda vez que surgisse um tipo novo, você teria que voltar ali e acrescentar mais um ramo — em **todos** os lugares que fazem esse tipo de decisão.

O polimorfismo inverte isso. Cada tipo de exame **sabe se imprimir**, e quem usa simplesmente pede:

```objectscript
write exame.Describe(), !
```

O IRIS descobre, em tempo de execução, qual é a classe real do objeto e chama a implementação certa. Um tipo novo entra no sistema **sem alterar uma linha** de quem consome.

A analogia é a de um balcão de atendimento: você entrega o pedido e diz *"imprima"*. Cada tipo de ficha tem seu próprio procedimento colado no verso. O atendente não precisa conhecer todos os tipos — ele só lê o verso da ficha que está na mão.

Você já viu isso funcionando no Capítulo 3, quando `LabStudy.UrgentExam` sobrescreveu `Describe()` e chamou `##super()`.

### 2.3 Consultar objetos: três portas

Quando você precisa **percorrer** os objetos de uma classe, existem três caminhos, e a escolha depende do que você vai fazer com eles:

**Porta 1 — a extensão (*extent*).**

Toda classe persistente tem uma query pronta chamada **`Extent`**, que devolve os IDs de todos os objetos daquela classe. É a forma "oficial" de dizer "me dê todos".

**Porta 2 — SQL.**

Você escreve a consulta que quiser, com filtros, ordenação e junções. É a porta mais poderosa para **conjuntos**.

**Porta 3 — a global de armazenamento, com `$ORDER`.**

Rápida, mas depende do nome interno da global e quebra se o armazenamento for customizado. Você usou isso em capítulos iniciais e depois substituiu por SQL — corretamente.

A regra de decisão:

> **Precisa filtrar, ordenar, agrupar ou juntar? SQL.**
> **Precisa de todos os objetos, para aplicar regras de negócio um a um? `Extent`.**
> **Precisa de um objeto conhecido pelo ID? `%OpenId`.**

E o alerta do Capítulo 12 continua valendo: **abrir objeto dentro de laço é caro**. Use objetos quando precisar das regras de negócio; use SQL quando só precisar dos dados.

### 2.4 `JOB`: mandar alguém fazer enquanto você continua

Alguns trabalhos demoram: reconstruir índices, gerar um relatório pesado, importar um arquivo grande. Se o usuário tiver que esperar na tela, a experiência é ruim.

O comando **`JOB`** cria um **processo separado** que executa o trabalho, enquanto o seu processo continua.

A analogia é a de delegar: você entrega a tarefa a um colega e volta a atender o balcão. Mas — e aqui está a parte importante — **o colega vai para outra sala**. Ele não enxerga as suas variáveis, e não pode responder falando com você; se quiser deixar um recado, tem que escrever num lugar que os dois enxerguem.

Consequências práticas do processo em segundo plano:

- **Não compartilha variáveis locais nem PPG** com quem o disparou.
- **Não escreve na sua tela** — a saída dele vai para outro lugar (ou para lugar nenhum).
- **Não devolve valor.** Para comunicar resultado, use uma **global**.
- **Tem o próprio contexto de segurança e o próprio namespace**, que precisam ser especificados se forem diferentes.

---

## 3. Chamando métodos

### 3.1 As formas básicas

```objectscript
// método de classe, de fora
do ##class(LabStudy.Patient).Show(1)

// método de instância, de fora
set p = ##class(LabStudy.Patient).%OpenId(1)
write p.DisplayName(), !

// de dentro da própria classe
quit ..Name_" ("_..RecordNumber_")"

// método de classe, de dentro
quit ..FindByRecord(rec)

// implementação da superclasse
quit "[URGENTE] "_##super()
```

Um detalhe que gera confusão: **`..Metodo()` funciona tanto para métodos de instância quanto para métodos de classe**, desde que você esteja dentro da classe. A diferença é que, num `ClassMethod`, `..` não dá acesso a `..Propriedade` — porque não há objeto atual, como visto no Capítulo 3.

### 3.2 Polimorfismo em ação

```objectscript
Class LabStudy.Demo.Base Extends %RegisteredObject
{
Method Describe() As %String
{
    quit "base"
}

Method Announce() As %String
{
    // ".." respects polymorphism: the subclass version runs
    quit "Eu sou: "_..Describe()
}
}
```

```objectscript
Class LabStudy.Demo.Child Extends LabStudy.Demo.Base
{
Method Describe() As %String
{
    quit "filho (que estende "_##super()_")"
}
}
```

```
LABSTUDY>WRITE ##class(LabStudy.Demo.Child).%New().Announce(), !
Eu sou: filho (que estende base)
```

O ponto essencial: **`Announce()` está definido na classe base e nunca foi alterado**, mas chamou a versão de `Describe()` da subclasse. É o `..` fazendo a resolução em tempo de execução.

Se `Announce()` tivesse escrito `##class(LabStudy.Demo.Base).Describe()`, ele chamaria sempre a versão da base — quebrando o polimorfismo. **Dentro de uma classe, prefira `..` justamente por isso.**

### 3.3 Descobrindo a classe e a herança

```objectscript
set exam = ##class(LabStudy.UrgentExam).%OpenId(1)

write $CLASSNAME(exam), !                    // LabStudy.UrgentExam
write exam.%ClassName(1), !                  // LabStudy.UrgentExam
write exam.%ClassName(), !                   // UrgentExam
write exam.%Extends("LabStudy.Exam"), !      // 1
write exam.%Extends("%Persistent"), !        // 1
write exam.%Extends("LabStudy.Patient"), !   // 0
write $ISOBJECT(exam), !                     // 1
```

- **`$CLASSNAME(oref)`** — nome completo da classe real do objeto.
- **`oref.%ClassName(1)`** — o mesmo; sem o argumento, devolve o nome curto.
- **`oref.%Extends("Classe")`** — testa se o objeto é daquela classe **ou de uma descendente**. É o teste correto para "isto é um exame?".
- **`$ISOBJECT(x)`** — testa se é uma OREF válida.

Um padrão útil quando você precisa tratar tipos diferentes:

```objectscript
if exam.%Extends("LabStudy.UrgentExam") {
    do ..NotifyImmediately(exam)
}
```

Mas cuidado: **se você se pega escrevendo muitos testes de tipo, provavelmente falta um método polimórfico.** Em vez de perguntar "que tipo é você?", peça "faça o que você sabe fazer".

### 3.4 Rotinas e funções extrínsecas

Além de classes, o IRIS tem **rotinas** — arquivos `.mac` com código organizado em **rótulos**. Você as encontrará em sistemas legados e em utilitários do próprio produto.

Uma rotina `MyUtil.mac`:

```objectscript
MyUtil ; utility routine

Greet(name) ; prints a greeting
 write "Olá, ", name, !
 quit

Double(n) ; extrinsic function: returns a value
 quit n * 2
```

Chamando:

```objectscript
do ^MyUtil                         // executa a rotina desde o início
do Greet^MyUtil("Maria")           // executa a partir do rótulo
write $$Double^MyUtil(21), !       // 42 -- função extrínseca
```

- **`DO ^Rotina`** — executa a rotina inteira, do começo.
- **`DO rótulo^Rotina(args)`** — executa a partir de um rótulo específico.
- **`$$rótulo^Rotina(args)`** — chama como **função extrínseca**, que **devolve um valor**. Os dois cifrões são a marca.

O `$$` é o que o Capítulo 0 prometeu explicar: **dois cifrões chamam uma função escrita numa rotina e usam o valor devolvido**.

Em código novo, prefira classes. Conheça rotinas para ler o legado e para entender chamadas a utilitários do sistema.

### 3.5 O comando `JOB`

```objectscript
job ##class(LabStudy.Reports).HeavyReport()

if $TEST {
    write "processo iniciado", !
} else {
    write "não foi possível iniciar", !
}
```

- **`JOB`** cria um processo separado que executa a chamada indicada.
- Confira **`$TEST`** logo depois: `1` se o processo foi criado, `0` se não.
- **`$ZCHILD`** contém o número do último processo criado por `JOB`.

Com parâmetros de ambiente:

```objectscript
job ##class(X).Metodo(arg):("LABSTUDY"):5
```

Os parênteses após os dois-pontos permitem especificar o **namespace** e outras características do processo; o número após o segundo dois-pontos é o **tempo limite** para conseguir criar o processo. A lista completa de parâmetros varia por versão: **verificar na documentação oficial**.

**Como o processo em segundo plano se comunica:**

```objectscript
// no processo que dispara
kill ^JobStatus(id)
set ^JobStatus(id, "status") = "iniciado"
job ##class(X).Work(id)

// no processo em segundo plano
ClassMethod Work(id As %String) As %Status
{
    set ^JobStatus(id, "status") = "executando"
    set ^JobStatus(id, "start") = ##class(LabStudy.DateTime).NowTimestamp()

    // ... trabalho ...

    set ^JobStatus(id, "status") = "concluido"
    set ^JobStatus(id, "end") = ##class(LabStudy.DateTime).NowTimestamp()
    quit $$$OK
}
```

Três cuidados essenciais:

1. **Use uma global, não PPG nem variável local** — o processo em segundo plano não as enxerga.
2. **Trate erros dentro do método** e registre-os na global. Um erro num processo em segundo plano some silenciosamente.
3. **Nunca use `WRITE` esperando ver na sua tela** — o dispositivo é outro.

---

## 4. Consultando objetos

### 4.1 A query `Extent`

Toda classe persistente ganha, de graça, uma query chamada **`Extent`** que devolve os IDs de todos os seus objetos.

Com a API moderna:

```objectscript
set stmt = ##class(%SQL.Statement).%New()
set sc = stmt.%PrepareClassQuery("LabStudy.Patient", "Extent")
if $$$ISERR(sc) { do $SYSTEM.Status.DisplayError(sc) quit }

set rs = stmt.%Execute()

while rs.%Next() {
    set id = rs.%Get("ID")
    set patient = ##class(LabStudy.Patient).%OpenId(id)
    continue:'$ISOBJECT(patient)

    // ... regras de negócio sobre o objeto ...
}
```

**`%PrepareClassQuery(classe, nomeDaQuery)`** prepara qualquer query declarada numa classe — inclusive as suas próprias, do Capítulo 9, e as geradas pelo sistema.

Quando usar `Extent` em vez de `SELECT %ID FROM ...`?

- **`Extent`** deixa claro que a intenção é "todos os objetos desta classe", sem depender do nome da tabela SQL nem do `SqlTableName`.
- **SQL** é melhor quando há filtro, ordenação ou junção.

Na prática, os dois funcionam. `Extent` é a forma canônica do mundo dos objetos.

### 4.2 A API antiga: `%ResultSet`

Você vai encontrar muito código assim:

```objectscript
set rs = ##class(%ResultSet).%New("LabStudy.Patient:Extent")

set sc = rs.Execute()
if $$$ISERR(sc) { do $SYSTEM.Status.DisplayError(sc) quit }

while rs.Next() {
    write rs.Get("ID"), !
}

do rs.Close()
```

Diferenças em relação à API moderna:

| | `%SQL.Statement` | `%ResultSet` |
|---|---|---|
| Criação | `%New()` + `%Prepare...` | `%New("Classe:Query")` |
| Execução | `%Execute(args...)` | `Execute(args...)` |
| Avanço | `%Next()` | `Next()` |
| Leitura | `%Get("Coluna")` | `Get("Coluna")` |
| Fechamento | automático | **`Close()`** recomendado |

Repare no padrão: **a API moderna usa `%` nos nomes dos métodos; a antiga, não.** Isso ajuda a identificar rapidamente qual está sendo usada.

**Em código novo, use `%SQL.Statement`.** Conheça `%ResultSet` para ler o legado — e a prova pode cobrar as duas.

### 4.3 Executando as suas próprias class queries

Recapitulando o Capítulo 9, agora com as duas APIs:

```objectscript
// forma gerada pelo compilador
set rs = ##class(LabStudy.Reports).PatientListFunc(1)
while rs.%Next() { ... }

// forma explícita, útil quando o nome da query é dinâmico
set stmt = ##class(%SQL.Statement).%New()
set sc = stmt.%PrepareClassQuery("LabStudy.Reports", "PatientList")
set rs = stmt.%Execute(1)
while rs.%Next() { ... }

// forma antiga
set rs = ##class(%ResultSet).%New("LabStudy.Reports:PatientList")
do rs.Execute(1)
while rs.Next() { ... }
do rs.Close()
```

A segunda forma é a que permite **descobrir a query em tempo de execução** — por exemplo, um relatório configurável em que o nome da query vem de uma tabela.

### 4.4 Percorrendo coleções de objetos

Já visto nos capítulos 2 e 9, mas vale consolidar num só lugar:

```objectscript
// coleção list Of
for i = 1:1:patient.Allergies.Count() {
    write patient.Allergies.GetAt(i), !
}

// coleção array Of
set key = ""
for {
    set value = patient.Values.GetNext(.key)
    quit:key=""
    write key, " = ", value, !
}

// relacionamento (lado many)
for i = 1:1:patient.Exams.Count() {
    set exam = patient.Exams.GetAt(i)
    write exam.Describe(), !
}
```

**Alerta de desempenho:** `patient.Exams` carrega os objetos relacionados do disco. Num laço sobre muitos pacientes, isso é o N+1 do Capítulo 12. Para relatórios, use SQL; para regras de negócio sobre um paciente específico, a coleção é o caminho natural.

### 4.5 Combinando: SQL para achar, objeto para agir

O padrão recomendado, repetido aqui porque é o mais importante do capítulo:

```objectscript
set rs = ##class(%SQL.Statement).%ExecDirect(,
    "SELECT %ID AS Id FROM LabStudy.EXAM "
    _"WHERE ResultStatus = 'final' AND ResultValue > ?", 300)

while rs.%Next() {
    set exam = ##class(LabStudy.Exam).%OpenId(rs.%Get("Id"))
    continue:'$ISOBJECT(exam)

    do exam.Flag()              // regra de negócio, com callbacks e validação
    do exam.%Save()
}
```

O SQL faz o **filtro** aproveitando índices; o objeto faz a **ação** passando pelas regras.

E o contraponto honesto: se a ação for uma simples atualização de campo, sem regras, um `UPDATE` puro é ordens de grandeza mais rápido — ao custo de pular os callbacks. **Decida conscientemente.**

---

## 5. Exemplo comentado

Arquivo `src/LabStudy/Demo/Dispatch.cls`:

```objectscript
/// Calling methods and querying objects.
Class LabStudy.Demo.Dispatch Extends %RegisteredObject
{

/// The four ways of calling, side by side.
ClassMethod CallingForms() As %Status
{
    write "-- from outside, class method --", !
    write "  ", ##class(LabStudy.Text).ProperName("  joao DA silva  "), !

    write !, "-- from outside, instance method --", !
    set p = ##class(LabStudy.Patient).%OpenId(1)
    if $ISOBJECT(p) {
        write "  ", p.DisplayName(), !
    }

    write !, "-- dynamic dispatch --", !
    set class = "LabStudy.Text"
    for method = "ProperName", "Slug", "OnlyDigits" {
        write "  ", $JUSTIFY(method, 12), " -> ",
              $CLASSMETHOD(class, method, "  joao DA silva 42  "), !
    }

    write !, "-- dynamic dispatch on an instance --", !
    if $ISOBJECT(p) {
        for method = "DisplayName", "MaskedEmail" {
            write "  ", $JUSTIFY(method, 14), " -> ", $METHOD(p, method), !
        }
        write "  ", $JUSTIFY("Name (prop)", 14), " -> ", $PROPERTY(p, "Name"), !
    }

    quit $$$OK
}

/// Type inspection.
ClassMethod Inspect(obj As %RegisteredObject) As %Status
{
    if '$ISOBJECT(obj) {
        write "  não é um objeto", !
        quit $$$OK
    }

    write "  $CLASSNAME          : ", $CLASSNAME(obj), !
    write "  %ClassName(1)       : ", obj.%ClassName(1), !
    write "  %ClassName()        : ", obj.%ClassName(), !

    for class = "LabStudy.Exam", "LabStudy.UrgentExam", "%Persistent", "LabStudy.Patient" {
        write "  %Extends(", $JUSTIFY(class, 20), ") : ", obj.%Extends(class), !
    }

    quit $$$OK
}

/// Walks the whole extent of a class, using the Extent query.
ClassMethod WalkExtent(className As %String, limit As %Integer = 5) As %Integer
{
    set stmt = ##class(%SQL.Statement).%New()

    set sc = stmt.%PrepareClassQuery(className, "Extent")
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    set rs = stmt.%Execute()

    set n = 0
    while rs.%Next() {
        set n = n + 1
        quit:n>limit

        write "  ", $JUSTIFY(n, 3), ": id ", rs.%Get("ID"), !
    }

    // count the rest without printing
    while rs.%Next() { set n = n + 1 }

    write "  (", n, " objects in total)", !
    quit n
}

/// The same walk with the old %ResultSet API.
ClassMethod WalkExtentOldApi(className As %String, limit As %Integer = 5) As %Integer
{
    set rs = ##class(%ResultSet).%New(className_":Extent")

    set sc = rs.Execute()
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    set n = 0
    while rs.Next() {
        set n = n + 1
        if n <= limit {
            write "  ", $JUSTIFY(n, 3), ": id ", rs.Get("ID"), !
        }
    }

    do rs.Close()

    write "  (", n, " objects in total)", !
    quit n
}

/// Runs a class query whose name is decided at run time.
ClassMethod RunQuery(className As %String, queryName As %String, arg As %String = "") As %Integer
{
    set stmt = ##class(%SQL.Statement).%New()

    set sc = stmt.%PrepareClassQuery(className, queryName)
    if $$$ISERR(sc) {
        write "  query não encontrada: ", className, ":", queryName, !
        quit -1
    }

    set rs = $SELECT(arg = "": stmt.%Execute(), 1: stmt.%Execute(arg))

    write "  colunas: ", rs.%ResultColumnCount, !

    set n = 0
    while rs.%Next() {
        set n = n + 1
        quit:n>5
    }
    write "  ", n, " linhas (mostrando no máximo 5)", !

    quit n
}

/// SQL to find, objects to act: the recommended hybrid.
ClassMethod FlagHighResults(threshold As %Numeric = 350) As %Integer
{
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id FROM LabStudy.EXAM "
        _"WHERE ResultStatus = 'final' AND ResultValue > ?", threshold)

    set n = 0
    while rs.%Next() {
        set exam = ##class(LabStudy.Exam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)

        // the object carries the business rules
        continue:'exam.IsAbnormal()

        set n = n + 1
    }

    write "  ", n, " exames acima de ", threshold, " e fora da faixa", !
    quit n
}

ClassMethod Demo() As %Status
{
    do ..CallingForms()

    write !, "-- inspecting an Exam --", !
    do ..Inspect(##class(LabStudy.Exam).%OpenId(1))

    write !, "-- inspecting an UrgentExam --", !
    set u = ##class(LabStudy.UrgentExam).%New("TROP")
    do ..Inspect(u)

    write !, "-- extent, modern API --", !
    do ..WalkExtent("LabStudy.Patient", 3)

    write !, "-- extent, old API --", !
    do ..WalkExtentOldApi("LabStudy.Patient", 3)

    write !, "-- class query by name --", !
    do ..RunQuery("LabStudy.Reports", "PatientList", 1)
    do ..RunQuery("LabStudy.Reports", "NaoExiste")

    write !, "-- hybrid: SQL finds, object decides --", !
    do ..FlagHighResults(350)

    quit $$$OK
}

}
```

### 5.1 Executando

```
LABSTUDY>DO ##class(LabStudy.Demo.Dispatch).Demo()
-- from outside, class method --
  Joao da Silva

-- from outside, instance method --
  Paciente 00001

-- dynamic dispatch --
    ProperName -> Joao da Silva 42
          Slug -> joao-da-silva-42
    OnlyDigits -> 42

-- dynamic dispatch on an instance --
     DisplayName -> Paciente 00001
     MaskedEmail ->
     Name (prop) -> Paciente 00001

-- inspecting an Exam --
  $CLASSNAME          : LabStudy.Exam
  %ClassName(1)       : LabStudy.Exam
  %ClassName()        : Exam
  %Extends(        LabStudy.Exam) : 1
  %Extends(  LabStudy.UrgentExam) : 0
  %Extends(          %Persistent) : 1
  %Extends(     LabStudy.Patient) : 0

-- inspecting an UrgentExam --
  $CLASSNAME          : LabStudy.UrgentExam
  %ClassName(1)       : LabStudy.UrgentExam
  %ClassName()        : UrgentExam
  %Extends(        LabStudy.Exam) : 1
  %Extends(  LabStudy.UrgentExam) : 1
  %Extends(          %Persistent) : 1
  %Extends(     LabStudy.Patient) : 0

-- extent, modern API --
    1: id 1
    2: id 2
    3: id 3
  (201 objects in total)

-- extent, old API --
    1: id 1
    2: id 2
    3: id 3
  (201 objects in total)

-- class query by name --
  colunas: 6
  6 linhas (mostrando no máximo 5)
  query não encontrada: LabStudy.Reports:NaoExiste

-- hybrid: SQL finds, object decides --
  27 exames acima de 350 e fora da faixa
```

O que observar:

- **O `UrgentExam` respondeu `1` para `%Extends("LabStudy.Exam")`.** Ele **é** um exame, mesmo sendo de uma subclasse. Este é o teste correto para "posso tratar isto como exame?".
- **O `Exam` respondeu `0` para `%Extends("LabStudy.UrgentExam")`.** A relação é de mão única: o filho é o pai, mas o pai não é o filho.
- **As duas APIs de `Extent` deram o mesmo total.** A escolha entre elas é de estilo e de época, não de resultado.
- **A query inexistente foi tratada**, não explodiu. `%PrepareClassQuery` devolve `%Status`, e conferi-lo é o que transforma um erro fatal numa mensagem.
- **O híbrido usou SQL para reduzir de 2000 para poucas dezenas de candidatos**, e só então abriu objetos para aplicar `IsAbnormal()` — que depende da faixa de referência guardada como lista, invisível ao SQL. **Essa divisão de trabalho é a lição prática do capítulo.**

---

## 6. Variações e detalhes

### 6.1 Quando `##class()` quebra o polimorfismo

```objectscript
// Dentro de LabStudy.Exam
Method Report() As %String
{
    // ERRADO: sempre chama a versão desta classe
    quit ##class(LabStudy.Exam).Describe()
}

Method ReportCorrect() As %String
{
    // CERTO: chama a versão da classe real do objeto
    quit ..Describe()
}
```

Se o objeto for um `UrgentExam`, o primeiro método devolve a descrição **do exame comum**, ignorando a sobrescrita. É um bug difícil de encontrar, porque o código "parece" certo.

**Regra:** dentro de uma classe, use `..` salvo quando você **quer explicitamente** a implementação de uma classe específica.

### 6.2 Métodos abstratos como contrato

```objectscript
Class LabStudy.Demo.Formatter Extends %RegisteredObject [ Abstract ]
{
Method Format(value As %String) As %String [ Abstract ]
{
}

Method FormatAll(values As %List) As %String
{
    set out = "", ptr = 0
    while $LISTNEXT(values, ptr, v) {
        set out = out_..Format($GET(v))_" "     // chama a versão da subclasse
    }
    quit out
}
}
```

A classe base define **o que** as filhas precisam saber fazer (`Format`) e já entrega o **como** de operações derivadas (`FormatAll`). Cada subclasse implementa só o essencial e ganha o resto.

Este é o padrão que sustenta boa parte das bibliotecas do próprio IRIS.

### 6.3 Criando objetos por nome

```objectscript
set className = "LabStudy.UrgentExam"
set obj = $CLASSMETHOD(className, "%New")

if '$ISOBJECT(obj) {
    write "não foi possível criar ", className, !
    quit
}
```

Isso permite fábricas configuráveis: uma tabela diz qual classe usar para cada tipo de exame, e o código cria a instância certa sem conhecer os nomes.

**Sempre valide o nome da classe** antes, contra uma lista permitida ou contra `%Dictionary.CompiledClass` — pelos mesmos motivos do Capítulo 17.

### 6.4 Verificando se uma classe existe

```objectscript
if ##class(%Dictionary.CompiledClass).%ExistsId("LabStudy.Patient") {
    write "classe existe e está compilada", !
}
```

E para um método específico:

```objectscript
if ##class(%Dictionary.CompiledMethod).%ExistsId("LabStudy.Patient||Show") {
    write "método existe", !
}
```

O identificador do método usa `Classe||Metodo`. Você já usou isso no Capítulo 17, no `SafeDispatch`.

### 6.5 Cuidados com `JOB`

Uma lista de verificação antes de disparar um processo em segundo plano:

- **O método é autossuficiente?** Ele não pode depender de nada que esteja só na memória de quem o disparou.
- **Os erros são tratados e registrados?** Um erro lá dentro não aparece em lugar nenhum por padrão.
- **Há como saber que terminou?** Grave o estado numa global.
- **Há como saber que falhou?** Estado "executando" que nunca muda é a assinatura de um processo morto.
- **O trabalho é idempotente?** Se o processo cair e for disparado de novo, o resultado deve ser o mesmo (Capítulo 11).
- **Quantos processos você vai criar?** Disparar um `JOB` por registro num laço derruba o servidor.

Para trabalho paralelo estruturado, o gerenciador de trabalho do Capítulo 12 é preferível ao `JOB` cru: ele controla quantos processos existem e espera pelo término.

---

## 7. Pegadinhas e erros comuns

**1) Usar `##class(MinhaClasse).Metodo()` dentro da própria classe.**
Quebra o polimorfismo: a versão da subclasse nunca roda. Use `..`.

**2) Esquecer `##super()` ao sobrescrever.**
Substitui em vez de estender, perdendo o comportamento da base.

**3) Achar que `%Extends` testa igualdade de classe.**
Ele testa "é desta classe **ou de uma descendente**". Para igualdade exata, compare `$CLASSNAME`.

**4) Encher o código de testes de tipo.**
Se há muitos `%Extends`, provavelmente falta um método polimórfico.

**5) Chamar um método abstrato diretamente.**
Ele não tem implementação. A subclasse precisa fornecê-la.

**6) Confundir `$$rótulo^Rotina` com `DO rótulo^Rotina`.**
`$$` chama como função e **usa o valor devolvido**; `DO` executa e descarta.

**7) Esperar que um processo `JOB` escreva na sua tela.**
Ele tem outro dispositivo. Comunique por global.

**8) Esperar que um processo `JOB` enxergue variáveis locais ou PPG.**
Não enxerga. Passe tudo por argumento ou por global.

**9) Não conferir `$TEST` depois de um `JOB`.**
Você não saberá se o processo foi criado.

**10) Não tratar erros dentro do método executado por `JOB`.**
O erro some, e o processo morre sem deixar rastro.

**11) Disparar um `JOB` por registro dentro de um laço.**
Cria centenas de processos e derruba o servidor. Use lotes ou o gerenciador de trabalho.

**12) Esquecer o `Close()` do `%ResultSet` antigo.**
Vazamento de recurso.

**13) Confundir as duas APIs de resultado.**
`%SQL.Statement` usa `%Next()`/`%Get()`; `%ResultSet` usa `Next()`/`Get()`. Misturar não compila ou não funciona.

**14) Não conferir o `%Status` de `%PrepareClassQuery`.**
Query inexistente produz erro que só aparece na execução.

**15) Abrir objetos dentro de um laço só para ler campos.**
É o N+1 do Capítulo 12. Se não há regra de negócio, leia por SQL.

**16) Percorrer `Extent` quando bastava um `SELECT` com filtro.**
Você traz tudo e descarta a maior parte. Filtre no banco.

---

## 8. MÃO NA MASSA

---

### Exercício 18.1 — As formas de chamar

**a) Enunciado:** Crie uma hierarquia de três níveis para observar o polimorfismo:

1. `LabStudy.Demo.Doc` (base) — com `Method Kind()` devolvendo `"documento"`, `Method Header()` devolvendo `"=== "_..Kind()_" ==="`, e `Method Render()` que monta cabeçalho + corpo, chamando `..Body()`.
2. `LabStudy.Demo.Doc` também define `Method Body()` devolvendo `"(vazio)"`.
3. `LabStudy.Demo.Report` estende `Doc`, sobrescreve `Kind()` e `Body()`.
4. `LabStudy.Demo.UrgentReport` estende `Report`, sobrescreve `Kind()` usando `##super()`, e sobrescreve `Body()` acrescentando um aviso ao `##super()`.
5. `ClassMethod Compare()` — cria um objeto de cada classe, chama `Render()` em todos, e mostra que o **mesmo** método da base produz saídas diferentes.
6. `ClassMethod Broken()` — uma versão de `Render()` que usa `##class(LabStudy.Demo.Doc).Body()` em vez de `..Body()`, para você ver o polimorfismo quebrar.

**b) Dica:** `Render()` deve ficar **só** na base e nunca ser sobrescrito. É ele que prova o polimorfismo.

**c) Como testar:** As três classes devem produzir saídas diferentes sem que `Render()` tenha sido alterado.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Doc.cls`:

```objectscript
/// Base document. Render() is defined here and never overridden:
/// everything it produces comes from polymorphic calls.
Class LabStudy.Demo.Doc Extends %RegisteredObject
{

Method Kind() As %String
{
    quit "documento"
}

Method Body() As %String
{
    quit "(vazio)"
}

Method Header() As %String
{
    quit "=== "_..Kind()_" ==="
}

/// Never overridden. Its output changes because ".." resolves at run time.
Method Render() As %String
{
    quit ..Header()_$CHAR(13,10)_..Body()
}

/// Wrong on purpose: hard codes the base implementation.
Method RenderBroken() As %String
{
    quit ..Header()_$CHAR(13,10)_##class(LabStudy.Demo.Doc).Body()
}

}
```

`src/LabStudy/Demo/Report.cls`:

```objectscript
/// A report. Overrides Kind and Body.
Class LabStudy.Demo.Report Extends LabStudy.Demo.Doc
{

Method Kind() As %String
{
    quit "relatorio"
}

Method Body() As %String
{
    quit "conteudo do relatorio"
}

}
```

`src/LabStudy/Demo/UrgentReport.cls`:

```objectscript
/// An urgent report. Extends rather than replaces the parent behaviour.
Class LabStudy.Demo.UrgentReport Extends LabStudy.Demo.Report
{

Method Kind() As %String
{
    quit "URGENTE: "_##super()
}

Method Body() As %String
{
    quit ##super()_$CHAR(13,10)_"*** ATENCAO: leitura imediata ***"
}

}
```

E a classe de comparação, `src/LabStudy/Demo/Poly.cls`:

```objectscript
/// Demonstrates polymorphism and how easily it can be broken.
Class LabStudy.Demo.Poly Extends %RegisteredObject
{

ClassMethod Compare() As %Status
{
    set classes = $LISTBUILD("LabStudy.Demo.Doc",
                             "LabStudy.Demo.Report",
                             "LabStudy.Demo.UrgentReport")

    write "=== Render() -- defined ONCE, in the base class ===", !, !

    set ptr = 0
    while $LISTNEXT(classes, ptr, class) {
        set obj = $CLASSMETHOD(class, "%New")
        continue:'$ISOBJECT(obj)

        write "-- ", class, " --", !
        write obj.Render(), !, !
    }

    write "=== RenderBroken() -- hard codes the base ===", !, !

    set ptr = 0
    while $LISTNEXT(classes, ptr, class) {
        set obj = $CLASSMETHOD(class, "%New")
        continue:'$ISOBJECT(obj)

        write "-- ", class, " --", !
        write obj.RenderBroken(), !, !
    }

    quit $$$OK
}

/// Shows the type information of each object.
ClassMethod Types() As %Status
{
    for class = "LabStudy.Demo.Doc", "LabStudy.Demo.Report", "LabStudy.Demo.UrgentReport" {
        set obj = $CLASSMETHOD(class, "%New")

        write $JUSTIFY(obj.%ClassName(), 14), " : "
        write "Doc=", obj.%Extends("LabStudy.Demo.Doc"), " "
        write "Report=", obj.%Extends("LabStudy.Demo.Report"), " "
        write "Urgent=", obj.%Extends("LabStudy.Demo.UrgentReport"), !
    }
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Poly).Compare()
=== Render() -- defined ONCE, in the base class ===

-- LabStudy.Demo.Doc --
=== documento ===
(vazio)

-- LabStudy.Demo.Report --
=== relatorio ===
conteudo do relatorio

-- LabStudy.Demo.UrgentReport --
=== URGENTE: relatorio ===
conteudo do relatorio
*** ATENCAO: leitura imediata ***

=== RenderBroken() -- hard codes the base ===

-- LabStudy.Demo.Doc --
=== documento ===
(vazio)

-- LabStudy.Demo.Report --
=== relatorio ===
(vazio)

-- LabStudy.Demo.UrgentReport --
=== URGENTE: relatorio ===
(vazio)

LABSTUDY>DO ##class(LabStudy.Demo.Poly).Types()
           Doc : Doc=1 Report=0 Urgent=0
        Report : Doc=1 Report=1 Urgent=0
  UrgentReport : Doc=1 Report=1 Urgent=1
```

**Por que cada resultado:**

- **`Render()` existe numa classe só e produziu três saídas diferentes.** Nenhuma subclasse o tocou. Isso é polimorfismo: o código que **consome** não muda quando surge um tipo novo.
- **`RenderBroken()` produziu o mesmo corpo `(vazio)` para os três.** O `##class(LabStudy.Demo.Doc).Body()` fixou a implementação da base, e as sobrescritas foram ignoradas. **Repare que o cabeçalho continuou variando**, porque ele usa `..Kind()`. Metade do método é polimórfica e metade não — o tipo de inconsistência que passa despercebida em revisão de código.
- **`UrgentReport.Kind()` chamou `##super()`**, e o resultado foi `"URGENTE: relatorio"` — não `"URGENTE: documento"`. O `##super()` sobe **um** nível, para `Report`, e não até a base. A cadeia de herança é respeitada.
- **`UrgentReport.Body()` acrescentou ao `##super()`** em vez de substituir. Se `Report.Body()` mudar amanhã, o urgente acompanha automaticamente.
- **A tabela de tipos mostra a relação de mão única:** `UrgentReport` é tudo; `Doc` é só `Doc`. É por isso que `%Extends` é o teste certo para "posso tratar isto como um documento?".
- **`$CLASSMETHOD(class, "%New")` criou objetos de classes decididas por uma lista.** Uma fábrica configurável em três linhas.

---

### Exercício 18.2 — Percorrendo a extensão

**a) Enunciado:** Crie `LabStudy.Demo.Walk2` com quatro formas de percorrer todos os pacientes, medindo cada uma com a bancada do Capítulo 12:

1. `ByExtentModern()` — `%PrepareClassQuery` com `Extent`, abrindo cada objeto.
2. `ByExtentOldApi()` — `%ResultSet` com `Extent`, abrindo cada objeto.
3. `BySql()` — `SELECT` trazendo as colunas necessárias, **sem** abrir objetos.
4. `ByGlobal()` — `$ORDER` sobre a global de armazenamento, **sem** abrir objetos.

Todas devem produzir o **mesmo** resultado: a soma dos comprimentos dos nomes. Compare correção e tempo.

5. `Compare()` — verifica que as quatro concordam e mede.

**b) Dica:** A soma dos comprimentos é um resultado fácil de verificar e independente de ordem.

**c) Como testar:** As quatro devem dar o mesmo número. Os tempos devem diferir bastante.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Walk2.cls`:

```objectscript
/// Four ways to walk every object of a class.
Class LabStudy.Demo.Walk2 Extends %RegisteredObject
{

/// Extent query, modern API, opening each object.
ClassMethod ByExtentModern() As %Integer
{
    set stmt = ##class(%SQL.Statement).%New()
    set sc = stmt.%PrepareClassQuery("LabStudy.Patient", "Extent")
    quit:$$$ISERR(sc) -1

    set rs = stmt.%Execute()
    set total = 0

    while rs.%Next() {
        set p = ##class(LabStudy.Patient).%OpenId(rs.%Get("ID"))
        continue:'$ISOBJECT(p)
        set total = total + $LENGTH(p.Name)
    }
    quit total
}

/// Extent query, old API, opening each object.
ClassMethod ByExtentOldApi() As %Integer
{
    set rs = ##class(%ResultSet).%New("LabStudy.Patient:Extent")
    set sc = rs.Execute()
    quit:$$$ISERR(sc) -1

    set total = 0
    while rs.Next() {
        set p = ##class(LabStudy.Patient).%OpenId(rs.Get("ID"))
        continue:'$ISOBJECT(p)
        set total = total + $LENGTH(p.Name)
    }

    do rs.Close()
    quit total
}

/// SQL bringing only what is needed. No objects opened.
ClassMethod BySql() As %Integer
{
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT Name FROM LabStudy.PATIENT")

    set total = 0
    while rs.%Next() {
        set total = total + $LENGTH(rs.%Get("Name"))
    }
    quit total
}

/// Direct global traversal. Fast, but tied to the storage layout.
ClassMethod ByGlobal() As %Integer
{
    set total = 0, id = ""
    for {
        set id = $ORDER(^LabStudy.PatientD(id), 1, row)
        quit:id=""
        continue:'$LISTVALID(row)

        // position 2 is Name, per the Storage map (chapter 8)
        set total = total + $LENGTH($LISTGET(row, 2))
    }
    quit total
}

ClassMethod Compare() As %Status
{
    write "-- correctness --", !

    set a = ..ByExtentModern()
    set b = ..ByExtentOldApi()
    set c = ..BySql()
    set d = ..ByGlobal()

    write "  extent (modern) : ", a, !
    write "  extent (old)    : ", b, !
    write "  sql             : ", c, !
    write "  global          : ", d, !
    write "  all agree       : ", $SELECT((a = b) && (b = c) && (c = d): "yes", 1: "NO <--"), !

    write !, "-- timing --", !
    do ##class(LabStudy.Bench).Run($CLASSNAME(), "ByExtentModern", 3)
    do ##class(LabStudy.Bench).Run($CLASSNAME(), "ByExtentOldApi", 3)
    do ##class(LabStudy.Bench).Run($CLASSNAME(), "BySql", 3)
    do ##class(LabStudy.Bench).Run($CLASSNAME(), "ByGlobal", 3)

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Walk2).Compare()
-- correctness --
  extent (modern) : 2814
  extent (old)    : 2814
  sql             : 2814
  global          : 2814
  all agree       : yes

-- timing --
-- LabStudy.Demo.Walk2.ByExtentModern (3 runs) --
   run 1: 0.3218s
   run 2: 0.2874s
   run 3: 0.2901s
   best 0.2874s | worst 0.3218s | avg 0.2998s
-- LabStudy.Demo.Walk2.ByExtentOldApi (3 runs) --
   best 0.3102s | worst 0.3455s | avg 0.3221s
-- LabStudy.Demo.Walk2.BySql (3 runs) --
   best 0.0118s | worst 0.0163s | avg 0.0134s
-- LabStudy.Demo.Walk2.ByGlobal (3 runs) --
   best 0.0041s | worst 0.0072s | avg 0.0051s
```

**Por que cada resultado:**

- **As quatro concordaram**, o que é o pré-requisito antes de comparar tempos — a lição do Capítulo 12.
- **As duas versões com `Extent` foram cerca de 25 vezes mais lentas que o SQL.** A diferença **não** está na query `Extent`: está no `%OpenId` dentro do laço. É o N+1, agora medido sobre o projeto real.
- **A API antiga foi ligeiramente mais lenta que a moderna**, o que é secundário: as duas fazem o mesmo trabalho pesado.
- **A varredura direta da global foi a mais rápida — e é a pior escolha.** Ela depende do nome interno da global e da **posição** da propriedade no mapa de armazenamento. Basta alguém reordenar as propriedades, ou a classe ganhar armazenamento customizado, para ela passar a ler o campo errado **sem erro nenhum**. Velocidade comprada com fragilidade.
- **A escolha correta depende do objetivo**, e é isso que o exercício ensina: para **ler dados**, SQL. Para **aplicar regras de negócio a cada objeto**, `Extent` mais `%OpenId` — o custo é o preço das validações e callbacks, e vale a pena. Para nada disso a varredura de global é recomendável em código de aplicação.

---

### Exercício 18.3 — Processamento em segundo plano

**a) Enunciado:** Crie `LabStudy.Demo.Background` com:

1. `ClassMethod Start(quantidade) As %String` — gera um identificador, prepara a global de estado, dispara um `JOB` e devolve o identificador.
2. `ClassMethod Work(jobId, quantidade) As %Status` — o método executado em segundo plano: registra início, processa em passos com `HANG`, atualiza o progresso, trata erros, registra o fim.
3. `ClassMethod Status(jobId) As %Status` — mostra o estado atual.
4. `ClassMethod Watch(jobId, segundos) As %Status` — acompanha o progresso até terminar ou esgotar o tempo.
5. `ClassMethod Cleanup()` — limpa a global de estado.
6. `ClassMethod WorkFailing(jobId)` — versão que **falha de propósito**, para você ver como um erro em segundo plano se manifesta.

**b) Dica:** Registre `$JOB` na global dentro do método de trabalho, para provar que é outro processo.

**c) Como testar:** Compare `$JOB` do seu Terminal com o registrado pelo processo.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Background.cls`:

```objectscript
/// Background processing with JOB, communicating through a global.
Class LabStudy.Demo.Background Extends %RegisteredObject
{

Parameter STATEGLOBAL = "^LabStudyJobs";

/// Starts a background job. Returns the job id, or "" on failure.
ClassMethod Start(amount As %Integer = 20) As %String
{
    set jobId = "J"_$INCREMENT(@..#STATEGLOBAL)

    // prepare the state BEFORE starting: the job may begin immediately
    set @..#STATEGLOBAL@(jobId, "status") = "starting"
    set @..#STATEGLOBAL@(jobId, "amount") = amount
    set @..#STATEGLOBAL@(jobId, "done") = 0
    set @..#STATEGLOBAL@(jobId, "startedBy") = $JOB
    set @..#STATEGLOBAL@(jobId, "requestedOn") = ##class(LabStudy.DateTime).NowTimestamp()

    job ##class(LabStudy.Demo.Background).Work(jobId, amount)

    if '$TEST {
        set @..#STATEGLOBAL@(jobId, "status") = "failed to start"
        write "  não foi possível iniciar o processo", !
        quit ""
    }

    set @..#STATEGLOBAL@(jobId, "childJob") = $ZCHILD

    write "  job ", jobId, " iniciado (processo ", $ZCHILD, ")", !
    quit jobId
}

/// The background work itself. Runs in another process.
ClassMethod Work(jobId As %String, amount As %Integer) As %Status
{
    // this process is NOT the one that called Start
    set @..#STATEGLOBAL@(jobId, "runningIn") = $JOB
    set @..#STATEGLOBAL@(jobId, "status") = "running"
    set @..#STATEGLOBAL@(jobId, "startedOn") = ##class(LabStudy.DateTime).NowTimestamp()

    try {
        for i = 1:1:amount {
            // pretend to do something useful
            hang 0.2

            set @..#STATEGLOBAL@(jobId, "done") = i
            set @..#STATEGLOBAL@(jobId, "lastUpdate") = ##class(LabStudy.DateTime).NowTimestamp()
        }

        set @..#STATEGLOBAL@(jobId, "status") = "finished"
    }
    catch e {
        // an untreated error here would vanish without a trace
        set @..#STATEGLOBAL@(jobId, "status") = "error"
        set @..#STATEGLOBAL@(jobId, "error") = e.DisplayString()
    }

    set @..#STATEGLOBAL@(jobId, "finishedOn") = ##class(LabStudy.DateTime).NowTimestamp()
    quit $$$OK
}

/// A version that fails on purpose.
ClassMethod WorkFailing(jobId As %String) As %Status
{
    set @..#STATEGLOBAL@(jobId, "runningIn") = $JOB
    set @..#STATEGLOBAL@(jobId, "status") = "running"

    try {
        set x = 1 / 0                    // <DIVIDE>
    }
    catch e {
        set @..#STATEGLOBAL@(jobId, "status") = "error"
        set @..#STATEGLOBAL@(jobId, "error") = e.DisplayString()
    }

    set @..#STATEGLOBAL@(jobId, "finishedOn") = ##class(LabStudy.DateTime).NowTimestamp()
    quit $$$OK
}

/// Starts the failing version.
ClassMethod StartFailing() As %String
{
    set jobId = "F"_$INCREMENT(@..#STATEGLOBAL)
    set @..#STATEGLOBAL@(jobId, "status") = "starting"
    set @..#STATEGLOBAL@(jobId, "startedBy") = $JOB

    job ##class(LabStudy.Demo.Background).WorkFailing(jobId)
    quit:'$TEST ""

    write "  job ", jobId, " iniciado", !
    quit jobId
}

/// Shows the current state of a job.
ClassMethod Status(jobId As %String) As %Status
{
    if '$DATA(@..#STATEGLOBAL@(jobId)) {
        write "  job desconhecido: ", jobId, !
        quit $$$OK
    }

    write "  job         : ", jobId, !
    write "  status      : ", $GET(@..#STATEGLOBAL@(jobId, "status")), !
    write "  started by  : process ", $GET(@..#STATEGLOBAL@(jobId, "startedBy")), !
    write "  running in  : process ", $GET(@..#STATEGLOBAL@(jobId, "runningIn")), !

    set done = $GET(@..#STATEGLOBAL@(jobId, "done"), 0)
    set amount = $GET(@..#STATEGLOBAL@(jobId, "amount"), 0)

    if amount > 0 {
        set pct = ##class(LabStudy.Demo.Md2).Percent(done, amount, 0)
        write "  progress    : ", done, "/", amount, "  (", pct, "%)", !
    }

    if $GET(@..#STATEGLOBAL@(jobId, "error")) '= "" {
        write "  error       : ", @..#STATEGLOBAL@(jobId, "error"), !
    }

    quit $$$OK
}

/// Follows a job until it finishes or the timeout expires.
ClassMethod Watch(jobId As %String, seconds As %Integer = 15) As %Status
{
    set deadline = $ZHOROLOG + seconds

    for {
        set status = $GET(@..#STATEGLOBAL@(jobId, "status"))
        set done = $GET(@..#STATEGLOBAL@(jobId, "done"), 0)
        set amount = $GET(@..#STATEGLOBAL@(jobId, "amount"), 0)

        set bar = $TRANSLATE($JUSTIFY("", done), " ", "#")
        write "  [", $JUSTIFY(bar, 20), "] ", done, "/", amount, "  ", status, !

        quit:status="finished"
        quit:status="error"
        quit:$ZHOROLOG>deadline

        hang 1
    }

    write !
    do ..Status(jobId)
    quit $$$OK
}

ClassMethod Cleanup() As %Status
{
    kill @..#STATEGLOBAL
    write "  estado limpo", !
    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Cleanup()

    write "meu processo: ", $JOB, !, !

    write "-- disparando um job --", !
    set id = ..Start(15)
    quit:id="" $$$OK

    write !, "-- estado logo apos disparar --", !
    do ..Status(id)

    write !, "-- acompanhando --", !
    do ..Watch(id, 20)

    write !, "-- agora um job que falha --", !
    set f = ..StartFailing()
    hang 2
    do ..Status(f)

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Background).Demo()
  estado limpo
meu processo: 12345

-- disparando um job --
  job J1 iniciado (processo 12389)

-- estado logo apos disparar --
  job         : J1
  status      : running
  started by  : process 12345
  running in  : process 12389
  progress    : 1/15  (7%)

-- acompanhando --
  [#                   ] 1/15  running
  [######              ] 6/15  running
  [###########         ] 11/15  running
  [###############     ] 15/15  finished

  job         : J1
  status      : finished
  started by  : process 12345
  running in  : process 12389
  progress    : 15/15  (100%)

-- agora um job que falha --
  job F2 iniciado
  job         : F2
  status      : error
  started by  : process 12345
  running in  : process 12401
  error       : <DIVIDE>zWorkFailing+5^LabStudy.Demo.Background.1
```

**Por que cada decisão:**

- **`startedBy` e `runningIn` são processos diferentes** — 12345 e 12389. Essa é a prova visual de que o `JOB` criou outro processo. Tudo o mais decorre disso: memória separada, dispositivo separado, contexto separado.
- **O estado é preparado ANTES do `JOB`.** O processo em segundo plano pode começar imediatamente; se a global ainda não existisse, ele poderia sobrescrever ou encontrar um estado incoerente.
- **O `$TEST` é conferido logo depois do `JOB`.** Sem isso, uma falha na criação do processo passaria despercebida e você ficaria esperando um trabalho que nunca começou.
- **Todo o trabalho está dentro de `try`/`catch`, e o erro vai para a global.** Compare com o que aconteceria sem isso: o processo morreria, o status ficaria eternamente em `"running"`, e **não haveria nenhuma mensagem em lugar nenhum**. O job que falha demonstra exatamente isso funcionando: `<DIVIDE>` foi capturado, registrado e ficou visível.
- **Status `"running"` que nunca muda é a assinatura de um processo morto.** Um sistema de produção precisa de um monitor que detecte isso — por exemplo, comparando `lastUpdate` com a hora atual.
- **`Watch` tem tempo limite.** Um acompanhamento sem limite trava o Terminal se o processo travar.
- **O `HANG 0.2` simula trabalho** e, de quebra, permite ver o progresso avançar. Num processo real, o `HANG` entre lotes é a cortesia com os outros usuários, como visto no Capítulo 12.
- **Nada foi escrito na tela pelo processo em segundo plano.** Toda a saída que você vê veio do **seu** processo, lendo a global. Se `Work` tivesse um `WRITE`, ele iria para o dispositivo padrão daquele processo — e você não veria nada.

---

### Exercício 18.4 — PROJETO CONTÍNUO: exames polimórficos e relatório em segundo plano

**a) Enunciado:** Evolua o sistema:

1. Reorganize a hierarquia de exames:
   - `LabStudy.Exam` ganha `Method Priority() As %Integer` devolvendo `100`, `Method Label() As %String` devolvendo o código, e `Method Summary() As %String` — definido **uma vez** — que monta a saída usando `..Label()`, `..ResultText()` e `..Priority()`.
   - `LabStudy.UrgentExam` sobrescreve `Priority()` (devolvendo `10`) e `Label()` (usando `##super()`).
   - Crie `LabStudy.CriticalExam Extends LabStudy.UrgentExam`, com `Priority()` = `1`, `Label()` estendendo o pai, e `Property NotifiedTo As %String`.
2. Crie `LabStudy.ExamFactory` com `ClassMethod Create(tipo, codigo)` que devolve a instância da classe certa, validando o tipo contra uma lista permitida.
3. Em `LabStudy.Reports`, acrescente `ClassMethod ByPriority()` — percorre todos os exames pela `Extent`, agrupa por prioridade usando o `Sorter`, e imprime.
4. Crie `LabStudy.BackgroundReport` com `Start()`, `Work()`, `Status()` e `Watch()`, que gera o painel completo em segundo plano e grava o resultado numa global, para ser exibido depois.
5. Suba `LabStudy.App` para `"1.9"` e acrescente a opção correspondente ao menu.

**b) Dica:** No item 1, `Summary()` deve ficar **só** em `LabStudy.Exam`. É ele que prova o polimorfismo no projeto real.

**c) Como testar:**

```
LABSTUDY>SET e = ##class(LabStudy.ExamFactory).Create("critical", "TROP")
LABSTUDY>WRITE e.Summary(), !
LABSTUDY>DO ##class(LabStudy.Reports).ByPriority()
LABSTUDY>SET id = ##class(LabStudy.BackgroundReport).Start()
LABSTUDY>DO ##class(LabStudy.BackgroundReport).Watch(id)
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Acrescente a `src/LabStudy/Exam.cls`:

```objectscript
/// Priority of this exam. Lower means more urgent.
/// Subclasses override this; nothing else needs to know they exist.
Method Priority() As %Integer
{
    quit 100
}

/// Short label. Subclasses may decorate it.
Method Label() As %String
{
    quit ..TestCode
}

/// Defined ONCE, here. Never override it: everything it shows comes
/// from polymorphic calls, so new exam types work with no changes.
Method Summary() As %String
{
    quit "["_$JUSTIFY(..Priority(), 3)_"] "_..Label()_": "_..ResultText()
}
```

Acrescente a `src/LabStudy/UrgentExam.cls` (substituindo o `Describe` antigo, ou mantendo-o):

```objectscript
Method Priority() As %Integer
{
    quit 10
}

Method Label() As %String
{
    quit "URGENTE "_##super()
}
```

`src/LabStudy/CriticalExam.cls`:

```objectscript
/// A critical exam: highest priority, and someone must be notified.
Class LabStudy.CriticalExam Extends LabStudy.UrgentExam
{

/// Who was notified about this result.
Property NotifiedTo As %String(MAXLEN = 120);

/// When the notification was sent.
Property NotifiedOn As %TimeStamp;

Method Priority() As %Integer
{
    quit 1
}

Method Label() As %String
{
    quit "CRITICO "_##super()
}

/// Records that someone was notified.
Method Notify(who As %String) As %Status
{
    if who = "" {
        quit $$$ERROR($$$GeneralError, "É obrigatório informar quem foi notificado")
    }

    set ..NotifiedTo = ##class(LabStudy.Text).Clean(who)
    set ..NotifiedOn = ##class(LabStudy.DateTime).NowTimestamp()
    quit $$$OK
}

/// A critical exam with a final result must have been notified.
Method %OnValidateObject() As %Status [ Private, ServerOnly = 1 ]
{
    set sc = ##super()
    quit:$$$ISERR(sc) sc

    if (..ResultStatus = "final") && (..NotifiedTo = "") {
        quit $$$ERROR($$$GeneralError,
            "Exame critico liberado exige notificacao registrada")
    }
    quit $$$OK
}

}
```

`src/LabStudy/ExamFactory.cls`:

```objectscript
/// Creates the right exam class for a given type.
/// The mapping lives here, so nothing else needs to know the class names.
Class LabStudy.ExamFactory Extends %RegisteredObject
{

/// type -> class name
Parameter TYPES = "normal:LabStudy.Exam,urgent:LabStudy.UrgentExam,critical:LabStudy.CriticalExam";

/// Class name for a type, or "" when the type is unknown.
ClassMethod ClassFor(type As %String) As %String
{
    set type = $ZCONVERT(##class(LabStudy.Text).Clean(type), "L")
    quit:type="" ""

    for i = 1:1:$LENGTH(..#TYPES, ",") {
        set entry = $PIECE(..#TYPES, ",", i)

        // RETURN leaves the loop AND the method: no need to inspect
        // the loop variable afterwards (chapter 17)
        return:$PIECE(entry, ":", 1) = type $PIECE(entry, ":", 2)
    }

    quit ""
}

/// Creates an exam of the requested type.
ClassMethod Create(type As %String, testCode As %String = "") As LabStudy.Exam
{
    set class = ..ClassFor(type)

    if class = "" {
        write "  tipo desconhecido: [", type, "]", !
        quit ""
    }

    if '##class(%Dictionary.CompiledClass).%ExistsId(class) {
        write "  classe não compilada: ", class, !
        quit ""
    }

    quit $CLASSMETHOD(class, "%New", testCode)
}

/// Lists the known types.
ClassMethod ListTypes() As %Status
{
    write "  tipos disponíveis:", !
    for i = 1:1:$LENGTH(..#TYPES, ",") {
        set entry = $PIECE(..#TYPES, ",", i)
        write "    ", $JUSTIFY($PIECE(entry, ":", 1), 10), " -> ", $PIECE(entry, ":", 2), !
    }
    quit $$$OK
}

}
```

Acrescente a `src/LabStudy/Reports.cls`:

```objectscript
/// Groups every exam by priority, using the Extent query and polymorphism.
ClassMethod ByPriority(limit As %Integer = 20) As %Integer
{
    kill idx

    set stmt = ##class(%SQL.Statement).%New()
    set sc = stmt.%PrepareClassQuery("LabStudy.Exam", "Extent")
    if $$$ISERR(sc) {
        do $SYSTEM.Status.DisplayError(sc)
        quit -1
    }

    set rs = stmt.%Execute()
    set total = 0

    while rs.%Next() {
        set exam = ##class(LabStudy.Exam).%OpenId(rs.%Get("ID"))
        continue:'$ISOBJECT(exam)

        set total = total + 1

        // Priority() and Summary() resolve to the real class of each object
        do ##class(LabStudy.Sorter).Add(.idx, exam.Priority(), exam.%Id(),
            $LISTBUILD(exam.%ClassName(), exam.Summary()))
    }

    write "=== exames por prioridade ===", !

    set W = $LISTBUILD(14, 40)
    set A = $LISTBUILD("L", "L")
    do ##class(LabStudy.Formatter).Header($LISTBUILD("classe", "resumo"), W, A)

    set shown = ##class(LabStudy.Sorter).TopN(.idx, limit, 0, .ordered)

    for i = 1:1:shown {
        set info = $LIST(ordered(i), 3)
        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD($LISTGET(info, 1), $LISTGET(info, 2)), W, A)
    }

    do ##class(LabStudy.Formatter).Line(56)
    write "  mostrando ", shown, " de ", total, " exames", !
    quit total
}
```

`src/LabStudy/BackgroundReport.cls`:

```objectscript
/// Generates the dashboard in a background process.
Class LabStudy.BackgroundReport Extends %RegisteredObject
{

Parameter STATEGLOBAL = "^LabStudyBgReport";

/// Starts the background generation. Returns the job id.
ClassMethod Start() As %String
{
    set jobId = "R"_$INCREMENT(@..#STATEGLOBAL)

    set @..#STATEGLOBAL@(jobId, "status") = "starting"
    set @..#STATEGLOBAL@(jobId, "startedBy") = $JOB
    set @..#STATEGLOBAL@(jobId, "requestedOn") = ##class(LabStudy.DateTime).NowTimestamp()

    job ##class(LabStudy.BackgroundReport).Work(jobId)

    if '$TEST {
        set @..#STATEGLOBAL@(jobId, "status") = "failed to start"
        quit ""
    }

    write "  relatorio ", jobId, " em geracao (processo ", $ZCHILD, ")", !
    quit jobId
}

/// Runs in the background. Collects numbers into the global.
ClassMethod Work(jobId As %String) As %Status
{
    set @..#STATEGLOBAL@(jobId, "runningIn") = $JOB
    set @..#STATEGLOBAL@(jobId, "status") = "running"
    set @..#STATEGLOBAL@(jobId, "step") = "iniciando"

    try {
        set @..#STATEGLOBAL@(jobId, "step") = "contando pacientes e exames"
        set withExams = ##class(LabStudy.Patient).Statistics(.patients, .exams)

        set @..#STATEGLOBAL@(jobId, "data", "patients") = patients
        set @..#STATEGLOBAL@(jobId, "data", "exams") = exams
        set @..#STATEGLOBAL@(jobId, "data", "withExams") = withExams

        set @..#STATEGLOBAL@(jobId, "step") = "agrupando por codigo"

        set rs = ##class(%SQL.Statement).%ExecDirect(,
            "SELECT TestCode, COUNT(*) AS Total, AVG(ResultValue) AS Average "
            _"FROM LabStudy.EXAM WHERE ResultStatus = 'final' GROUP BY TestCode")

        while rs.%Next() {
            set code = rs.%Get("TestCode")
            set @..#STATEGLOBAL@(jobId, "byCode", code, "count") = rs.%Get("Total")
            set @..#STATEGLOBAL@(jobId, "byCode", code, "avg") = rs.%Get("Average")
        }

        set @..#STATEGLOBAL@(jobId, "step") = "contando pendentes"

        set rs2 = ##class(%SQL.Statement).%ExecDirect(,
            "SELECT COUNT(*) AS N FROM LabStudy.EXAM WHERE ResultStatus = 'pending'")
        if rs2.%Next() {
            set @..#STATEGLOBAL@(jobId, "data", "pending") = rs2.%Get("N")
        }

        set @..#STATEGLOBAL@(jobId, "status") = "finished"
        set @..#STATEGLOBAL@(jobId, "step") = "concluido"
    }
    catch e {
        set @..#STATEGLOBAL@(jobId, "status") = "error"
        set @..#STATEGLOBAL@(jobId, "error") = e.DisplayString()
    }

    set @..#STATEGLOBAL@(jobId, "finishedOn") = ##class(LabStudy.DateTime).NowTimestamp()
    quit $$$OK
}

ClassMethod Status(jobId As %String) As %Status
{
    if '$DATA(@..#STATEGLOBAL@(jobId)) {
        write "  relatorio desconhecido: ", jobId, !
        quit $$$OK
    }

    write "  status : ", $GET(@..#STATEGLOBAL@(jobId, "status")), !
    write "  etapa  : ", $GET(@..#STATEGLOBAL@(jobId, "step")), !

    if $GET(@..#STATEGLOBAL@(jobId, "error")) '= "" {
        write "  erro   : ", @..#STATEGLOBAL@(jobId, "error"), !
    }
    quit $$$OK
}

/// Follows until it finishes, then shows the result.
ClassMethod Watch(jobId As %String, seconds As %Integer = 30) As %Status
{
    set deadline = $ZHOROLOG + seconds

    for {
        set status = $GET(@..#STATEGLOBAL@(jobId, "status"))
        write "  ", $GET(@..#STATEGLOBAL@(jobId, "step")), " ...", !

        quit:status="finished"
        quit:status="error"
        quit:$ZHOROLOG>deadline

        hang 1
    }

    write !
    do ..Show(jobId)
    quit $$$OK
}

/// Displays the collected result.
ClassMethod Show(jobId As %String) As %Status
{
    if $GET(@..#STATEGLOBAL@(jobId, "status")) '= "finished" {
        do ..Status(jobId)
        quit $$$OK
    }

    write "=== relatorio ", jobId, " ===", !
    write "  gerado pelo processo ", $GET(@..#STATEGLOBAL@(jobId, "runningIn")), !
    write "  em ", $GET(@..#STATEGLOBAL@(jobId, "finishedOn")), !, !

    set W = $LISTBUILD(22, 12)
    set A = $LISTBUILD("L", "R")

    for key = "patients", "exams", "withExams", "pending" {
        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(key, $GET(@..#STATEGLOBAL@(jobId, "data", key), 0)), W, A)
    }

    write !
    set W2 = $LISTBUILD(10, 8, 12)
    set A2 = $LISTBUILD("L", "R", "R")
    do ##class(LabStudy.Formatter).Header($LISTBUILD("codigo", "n", "media"), W2, A2)

    set code = ""
    for {
        set code = $ORDER(@..#STATEGLOBAL@(jobId, "byCode", code))
        quit:code=""

        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(code,
                       $GET(@..#STATEGLOBAL@(jobId, "byCode", code, "count")),
                       ##class(LabStudy.Text).Number(
                           $GET(@..#STATEGLOBAL@(jobId, "byCode", code, "avg")), 2)),
            W2, A2)
    }

    do ##class(LabStudy.Formatter).Line(32)
    quit $$$OK
}

ClassMethod Cleanup() As %Status
{
    kill @..#STATEGLOBAL
    quit $$$OK
}

}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.9";
```

Acrescente à tabela `LabStudy.Menu.Dispatch()`:

```objectscript
        "9":  "OptByPriority",
        "10": "OptBackgroundReport",
```

E os métodos correspondentes:

```objectscript
ClassMethod OptByPriority() As %Status
{
    do ##class(LabStudy.Reports).ByPriority(20)
    quit $$$OK
}

ClassMethod OptBackgroundReport() As %Status
{
    set id = ##class(LabStudy.BackgroundReport).Start()

    if id = "" {
        write "  Nao foi possivel iniciar o relatorio.", !
        quit $$$OK
    }

    if ..Confirm("  Acompanhar a geracao?") {
        do ##class(LabStudy.BackgroundReport).Watch(id, 30)
    } else {
        write "  consulte depois com BackgroundReport.Show(""", id, """)", !
    }
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.ExamFactory).ListTypes()
  tipos disponíveis:
        normal -> LabStudy.Exam
        urgent -> LabStudy.UrgentExam
      critical -> LabStudy.CriticalExam

LABSTUDY>SET p = ##class(LabStudy.Patient).%OpenId(1)

LABSTUDY>SET e1 = ##class(LabStudy.ExamFactory).Create("normal", "GLU")
LABSTUDY>SET e2 = ##class(LabStudy.ExamFactory).Create("urgent", "TROP")
LABSTUDY>SET e3 = ##class(LabStudy.ExamFactory).Create("critical", "K")
LABSTUDY>SET e4 = ##class(LabStudy.ExamFactory).Create("inexistente", "X")
  tipo desconhecido: [inexistente]

LABSTUDY>FOR i=1:1:3 { SET e=$CASE(i,1:e1,2:e2,:e3) SET e.Patient=p DO e.SetResult(7.2,"mmol/L") WRITE e.Summary(),! }
[100] GLU: 7.2 mmol/L
[ 10] URGENTE TROP: 7.2 mmol/L
[  1] CRITICO URGENTE K: 7.2 mmol/L

LABSTUDY>DO $SYSTEM.Status.DisplayError(e3.%Save())
ERROR #5001: Exame critico liberado exige notificacao registrada

LABSTUDY>DO e3.Notify("Dr. Souza")
LABSTUDY>WRITE $$$ISOK(e3.%Save()), !
1

LABSTUDY>DO ##class(LabStudy.Reports).ByPriority(10)
=== exames por prioridade ===
classe         resumo
--------------------------------------------------------
CriticalExam   [  1] CRITICO URGENTE K: 7.2 mmol/L
UrgentExam     [ 10] URGENTE TROP: 7.2 mmol/L
Exam           [100] GLU: 305 mg/dL
Exam           [100] HGB: 13.5 g/dL
...
```

**Por que cada decisão:**

- **`Summary()` está definido uma vez, em `LabStudy.Exam`, e produziu três saídas diferentes.** Nenhuma subclasse o tocou. Quando surgir um quarto tipo de exame, `Summary()` continuará funcionando — e o relatório `ByPriority` também.
- **`CriticalExam.Label()` chamou `##super()`, que chamou `##super()` de novo**, produzindo `"CRITICO URGENTE K"`. A cadeia de três níveis se compõe sozinha. Se `UrgentExam` mudar o prefixo amanhã, o crítico acompanha.
- **`%OnValidateObject` de `CriticalExam` chama `##super()` primeiro.** Sem isso, as validações herdadas de `LabStudy.Exam` — como a coerência entre estado e valor, do Capítulo 10 — seriam **perdidas**. Esquecer o `##super()` num callback sobrescrito é um erro sutil e grave, porque o sistema continua funcionando e só as regras da subclasse valem.
- **O exame crítico foi recusado até haver notificação registrada.** Uma regra de negócio que só existe naquele tipo, sem que ninguém fora dele precise saber disso.
- **A fábrica concentra o mapeamento tipo → classe num `Parameter`.** Acrescentar um tipo é acrescentar uma entrada e criar a classe. Nenhum `if` espalhado pelo sistema.
- **`ClassFor` usa `RETURN` de dentro do laço**, e não `QUIT` seguido de leitura da variável de controle. A diferença importa: como você viu no Capítulo 17, depois de um `QUIT` a variável do laço fica com o valor da parada, e depois de um fim natural fica com um a mais — descobrir qual dos dois aconteceu exige uma comparação extra e frágil. Com `RETURN`, a resposta sai no ponto em que foi encontrada, e o `quit ""` final cobre o caso de não encontrar nada.
- **A fábrica valida contra a lista permitida e contra o dicionário** antes de instanciar — as duas defesas do Capítulo 17.
- **`ByPriority` percorre a `Extent` de `LabStudy.Exam` e recebe objetos das subclasses.** É o `%OpenId` da classe base devolvendo a instância da classe **real** — o IRIS sabe qual é, porque grava o nome da classe na primeira posição do nó de dados, como você viu no Capítulo 8. **Aquela posição vazia que parecia inútil é o que sustenta o polimorfismo do armazenamento.**
- **O relatório em segundo plano grava resultados estruturados na global**, não texto formatado. Assim, quem exibe decide o formato — e o mesmo resultado pode virar tela, arquivo ou JSON. **Separar coleta de apresentação** é o que torna um relatório reutilizável.
- **O menu oferece acompanhar ou não.** O usuário que não quer esperar recebe o identificador e consulta depois. É o benefício concreto do processamento em segundo plano.

---

## 9. Quiz do capítulo

**Q1.** Dentro de uma classe, qual forma de chamada respeita o polimorfismo?

- A) `##class(MinhaClasse).Metodo()`
- B) `..Metodo()`
- C) `$CLASSMETHOD("MinhaClasse", "Metodo")`
- D) `##super()`

---

**Q2.** O que `##super()` chama?

- A) A implementação da própria classe.
- B) A implementação da superclasse imediata do mesmo método.
- C) A implementação da classe base da hierarquia.
- D) Um método de nome `super`.

---

**Q3.** `obj.%Extends("LabStudy.Exam")` devolve `1` quando o objeto é:

- A) Exatamente da classe `LabStudy.Exam`.
- B) Da classe `LabStudy.Exam` **ou de uma descendente**.
- C) Da superclasse de `LabStudy.Exam`.
- D) De qualquer classe persistente.

---

**Q4.** Você sobrescreveu `%OnValidateObject()` numa subclasse e não chamou `##super()`. O que acontece?

- A) Nada de especial.
- B) As validações herdadas da superclasse deixam de ser aplicadas.
- C) Erro de compilação.
- D) A subclasse não pode ser salva.

---

**Q5.** O que `$$Rotulo^Rotina(args)` faz?

- A) Executa a rotina e descarta o retorno.
- B) Chama como **função extrínseca** e usa o valor devolvido.
- C) Cria um processo em segundo plano.
- D) Não é sintaxe válida.

---

**Q6.** Qual query pronta devolve os IDs de todos os objetos de uma classe persistente?

- A) `All`
- B) `Extent`
- C) `Objects`
- D) `Instances`

---

**Q7.** Como preparar uma class query com a API moderna?

- A) `stmt.%Prepare("Classe:Query")`
- B) `stmt.%PrepareClassQuery("Classe", "Query")`
- C) `##class(%ResultSet).%New("Classe:Query")`
- D) `stmt.%Execute("Classe", "Query")`

---

**Q8.** Qual a diferença de nomenclatura entre `%SQL.Statement` e `%ResultSet`?

- A) Nenhuma.
- B) `%SQL.Statement` usa métodos com `%` (`%Next`, `%Get`); `%ResultSet` usa sem (`Next`, `Get`).
- C) `%ResultSet` usa `%` e `%SQL.Statement` não.
- D) Diferem apenas no construtor.

---

**Q9.** Um processo criado com `JOB` enxerga as variáveis locais de quem o disparou?

- A) Sim, todas.
- B) Não: é outro processo, com memória separada.
- C) Apenas as que começam com `%`.
- D) Apenas as PPG.

---

**Q10.** Como um processo em segundo plano comunica o resultado?

- A) Devolvendo um valor.
- B) Gravando numa **global**.
- C) Escrevendo na tela de quem o disparou.
- D) Por PPG.

---

**Q11.** Depois de um comando `JOB`, o que se deve conferir?

- A) `$ZERROR`
- B) `$TEST`, para saber se o processo foi criado.
- C) `$JOB`
- D) Nada.

---

**Q12.** O que acontece com um erro não tratado dentro de um método executado por `JOB`?

- A) Aparece no Terminal de quem disparou.
- B) O processo morre e o erro não aparece em lugar nenhum por padrão.
- C) O processo tenta de novo automaticamente.
- D) Gera erro de compilação.

---

**Q13.** Qual é o problema de percorrer a `Extent` abrindo cada objeto, para apenas ler campos?

- A) Nenhum.
- B) É o problema N+1: cada objeto custa uma abertura completa. Se não há regra de negócio, use SQL.
- C) `Extent` não devolve todos os objetos.
- D) `%OpenId` não funciona em laço.

---

**Q14.** `%OpenId` chamado na classe base devolve um objeto da subclasse quando o registro é de uma subclasse?

- A) Não, sempre devolve a classe base.
- B) Sim: o IRIS grava o nome da classe real no nó de dados e instancia a classe correta.
- C) Só se você usar `%OpenId` da subclasse.
- D) Gera erro.

---

**Q15.** Você percebe que o código tem muitos testes `%Extends` para decidir o que fazer. O que isso sugere?

- A) Que o código está bem escrito.
- B) Que provavelmente falta um método polimórfico na hierarquia.
- C) Que a hierarquia é rasa demais.
- D) Que se deve usar `$CLASSNAME` em vez de `%Extends`.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** `..` resolve em tempo de execução para a classe real do objeto.
- **A está errada:** fixa a implementação daquela classe e quebra o polimorfismo.
- **C está errada:** o nome da classe é fixado no argumento.
- **D está errada:** `##super()` chama a superclasse, não a classe real.

**Q2 — Resposta: B.**
- **B está certa:** sobe **um** nível na hierarquia, para a implementação imediatamente acima.
- **A está errada:** isso causaria recursão infinita.
- **C está errada:** ele não pula direto para a base.
- **D está errada:** não existe um método com esse nome.

**Q3 — Resposta: B.**
- **B está certa:** `%Extends` testa "é desta classe ou de uma descendente".
- **A está errada:** para igualdade exata, compare `$CLASSNAME`.
- **C está errada:** a relação é de mão única, do filho para o pai.
- **D está errada:** ele testa uma classe específica.

**Q4 — Resposta: B.**
- **B está certa:** sobrescrever sem `##super()` substitui completamente, perdendo as validações herdadas.
- **A está errada:** o efeito é significativo e silencioso.
- **C está errada:** compila normalmente.
- **D está errada:** pode ser salva — e é justamente isso que é perigoso.

**Q5 — Resposta: B.**
- **B está certa:** `$$` chama uma função de rotina e devolve o valor.
- **A está errada:** isso é `DO rótulo^Rotina`.
- **C está errada:** isso é `JOB`.
- **D está errada:** é sintaxe clássica do ObjectScript.

**Q6 — Resposta: B.**
- **B está certa:** toda classe persistente ganha a query `Extent`.
- **A, C e D estão erradas:** essas queries não existem.

**Q7 — Resposta: B.**
- **B está certa:** `%PrepareClassQuery(classe, query)` é a forma moderna.
- **A está errada:** `%Prepare` recebe texto SQL.
- **C está errada:** essa é a API antiga.
- **D está errada:** `%Execute` recebe os argumentos da query, não seu nome.

**Q8 — Resposta: B.**
- **B está certa:** a API moderna usa `%` nos nomes; a antiga não.
- **A está errada:** os nomes de método diferem.
- **C está errada:** inverte.
- **D está errada:** a diferença vai além do construtor.

**Q9 — Resposta: B.**
- **B está certa:** é um processo separado, com memória própria.
- **A está errada:** nada de memória é compartilhado.
- **C está errada:** o prefixo `%` não muda isso.
- **D está errada:** PPG é privada do processo — e o processo é outro.

**Q10 — Resposta: B.**
- **B está certa:** globais são o único meio compartilhado entre processos.
- **A está errada:** `JOB` não devolve valor.
- **C está errada:** o dispositivo é outro.
- **D está errada:** PPG não atravessa processos.

**Q11 — Resposta: B.**
- **B está certa:** `$TEST` vale 1 se o processo foi criado.
- **A está errada:** `$ZERROR` guarda o último erro do processo atual.
- **C está errada:** `$JOB` é o número do processo atual.
- **D está errada:** sem conferir, uma falha na criação passa despercebida.

**Q12 — Resposta: B.**
- **B está certa:** sem `try`/`catch` e sem registro em global, o erro desaparece.
- **A está errada:** o dispositivo é outro.
- **C está errada:** não há retentativa automática.
- **D está errada:** é comportamento de execução.

**Q13 — Resposta: B.**
- **B está certa:** abrir objeto custa muito mais que ler colunas por SQL. Use objetos quando precisar das regras de negócio.
- **A está errada:** o custo é medível e grande.
- **C está errada:** `Extent` devolve todos.
- **D está errada:** funciona; é apenas caro.

**Q14 — Resposta: B.**
- **B está certa:** a primeira posição do nó de dados guarda o nome da classe, e o IRIS instancia a classe correta.
- **A está errada:** seria a quebra do polimorfismo no armazenamento.
- **C está errada:** funciona pela base.
- **D está errada:** não há erro.

**Q15 — Resposta: B.**
- **B está certa:** muitos testes de tipo são sinal de que o comportamento deveria estar nos próprios objetos.
- **A está errada:** é um cheiro de código clássico.
- **C está errada:** a profundidade não é o problema.
- **D está errada:** trocar a função de teste não resolve o desenho.

---

## 10. Resumo relâmpago

1. **`##class()`** de fora, **`..`** de dentro, **`##super()`** para estender, **`$CLASSMETHOD`/`$METHOD`** quando o nome é dinâmico.
2. **Dentro de uma classe, use `..`**: é o que respeita o polimorfismo. `##class(MinhaClasse).Metodo()` fixa a implementação e quebra a sobrescrita.
3. **Polimorfismo**: um método definido **uma vez** na base produz comportamentos diferentes conforme a classe real do objeto.
4. **`##super()` sobe um nível.** Numa hierarquia de três, ele encadeia naturalmente.
5. **Ao sobrescrever um callback, chame `##super()`** — senão as validações e comportamentos herdados são perdidos silenciosamente.
6. **`$CLASSNAME(oref)`** e **`oref.%ClassName(1)`** dão a classe real; **`%ClassName()`** dá o nome curto.
7. **`oref.%Extends("Classe")`** testa "é desta classe **ou de uma descendente**". Para igualdade exata, compare `$CLASSNAME`.
8. **Muitos testes de tipo indicam que falta um método polimórfico.**
9. Rotinas: **`DO ^Rotina`**, **`DO rótulo^Rotina`**, e **`$$rótulo^Rotina`** como **função extrínseca** que devolve valor.
10. **`JOB`** cria um processo separado: **sem** variáveis locais, **sem** PPG, **sem** a sua tela, **sem** retorno.
11. Confira **`$TEST`** depois do `JOB`; **`$ZCHILD`** traz o número do processo criado.
12. **Comunique por global**, trate erros com `try`/`catch` dentro do método, e registre o estado — senão a falha desaparece.
13. Status `"running"` que nunca muda é a assinatura de um processo morto.
14. Toda classe persistente tem a query **`Extent`**, executada com **`%PrepareClassQuery(classe, "Extent")`**.
15. API moderna: **`%Execute`/`%Next`/`%Get`**. API antiga (`%ResultSet`): **`Execute`/`Next`/`Get`** mais **`Close()`**.
16. **`%OpenId` da classe base devolve a instância da classe real**, porque o nome da classe está gravado no nó de dados.
17. **SQL para achar, objeto para agir.** Abrir objetos só para ler campos é o N+1 do Capítulo 12.
18. Uma **fábrica** com o mapeamento tipo → classe num só lugar evita `if` espalhados e permite acrescentar tipos sem tocar em quem consome.

---

## 11. Cartões de memorização

**Frente:** Dentro de uma classe, `..Metodo()` ou `##class(MinhaClasse).Metodo()`?
**Verso:** `..Metodo()` — é o que respeita o polimorfismo. A outra forma fixa a implementação da classe indicada.

**Frente:** O que `##super()` chama?
**Verso:** A implementação do mesmo método na **superclasse imediata**. Sobe um nível.

**Frente:** O que acontece se você sobrescrever um callback sem chamar `##super()`?
**Verso:** As validações e comportamentos herdados deixam de ser aplicados — silenciosamente.

**Frente:** O que `%Extends("Classe")` testa?
**Verso:** Se o objeto é daquela classe **ou de uma descendente**.

**Frente:** Como saber a classe real de um objeto?
**Verso:** `$CLASSNAME(oref)` ou `oref.%ClassName(1)`. Sem o argumento, `%ClassName()` dá o nome curto.

**Frente:** O que significa `$$Rotulo^Rotina(args)`?
**Verso:** Chamada de **função extrínseca**: executa o rótulo da rotina e **usa o valor devolvido**.

**Frente:** Qual query devolve todos os IDs de uma classe persistente?
**Verso:** `Extent`, executada com `%PrepareClassQuery(classe, "Extent")`.

**Frente:** Como se chamam os métodos na API antiga `%ResultSet`?
**Verso:** `Execute()`, `Next()`, `Get()` e `Close()` — **sem** o `%`. A moderna usa `%Execute`, `%Next`, `%Get`.

**Frente:** Um processo criado com `JOB` enxerga suas variáveis?
**Verso:** Não. Nem locais, nem PPG. É outro processo, com memória própria.

**Frente:** Como um processo em segundo plano devolve resultado?
**Verso:** Gravando numa **global**. `JOB` não devolve valor e não escreve na sua tela.

**Frente:** O que conferir logo depois de um `JOB`?
**Verso:** `$TEST` — 1 se o processo foi criado. `$ZCHILD` traz o número dele.

**Frente:** O que acontece com um erro não tratado num processo `JOB`?
**Verso:** Ele desaparece: o processo morre sem mensagem. Envolva o trabalho em `try`/`catch` e registre na global.

**Frente:** Como detectar um processo em segundo plano que morreu?
**Verso:** Status `"running"` que nunca muda. Registre um carimbo de última atualização e monitore-o.

**Frente:** `%OpenId` da classe base devolve o quê para um registro de subclasse?
**Verso:** Uma instância da **subclasse**. O nome da classe real está gravado no nó de dados.

**Frente:** Qual o padrão recomendado para alterar muitos objetos?
**Verso:** SQL para **achar** os IDs (aproveita índices), objeto para **agir** (passa pelas regras).

**Frente:** Quando percorrer `Extent` em vez de usar SQL?
**Verso:** Quando você precisa aplicar regras de negócio a cada objeto. Para apenas ler campos, SQL.

**Frente:** O que sugere um código cheio de testes `%Extends`?
**Verso:** Que falta um método polimórfico: em vez de perguntar "que tipo é você?", peça "faça o que você sabe fazer".

**Frente:** Para que serve uma classe fábrica?
**Verso:** Concentrar o mapeamento tipo → classe num só lugar, permitindo acrescentar tipos sem alterar quem consome.

---

Digite CONTINUAR para o próximo capítulo.
