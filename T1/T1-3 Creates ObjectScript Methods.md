# Apostila InterSystems ObjectScript Specialist
## Capítulo 3 — T1.3 Creates ObjectScript Methods (Métodos)

> Ainda em **T1 — Manages Data Model** (23 questões de 76). Este capítulo fecha a parte de "como se escreve uma classe" e abre a porta para todo o resto da apostila: praticamente tudo que você fizer daqui para a frente será escrito dentro de um método.

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Diferenciar **`Method`** de **`ClassMethod`** e saber quando cada um é o certo.
2. Escrever a assinatura completa de um método: nome, parâmetros, tipos, valores padrão e tipo de retorno.
3. Devolver valores com **`QUIT`** e com **`RETURN`**, e saber exatamente por que os dois são diferentes.
4. Passar argumentos **por valor**, **por referência (`ByRef`)** e **de saída (`Output`)** — e usar o ponto na chamada.
5. Escrever métodos com **número variável de argumentos** (`args...`).
6. Entender o **escopo de variáveis** dentro de um método: bloco de procedimento, variáveis privadas, `NEW` e `PublicList`.
7. Usar as **palavras-chave de método**: `Private`, `Final`, `Abstract`, `CodeMode`, `SqlProc`, `ProcedureBlock`.
8. Sobrescrever um método herdado e chamar a versão da superclasse com **`##super()`**.
9. Usar os **callbacks** do IRIS: `%OnNew`, `%OnBeforeSave`, `%OnAfterSave`, `%OnValidateObject`, `%OnOpen`, `%OnDelete`.
10. Criar erros próprios com **`$$$ERROR($$$GeneralError, "mensagem")`**.
11. Chamar métodos dinamicamente com **`$CLASSMETHOD`**, **`$METHOD`** e **`$PROPERTY`**.
12. Evoluir o projeto com métodos de negócio, normalização automática no salvamento e uma subclasse de exame urgente.

---

## 2. O conceito em linguagem de gente

### 2.1 Método é uma ação escrita no manual

A classe é o formulário; as propriedades são os campos. Os **métodos** são o **manual de procedimentos** que acompanha o formulário: "como calcular a idade", "como imprimir a ficha", "como registrar um exame".

Um método tem três partes, como qualquer receita:

1. **O que ele precisa receber** — os ingredientes. São os **parâmetros**.
2. **O que ele faz** — o preparo. É o **corpo**.
3. **O que ele devolve** — o prato pronto. É o **retorno**.

Nem toda receita devolve algo. Algumas só executam uma ação ("imprima esta ficha") e não entregam nada de volta. Isso é normal e legítimo.

### 2.2 A pergunta que decide entre `Method` e `ClassMethod`

Faça sempre esta pergunta: **essa ação precisa de uma ficha preenchida na mão?**

- *"Calcular a idade"* — precisa. De quem é a idade? Do paciente que está na sua mão. → **`Method`** (método de instância).
- *"Contar quantos pacientes existem"* — não precisa. Não é a contagem de ninguém em particular. → **`ClassMethod`** (método de classe).
- *"Imprimir esta ficha"* — precisa. → `Method`.
- *"Criar um paciente novo"* — não precisa, porque o paciente ainda não existe. → `ClassMethod`.

Analogia do balcão do laboratório:

- Um `ClassMethod` é uma **regra afixada na parede**: "para cadastrar um paciente, faça assim". Vale sempre, para todos, não depende de ninguém específico.
- Um `Method` é uma **instrução escrita no verso de uma ficha**: "para calcular a idade **deste** paciente, faça assim". Só faz sentido com a ficha na mão.

Consequência prática: um `Method` pode usar `..Name`, `..BirthDate`, `..%Save()`. Um `ClassMethod` **não pode**, porque não existe "este objeto" ali dentro. Se você tentar, o erro é claro e imediato.

### 2.3 O tema mais difícil do capítulo: passar por valor ou por referência

Esta é a ideia que mais derruba gente na prova. Vamos com uma analogia bem concreta.

Você tem um caderno com uma anotação. Um colega pede a informação. Você tem duas escolhas:

**Escolha 1 — você tira uma fotocópia e entrega.** O colega pode rabiscar a fotocópia à vontade: o seu caderno continua intacto. Isso é **passar por valor**. É o comportamento **padrão** no ObjectScript.

**Escolha 2 — você entrega o caderno.** O que o colega escrever, escreveu no seu caderno. Isso é **passar por referência**.

No IRIS, a diferença aparece em dois lugares ao mesmo tempo, e é preciso os dois combinarem:

- Na **declaração** do método, você marca o parâmetro com `ByRef` ou `Output`.
- Na **chamada**, quem chama coloca um **ponto** antes do nome da variável.

```objectscript
ClassMethod Double(ByRef number As %Integer)
{
    set number = number * 2
}
```

```
LABSTUDY>SET n = 5
LABSTUDY>DO ##class(X).Double(n)      ; SEM ponto
LABSTUDY>WRITE n
5

LABSTUDY>DO ##class(X).Double(.n)     ; COM ponto
LABSTUDY>WRITE n
10
```

Repare: **é o ponto na chamada que decide de verdade**. Sem o ponto, o método recebeu uma fotocópia, dobrou a fotocópia, e você não viu diferença nenhuma. Esquecer o ponto é a origem de bugs silenciosos e é exatamente o que a prova testa.

E qual a diferença entre `ByRef` e `Output`?

- **`ByRef`** — mão dupla. O método **recebe** o valor que estava lá **e** pode alterá-lo.
- **`Output`** — mão única para fora. O método **ignora** o que estava lá e usa o parâmetro só para **devolver** algo.

As duas palavras, sozinhas, são apenas **documentação para quem lê o código**: tecnicamente, quem faz a mágica é o ponto na chamada. Mas escrevê-las corretamente comunica intenção e é o que se espera de código profissional.

### 2.4 Por que existe passagem por referência, se já existe retorno?

Porque um método só devolve **um** valor pelo `QUIT`. Quando você precisa entregar três informações — o total, a média e o maior valor — as opções são: criar um objeto só para isso, ou usar parâmetros `Output`. Em ObjectScript, `Output` é o caminho idiomático.

Muitos métodos do próprio IRIS usam esse padrão: devolvem o resultado principal pelo `QUIT` e um `%Status` detalhado por um parâmetro `Output`. Você já viu isso no Capítulo 1, com `%OpenId(id, concurrency, .status)`.

### 2.5 Callback é um método que o IRIS chama sozinho

Imagine que o laboratório tem uma regra: *"toda vez que uma ficha for arquivada, carimbe a data e confira se o nome está em maiúsculas"*.

Você poderia escrever esse carimbo em todos os lugares do sistema que arquivam fichas. Seria repetitivo e um dia alguém esqueceria.

O IRIS oferece coisa melhor: você escreve um método com um **nome combinado**, e o IRIS **o chama sozinho** no momento certo. Esses métodos de nome combinado se chamam **callbacks**.

- `%OnNew()` — o IRIS chama logo depois de criar o objeto.
- `%OnBeforeSave()` — o IRIS chama **antes** de gravar.
- `%OnAfterSave()` — o IRIS chama **depois** de gravar com sucesso.
- `%OnValidateObject()` — o IRIS chama durante a validação, para você acrescentar regras próprias.
- `%OnOpen()` — o IRIS chama depois de trazer o objeto do disco.
- `%OnDelete()` — o IRIS chama antes de apagar.

Você não chama nenhum deles. Você apenas **escreve**, e o IRIS aciona no momento certo. É como deixar um bilhete na gaveta que o arquivista lê automaticamente.

---

## 3. A sintaxe explicada

### 3.1 A forma geral de um método

```
Method NomeDoMetodo(param1 As Tipo, param2 As Tipo = padrao) As TipoRetorno [ PalavrasChave ]
{
    ... corpo ...
}
```

- **`Method`** ou **`ClassMethod`** — palavra fixa. **Obrigatória.** `Method` age sobre um objeto; `ClassMethod` é chamado pela classe.
- **`NomeDoMetodo`** — **obrigatório**, diferencia maiúsculas de minúsculas.
- **`(...)`** — a lista de parâmetros. **Os parênteses são obrigatórios**, mesmo vazios.
- **`As Tipo`** dentro dos parênteses — o tipo esperado do parâmetro. **Opcional**; se omitido, assume-se `%String`.
- **`= padrao`** — valor usado quando quem chama não passa o argumento. **Opcional.**
- **`As TipoRetorno`** — o tipo do valor devolvido. **Opcional**; omita se o método não devolve nada.
- **`[ PalavrasChave ]`** — características do método. **Opcional.**
- **`{ ... }`** — o corpo. **Obrigatório.** Diferente de propriedades, métodos **não** terminam com ponto e vírgula.

Exemplos das quatro formas mais comuns:

```objectscript
/// Instance method that returns a value.
Method GetLabel() As %String
{
    quit ..Name_" ("_..RecordNumber_")"
}

/// Instance method that returns nothing.
Method Print()
{
    write ..GetLabel(), !
}

/// Class method with a default argument.
ClassMethod Greet(name As %String = "World")
{
    write "Hello, ", name, !
}

/// Class method that returns a status.
ClassMethod DoWork() As %Status
{
    quit $$$OK
}
```

### 3.2 `QUIT` e `RETURN`: a diferença que cai na prova

Os dois encerram alguma coisa e podem devolver um valor. A diferença está no **alcance**.

- **`QUIT`** encerra o **bloco atual**. Se você estiver dentro de um `FOR`, o `QUIT` sai do `FOR` — **não** do método.
- **`RETURN`** encerra o **método inteiro**, não importa quantos blocos aninhados existam.

Veja a diferença na prática:

```objectscript
ClassMethod TestQuit() As %String
{
    for i = 1:1:5 {
        if i = 3 {
            quit               // sai apenas do FOR
        }
    }
    write "still here", !
    quit "finished with quit"
}

ClassMethod TestReturn() As %String
{
    for i = 1:1:5 {
        if i = 3 {
            return "left early"   // sai do MÉTODO inteiro
        }
    }
    write "never printed", !
    quit "finished with return"
}
```

```
LABSTUDY>WRITE ##class(X).TestQuit(), !
still here
finished with quit

LABSTUDY>WRITE ##class(X).TestReturn(), !
left early
```

Regra prática: **para devolver do método a partir de dentro de um laço, use `RETURN`.** Para interromper só o laço, use `QUIT`.

Fora de laços, no nível principal do método, `QUIT valor` e `RETURN valor` fazem exatamente a mesma coisa. Muito código antigo usa `QUIT` porque `RETURN` é mais recente.

### 3.3 Argumentos: valor, `ByRef` e `Output`

```objectscript
/// value  : passed by value, changes are not visible to the caller
/// counter: passed by reference, comes in and goes out
/// message: output only, used just to send something back
ClassMethod Process(value As %Integer, ByRef counter As %Integer, Output message As %String) As %Status
{
    set value = value * 100        // affects only the local copy
    set counter = counter + 1      // affects the caller's variable
    set message = "processed"      // fills the caller's variable
    quit $$$OK
}
```

Uso:

```
LABSTUDY>SET v = 5, c = 10

LABSTUDY>SET sc = ##class(X).Process(v, .c, .msg)

LABSTUDY>WRITE v, " / ", c, " / ", msg, !
5 / 11 / processed
```

- `v` continua `5`: foi passado **sem ponto**, então o método mexeu numa cópia.
- `c` virou `11`: foi passado **com ponto**, então o método mexeu no original.
- `msg` **nem existia** antes da chamada, e agora existe: o parâmetro `Output` criou a variável.

Um detalhe importante sobre `Output`: como o método não deve confiar no que entrou, o padrão profissional é o método **limpar** o parâmetro logo no começo:

```objectscript
ClassMethod Load(Output result As %String) As %Status
{
    set result = ""      // never trust what came in
    ...
}
```

### 3.4 Número variável de argumentos

```objectscript
/// Sums any number of values.
ClassMethod Sum(values... ) As %Numeric
{
    set total = 0
    for i = 1:1:values {
        set total = total + values(i)
    }
    quit total
}
```

Como funciona:

- **`values...`** — os três pontos dizem "aqui entram quantos argumentos vierem".
- Dentro do método, `values` (sem parênteses) contém a **quantidade** de argumentos recebidos.
- `values(1)`, `values(2)`, ... contêm os valores, na ordem.

```
LABSTUDY>WRITE ##class(X).Sum(10, 20, 30), !
60
LABSTUDY>WRITE ##class(X).Sum(5), !
5
LABSTUDY>WRITE ##class(X).Sum(), !
0
```

Um cuidado: se um argumento não for passado no meio da lista (por exemplo `Sum(10, , 30)`), aquele subscrito fica **indefinido**, e usá-lo direto causa `<UNDEFINED>`. Para proteger, use `$GET(values(i))`, que devolve vazio em vez de dar erro. A função `$GET` é do Capítulo 4.1, mas guarde a receita.

### 3.5 Escopo de variáveis dentro de um método

Por padrão, todo método de classe é um **bloco de procedimento** (*procedure block*). Isso significa:

- As variáveis que você cria dentro do método são **privadas dele**. Ninguém de fora as enxerga.
- Quando o método termina, elas **desaparecem sozinhas**.
- Um método **não enxerga** as variáveis de quem o chamou.

Isso é ótimo: cada método é uma caixa fechada e você não corre risco de estragar variáveis alheias sem querer.

Demonstração:

```objectscript
ClassMethod Outer()
{
    set secret = "outer value"
    do ..Inner()
    write "after inner, secret is still: ", secret, !
}

ClassMethod Inner()
{
    // "secret" here is a brand new, empty variable
    set secret = "inner value"
}
```

```
LABSTUDY>DO ##class(X).Outer()
after inner, secret is still: outer value
```

Se você **precisar** de uma variável compartilhada entre chamadas — algo raro e a ser evitado —, declare-a na palavra-chave `PublicList` do método e marque-a. A forma completa é assim:

```objectscript
ClassMethod UsesPublic() [ ProcedureBlock = 1, PublicList = (sharedValue) ]
{
    set sharedValue = 42
}
```

Existe ainda `[ ProcedureBlock = 0 ]`, que desliga o isolamento e faz o método enxergar tudo. Você vai ver isso em código antigo. **Não use em código novo**: o isolamento é uma proteção, não um estorvo.

E existe o comando **`NEW`**, herdado da tradição da linguagem:

```
NEW temp
```

`NEW temp` guarda o valor atual de `temp`, deixa `temp` vazia para você usar, e restaura o valor antigo quando o bloco termina. Dentro de blocos de procedimento isso raramente é necessário, porque o isolamento já é automático. Em rotinas (arquivos `.mac`), é essencial. Voltaremos a isso no Capítulo 4.

### 3.6 Palavras-chave de método

```objectscript
Method Helper() [ Private ]
{
}

Method核心() [ Final ]
{
}

Method Calculate() As %Numeric [ Abstract ]
{
}

ClassMethod Label() As %String [ CodeMode = expression ]
{
"LabStudy System"
}

ClassMethod CountAll() As %Integer [ SqlProc, SqlName = COUNT_ALL ]
{
    quit 0
}
```

- **`Private`** — o método só pode ser chamado de dentro da própria classe (e das subclasses). De fora, dá erro.
- **`Final`** — nenhuma subclasse pode sobrescrever esse método.
- **`Abstract`** — o método não tem implementação; existe apenas para obrigar as subclasses a implementarem. Chamar um método abstrato diretamente é erro.
- **`CodeMode = expression`** — o corpo do método é **uma única expressão**, sem `quit`. O IRIS devolve o resultado dela. É mais enxuto para métodos triviais. Os outros valores de `CodeMode` são `code` (o padrão) e `objectgenerator` (geração de código na compilação, tema avançado).
- **`SqlProc`** — expõe o método como **procedimento armazenado** do SQL, podendo ser chamado de uma consulta. **`SqlName`** dá a ele um nome diferente no mundo SQL.
- **`ProcedureBlock`** e **`PublicList`** — vistos acima.
- **`ReturnResultsets`** — declara que o método devolve conjuntos de resultados para o cliente. Assunto do Capítulo 4.6.

*(Observação: use apenas letras do alfabeto latino em nomes de método. O exemplo acima está apenas ilustrando a posição dos colchetes.)*

### 3.7 Sobrescrever e chamar a superclasse com `##super()`

Quando uma subclasse declara um método com o **mesmo nome** de um método da superclasse, ela **sobrescreve** o comportamento. Se você quiser **acrescentar** em vez de substituir, chame a versão original:

```objectscript
Class LabStudy.Demo.Animal Extends %RegisteredObject
{
Method Describe() As %String
{
    quit "I am an animal"
}
}
```

```objectscript
Class LabStudy.Demo.Dog Extends LabStudy.Demo.Animal
{
Method Describe() As %String
{
    quit ##super() _ ", specifically a dog"
}
}
```

```
LABSTUDY>WRITE ##class(LabStudy.Demo.Dog).%New().Describe(), !
I am an animal, specifically a dog
```

- **`##super()`** — chama a implementação **da superclasse** do mesmo método em que você está. Os parênteses são obrigatórios; se o método tiver argumentos, repasse-os: `##super(a, b)`.
- Repare no encadeamento `##class(...).%New().Describe()`: cria o objeto e já chama o método nele, sem variável intermediária. É válido e comum.

O uso mais frequente de `##super()` é justamente nos callbacks: você acrescenta a sua regra e depois deixa o IRIS fazer o que ele já fazia.

### 3.8 Os callbacks, um por um

**`%OnNew()`** — chamado pelo `%New()`, logo após a criação.

```objectscript
Method %OnNew(initialCode As %String = "") As %Status [ Private, ServerOnly = 1 ]
{
    set ..TestCode = initialCode
    quit $$$OK
}
```

- É um **`Method`** (de instância): o objeto já existe quando ele roda.
- Recebe o que quer que tenha sido passado ao `%New()`. `##class(X).%New("HGB")` faz `initialCode` valer `"HGB"`.
- Devolve `%Status`. **Se devolver erro, o `%New()` devolve OREF nula** e o objeto não nasce.
- `ServerOnly = 1` indica que só roda no servidor. É a convenção para callbacks.

**`%OnBeforeSave(insert As %Boolean)`** — chamado antes de gravar.

```objectscript
Method %OnBeforeSave(insert As %Boolean) As %Status [ Private, ServerOnly = 1 ]
{
    set ..RecordNumber = $ZCONVERT(..RecordNumber, "U")
    quit $$$OK
}
```

- O argumento `insert` vale `1` se é a **primeira** gravação daquele objeto e `0` se é uma **atualização**. Isso permite tratar os dois casos de forma diferente.
- **Se devolver erro, a gravação é cancelada.** É o lugar certo para regras de negócio que impedem o salvamento.
- É o lugar certo também para **normalizar** dados: colocar em maiúsculas, tirar espaços, completar campos.

**`%OnAfterSave(insert As %Boolean)`** — chamado depois de gravar com sucesso. Bom para registrar log, disparar notificações, atualizar contadores.

**`%OnValidateObject()`** — chamado durante a validação, para você acrescentar regras que não cabem numa declaração de propriedade.

```objectscript
Method %OnValidateObject() As %Status [ Private, ServerOnly = 1 ]
{
    if (..ResultValue < 0) {
        quit $$$ERROR($$$GeneralError, "ResultValue cannot be negative")
    }
    quit $$$OK
}
```

**`%OnOpen()`** — chamado após o objeto ser trazido do disco por `%OpenId()`.

**`%OnDelete(oid)`** — chamado antes de apagar. Atenção: é um **`ClassMethod`**, e recebe o **OID** (não o ID). Se devolver erro, o apagamento é cancelado.

```objectscript
ClassMethod %OnDelete(oid As %ObjectIdentity) As %Status [ Private, ServerOnly = 1 ]
{
    quit $$$OK
}
```

Existem outros callbacks (`%OnAddToSaveSet`, `%OnClose`, `%OnAfterDelete`, entre outros). Os seis acima cobrem o que a prova cobra e o que você usará no dia a dia; para a lista completa, **verificar na documentação oficial**.

### 3.9 Criando os seus próprios erros

```objectscript
quit $$$ERROR($$$GeneralError, "Patient must have a record number")
```

- **`$$$ERROR(código, texto)`** — macro que constrói um `%Status` de erro.
- **`$$$GeneralError`** — o código genérico, para erros de regra de negócio próprios. Existem dezenas de outros códigos específicos do sistema.
- O texto é livre e vai aparecer no `DisplayError`.

Você pode passar valores para dentro da mensagem:

```objectscript
quit $$$ERROR($$$GeneralError, "Invalid test code: "_code)
```

### 3.10 Chamando métodos dinamicamente

Às vezes o nome do método ou da classe só é conhecido em tempo de execução (por exemplo, veio de uma configuração). Para isso existem três funções:

```
LABSTUDY>SET className = "LabStudy.Patient"
LABSTUDY>SET methodName = "Show"

LABSTUDY>DO $CLASSMETHOD(className, methodName, 1)

LABSTUDY>SET p = ##class(LabStudy.Patient).%OpenId(1)
LABSTUDY>WRITE $METHOD(p, "GetLabel"), !
LABSTUDY>WRITE $PROPERTY(p, "Name"), !
```

- **`$CLASSMETHOD(classe, metodo, args...)`** — chama um **método de classe** cujo nome está numa variável.
- **`$METHOD(oref, metodo, args...)`** — chama um **método de instância** sobre uma OREF.
- **`$PROPERTY(oref, propriedade)`** — lê uma propriedade cujo nome está numa variável. Também serve para escrever: `SET $PROPERTY(p, "Name") = "Ana"`.

Isso é poderoso, mas custa desempenho e tira a verificação do compilador. Use quando o nome for realmente dinâmico, nunca por preguiça.

---

## 4. Exemplo comentado

Uma classe que usa quase tudo do capítulo:

Arquivo `src/LabStudy/Demo/Account.cls`:

```objectscript
/// Demonstrates instance methods, class methods, argument passing and callbacks.
Class LabStudy.Demo.Account Extends %Persistent
{

Property Owner As %String(MAXLEN = 100) [ Required ];

Property Balance As %Numeric(SCALE = 2) [ InitialExpression = 0 ];

Property Closed As %Boolean [ InitialExpression = 0 ];

/// Runs right after %New(). Optionally sets the owner.
Method %OnNew(owner As %String = "") As %Status [ Private, ServerOnly = 1 ]
{
    if owner '= "" {
        set ..Owner = owner
    }
    quit $$$OK
}

/// Runs before every save. Normalises the owner name.
Method %OnBeforeSave(insert As %Boolean) As %Status [ Private, ServerOnly = 1 ]
{
    set ..Owner = $ZSTRIP(..Owner, "<>W")
    if insert {
        write "[callback] inserting a new account", !
    } else {
        write "[callback] updating an existing account", !
    }
    quit $$$OK
}

/// Extra validation rule that no property parameter could express.
Method %OnValidateObject() As %Status [ Private, ServerOnly = 1 ]
{
    if (..Closed = 1) && (..Balance '= 0) {
        quit $$$ERROR($$$GeneralError, "A closed account must have zero balance")
    }
    quit $$$OK
}

/// Adds an amount to this account. Instance method: needs an object.
Method Deposit(amount As %Numeric) As %Status
{
    if amount <= 0 {
        quit $$$ERROR($$$GeneralError, "Deposit amount must be positive")
    }
    set ..Balance = ..Balance + amount
    quit $$$OK
}

/// Returns a printable label. Single expression, no quit needed.
Method Label() As %String [ CodeMode = expression ]
{
..Owner_": "_..Balance
}

/// Class method: does not belong to any single account.
/// Returns the count through QUIT and extra data through Output parameters.
ClassMethod Summary(Output total As %Numeric, Output largest As %Numeric) As %Integer
{
    set total = 0
    set largest = 0
    set count = 0

    set id = ""
    for {
        set id = $ORDER(^LabStudy.Demo.AccountD(id))
        quit:id=""

        set account = ..%OpenId(id)
        continue:'$ISOBJECT(account)

        set count = count + 1
        set total = total + account.Balance
        if account.Balance > largest {
            set largest = account.Balance
        }
    }

    quit count
}

}
```

Comentando as decisões:

- **`%OnNew(owner)`** permite `%New("Maria")`, encurtando o código de quem usa. O `if owner '= ""` evita sobrescrever com vazio quando ninguém passa nada. O apóstrofo é a negação: `'=` significa "diferente de".
- **`%OnBeforeSave`** normaliza o nome com `$ZSTRIP(texto, "<>W")` — essa função remove caracteres; `<` significa "do início", `>` significa "do fim" e `W` significa "espaços em branco". Junto: tira os espaços das duas pontas. O `if insert` mostra na prática a diferença entre inserção e atualização.
- **`%OnValidateObject`** expressa uma regra que **nenhum parâmetro de propriedade conseguiria**: ela envolve duas propriedades ao mesmo tempo. Esse é exatamente o critério para escolher entre declarar na propriedade e escrever no callback. O `&&` é o "e" lógico (Capítulo 4.4).
- **`Deposit`** é um `Method` porque deposita **nesta** conta. Ele devolve `%Status` porque pode recusar.
- **`Label`** usa `CodeMode = expression`: o corpo é uma expressão só, sem `quit`, sem chaves internas. Repare que a expressão fica na coluna 1 — nesse modo, o corpo é literalmente uma expressão, não um bloco de comandos.
- **`Summary`** é um `ClassMethod` que devolve **três informações**: a contagem pelo `QUIT` e mais duas por parâmetros `Output`. É o padrão idiomático do IRIS.
- Dentro de `Summary` usamos `$ORDER` sobre a global de dados da classe para percorrer os IDs. `$ORDER` é o assunto principal do Capítulo 4.1; aqui, entenda apenas que ele caminha pelos subscritos existentes, um a um. O nome `^LabStudy.Demo.AccountD` é a global onde o IRIS grava os dados dessa classe — o sufixo `D` é a convenção do armazenamento padrão. (No Capítulo 4.6 substituiremos isso por SQL, que é a forma correta e portável.)
- `continue:'$ISOBJECT(account)` — o `CONTINUE` com pós-condicional pula para a próxima volta do laço se a condição for verdadeira. Aqui: "se não abriu, pule".

### 4.1 Usando no Terminal

```
LABSTUDY>SET a = ##class(LabStudy.Demo.Account).%New("  Maria Silva  ")

LABSTUDY>WRITE "[", a.Owner, "]", !
[  Maria Silva  ]

LABSTUDY>WRITE $$$ISOK(a.Deposit(100)), !
1

LABSTUDY>DO $SYSTEM.Status.DisplayError(a.Deposit(-5))
ERROR #5001: Deposit amount must be positive

LABSTUDY>SET sc = a.%Save()
[callback] inserting a new account

LABSTUDY>WRITE "[", a.Owner, "]", !
[Maria Silva]

LABSTUDY>WRITE a.Label(), !
Maria Silva: 100

LABSTUDY>SET a.Balance = 50, a.Closed = 1

LABSTUDY>DO $SYSTEM.Status.DisplayError(a.%Save())
[callback] updating an existing account
ERROR #5001: A closed account must have zero balance

LABSTUDY>SET a.Balance = 0

LABSTUDY>WRITE $$$ISOK(a.%Save()), !
[callback] updating an existing account
1

LABSTUDY>SET count = ##class(LabStudy.Demo.Account).Summary(.total, .largest)

LABSTUDY>WRITE count, " accounts, total ", total, ", largest ", largest, !
1 accounts, total 0, largest 0
```

O que observar:

- Logo após o `%New("  Maria Silva  ")`, o nome ainda tem os espaços. **O `%OnBeforeSave` só roda no `%Save()`**, não antes.
- A mensagem `[callback] inserting...` apareceu sozinha. Ninguém a chamou.
- A segunda gravação disse `updating`, porque o objeto já existia. Foi o argumento `insert` valendo `0`.
- A validação do `%OnValidateObject` **impediu** a gravação. Repare na ordem: o `%OnBeforeSave` já tinha rodado e impresso a mensagem antes de a validação recusar. Ou seja: **o callback rodar não garante que a gravação vá acontecer.** É um ponto sutil e útil de saber.
- `Summary(.total, .largest)` — os pontos entregam as variáveis, e elas voltam preenchidas. O valor de retorno (`count`) veio pelo caminho normal.

---

## 5. Variações e detalhes

### 5.1 Método com o mesmo nome numa subclasse

Sobrescrever é livre, com duas restrições:

- Não se pode sobrescrever um método marcado como `Final`.
- A **assinatura** deve permanecer compatível. Mudar o número de parâmetros de forma incompatível causa erro de compilação.

Sobrescrever **sem** chamar `##super()` substitui totalmente o comportamento. Sobrescrever **com** `##super()` estende. As duas coisas são legítimas; a escolha depende da intenção.

### 5.2 Método abstrato obriga a subclasse

```objectscript
Class LabStudy.Demo.Shape Extends %RegisteredObject [ Abstract ]
{
Method Area() As %Numeric [ Abstract ]
{
}
}
```

A subclasse **precisa** implementar `Area()` para ser útil. Isso é um contrato: quem herdar de `Shape` promete saber calcular a própria área.

Note que a classe abstrata pode ter métodos concretos também. Abstrato é uma marca de membro, não uma condição da classe inteira.

### 5.3 Passando objetos como argumento

Quando você passa uma OREF para um método, a **OREF** é copiada, mas ela continua apontando para o **mesmo objeto**. Ou seja: alterações feitas nas propriedades **são** visíveis para quem chamou, mesmo sem `ByRef`.

```objectscript
ClassMethod Rename(patient As LabStudy.Patient)
{
    set patient.Name = "changed"     // visível para quem chamou
    set patient = ""                 // NÃO visível para quem chamou
}
```

Entregar a fotocópia do **bilhete** não impede o colega de ir até o objeto e rabiscá-lo. O que a fotocópia protege é o bilhete em si: se o método apontar a sua variável local para outro lugar, quem chamou não percebe.

Esse é um ponto conceitual que a prova gosta de explorar.

### 5.4 Verificando se um argumento foi passado

Valor padrão resolve a maioria dos casos. Quando você precisa distinguir "não passou" de "passou vazio", use `$DATA`:

```objectscript
ClassMethod Test(value As %String)
{
    if $DATA(value) {
        write "argument was provided: [", value, "]", !
    } else {
        write "argument was not provided", !
    }
}
```

`$DATA(variavel)` devolve `0` se a variável não existe. É o assunto do Capítulo 4.1, mas a receita é essa.

### 5.5 Método como procedimento armazenado do SQL

```objectscript
ClassMethod PatientLabel(id As %String) As %String [ SqlProc, SqlName = PATIENT_LABEL ]
{
    set p = ##class(LabStudy.Patient).%OpenId(id)
    if '$ISOBJECT(p) {
        quit ""
    }
    quit p.Name_" ("_p.RecordNumber_")"
}
```

Depois de compilar, isso pode ser chamado de dentro de uma consulta SQL:

```sql
SELECT LabStudy.PATIENT_LABEL(ID) AS Label FROM LabStudy.PATIENT
```

O nome no SQL é `schema.SqlName` — no caso, `LabStudy.PATIENT_LABEL`. Sem `SqlName`, o nome seria o do método.

### 5.6 Ordem dos callbacks num `%Save()`

Saber a ordem ajuda a entender comportamentos e é material de prova. Numa gravação bem-sucedida, a sequência essencial é:

1. `%OnAddToSaveSet()` — o objeto entra no conjunto a ser gravado.
2. `%OnValidateObject()` — junto com a validação declarativa das propriedades.
3. `%OnBeforeSave(insert)`.
4. Gravação física em disco e atualização dos índices.
5. `%OnAfterSave(insert)`.

O detalhe que o exemplo da seção 4.1 revelou: como `%OnBeforeSave` e a validação estão ambos antes da gravação, ver a mensagem do `%OnBeforeSave` na tela **não** significa que os dados foram gravados. Para ter certeza, confira o `%Status`.

A ordem completa e exata, com todos os callbacks e o comportamento em objetos relacionados, é longa: **verificar na documentação oficial** quando precisar do detalhe fino.

---

## 6. Pegadinhas e erros comuns

**1) Usar `..Propriedade` dentro de um `ClassMethod`.**
Sintaxe aceita na escrita, mas quebra na execução: não existe "este objeto" num método de classe. Se a ação precisa do objeto, o método tem que ser `Method`.

**2) Esquecer o ponto na chamada de um `ByRef`/`Output`.**
O método executa normalmente, sem erro nenhum, e a sua variável não muda. Bug silencioso, o pior tipo.

**3) Usar `QUIT` dentro de um `FOR` esperando sair do método.**
`QUIT` sai só do laço. Para sair do método, use `RETURN`.

**4) Achar que um método enxerga as variáveis de quem o chamou.**
Não enxerga: cada método é um bloco de procedimento isolado. Passe explicitamente o que ele precisa.

**5) Chamar um callback manualmente.**
`DO obj.%OnBeforeSave(1)` é sinal de que algo está errado no desenho. Callbacks são acionados pelo IRIS. Se a lógica precisa ser chamada à mão, ela merece um método próprio.

**6) Errar o nome do callback.**
`%OnBeforeSave` com um `S` minúsculo, ou `OnBeforeSave` sem o `%`, viram apenas métodos comuns que ninguém chama. Não há aviso: o código simplesmente nunca roda.

**7) Devolver erro de `%OnNew` sem perceber a consequência.**
Se `%OnNew` devolve erro, o `%New()` devolve OREF nula. Quem chamou fica com `""` na mão e frequentemente não confere.

**8) Achar que `%OnBeforeSave` rodou significa que gravou.**
Não significa. A validação e a própria gravação podem falhar depois. Confira sempre o `%Status`.

**9) Confundir `%OnDelete` com método de instância.**
`%OnDelete` é `ClassMethod` e recebe um **OID**, não um ID.

**10) Esquecer os parênteses do `##super()`.**
`##super` sem parênteses não compila. E, se o método tem argumentos, eles precisam ser repassados: `##super(a, b)`.

**11) Achar que passar OREF sem ponto protege o objeto.**
Não protege as propriedades. Protege apenas a variável que guarda o bilhete.

**12) Usar `$CLASSMETHOD` quando o nome é conhecido.**
Perde a verificação do compilador e o desempenho, sem ganho nenhum. Use `##class(...)` sempre que o nome for fixo.

**13) Deixar um parâmetro `Output` sem inicializar.**
Se o método sai por um caminho alternativo sem preencher o parâmetro, quem chamou fica com lixo da chamada anterior. Limpe no começo: `set result = ""`.

**14) Terminar um método com ponto e vírgula.**
Propriedades e índices terminam com `;`. Métodos terminam com `}` e nada mais.

---

## 7. MÃO NA MASSA

---

### Exercício 3.1 — `Method` contra `ClassMethod`

**a) Enunciado:** Crie `LabStudy.Demo.Counter`, persistente, com `Label As %String(MAXLEN = 50)` e `Value As %Integer [ InitialExpression = 0 ]`. Escreva:

1. Um `Method Increment(step)` que soma ao valor **deste** contador e devolve o novo valor.
2. Um `ClassMethod Describe()` que apenas escreve na tela o nome da classe, sem depender de objeto nenhum.
3. Um `ClassMethod Broken()` que tente usar `..Value` — compile, execute e observe o erro.

**b) Dica:** O erro do item 3 acontece na **execução**, não na compilação.

**c) Como testar:** `Increment` deve funcionar sobre um objeto; `Describe` deve funcionar chamado direto pela classe; `Broken` deve falhar.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Counter.cls`:

```objectscript
/// Shows the difference between instance and class methods.
Class LabStudy.Demo.Counter Extends %Persistent
{

Property Label As %String(MAXLEN = 50);

Property Value As %Integer [ InitialExpression = 0 ];

/// Instance method: acts on THIS counter.
Method Increment(step As %Integer = 1) As %Integer
{
    set ..Value = ..Value + step
    quit ..Value
}

/// Class method: belongs to no particular counter.
ClassMethod Describe() As %Status
{
    write "This is the class ", ..%ClassName(1), !
    quit $$$OK
}

/// Wrong on purpose: a class method has no "this object".
ClassMethod Broken() As %Integer
{
    quit ..Value
}

}
```

```
LABSTUDY>SET c = ##class(LabStudy.Demo.Counter).%New()
LABSTUDY>SET c.Label = "daily samples"

LABSTUDY>WRITE c.Increment(), !
1

LABSTUDY>WRITE c.Increment(10), !
11

LABSTUDY>DO ##class(LabStudy.Demo.Counter).Describe()
This is the class LabStudy.Demo.Counter

LABSTUDY>WRITE ##class(LabStudy.Demo.Counter).Broken()

WRITE ##class(LabStudy.Demo.Counter).Broken()
                                    ^
<INVALID OREF>
```

**Por que cada decisão:**

- `Increment(step As %Integer = 1)` com valor padrão permite `c.Increment()` sem argumento, o que é o uso mais comum. Valor padrão em parâmetro é sempre uma cortesia com quem chama.
- `Increment` devolve o novo valor. Poderia não devolver nada, mas devolver economiza uma leitura para quem chamou.
- `Describe` usa `..%ClassName(1)` — e isso funciona num `ClassMethod`, porque `%ClassName` também existe no nível da classe. A diferença é que `..Value` precisa de um objeto, e `..%ClassName(1)` não.
- `Broken` falha com `<INVALID OREF>`: o IRIS tentou acessar "a propriedade do objeto atual" e não havia objeto atual. Guarde essa mensagem: quando ela aparecer, quase sempre é isso ou um `%OpenId` que não encontrou nada.

---

### Exercício 3.2 — `ByRef` e `Output` na prática

**a) Enunciado:** Escreva `LabStudy.Demo.Calc` com um `ClassMethod Analyse(list, ByRef callCount, Output average, Output biggest)` que:

- recebe uma string com números separados por vírgula (por exemplo `"3,9,4,12"`);
- soma 1 ao `callCount` recebido;
- devolve pelo `QUIT` a **quantidade** de números;
- devolve pelos parâmetros de saída a **média** e o **maior** valor.

Depois, chame o método **duas vezes**: a primeira **sem** os pontos e a segunda **com** os pontos, e compare os resultados.

**b) Dica:** Para separar a string, use `$LENGTH(texto, ",")` para saber quantos pedaços existem e `$PIECE(texto, ",", i)` para pegar o pedaço `i`. Essas funções são o Capítulo 4.3; aqui use como receita.

**c) Como testar:** Na chamada sem pontos, `average` e `biggest` devem continuar indefinidas ou vazias e `callCount` não deve mudar. Na chamada com pontos, tudo deve vir preenchido.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Calc.cls`:

```objectscript
/// Demonstrates by-value, ByRef and Output argument passing.
Class LabStudy.Demo.Calc Extends %RegisteredObject
{

/// Analyses a comma separated list of numbers.
/// Returns how many numbers were found.
ClassMethod Analyse(list As %String, ByRef callCount As %Integer, Output average As %Numeric, Output biggest As %Numeric) As %Integer
{
    // never trust what came in through Output parameters
    set average = ""
    set biggest = ""

    set callCount = $GET(callCount, 0) + 1

    set count = $LENGTH(list, ",")
    if (list = "") {
        quit 0
    }

    set total = 0
    set biggest = $PIECE(list, ",", 1)

    for i = 1:1:count {
        set value = $PIECE(list, ",", i)
        set total = total + value
        if value > biggest {
            set biggest = value
        }
    }

    set average = total / count
    quit count
}

}
```

```
LABSTUDY>KILL calls, avg, max

LABSTUDY>SET calls = 0

LABSTUDY>SET n = ##class(LabStudy.Demo.Calc).Analyse("3,9,4,12", calls, avg, max)

LABSTUDY>WRITE "n=", n, " calls=", calls, !
n=4 calls=0

LABSTUDY>ZWRITE avg, max
LABSTUDY>

LABSTUDY>SET n = ##class(LabStudy.Demo.Calc).Analyse("3,9,4,12", .calls, .avg, .max)

LABSTUDY>WRITE "n=", n, " calls=", calls, !
n=4 calls=1

LABSTUDY>WRITE "avg=", avg, " max=", max, !
avg=7 max=12
```

**Por que cada decisão:**

- **Limpar os `Output` na primeira linha.** Se o método saísse cedo (lista vazia), quem chamou poderia ficar com valores de uma chamada anterior. Limpar é disciplina, não paranoia.
- **`$GET(callCount, 0)`** — se quem chamou não passou nada, `callCount` está indefinida, e somar 1 a uma variável indefinida dá `<UNDEFINED>`. `$GET(x, padrao)` devolve o valor se existir e o padrão se não existir. É uma das funções mais úteis do ObjectScript.
- **A primeira chamada, sem pontos, é o ponto do exercício.** O método rodou inteiro, calculou tudo certinho, devolveu `4` pelo `QUIT`... e nada saiu pelos parâmetros. `calls` continuou `0`, `avg` e `max` continuaram inexistentes (o `ZWRITE` não imprimiu nada). Nenhum erro foi levantado. **É assim que esse bug se esconde.**
- **`biggest` começa com o primeiro elemento**, não com zero. Se a lista fosse `"-5,-3"`, começar com zero daria a resposta errada.

---

### Exercício 3.3 — `QUIT` contra `RETURN`, e argumentos variáveis

**a) Enunciado:** Escreva `LabStudy.Demo.Flow` com:

1. `ClassMethod FindFirstNegativeWithQuit(values...)` que percorre os argumentos e, ao achar o primeiro negativo, executa `quit` dentro do `for`. Ao final, devolve o que achou.
2. `ClassMethod FindFirstNegativeWithReturn(values...)` que faz o mesmo, mas usando `return` dentro do `for`.
3. Faça os dois escreverem uma mensagem **depois** do laço, para você ver qual chega lá.

**b) Dica:** Dentro do método, `values` é a quantidade e `values(i)` são os valores.

**c) Como testar:** Chame os dois com `(5, 8, -3, 7, -1)`. O primeiro deve imprimir a mensagem pós-laço; o segundo não.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Flow.cls`:

```objectscript
/// Shows the difference between QUIT and RETURN inside a loop.
Class LabStudy.Demo.Flow Extends %RegisteredObject
{

/// QUIT leaves only the FOR loop.
ClassMethod FindFirstNegativeWithQuit(values...) As %String
{
    set found = ""
    for i = 1:1:values {
        if $GET(values(i)) < 0 {
            set found = values(i)
            quit
        }
    }
    write "[quit version] reached the code after the loop", !
    quit found
}

/// RETURN leaves the whole method.
ClassMethod FindFirstNegativeWithReturn(values...) As %String
{
    for i = 1:1:values {
        if $GET(values(i)) < 0 {
            return values(i)
        }
    }
    write "[return version] reached the code after the loop", !
    quit ""
}

}
```

```
LABSTUDY>WRITE ##class(LabStudy.Demo.Flow).FindFirstNegativeWithQuit(5,8,-3,7,-1), !
[quit version] reached the code after the loop
-3

LABSTUDY>WRITE ##class(LabStudy.Demo.Flow).FindFirstNegativeWithReturn(5,8,-3,7,-1), !
-3

LABSTUDY>WRITE ##class(LabStudy.Demo.Flow).FindFirstNegativeWithReturn(5,8,7), !
[return version] reached the code after the loop

LABSTUDY>WRITE ##class(LabStudy.Demo.Flow).FindFirstNegativeWithQuit(), !
[quit version] reached the code after the loop
```

**Por que cada decisão:**

- A versão com `quit` precisou de uma **variável auxiliar** (`found`) para carregar o resultado até depois do laço. A versão com `return` não precisou de nada: devolveu na hora.
- Repare na terceira chamada: quando **não há** negativo, a versão com `return` chega à linha depois do laço. Isso prova que o `return` não é um "sempre sai", e sim "sai quando executado".
- A quarta chamada, sem argumento nenhum, funciona: `values` vale `0`, o `for` não executa nenhuma volta, e o método devolve vazio. Métodos com argumentos variáveis precisam sobreviver ao caso de zero argumentos, e este sobrevive.
- `$GET(values(i))` protege contra argumentos pulados (`Flow(5, , 7)`).

---

### Exercício 3.4 — Callbacks em ação

**a) Enunciado:** Crie `LabStudy.Demo.Ticket`, persistente, com `Title As %String(MAXLEN = 100) [ Required ]`, `Priority As %Integer [ InitialExpression = 3 ]` e `Log As %String(MAXLEN = 300)`.

Implemente **quatro** callbacks, cada um acrescentando uma linha na tela dizendo que rodou:

1. `%OnNew(title)` — se receber um título, já o define.
2. `%OnValidateObject()` — recusa prioridade fora da faixa 1 a 5, com mensagem própria.
3. `%OnBeforeSave(insert)` — coloca o título em maiúsculas e informa se é inserção ou atualização.
4. `%OnAfterSave(insert)` — informa o ID gravado.

Depois, no Terminal, execute uma criação bem-sucedida e uma tentativa com prioridade inválida, observando quais callbacks rodam em cada caso.

**b) Dica:** Para maiúsculas, `$ZCONVERT(texto, "U")`.

**c) Como testar:** Na gravação válida, você deve ver as quatro mensagens na ordem certa. Na inválida, o `%OnAfterSave` **não** deve aparecer.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Ticket.cls`:

```objectscript
/// Demonstrates the persistence callbacks and their order.
Class LabStudy.Demo.Ticket Extends %Persistent
{

Property Title As %String(MAXLEN = 100) [ Required ];

Property Priority As %Integer [ InitialExpression = 3 ];

Property Log As %String(MAXLEN = 300);

Method %OnNew(title As %String = "") As %Status [ Private, ServerOnly = 1 ]
{
    write "1) %OnNew", !
    if title '= "" {
        set ..Title = title
    }
    quit $$$OK
}

Method %OnValidateObject() As %Status [ Private, ServerOnly = 1 ]
{
    write "2) %OnValidateObject", !
    if (..Priority < 1) || (..Priority > 5) {
        quit $$$ERROR($$$GeneralError, "Priority must be between 1 and 5, got "_..Priority)
    }
    quit $$$OK
}

Method %OnBeforeSave(insert As %Boolean) As %Status [ Private, ServerOnly = 1 ]
{
    write "3) %OnBeforeSave (insert=", insert, ")", !
    set ..Title = $ZCONVERT(..Title, "U")
    quit $$$OK
}

Method %OnAfterSave(insert As %Boolean) As %Status [ Private, ServerOnly = 1 ]
{
    write "4) %OnAfterSave, id=", ..%Id(), !
    quit $$$OK
}

}
```

```
LABSTUDY>SET t = ##class(LabStudy.Demo.Ticket).%New("broken centrifuge")
1) %OnNew

LABSTUDY>WRITE t.Title, !
broken centrifuge

LABSTUDY>SET sc = t.%Save()
2) %OnValidateObject
3) %OnBeforeSave (insert=1)
4) %OnAfterSave, id=1

LABSTUDY>WRITE $$$ISOK(sc), " / ", t.Title, !
1 / BROKEN CENTRIFUGE

LABSTUDY>SET t.Priority = 9

LABSTUDY>DO $SYSTEM.Status.DisplayError(t.%Save())
2) %OnValidateObject
ERROR #5001: Priority must be between 1 and 5, got 9

LABSTUDY>SET t.Priority = 1

LABSTUDY>SET sc = t.%Save()
2) %OnValidateObject
3) %OnBeforeSave (insert=0)
4) %OnAfterSave, id=1
```

**Por que cada decisão:**

- **A numeração nas mensagens** transforma o exercício num experimento visual. Você não precisa acreditar na ordem: você a vê.
- **`%OnNew` rodou sozinho** no `%New()`, e ninguém o chamou. Esse é o ponto do capítulo sobre callbacks.
- **Na tentativa inválida, só o callback 2 apareceu.** A validação recusou antes de tudo o mais. Compare com o exemplo da seção 4, onde a mensagem do `%OnBeforeSave` apareceu apesar da falha — a diferença está em **qual** validação recusou e em que momento. Isso mostra por que confiar em mensagens de tela para saber se gravou é péssima ideia: confie no `%Status`.
- **`insert=1` na primeira gravação e `insert=0` na segunda** — exatamente o comportamento documentado, e o que permite tratar inserção e atualização de forma diferente.
- **`$ZCONVERT(texto, "U")`** — o `"U"` significa *upper*. Existem também `"L"` (minúsculas) e outras conversões. Detalhes no Capítulo 4.3.

---

### Exercício 3.5 — PROJETO CONTÍNUO: métodos de negócio no laboratório

**a) Enunciado:** Evolua o projeto:

1. Em `LabStudy.Patient`:
   - Acrescente `%OnBeforeSave` que coloca o `RecordNumber` em maiúsculas e remove os espaços das pontas do `Name`.
   - Acrescente `%OnValidateObject` que recusa data de nascimento no futuro, com mensagem própria.
   - Acrescente `Method AddAllergy(name)` que só insere se ainda não estiver na lista, devolvendo `%Status`.
   - Acrescente `ClassMethod Statistics(Output totalPatients, Output totalExams)` que percorre os pacientes e devolve, pelo `QUIT`, quantos têm pelo menos um exame.
2. Em `LabStudy.Exam`:
   - Acrescente `%OnNew(testCode)` que já define o código do exame.
   - Acrescente `Method Describe() As %String` que devolve algo como `HGB: 13.5 g/dL`.
   - Acrescente `Method IsAbnormal() As %Boolean [ Abstract ]`? **Não** — em vez disso, acrescente um `Method IsAbnormal() As %Boolean` concreto que sempre devolve `0`, para as subclasses sobrescreverem.
3. Crie `LabStudy.UrgentExam Extends LabStudy.Exam`, com `Property RequestedBy As %String(MAXLEN = 100)`, sobrescrevendo `Describe()` para acrescentar o prefixo `[URGENT]` **usando `##super()`**.
4. Suba `LabStudy.App` para a versão `"0.4"` e faça o `Status` usar o novo `Statistics`.

**b) Dica:** Para percorrer todos os pacientes sem SQL (que ainda não vimos), use o mesmo padrão do `$ORDER` sobre a global de dados: `^LabStudy.PatientD`. Lembre que definimos `SqlTableName = PATIENT`, mas o nome da **global** continua derivado do nome da **classe**.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Patient).%KillExtent()
LABSTUDY>DO ##class(LabStudy.Exam).%KillExtent()
LABSTUDY>SET id = ##class(LabStudy.Patient).Create("  Maria Silva  ","1990-05-17","reg-001","F")
LABSTUDY>DO ##class(LabStudy.Exam).Register(id,"HGB",13.5,"g/dL")
LABSTUDY>DO ##class(LabStudy.Patient).Show(id)
LABSTUDY>DO ##class(LabStudy.App).Status()
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Acrescente a `src/LabStudy/Patient.cls` (mantendo tudo o que já existe):

```objectscript
/// Normalises data before every save.
Method %OnBeforeSave(insert As %Boolean) As %Status [ Private, ServerOnly = 1 ]
{
    set ..Name = $ZSTRIP(..Name, "<>W")
    set ..RecordNumber = $ZCONVERT($ZSTRIP(..RecordNumber, "<>W"), "U")
    quit $$$OK
}

/// Business rule that no property parameter could express.
Method %OnValidateObject() As %Status [ Private, ServerOnly = 1 ]
{
    if (..BirthDate '= "") && (..BirthDate > +$HOROLOG) {
        quit $$$ERROR($$$GeneralError, "BirthDate cannot be in the future")
    }
    quit $$$OK
}

/// Adds an allergy only if it is not already present.
Method AddAllergy(name As %String) As %Status
{
    set name = $ZSTRIP(name, "<>W")
    if name = "" {
        quit $$$ERROR($$$GeneralError, "Allergy name cannot be empty")
    }

    if ..Allergies.Find(name) '= "" {
        quit $$$ERROR($$$GeneralError, "Allergy already registered: "_name)
    }

    do ..Allergies.Insert(name)
    quit $$$OK
}

/// Counts patients and exams.
/// Returns, through QUIT, how many patients have at least one exam.
ClassMethod Statistics(Output totalPatients As %Integer, Output totalExams As %Integer) As %Integer
{
    set totalPatients = 0
    set totalExams = 0
    set withExams = 0

    set id = ""
    for {
        set id = $ORDER(^LabStudy.PatientD(id))
        quit:id=""

        set patient = ..%OpenId(id)
        continue:'$ISOBJECT(patient)

        set totalPatients = totalPatients + 1
        set examCount = patient.Exams.Count()
        set totalExams = totalExams + examCount
        if examCount > 0 {
            set withExams = withExams + 1
        }
    }

    quit withExams
}
```

Acrescente a `src/LabStudy/Exam.cls`:

```objectscript
/// Allows ##class(LabStudy.Exam).%New("HGB").
Method %OnNew(testCode As %String = "") As %Status [ Private, ServerOnly = 1 ]
{
    if testCode '= "" {
        set ..TestCode = testCode
    }
    quit $$$OK
}

/// Short readable description of this exam.
Method Describe() As %String
{
    quit ..TestCode_": "_..ResultValue_" "_..Unit
}

/// Base implementation. Subclasses may override with real reference ranges.
Method IsAbnormal() As %Boolean
{
    quit 0
}
```

`src/LabStudy/UrgentExam.cls`:

```objectscript
/// An exam requested with priority. Inherits everything from Exam.
Class LabStudy.UrgentExam Extends LabStudy.Exam
{

Property RequestedBy As %String(MAXLEN = 100);

/// Extends, rather than replaces, the parent description.
Method Describe() As %String
{
    quit "[URGENT] "_##super()
}

}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "0.4";

/// Real counters, using the Statistics method of Patient.
ClassMethod Status() As %Status
{
    set withExams = ##class(LabStudy.Patient).Statistics(.patients, .exams)
    write "Patients:            ", patients, !
    write "Exams:               ", exams, !
    write "Patients with exams: ", withExams, !
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>SET id = ##class(LabStudy.Patient).Create("  Maria Silva  ","1990-05-17","reg-001","F")

LABSTUDY>SET p = ##class(LabStudy.Patient).%OpenId(id)

LABSTUDY>WRITE "[", p.Name, "] [", p.RecordNumber, "]", !
[Maria Silva] [REG-001]

LABSTUDY>WRITE $$$ISOK(p.AddAllergy("penicillin")), !
1

LABSTUDY>DO $SYSTEM.Status.DisplayError(p.AddAllergy("penicillin"))
ERROR #5001: Allergy already registered: penicillin

LABSTUDY>WRITE $$$ISOK(p.%Save()), !
1

LABSTUDY>DO ##class(LabStudy.Exam).Register(id,"HGB",13.5,"g/dL")

LABSTUDY>SET u = ##class(LabStudy.UrgentExam).%New("TROP")
LABSTUDY>SET u.ResultValue = 0.08, u.Unit = "ng/mL", u.RequestedBy = "Dr. Souza"
LABSTUDY>SET u.Patient = p
LABSTUDY>WRITE $$$ISOK(u.%Save()), !
1

LABSTUDY>WRITE u.Describe(), !
[URGENT] TROP: 0.08 ng/mL

LABSTUDY>DO ##class(LabStudy.App).Status()
Patients:            1
Exams:               2
Patients with exams: 1
```

**Por que cada decisão:**

- **A normalização foi para o `%OnBeforeSave`, não para o `Create`.** Isso é deliberado e importante. Se estivesse no `Create`, um paciente gravado por qualquer outro caminho — pelo SQL, por uma importação, por outro método — escaparia da regra. No callback, **toda** gravação passa por ali. Regra que precisa valer sempre pertence ao callback.
- **`..BirthDate > +$HOROLOG`** — comparação de números, porque o valor lógico de `%Date` é numérico. `+$HOROLOG` é o dia de hoje. Simples e correto, sem conversão de texto.
- **`AddAllergy` devolve `%Status` em vez de silenciosamente ignorar duplicatas.** Ignorar sem avisar esconde erro de quem chamou. Recusar explicando é mais honesto.
- **`Allergies.Find(name)` devolve a posição ou vazio.** Testar `'= ""` é o jeito de perguntar "já existe?".
- **`Statistics` devolve três números por três caminhos diferentes** — dois `Output` e um `QUIT`. É exatamente o padrão que a prova cobra sobre passagem de argumentos, aplicado a um caso real. Repare que quem chama, no `App.Status()`, usa os pontos: `Statistics(.patients, .exams)`.
- **`IsAbnormal` concreto devolvendo `0`, em vez de abstrato.** Um método abstrato obrigaria **toda** subclasse a implementar. Como nem todo exame tem faixa de referência conhecida, uma implementação base neutra é mais prática. Essa é uma decisão de projeto legítima, e saber justificá-la vale mais do que decorar a regra.
- **`UrgentExam.Describe()` usa `##super()`.** Se um dia o `Describe` da classe base mudar de formato, o exame urgente acompanha automaticamente. Reescrever a formatação inteira na subclasse criaria duas verdades para manter.
- **`^LabStudy.PatientD` continua sendo o nome da global** mesmo com `SqlTableName = PATIENT`. Isso porque `SqlTableName` só afeta o mundo SQL. Confundir os dois é pegadinha de prova, e o exercício obriga você a encarar isso.

---

## 8. Quiz do capítulo

**Q1.** Qual afirmação sobre `Method` e `ClassMethod` está correta?

- A) Ambos podem usar `..Propriedade`.
- B) `Method` age sobre um objeto e pode usar `..Propriedade`; `ClassMethod` é chamado pela classe e não tem objeto atual.
- C) `ClassMethod` só pode existir em classes persistentes.
- D) `Method` não pode devolver valor.

---

**Q2.** Analise:

```objectscript
ClassMethod Change(ByRef x As %Integer)
{
    set x = 99
}
```

E a chamada:

```
SET a = 1
DO ##class(X).Change(a)
WRITE a
```

O que é impresso?

- A) `99`
- B) `1`
- C) Vazio
- D) Erro `<UNDEFINED>`

---

**Q3.** Qual é a diferença entre `ByRef` e `Output`?

- A) Não há diferença técnica; `ByRef` indica entrada e saída, `Output` indica apenas saída.
- B) `Output` funciona sem o ponto na chamada; `ByRef` exige o ponto.
- C) `ByRef` só aceita números.
- D) `Output` cria uma cópia do valor.

---

**Q4.** Considere:

```objectscript
ClassMethod Test() As %String
{
    for i = 1:1:5 {
        if i = 2 { quit }
    }
    write "A"
    quit "B"
}
```

O que acontece ao chamar `WRITE ##class(X).Test()`?

- A) Imprime apenas `B`.
- B) Imprime `AB`.
- C) Não imprime nada.
- D) Erro de compilação.

---

**Q5.** Num método declarado como `ClassMethod Sum(values...)`, o que a variável `values` (sem parênteses) contém?

- A) O primeiro argumento.
- B) A quantidade de argumentos recebidos.
- C) Uma lista com todos os valores concatenados.
- D) Sempre vazio.

---

**Q6.** Um método cria a variável `temp`. Outro método da mesma classe é chamado logo em seguida. Ele enxerga `temp`?

- A) Sim, variáveis de método são compartilhadas na classe.
- B) Não; métodos são blocos de procedimento e suas variáveis são privadas.
- C) Só se `temp` começar com `%`.
- D) Só em classes persistentes.

---

**Q7.** O que acontece se `%OnNew()` devolver um `%Status` de erro?

- A) Nada; o objeto é criado normalmente.
- B) `%New()` devolve uma OREF nula e o objeto não é criado.
- C) O erro só aparece no `%Save()`.
- D) Erro de compilação.

---

**Q8.** No callback `%OnBeforeSave(insert As %Boolean)`, o que significa `insert = 0`?

- A) A gravação falhou.
- B) O objeto está sendo atualizado, não inserido pela primeira vez.
- C) O objeto será apagado.
- D) A validação foi desligada.

---

**Q9.** Qual afirmação sobre `%OnDelete` está correta?

- A) É um `Method` de instância e recebe o ID.
- B) É um `ClassMethod` e recebe o OID.
- C) É chamado depois que o objeto já foi apagado.
- D) Não pode cancelar o apagamento.

---

**Q10.** Numa subclasse, como chamar a implementação da superclasse do mesmo método?

- A) `##class(Superclasse).Metodo()`
- B) `..Metodo()`
- C) `##super()`
- D) `$METHOD("super")`

---

**Q11.** Um método recebe uma OREF como argumento **sem** `ByRef` e altera uma propriedade do objeto. Quem chamou enxerga a alteração?

- A) Não, porque a passagem foi por valor.
- B) Sim: a cópia é da referência, mas ela aponta para o mesmo objeto.
- C) Só se o objeto tiver sido gravado antes.
- D) Só se a propriedade for `Transient`.

---

**Q12.** Como criar um `%Status` de erro com uma mensagem própria?

- A) `quit "error"`
- B) `quit $$$ERROR($$$GeneralError, "minha mensagem")`
- C) `quit 0`
- D) `write "erro" quit $$$OK`

---

**Q13.** Qual é o efeito da palavra-chave `[ CodeMode = expression ]`?

- A) O corpo do método é uma única expressão cujo resultado é devolvido, sem `quit`.
- B) O método passa a ser chamado por SQL.
- C) O método vira abstrato.
- D) O método não pode ser sobrescrito.

---

**Q14.** Qual chamada executa dinamicamente um método de instância cujo nome está numa variável?

- A) `$CLASSMETHOD(oref, nome)`
- B) `$METHOD(oref, nome)`
- C) `$PROPERTY(oref, nome)`
- D) `##class(oref).nome()`

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** `Method` roda sobre um objeto e tem acesso a `..Propriedade`; `ClassMethod` não tem objeto atual.
- **A está errada:** num `ClassMethod`, `..Propriedade` falha em execução com `<INVALID OREF>`.
- **C está errada:** `ClassMethod` existe em qualquer classe.
- **D está errada:** métodos de instância podem devolver valor normalmente.

**Q2 — Resposta: B.**
- **B está certa:** faltou o **ponto** na chamada. Sem ele, o método recebeu uma cópia, e a variável original não mudou. A declaração `ByRef` sozinha não basta.
- **A está errada:** seria `99` apenas com `Change(.a)`.
- **C está errada:** `a` continua existindo com o valor original.
- **D está errada:** nenhum erro é levantado — e é justamente por isso que esse bug é perigoso.

**Q3 — Resposta: A.**
- **A está certa:** a diferença é de intenção documentada. `ByRef` = entra e sai; `Output` = só sai. Ambos exigem o ponto na chamada.
- **B está errada:** os dois precisam do ponto.
- **C está errada:** não há restrição de tipo.
- **D está errada:** `Output` não copia; ele escreve na variável de quem chamou.

**Q4 — Resposta: B.**
- **B está certa:** o `quit` dentro do `for` encerra apenas o laço. O fluxo segue, escreve `A` e devolve `B`. O `WRITE` externo imprime o `B`.
- **A está errada:** o `write "A"` é alcançado.
- **C está errada:** as duas saídas ocorrem.
- **D está errada:** o código compila normalmente.

**Q5 — Resposta: B.**
- **B está certa:** `values` contém a contagem; os valores ficam em `values(1)`, `values(2)`, etc.
- **A está errada:** o primeiro argumento é `values(1)`.
- **C está errada:** não há concatenação automática.
- **D está errada:** vale `0` quando nada é passado, o que ainda é um valor útil.

**Q6 — Resposta: B.**
- **B está certa:** por padrão, métodos de classe são blocos de procedimento com variáveis privadas, criadas e destruídas a cada chamada.
- **A está errada:** não há compartilhamento automático.
- **C está errada:** o prefixo `%` é reservado ao sistema e não deve ser usado para isso.
- **D está errada:** o comportamento independe do tipo da classe.

**Q7 — Resposta: B.**
- **B está certa:** um erro em `%OnNew` aborta a criação e `%New()` devolve OREF nula.
- **A está errada:** o objeto não é entregue.
- **C está errada:** a falha é imediata, na criação.
- **D está errada:** é comportamento de execução, não de compilação.

**Q8 — Resposta: B.**
- **B está certa:** `insert` vale `1` na primeira gravação e `0` nas atualizações.
- **A está errada:** o callback roda antes de saber se vai dar certo.
- **C está errada:** apagamento tem callbacks próprios.
- **D está errada:** validação não é controlada por esse argumento.

**Q9 — Resposta: B.**
- **B está certa:** `%OnDelete` é `ClassMethod` e recebe o OID.
- **A está errada:** não é método de instância nem recebe ID simples.
- **C está errada:** é chamado antes do apagamento.
- **D está errada:** devolvendo erro, ele cancela o apagamento.

**Q10 — Resposta: C.**
- **C está certa:** `##super()` chama a implementação da superclasse do mesmo método, repassando argumentos se houver.
- **A está errada:** chamar pela superclasse dessa forma perde o contexto do objeto atual e não é o mecanismo previsto.
- **B está errada:** `..Metodo()` chamaria a própria implementação, gerando recursão infinita.
- **D está errada:** não existe essa sintaxe.

**Q11 — Resposta: B.**
- **B está certa:** o que foi copiado é a referência; ela continua apontando para o mesmo objeto, então mudanças nas propriedades são visíveis.
- **A está errada:** passagem por valor protege a variável, não o objeto apontado.
- **C está errada:** independe de ter sido gravado.
- **D está errada:** independe do tipo da propriedade.

**Q12 — Resposta: B.**
- **B está certa:** `$$$ERROR($$$GeneralError, texto)` constrói um `%Status` de erro com mensagem própria.
- **A está errada:** uma string comum não é um `%Status` válido.
- **C está errada:** `0` não carrega mensagem nem estrutura de status.
- **D está errada:** escrever na tela e devolver sucesso engana quem chamou.

**Q13 — Resposta: A.**
- **A está certa:** com `CodeMode = expression`, o corpo é uma expressão única cujo valor é devolvido.
- **B está errada:** quem faz isso é `SqlProc`.
- **C está errada:** quem faz isso é `Abstract`.
- **D está errada:** quem faz isso é `Final`.

**Q14 — Resposta: B.**
- **B está certa:** `$METHOD(oref, "NomeDoMetodo", args...)` chama um método de instância dinamicamente.
- **A está errada:** `$CLASSMETHOD` recebe o **nome da classe**, não uma OREF.
- **C está errada:** `$PROPERTY` acessa propriedades, não métodos.
- **D está errada:** `##class()` espera um nome de classe literal, não uma OREF.

---

## 9. Resumo relâmpago

1. `Method` age sobre um objeto e pode usar `..Propriedade`; `ClassMethod` é chamado pela classe e **não tem objeto atual**.
2. Assinatura: `Method Nome(param As Tipo = padrao) As TipoRetorno [ palavrasChave ] { }`. Métodos **não** terminam com `;`.
3. `QUIT` sai do **bloco atual** (inclusive de um `FOR`); `RETURN` sai do **método inteiro**.
4. O padrão é passagem **por valor**. `ByRef` (entra e sai) e `Output` (só sai) documentam a intenção.
5. **É o ponto na chamada** (`.variavel`) que efetivamente entrega a variável ao método. Sem ponto, nada volta — e nenhum erro é levantado.
6. Sempre limpe os parâmetros `Output` no início do método.
7. `args...` recebe quantos argumentos vierem: `args` é a **contagem**, `args(i)` são os valores. Proteja com `$GET`.
8. Métodos são **blocos de procedimento**: variáveis são privadas e somem no fim. Não enxergam as de quem chamou.
9. Palavras-chave úteis: `Private`, `Final`, `Abstract`, `CodeMode = expression`, `SqlProc` + `SqlName`, `PublicList`.
10. `##super()` chama a implementação da superclasse; parênteses obrigatórios, argumentos repassados.
11. Callbacks são chamados pelo IRIS: `%OnNew`, `%OnValidateObject`, `%OnBeforeSave(insert)`, `%OnAfterSave(insert)`, `%OnOpen`, `%OnDelete(oid)`.
12. `%OnNew` devolvendo erro faz `%New()` entregar OREF nula.
13. `%OnBeforeSave` devolvendo erro **cancela** a gravação. `insert = 1` é inserção; `insert = 0` é atualização.
14. `%OnDelete` é **`ClassMethod`** e recebe **OID**.
15. Regra que precisa valer sempre vai no **callback**, não num método de conveniência.
16. Erro próprio: `quit $$$ERROR($$$GeneralError, "mensagem")`.
17. Chamada dinâmica: `$CLASSMETHOD(classe, metodo, ...)`, `$METHOD(oref, metodo, ...)`, `$PROPERTY(oref, prop)`.
18. Passar OREF sem ponto **não** protege o objeto: alterações nas propriedades são visíveis para quem chamou.

---

## 10. Cartões de memorização

**Frente:** Quando usar `ClassMethod` em vez de `Method`?
**Verso:** Quando a ação não depende de um objeto específico — criar, contar, buscar, utilidades.

**Frente:** O que acontece ao usar `..Propriedade` dentro de um `ClassMethod`?
**Verso:** Erro `<INVALID OREF>` em execução: não existe objeto atual.

**Frente:** Diferença entre `QUIT` e `RETURN`.
**Verso:** `QUIT` encerra o bloco atual (um `FOR`, por exemplo). `RETURN` encerra o método inteiro.

**Frente:** O que realmente faz um argumento voltar preenchido?
**Verso:** O **ponto** antes da variável na chamada: `Metodo(.minhaVar)`.

**Frente:** Diferença entre `ByRef` e `Output`.
**Verso:** `ByRef` entra e sai; `Output` serve só para devolver. Ambos exigem o ponto na chamada.

**Frente:** O que fazer no início de um método com parâmetro `Output`?
**Verso:** Limpá-lo (`set result = ""`), para não devolver lixo de chamadas anteriores.

**Frente:** Em `ClassMethod Sum(values...)`, o que é `values`?
**Verso:** A quantidade de argumentos recebidos. Os valores estão em `values(1)`, `values(2)`, ...

**Frente:** Um método enxerga as variáveis de quem o chamou?
**Verso:** Não. Métodos são blocos de procedimento com variáveis privadas.

**Frente:** Como chamar a versão da superclasse de um método sobrescrito?
**Verso:** `##super()` — com parênteses, repassando os argumentos se houver.

**Frente:** Quando o IRIS chama `%OnNew()`?
**Verso:** Logo depois do `%New()`, passando a ele os argumentos dados ao `%New()`.

**Frente:** O que acontece se `%OnNew()` devolver erro?
**Verso:** O `%New()` devolve OREF nula e o objeto não é criado.

**Frente:** O que significa `insert = 1` em `%OnBeforeSave`?
**Verso:** É a primeira gravação do objeto (inserção). `0` significa atualização.

**Frente:** Onde colocar uma regra de normalização que precisa valer em toda gravação?
**Verso:** No `%OnBeforeSave`, não num método de conveniência que alguém pode contornar.

**Frente:** `%OnDelete` é de classe ou de instância? Recebe o quê?
**Verso:** É `ClassMethod` e recebe o **OID**.

**Frente:** Onde colocar uma validação que envolve duas propriedades ao mesmo tempo?
**Verso:** No `%OnValidateObject()`, devolvendo `$$$ERROR(...)` quando a regra for violada.

**Frente:** Como criar um erro com mensagem própria?
**Verso:** `quit $$$ERROR($$$GeneralError, "texto da mensagem")`.

**Frente:** O que faz `[ CodeMode = expression ]`?
**Verso:** O corpo do método é uma única expressão, cujo valor é devolvido sem `quit`.

**Frente:** O que faz `[ SqlProc ]`?
**Verso:** Expõe o método como procedimento armazenado, chamável de dentro do SQL. `SqlName` define o nome usado lá.

**Frente:** Como chamar dinamicamente um método de classe e um de instância?
**Verso:** `$CLASSMETHOD(nomeDaClasse, nomeDoMetodo, ...)` e `$METHOD(oref, nomeDoMetodo, ...)`.

**Frente:** Passei uma OREF sem ponto e alterei uma propriedade dentro do método. Quem chamou vê?
**Verso:** Vê sim. A cópia é do bilhete; o objeto apontado é o mesmo.

---

Digite CONTINUAR para o próximo capítulo.
