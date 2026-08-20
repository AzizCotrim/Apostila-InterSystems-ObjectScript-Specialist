# Apostila InterSystems ObjectScript Specialist
## Capítulo 21 — T5.2 Handles and Logs Runtime Errors (Tratando e registrando erros)

> Ainda em **T5 — Handles and Resolves Errors**. O capítulo anterior tratou de **enxergar** o erro. Este trata de **reagir** a ele: capturar, decidir, registrar e propagar — sem esconder nada e sem deixar o sistema em estado inconsistente.

---

## 1. O que você vai saber fazer ao terminar

1. Usar **`TRY`** e **`CATCH`** corretamente, sabendo o que cada bloco protege.
2. Ler o **objeto de exceção**: `Name`, `Code`, `Location`, `Data`, `DisplayString()`.
3. Lançar exceções com **`THROW`**, inclusive **relançar** a atual.
4. Criar exceções próprias com **`%Exception.General`** e **`%Exception.StatusException`**.
5. Converter entre os dois mundos: **`%Status`** e **exceção**.
6. Usar **`$$$ThrowOnError`**, **`AsStatus()`** e **`CreateFromStatus()`**.
7. Combinar e decompor `%Status` com `AppendStatus` e `DecomposeStatus`.
8. Decidir **onde** tratar: propagar ou resolver.
9. Combinar **transação** e `try`/`catch` com o padrão do `entryLevel`.
10. Implementar **repetição com espera** para falhas transitórias.
11. Reconhecer o tratamento legado com **`$ZTRAP`** e o comando **`ZTRAP`**.
12. Registrar erros com contexto suficiente para investigar depois.
13. Levar o projeto à versão **2.2**, com tratamento de erros consistente em todas as camadas.

---

## 2. O conceito em linguagem de gente

### 2.1 Dois mundos de sinalização de erro

O ObjectScript tem **duas** formas de sinalizar que algo deu errado, e você precisa transitar entre elas com naturalidade.

**Mundo 1 — o `%Status`: o erro como valor devolvido.**

O método devolve um valor que diz "deu certo" ou "deu errado, e o motivo foi este". Quem chama **verifica**.

```objectscript
set sc = patient.%Save()
if $$$ISERR(sc) {
    // trata
}
```

É o mundo das APIs do próprio IRIS: `%Save()`, `%DeleteId()`, `%ValidateObject()`.

**Mundo 2 — a exceção: o erro como interrupção.**

Algo dá errado e a execução **para**, subindo pela pilha de chamadas até alguém capturar.

```objectscript
try {
    set x = 1 / 0
}
catch e {
    // trata
}
```

É o mundo dos erros de execução: `<DIVIDE>`, `<UNDEFINED>`, `<INVALID OREF>`.

A analogia dos dois mundos:

- **`%Status` é o bilhete devolvido pelo balcão:** *"não foi possível, motivo tal"*. Você tem que ler o bilhete. Se jogar fora sem ler, nunca saberá.
- **A exceção é o alarme de incêndio:** ele toca e todo mundo para, subindo pelas escadas até alguém que saiba o que fazer. Ninguém precisa lembrar de verificar.

Cada um tem uma vantagem: o `%Status` é explícito e não interrompe o fluxo; a exceção não pode ser ignorada por esquecimento.

**A regra prática:**

> **Use `%Status` para erros esperados de negócio.**
> **Use exceção para condições excepcionais e erros de programação.**
> **E converta entre os dois na fronteira das camadas.**

### 2.2 O erro mais grave: engolir a exceção

```objectscript
try {
    do ..OperacaoImportante()
}
catch e {
    // nada aqui
}
```

Este código é pior do que não ter tratamento nenhum. Sem o `try`, o erro apareceria, alguém veria, e o problema seria corrigido. Com ele, **o sistema continua como se tudo estivesse bem** — e o dado que deveria ter sido gravado não foi.

O nome disso é **engolir a exceção**, e é o antipadrão número um de tratamento de erros.

Um `catch` legítimo faz pelo menos uma destas coisas:

1. **Resolve** o problema (tenta outro caminho, usa um valor padrão documentado).
2. **Registra** o erro com contexto suficiente para investigação.
3. **Propaga** a falha para quem chamou, como exceção ou como `%Status`.

Um `catch` que não faz nenhuma das três é um bug.

### 2.3 Onde tratar

A pergunta não é *"devo tratar este erro?"* — é *"onde?"*.

A regra: **trate no nível que tem informação para decidir**.

Um método utilitário que converte texto em data **não sabe** o que fazer quando a data é inválida. Ignorar? Usar hoje? Recusar o registro inteiro? Ele não tem contexto para responder. Ele deve **propagar**.

Já o método que importa um arquivo **sabe**: pular a linha, registrar o problema, e continuar com as demais.

Analogia: o eletricista que encontra um fio rompido não decide sozinho se a obra para. Ele reporta a quem tem visão do todo.

Isso leva ao padrão das **camadas**:

| Camada | Papel diante do erro |
|---|---|
| utilitários | **propagam** — não têm contexto para decidir |
| serviços de negócio | **decidem** — traduzem para regras do domínio |
| fronteira (API, tela, importação) | **capturam tudo**, registram e devolvem resposta adequada |

**Nenhum erro deve escapar da fronteira sem tratamento.** Um erro que chega ao usuário como `<UNDEFINED>zMetodo+7^Classe.1` é uma falha de desenho, não só de código.

### 2.4 O erro transitório

Nem todo erro é permanente. Um serviço externo fora do ar por dois segundos, uma trava disputada, um pico de carga — todos falham agora e funcionam daqui a pouco.

Para esses, a resposta certa não é falhar nem ignorar: é **tentar de novo**.

E tentar de novo tem regras:

- **Um número máximo de tentativas.** Repetir indefinidamente transforma uma falha em travamento.
- **Espera entre as tentativas**, e preferencialmente **crescente**. Cem processos tentando de novo no mesmo instante pioram a situação que causou a falha.
- **Só para erros transitórios.** Repetir um `<UNDEFINED>` é desperdício: ele vai falhar igual nas dez tentativas.

Distinguir transitório de permanente é a parte difícil, e depende do domínio.

---

## 3. `TRY` e `CATCH`

### 3.1 A forma básica

```objectscript
try {
    // código que pode falhar
}
catch exception {
    // tratamento
}
```

- O bloco **`try`** delimita o trecho protegido.
- O bloco **`catch`** recebe o **objeto de exceção** numa variável.
- Se nenhuma exceção ocorrer, o `catch` **não** executa.
- Se ocorrer, a execução **salta** do ponto do erro para o `catch`. O restante do `try` **não** roda.

O nome da variável é livre. Também é possível escrever `catch { }` sem variável, quando a identidade do erro não importa — mas isso é raro e quase sempre indica que você vai engolir a exceção.

**Não existe `finally`** no ObjectScript. Se você precisa garantir uma limpeza, escreva-a nos dois caminhos ou use o padrão do `entryLevel` da seção 5.

### 3.2 O objeto de exceção

```objectscript
catch e {
    write "nome     : ", e.Name, !
    write "codigo   : ", e.Code, !
    write "local    : ", e.Location, !
    write "dados    : ", e.Data, !
    write "texto    : ", e.DisplayString(), !
    write "como sc  : ", $SYSTEM.Status.GetErrorText(e.AsStatus()), !
}
```

| Membro | Conteúdo |
|---|---|
| **`Name`** | o nome do erro, como `<DIVIDE>` |
| **`Code`** | o código numérico |
| **`Location`** | rotina e deslocamento onde ocorreu |
| **`Data`** | informação adicional, quando houver |
| **`DisplayString()`** | texto pronto para exibição |
| **`AsStatus()`** | converte a exceção em **`%Status`** |

Existem também métodos para obter a pilha de chamadas no momento do erro, cujo nome e formato variam por versão: **verificar na documentação oficial**.

A propriedade **`Location`** é a mais valiosa na investigação: ela traz o `Metodo+N^Classe.1` que você aprendeu a ler no Capítulo 20.

### 3.3 As classes de exceção

| Classe | Quando aparece |
|---|---|
| **`%Exception.AbstractException`** | a base de todas |
| **`%Exception.SystemException`** | erros de execução do sistema (`<DIVIDE>`, `<UNDEFINED>`) |
| **`%Exception.StatusException`** | uma exceção que carrega um `%Status` |
| **`%Exception.General`** | exceção própria, criada por você |

Você pode testar o tipo, como no Capítulo 18:

```objectscript
catch e {
    if e.%Extends("%Exception.SystemException") {
        write "erro do sistema", !
    } else {
        write "erro de aplicacao", !
    }
}
```

### 3.4 `THROW` — lançar uma exceção

**Criando uma exceção própria:**

```objectscript
throw ##class(%Exception.General).%New(
    "InvalidRecordNumber",          // nome
    5001,                           // código
    ,                               // localização (deixe vazio)
    "Numero de registro invalido: "_rec)   // dados
```

**A partir de um `%Status`:**

```objectscript
set sc = patient.%Save()
if $$$ISERR(sc) {
    throw ##class(%Exception.StatusException).CreateFromStatus(sc)
}
```

Ou, mais curto, com a macro:

```objectscript
$$$ThrowOnError(patient.%Save())
```

**`$$$ThrowOnError(expressão)`** avalia a expressão e, se o `%Status` for de erro, lança a exceção correspondente. É a ponte mais direta do mundo `%Status` para o mundo das exceções.

**Relançando a exceção atual:**

```objectscript
catch e {
    do ##class(LabStudy.Log).Error("import", "falha ao importar", e.DisplayString())
    throw                        // relança para quem chamou decidir
}
```

Um **`THROW` sem argumento**, dentro de um `catch`, relança a exceção que está sendo tratada. Esse é o padrão de "registro aqui, decisão lá em cima" — e preserva o erro original em vez de criar um novo.

A disponibilidade exata das macros e a assinatura de `%Exception.General.%New()` variam entre versões: **verificar na documentação oficial**.

---

## 4. Trabalhando com `%Status`

### 4.1 Criando e testando

```objectscript
quit $$$OK                                          // sucesso
quit $$$ERROR($$$GeneralError, "mensagem")          // erro próprio

if $$$ISOK(sc) { }
if $$$ISERR(sc) { }
```

Já visto no Capítulo 3. Vale acrescentar duas funções úteis:

```objectscript
write $SYSTEM.Status.GetErrorText(sc), !            // texto do erro
do $SYSTEM.Status.DisplayError(sc)                  // imprime na tela
write $SYSTEM.Status.IsError(sc), !                 // equivalente a $$$ISERR
```

### 4.2 Combinando erros

Uma operação pode acumular **vários** problemas. `%Status` suporta isso:

```objectscript
set sc = $$$OK

if nome = "" {
    set sc = $SYSTEM.Status.AppendStatus(sc, $$$ERROR($$$GeneralError, "nome obrigatorio"))
}

if data = "" {
    set sc = $SYSTEM.Status.AppendStatus(sc, $$$ERROR($$$GeneralError, "data obrigatoria"))
}

quit sc
```

**`AppendStatus(sc1, sc2)`** combina dois status num só. O resultado é de erro se qualquer um dos dois for.

Isso é muito melhor do que devolver o **primeiro** erro encontrado: o usuário corrige tudo de uma vez, em vez de descobrir um problema por tentativa.

### 4.3 Decompondo

```objectscript
do $SYSTEM.Status.DecomposeStatus(sc, .erros)

for i = 1:1:$GET(erros, 0) {
    write i, ": ", erros(i), !
}
```

**`DecomposeStatus(sc, .array)`** separa um `%Status` composto num array com cada mensagem. É o que permite exibir uma lista de problemas em vez de um texto único e longo.

### 4.4 Convertendo entre os mundos

```objectscript
// exceção -> %Status
catch e {
    quit e.AsStatus()
}

// %Status -> exceção
$$$ThrowOnError(sc)

// %Status -> exceção, forma explícita
if $$$ISERR(sc) {
    throw ##class(%Exception.StatusException).CreateFromStatus(sc)
}
```

O padrão profissional é escolher **um** mundo por camada e converter na fronteira:

```objectscript
/// Public API: returns %Status, never throws.
ClassMethod ImportFile(path As %String) As %Status
{
    try {
        do ..DoImport(path)          // internally, everything throws
    }
    catch e {
        do ##class(LabStudy.Log).Error("import", "falha na importacao", e.DisplayString())
        quit e.AsStatus()
    }
    quit $$$OK
}
```

Por dentro, o código usa exceções — mais limpo, sem verificar retorno a cada linha. Na fronteira, converte para `%Status` — que é o que a convenção do IRIS espera de uma API pública.

---

## 5. Padrões de tratamento

### 5.1 Transação com `try`/`catch`

Este é o padrão que o Capítulo 5 anunciou:

```objectscript
ClassMethod Operacao() As %Status
{
    set sc = $$$OK
    set entryLevel = $TLEVEL

    try {
        tstart

        // ... várias gravações ...
        $$$ThrowOnError(objeto1.%Save())
        $$$ThrowOnError(objeto2.%Save())

        tcommit
    }
    catch e {
        while $TLEVEL > entryLevel {
            trollback 1
        }
        set sc = e.AsStatus()

        do ##class(LabStudy.Log).Error("operacao", "desfeita por erro", e.DisplayString())
    }

    quit sc
}
```

Por que este padrão é superior ao do Capítulo 5:

- **Qualquer** erro cai no `catch`, inclusive os que você não previu.
- O rollback está num lugar só, em vez de repetido em cada caminho de saída.
- O `entryLevel` garante que a transação do chamador não seja derrubada.
- O erro é registrado **e** propagado.

Compare com o `CreateWithExams` do Capítulo 5, que tinha quatro caminhos de saída com o mesmo bloco de rollback repetido. Aqui, um.

### 5.2 Repetição com espera crescente

```objectscript
ClassMethod ComRepeticao(maxTentativas As %Integer = 3) As %Status
{
    set espera = 0.5

    for tentativa = 1:1:maxTentativas {
        try {
            do ..OperacaoQuePodeFalhar()
            quit                                  // sucesso: sai do laço
        }
        catch e {
            if '..EhTransitorio(e) {
                do ##class(LabStudy.Log).Error("retry", "erro permanente", e.DisplayString())
                quit
            }

            if tentativa = maxTentativas {
                do ##class(LabStudy.Log).Error("retry",
                    "falhou apos "_maxTentativas_" tentativas", e.DisplayString())
                quit
            }

            do ##class(LabStudy.Log).Warn("retry",
                "tentativa "_tentativa_" falhou, repetindo em "_espera_"s",
                e.DisplayString())

            hang espera
            set espera = espera * 2               // espera crescente
        }
    }

    quit $$$OK
}
```

Três decisões visíveis no código:

- **Erro permanente sai imediatamente.** Repetir um `<UNDEFINED>` é desperdiçar tempo.
- **A última tentativa registra como erro**, não como aviso.
- **A espera dobra a cada vez.** Se cem processos falharam juntos, eles não voltarão todos no mesmo instante.

E a parte difícil, que depende do domínio:

```objectscript
ClassMethod EhTransitorio(e As %Exception.AbstractException) As %Boolean
{
    // erros que costumam se resolver sozinhos
    quit:e.Name["LOCK" 1
    quit:e.Name["TIMEOUT" 1
    quit:e.DisplayString()["connection" 1

    quit 0
}
```

**Esta lista é uma decisão de projeto, não uma verdade universal.** Ela precisa ser combinada com quem conhece o ambiente.

### 5.3 Registro com contexto

Um registro de erro útil responde a cinco perguntas:

```objectscript
catch e {
    do ##class(LabStudy.Log).Error(
        "import",                                       // ONDE (área)
        "falha ao processar linha "_numeroLinha,        // O QUÊ
        e.DisplayString()                               // POR QUÊ
        _" | linha=["_conteudoLinha_"]"                 // COM QUAIS DADOS
        _" | "_##class(LabStudy.Diagnostics).Context()) // EM QUE CONTEXTO
}
```

O `Context()` do Capítulo 20 já traz usuário, processo, namespace e horário.

**O erro mais comum no registro de erros é registrar pouco.** "Falha na importação" não permite investigar nada. Registre o dado que causou o problema — respeitando, claro, o Capítulo 7: **nunca registre senhas nem dados sensíveis em claro**.

### 5.4 Não capture o que você não pode tratar

```objectscript
// Ruim: captura tudo e devolve um erro genérico, perdendo a informação
try {
    do ..Trabalho()
}
catch e {
    quit $$$ERROR($$$GeneralError, "erro na operacao")
}

// Bom: preserva a informação original
try {
    do ..Trabalho()
}
catch e {
    do ##class(LabStudy.Log).Error("area", "erro na operacao", e.DisplayString())
    quit e.AsStatus()
}
```

O primeiro descarta `Name`, `Location` e `Data`, substituindo por um texto inútil. Quem investigar depois não terá nada.

**Preserve a exceção original.** Se precisar acrescentar contexto, registre-o no log e relance com `throw`.

### 5.5 O tratamento legado: `$ZTRAP`

Antes do `try`/`catch`, o ObjectScript tratava erros assim:

```objectscript
MeuMetodo() {
    set $ZTRAP = "TratarErro"

    // ... código ...
    quit

TratarErro
    set $ZTRAP = ""
    write "erro: ", $ZERROR, !
    quit
}
```

**`$ZTRAP`** recebe o nome de um rótulo. Quando um erro ocorre, a execução salta para lá.

E o comando **`ZTRAP`** provoca um erro deliberadamente:

```objectscript
ztrap "MEUERRO"
```

**Não use `$ZTRAP` em código novo.** Ele não tem escopo de bloco, não traz objeto de exceção, e o fluxo é muito mais difícil de acompanhar. Conheça para ler código legado — e para a prova, que pode cobrar o reconhecimento.

Um detalhe: `$ZTRAP` e `try`/`catch` podem coexistir no mesmo sistema, mas misturá-los no mesmo método produz um fluxo que ninguém consegue prever. Escolha um por método.

---

## 6. Exemplo comentado

Arquivo `src/LabStudy/Demo/Err.cls`:

```objectscript
/// Error handling patterns.
Class LabStudy.Demo.Err Extends %RegisteredObject
{

/// Basic try/catch and the exception object.
ClassMethod Basics() As %Status
{
    write "-- erro do sistema --", !
    try {
        set x = 1 / 0
        write "  esta linha nunca executa", !
    }
    catch e {
        write "  nome     : ", e.Name, !
        write "  codigo   : ", e.Code, !
        write "  local    : ", e.Location, !
        write "  texto    : ", e.DisplayString(), !
        write "  classe   : ", e.%ClassName(1), !
        write "  sistema? : ", e.%Extends("%Exception.SystemException"), !
    }

    write !, "-- o resto do TRY nao executa --", !
    set marcador = "antes"
    try {
        set marcador = "dentro, antes do erro"
        set y = 1 / 0
        set marcador = "dentro, depois do erro"
    }
    catch e {
        write "  marcador ficou em: [", marcador, "]", !
    }

    write !, "-- sem erro, o CATCH nao executa --", !
    try {
        write "  tudo certo", !
    }
    catch e {
        write "  esta linha nunca executa", !
    }

    quit $$$OK
}

/// Throwing exceptions.
ClassMethod Throwing(kind As %String = "general") As %Status
{
    try {
        if kind = "general" {
            throw ##class(%Exception.General).%New(
                "RegistroInvalido", 5001, , "numero de registro fora do padrao")
        }
        elseif kind = "status" {
            set sc = $$$ERROR($$$GeneralError, "erro criado como %Status")
            throw ##class(%Exception.StatusException).CreateFromStatus(sc)
        }
        elseif kind = "macro" {
            $$$ThrowOnError($$$ERROR($$$GeneralError, "lancado pela macro"))
        }
        else {
            set z = 1 / 0
        }
    }
    catch e {
        write "  tipo lancado : ", kind, !
        write "    nome       : ", e.Name, !
        write "    codigo     : ", e.Code, !
        write "    dados      : ", e.Data, !
        write "    texto      : ", e.DisplayString(), !
        write "    como status: ", $SYSTEM.Status.GetErrorText(e.AsStatus()), !
    }

    quit $$$OK
}

/// Combining several validation errors into one %Status.
ClassMethod ValidateAll(name As %String, birth As %String, record As %String) As %Status
{
    set sc = $$$OK

    if $ZSTRIP(name, "<>W") = "" {
        set sc = $SYSTEM.Status.AppendStatus(sc,
            $$$ERROR($$$GeneralError, "nome e obrigatorio"))
    }

    if '##class(LabStudy.Text).IsIsoDate(birth) {
        set sc = $SYSTEM.Status.AppendStatus(sc,
            $$$ERROR($$$GeneralError, "data de nascimento invalida: ["_birth_"]"))
    }

    if '##class(LabStudy.Text).IsRecordNumber(record) {
        set sc = $SYSTEM.Status.AppendStatus(sc,
            $$$ERROR($$$GeneralError, "numero de registro invalido: ["_record_"]"))
    }

    quit sc
}

/// Shows a composite status decomposed into individual messages.
ClassMethod ShowErrors(sc As %Status) As %Status
{
    if $$$ISOK(sc) {
        write "  tudo valido", !
        quit $$$OK
    }

    do $SYSTEM.Status.DecomposeStatus(sc, .errors)

    set total = $GET(errors, 0)
    write "  ", total, " problema(s):", !

    for i = 1:1:total {
        write "    ", i, ". ", errors(i), !
    }

    quit $$$OK
}

/// The recommended transaction pattern.
ClassMethod TransactionalWork(shouldFail As %Boolean = 0) As %Status
{
    set sc = $$$OK
    set entryLevel = $TLEVEL

    kill ^DemoErrTx

    try {
        tstart

        set ^DemoErrTx(1) = "primeiro"
        set ^DemoErrTx(2) = "segundo"

        if shouldFail {
            set boom = 1 / 0
        }

        set ^DemoErrTx(3) = "terceiro"

        tcommit
        write "  transacao confirmada", !
    }
    catch e {
        // one rollback, covering every failure path
        while $TLEVEL > entryLevel {
            trollback 1
        }

        set sc = e.AsStatus()
        write "  transacao desfeita: ", e.DisplayString(), !
    }

    write "  conteudo de ^DemoErrTx apos a operacao:", !
    if $DATA(^DemoErrTx) {
        zwrite ^DemoErrTx
    } else {
        write "    (vazio)", !
    }

    quit sc
}

/// Retry with growing wait.
ClassMethod WithRetry(failuresBeforeSuccess As %Integer = 2, maxAttempts As %Integer = 4) As %Status
{
    kill ^||DemoErrAttempt
    set ^||DemoErrAttempt = 0

    set wait = 0.2
    set sc = $$$ERROR($$$GeneralError, "nao executado")

    for attempt = 1:1:maxAttempts {
        try {
            do ..FlakyOperation(failuresBeforeSuccess)

            write "  tentativa ", attempt, ": sucesso", !
            set sc = $$$OK
            quit
        }
        catch e {
            if attempt = maxAttempts {
                write "  tentativa ", attempt, ": falhou (ultima)", !
                set sc = e.AsStatus()
                quit
            }

            write "  tentativa ", attempt, ": falhou, repetindo em ", wait, "s", !
            hang wait
            set wait = wait * 2
        }
    }

    quit sc
}

/// Fails a given number of times, then succeeds.
ClassMethod FlakyOperation(failuresBeforeSuccess As %Integer) As %Status [ Private ]
{
    set n = $INCREMENT(^||DemoErrAttempt)

    if n <= failuresBeforeSuccess {
        throw ##class(%Exception.General).%New(
            "TransientFailure", 5002, , "falha transitoria numero "_n)
    }

    quit $$$OK
}

/// Swallowing exceptions: the antipattern.
ClassMethod Swallow() As %Status
{
    write "-- versao que engole o erro --", !

    kill ^DemoErrSwallow

    try {
        set ^DemoErrSwallow(1) = "gravado"
        set x = 1 / 0
        set ^DemoErrSwallow(2) = "nunca gravado"
    }
    catch e {
        // nothing. the caller will never know.
    }

    write "  o metodo terminou sem reclamar", !
    write "  mas os dados estao incompletos:", !
    zwrite ^DemoErrSwallow

    write !, "-- versao correta --", !

    kill ^DemoErrSwallow
    set sc = ..DoNotSwallow()

    write "  status devolvido: ", $SELECT($$$ISOK(sc): "OK", 1: "ERRO"), !
    write "  mensagem        : ", $SYSTEM.Status.GetErrorText(sc), !

    quit $$$OK
}

ClassMethod DoNotSwallow() As %Status [ Private ]
{
    try {
        set ^DemoErrSwallow(1) = "gravado"
        set x = 1 / 0
        set ^DemoErrSwallow(2) = "nunca gravado"
    }
    catch e {
        do ##class(LabStudy.Log).Error("demo", "falha em DoNotSwallow", e.DisplayString())
        quit e.AsStatus()
    }
    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Basics()
    write !

    for kind = "general", "status", "macro", "system" {
        do ..Throwing(kind)
        write !
    }

    write "-- validacao acumulada --", !
    do ..ShowErrors(..ValidateAll("", "31/02/2026", "REG-1"))
    write !
    do ..ShowErrors(..ValidateAll("Maria Silva", "1990-05-17", "REG-000001"))

    write !, "-- transacao que da certo --", !
    do ..TransactionalWork(0)

    write !, "-- transacao que falha --", !
    do ..TransactionalWork(1)

    write !, "-- repeticao --", !
    do ..WithRetry(2, 4)

    write !
    do ..Swallow()

    quit $$$OK
}

}
```

### 6.1 Executando

```
LABSTUDY>DO ##class(LabStudy.Demo.Err).Basics()
-- erro do sistema --
  nome     : <DIVIDE>
  codigo   : 9
  local    : zBasics+3^LabStudy.Demo.Err.1
  texto    : <DIVIDE>zBasics+3^LabStudy.Demo.Err.1
  classe   : %Exception.SystemException
  sistema? : 1

-- o resto do TRY nao executa --
  marcador ficou em: [dentro, antes do erro]

-- sem erro, o CATCH nao executa --
  tudo certo
```

- **O `marcador` provou que o `try` foi interrompido** exatamente no ponto do erro. A linha seguinte não executou. Isso parece óbvio, mas é a origem de bugs quando o código dentro do `try` faz várias coisas e alguém supõe que todas aconteceram.
- **A classe da exceção é `%Exception.SystemException`**, confirmada pelo `%Extends`. Erros de execução do sistema sempre chegam assim.

```
LABSTUDY>DO ##class(LabStudy.Demo.Err).Throwing("general")
  tipo lancado : general
    nome       : RegistroInvalido
    codigo     : 5001
    dados      : numero de registro fora do padrao
    texto      : RegistroInvalido
    como status: RegistroInvalido

LABSTUDY>DO ##class(LabStudy.Demo.Err).Throwing("status")
  tipo lancado : status
    nome       : <ThrowStatus>
    codigo     : 5001
    dados      : ERROR #5001: erro criado como %Status
    texto      : ERROR #5001: erro criado como %Status
    como status: ERROR #5001: erro criado como %Status
```

- **A exceção própria carrega o nome que você deu.** Isso permite ao `catch` distinguir tipos de problema pelo nome, sem depender de comparar texto de mensagem.
- **A exceção criada a partir de `%Status` preserva a mensagem completa** e volta a ser `%Status` intacta com `AsStatus()`. **A conversão de ida e volta não perde informação** — é o que torna viável trocar de mundo entre camadas.

```
-- validacao acumulada --
  3 problema(s):
    1. ERROR #5001: nome e obrigatorio
    2. ERROR #5001: data de nascimento invalida: [31/02/2026]
    3. ERROR #5001: numero de registro invalido: [REG-1]

  tudo valido
```

- **Três problemas foram acumulados e apresentados juntos.** Compare com a alternativa: o usuário corrige o nome, reenvia, descobre o problema da data, corrige, reenvia, descobre o do registro. Três idas e voltas em vez de uma.
- **Cada mensagem inclui o valor recebido.** `[31/02/2026]` diz exatamente o que estava errado.

```
-- transacao que da certo --
  transacao confirmada
  conteudo de ^DemoErrTx apos a operacao:
^DemoErrTx(1)="primeiro"
^DemoErrTx(2)="segundo"
^DemoErrTx(3)="terceiro"

-- transacao que falha --
  transacao desfeita: <DIVIDE>zTransactionalWork+9^LabStudy.Demo.Err.1
  conteudo de ^DemoErrTx apos a operacao:
    (vazio)
```

- **Na falha, os dois primeiros nós também sumiram**, embora tivessem sido gravados com sucesso antes do erro. Essa é a transação funcionando: **tudo ou nada**.
- **Um único bloco de rollback cobriu o caso.** Não foi preciso prever onde o erro ocorreria.

```
-- repeticao --
  tentativa 1: falhou, repetindo em 0.2s
  tentativa 2: falhou, repetindo em 0.4s
  tentativa 3: sucesso
```

- **A espera dobrou a cada tentativa**: 0,2 e depois 0,4. Com dez processos em situação semelhante, eles se espalham no tempo em vez de bater todos juntos.
- **A terceira tentativa funcionou**, e o método devolveu sucesso. Do ponto de vista de quem chamou, **não houve falha alguma** — que é exatamente o objetivo.

```
-- versao que engole o erro --
  o metodo terminou sem reclamar
  mas os dados estao incompletos:
^DemoErrSwallow(1)="gravado"

-- versao correta --
  status devolvido: ERRO
  mensagem        : <DIVIDE>zDoNotSwallow+3^LabStudy.Demo.Err.1
```

**Este é o par mais importante do capítulo.**

- **A primeira versão terminou "com sucesso" e deixou os dados pela metade.** Nada avisou. Quem chamou seguiu adiante confiante. Num sistema real, isso vira um registro incompleto que só será descoberto meses depois — se for.
- **A segunda registrou o erro e devolveu `%Status` de erro**, preservando nome e localização. Quem chamou pode decidir o que fazer, e o log guarda o rastro.
- **A diferença entre as duas são duas linhas.** E é a diferença entre um sistema confiável e um que mente.

---

## 7. Pegadinhas e erros comuns

**1) `catch` vazio.**
Engolir a exceção é pior que não tratar. Sempre: resolver, registrar ou propagar.

**2) Substituir o erro original por um genérico.**
Perde `Name`, `Location` e `Data`. Preserve a exceção ou registre-a antes de traduzir.

**3) Achar que o resto do `try` executa após o erro.**
A execução salta direto para o `catch`.

**4) Esperar um bloco `finally`.**
Não existe no ObjectScript. Escreva a limpeza nos dois caminhos.

**5) Ler `$ZERROR` no `catch` em vez do objeto de exceção.**
`$ZERROR` pode ter sido sobrescrita. O objeto é confiável.

**6) Esquecer o rollback no `catch`.**
A transação fica aberta, segurando travas e journal indefinidamente.

**7) Usar `TROLLBACK` sem argumento dentro de um método aninhado.**
Derruba a transação do chamador. Use o padrão do `entryLevel` com `TROLLBACK 1`.

**8) Repetir indefinidamente.**
Sempre um número máximo de tentativas.

**9) Repetir sem espera, ou com espera fixa.**
Muitos processos voltam juntos e agravam a causa. Use espera crescente.

**10) Repetir erro permanente.**
`<UNDEFINED>` vai falhar igual nas dez tentativas. Distinga transitório de permanente.

**11) Registrar erro sem contexto.**
"Falha na operação" não permite investigar. Registre o dado que causou o problema.

**12) Registrar dado sensível no log.**
Senhas e dados pessoais não vão para o log, nem em mensagem de erro.

**13) Devolver o primeiro erro de validação em vez de todos.**
Use `AppendStatus` e entregue a lista completa.

**14) Tratar erro na camada errada.**
Utilitários propagam; serviços decidem; a fronteira captura tudo.

**15) Deixar um erro escapar até o usuário.**
`<UNDEFINED>zMetodo+7^Classe.1` numa tela é falha de desenho.

**16) Misturar `$ZTRAP` com `try`/`catch` no mesmo método.**
O fluxo fica imprevisível. Escolha um.

**17) Usar exceção para fluxo normal de negócio.**
"Paciente não encontrado" numa busca é resultado, não exceção.

---

## 8. MÃO NA MASSA

---

### Exercício 21.1 — Anatomia da exceção

**a) Enunciado:** Crie `LabStudy.Demo.Er1` que catalogue exceções:

1. `ClassMethod Provoke(tipo)` — provoca oito tipos diferentes de erro.
2. `ClassMethod Catalog()` — captura cada um e monta uma tabela com: tipo, `Name`, `Code`, classe da exceção, e se é `%Exception.SystemException`.
3. `ClassMethod Custom()` — lança três exceções próprias com nomes distintos e mostra como o `catch` pode decidir pelo `Name`.
4. `ClassMethod Rethrow()` — demonstra o `throw` sem argumento: um método interno registra e relança, e o externo decide.
5. `ClassMethod NoFinally()` — demonstra que não há `finally`, e mostra o padrão de limpeza nos dois caminhos.

**b) Dica:** No item 3, use `e.Name` num `$CASE` para decidir o tratamento.

**c) Como testar:** No item 4, o registro deve aparecer uma vez e a decisão, no nível de cima.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Er1.cls`:

```objectscript
/// Anatomy of exceptions.
Class LabStudy.Demo.Er1 Extends %RegisteredObject
{

/// Produces one specific kind of error.
ClassMethod Provoke(kind As %String) As %Status [ Private ]
{
    if kind = "divide" {
        set x = 1 / 0
    } elseif kind = "undefined" {
        write naoDefinida
    } elseif kind = "oref" {
        set o = ""
        write o.Campo
    } elseif kind = "list" {
        write $LIST($LISTBUILD("a"), 9)
    } elseif kind = "select" {
        write $SELECT(1 = 2: "x")
    } elseif kind = "date" {
        set d = $ZDATEH("nao e data", 3)
    } elseif kind = "status" {
        $$$ThrowOnError($$$ERROR($$$GeneralError, "erro de negocio"))
    } else {
        throw ##class(%Exception.General).%New("MeuErro", 9001, , "detalhe qualquer")
    }
    quit $$$OK
}

ClassMethod Catalog() As %Status
{
    set kinds = $LISTBUILD("divide", "undefined", "oref", "list",
                           "select", "date", "status", "custom")

    set W = $LISTBUILD(12, 20, 6, 28, 6)
    set A = $LISTBUILD("L", "L", "R", "L", "C")
    do ##class(LabStudy.Formatter).Header(
        $LISTBUILD("tipo", "Name", "Code", "classe", "sist?"), W, A)

    set ptr = 0
    while $LISTNEXT(kinds, ptr, kind) {
        set name = "", code = "", class = "", isSystem = ""

        try {
            do ..Provoke(kind)
            set name = "(nao falhou)"
        }
        catch e {
            set name = e.Name
            set code = e.Code
            set class = e.%ClassName(1)
            set isSystem = e.%Extends("%Exception.SystemException")
        }

        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(kind, name, code, class, isSystem), W, A)
    }

    do ##class(LabStudy.Formatter).Line(76)
    quit $$$OK
}

/// Custom exceptions, and deciding by name.
ClassMethod Custom() As %Status
{
    for kind = "RegistroInvalido", "DataInvalida", "SemPermissao", "Desconhecido" {
        write "-- lancando ", kind, " --", !

        try {
            throw ##class(%Exception.General).%New(kind, 9001, , "dados do erro "_kind)
        }
        catch e {
            // the catch decides based on the NAME, not on the message text
            set acao = $CASE(e.Name,
                "RegistroInvalido": "pedir o numero de registro novamente",
                "DataInvalida":     "pedir a data no formato AAAA-MM-DD",
                "SemPermissao":     "registrar tentativa e recusar",
                :                   "registrar e propagar")

            write "  acao decidida: ", acao, !
        }
    }
    quit $$$OK
}

/// Inner method logs and rethrows; outer method decides.
ClassMethod Rethrow() As %Status
{
    write "-- nivel externo chamando o interno --", !

    try {
        do ..InnerThatRethrows()
    }
    catch e {
        write "  nivel externo recebeu: ", e.Name, !
        write "  e decidiu: devolver %Status para quem chamou", !
        quit e.AsStatus()
    }

    quit $$$OK
}

ClassMethod InnerThatRethrows() As %Status [ Private ]
{
    try {
        throw ##class(%Exception.General).%New("FalhaInterna", 9002, , "algo deu errado")
    }
    catch e {
        // registers here, where the context is known
        write "  nivel interno registrou o erro: ", e.Data, !

        // but does not decide: sends it upwards, intact
        throw
    }

    quit $$$OK
}

/// There is no finally: clean up on both paths.
ClassMethod NoFinally() As %Status
{
    write "-- caminho com sucesso --", !
    do ..WithCleanup(0)

    write !, "-- caminho com erro --", !
    do ..WithCleanup(1)

    quit $$$OK
}

ClassMethod WithCleanup(shouldFail As %Boolean) As %Status [ Private ]
{
    // pretend we acquired a resource
    set ^||Er1Resource = "adquirido"
    write "  recurso adquirido", !

    set sc = $$$OK

    try {
        if shouldFail {
            set x = 1 / 0
        }
        write "  trabalho concluido", !
    }
    catch e {
        write "  erro: ", e.Name, !
        set sc = e.AsStatus()
    }

    // the cleanup is written ONCE, after the try/catch,
    // because there is no finally block
    kill ^||Er1Resource
    write "  recurso liberado", !

    quit sc
}

ClassMethod Demo() As %Status
{
    do ..Catalog()
    write !
    do ..Custom()
    write !
    do ..Rethrow()
    write !
    do ..NoFinally()
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Er1).Demo()
tipo         Name                  Code  classe                       sist?
----------------------------------------------------------------------------
divide       <DIVIDE>                 9  %Exception.SystemException     1
undefined    <UNDEFINED>              6  %Exception.SystemException     1
oref         <INVALID OREF>           8  %Exception.SystemException     1
list         <LIST>                  26  %Exception.SystemException     1
select       <ILLEGAL VALUE>         16  %Exception.SystemException     1
date         <ILLEGAL VALUE>         16  %Exception.SystemException     1
status       <ThrowStatus>         5001  %Exception.StatusException     0
custom       MeuErro               9001  %Exception.General             0
----------------------------------------------------------------------------

-- lancando RegistroInvalido --
  acao decidida: pedir o numero de registro novamente
-- lancando DataInvalida --
  acao decidida: pedir a data no formato AAAA-MM-DD
-- lancando SemPermissao --
  acao decidida: registrar tentativa e recusar
-- lancando Desconhecido --
  acao decidida: registrar e propagar

-- nivel externo chamando o interno --
  nivel interno registrou o erro: algo deu errado
  nivel externo recebeu: FalhaInterna
  e decidiu: devolver %Status para quem chamou

-- caminho com sucesso --
  recurso adquirido
  trabalho concluido
  recurso liberado

-- caminho com erro --
  recurso adquirido
  erro: <DIVIDE>
  recurso liberado
```

**Por que cada resultado:**

- **Os seis primeiros erros vieram como `%Exception.SystemException`**, e os dois últimos, não. Essa distinção permite ao `catch` separar "erro de programação ou de ambiente" de "erro de negócio" — tratamentos frequentemente diferentes.
- **`$ZDATEH` com texto inválido produziu `<ILLEGAL VALUE>`, o mesmo do `$SELECT` sem ramo verdadeiro.** Erros diferentes podem compartilhar nome. **Não decida tratamento apenas pelo nome de erros do sistema** — use o contexto.
- **O `Custom` decide pelo `Name`, não pelo texto da mensagem.** Comparar texto de mensagem é frágil: basta alguém corrigir uma grafia para o tratamento parar de funcionar. **Nomes de exceção são contratos; textos de mensagem são para humanos.**
- **O `Rethrow` mostra a divisão de responsabilidades.** O nível interno **sabe o contexto** e registra; o nível externo **tem visão do todo** e decide. O `throw` sem argumento preservou a exceção intacta — o nível externo recebeu `FalhaInterna`, não um erro genérico.
- **`WithCleanup` libera o recurso nos dois caminhos**, com a limpeza escrita **uma vez**, depois do `try`/`catch`. Esse é o padrão que substitui o `finally` inexistente: capture, guarde o `%Status`, limpe, devolva.
- **Se a limpeza estivesse dentro do `try`**, ela não rodaria em caso de erro — e o recurso ficaria preso. Se estivesse duplicada nos dois blocos, uma das cópias acabaria desatualizada.

---

### Exercício 21.2 — PROJETO CONTÍNUO: tratamento consistente

**a) Enunciado:** Leve o sistema à versão **2.2**, com tratamento de erros coerente em todas as camadas:

1. Crie `LabStudy.ErrorHandler`:
   - `Handle(area, exception, contexto)` — registra o erro com contexto completo e devolve `%Status`;
   - `HandleAndRethrow(area, exception, contexto)` — registra e relança;
   - `IsTransient(exception)` — decide se vale repetir;
   - `Retry(classe, metodo, maxTentativas, arg)` — executa com repetição e espera crescente;
   - `Guard(area, classe, metodo, arg)` — executa protegido, devolvendo `%Status` e nunca lançando;
   - `Summarize(status)` — devolve o texto de todos os erros de um `%Status` composto, numa linha.
2. Reescreva `LabStudy.Patient.CreateWithExams()` usando o padrão `try`/`catch` + `entryLevel`, substituindo os quatro caminhos de rollback por um só.
3. Reescreva `LabStudy.FileIO.ImportPatientsCsv()` para:
   - acumular erros por linha com `AppendStatus`;
   - continuar após uma linha ruim;
   - registrar cada falha com o número da linha e o conteúdo;
   - devolver um `%Status` composto no fim.
4. Acrescente `LabStudy.Patient.ValidateAll()` que acumula todos os problemas de uma vez.
5. Em `LabStudy.Menu`, envolva o despacho num `try`/`catch` para que **nenhum erro escape ao usuário**.
6. Suba `LabStudy.App` para `"2.2"`.

**b) Dica:** No item 5, a fronteira é o único lugar que captura tudo — inclusive erros de programação.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Patient).ValidateAll("", "31/02/2026", "REG-1")
LABSTUDY>SET sc = ##class(LabStudy.ErrorHandler).Guard("teste", "LabStudy.Demo.Er1", "Provoke", "divide")
LABSTUDY>DO ##class(LabStudy.App).Menu()
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/ErrorHandler.cls`:

```objectscript
/// Central error handling for the LabStudy system.
///
/// Convention adopted here:
///   - internal code throws;
///   - public methods return %Status;
///   - the boundary (menu, API, import) catches everything.
Class LabStudy.ErrorHandler Extends %RegisteredObject
{

/// Names that usually indicate a transient failure worth retrying.
Parameter TRANSIENTNAMES = "LOCK,TIMEOUT,BUSY,CONNECTION,UNAVAILABLE";

/// Records an exception with full context and returns it as %Status.
ClassMethod Handle(area As %String, exception As %Exception.AbstractException, context As %String = "") As %Status
{
    set detail = exception.DisplayString()
    set:exception.Location'="" detail = detail_" | local="_exception.Location
    set:exception.Data'="" detail = detail_" | dados="_exception.Data
    set:context'="" detail = detail_" | "_context
    set detail = detail_" | "_##class(LabStudy.Diagnostics).Context()

    do ##class(LabStudy.Log).Error(area, exception.Name, detail)

    quit exception.AsStatus()
}

/// Records and rethrows, so the caller decides.
ClassMethod HandleAndRethrow(area As %String, exception As %Exception.AbstractException, context As %String = "") As %Status
{
    do ..Handle(area, exception, context)
    throw exception
}

/// Decides whether an exception is worth retrying.
/// This list is a project decision, not a universal truth.
ClassMethod IsTransient(exception As %Exception.AbstractException) As %Boolean
{
    set text = $ZCONVERT(exception.Name_" "_exception.DisplayString(), "U")

    for i = 1:1:$LENGTH(..#TRANSIENTNAMES, ",") {
        return:text[$PIECE(..#TRANSIENTNAMES, ",", i) 1
    }

    quit 0
}

/// Runs a class method with retry and growing wait.
ClassMethod Retry(class As %String, method As %String, maxAttempts As %Integer = 3, arg As %String = "") As %Status
{
    set wait = 0.5
    set sc = $$$ERROR($$$GeneralError, "nao executado")

    for attempt = 1:1:maxAttempts {
        try {
            if arg = "" {
                do $CLASSMETHOD(class, method)
            } else {
                do $CLASSMETHOD(class, method, arg)
            }

            do:attempt>1 ##class(LabStudy.Log).Info("retry",
                "sucesso na tentativa "_attempt_" de "_class_"."_method)

            quit $$$OK
        }
        catch e {
            if '..IsTransient(e) {
                do ..Handle("retry", e, "erro permanente em "_class_"."_method)
                quit e.AsStatus()
            }

            if attempt = maxAttempts {
                do ..Handle("retry", e,
                    "falhou apos "_maxAttempts_" tentativas em "_class_"."_method)
                quit e.AsStatus()
            }

            do ##class(LabStudy.Log).Warn("retry",
                "tentativa "_attempt_" falhou em "_class_"."_method,
                e.DisplayString()_" | proxima em "_wait_"s")

            hang wait
            set wait = wait * 2
        }
    }

    quit sc
}

/// Runs a class method protected. Never throws; always returns %Status.
/// This is what a boundary uses.
ClassMethod Guard(area As %String, class As %String, method As %String, arg As %String = "") As %Status
{
    try {
        if arg = "" {
            do $CLASSMETHOD(class, method)
        } else {
            do $CLASSMETHOD(class, method, arg)
        }
    }
    catch e {
        quit ..Handle(area, e, "chamada protegida: "_class_"."_method)
    }

    quit $$$OK
}

/// All messages of a composite %Status, in one line.
ClassMethod Summarize(sc As %Status, separator As %String = " / ") As %String
{
    quit:$$$ISOK(sc) ""

    do $SYSTEM.Status.DecomposeStatus(sc, .errors)

    set out = "", total = $GET(errors, 0)

    for i = 1:1:total {
        set out = out_$SELECT(i > 1: separator, 1: "")_errors(i)
    }

    quit out
}

/// Prints a composite %Status as a numbered list.
ClassMethod Print(sc As %Status) As %Status
{
    if $$$ISOK(sc) {
        write "  (sem erros)", !
        quit $$$OK
    }

    do $SYSTEM.Status.DecomposeStatus(sc, .errors)

    set total = $GET(errors, 0)
    write "  ", total, " problema(s):", !

    for i = 1:1:total {
        write "    ", i, ". ", errors(i), !
    }
    quit $$$OK
}

}
```

Acrescente a `src/LabStudy/Patient.cls`:

```objectscript
/// Validates every field at once, returning ALL problems.
ClassMethod ValidateAll(name As %String, birthDate As %String, recordNumber As %String, sex As %String = "") As %Status
{
    set sc = $$$OK

    if ##class(LabStudy.Text).Clean(name) = "" {
        set sc = $SYSTEM.Status.AppendStatus(sc,
            $$$ERROR($$$GeneralError, "nome e obrigatorio"))
    }

    if birthDate = "" {
        set sc = $SYSTEM.Status.AppendStatus(sc,
            $$$ERROR($$$GeneralError, "data de nascimento e obrigatoria"))
    } elseif '##class(LabStudy.Text).IsIsoDate(birthDate) {
        set sc = $SYSTEM.Status.AppendStatus(sc,
            $$$ERROR($$$GeneralError, "data invalida (use AAAA-MM-DD): ["_birthDate_"]"))
    } elseif ##class(LabStudy.DateTime).Parse(birthDate, 3) > ##class(LabStudy.DateTime).Today() {
        set sc = $SYSTEM.Status.AppendStatus(sc,
            $$$ERROR($$$GeneralError, "data de nascimento no futuro: ["_birthDate_"]"))
    }

    if recordNumber '= "", '##class(LabStudy.Text).IsRecordNumber(recordNumber) {
        set sc = $SYSTEM.Status.AppendStatus(sc,
            $$$ERROR($$$GeneralError, "numero de registro fora do padrao REG-000000: ["_recordNumber_"]"))
    }

    if sex '= "", (sex '= "M") && (sex '= "F") {
        set sc = $SYSTEM.Status.AppendStatus(sc,
            $$$ERROR($$$GeneralError, "sexo deve ser M ou F: ["_sex_"]"))
    }

    quit sc
}

/// Creates a patient with its exams, atomically.
/// One try/catch, one rollback, every failure path covered.
ClassMethod CreateWithExams(name As %String, birthDate As %String, sex As %String, examList As %DynamicArray) As %String
{
    set entryLevel = $TLEVEL
    set patientId = ""

    try {
        // validate everything BEFORE touching the database
        $$$ThrowOnError(..ValidateAll(name, birthDate, "", sex))

        tstart

        set patient = ..%New()
        set patient.Name = name
        set patient.BirthDate = ##class(LabStudy.DateTime).Parse(birthDate, 3)
        set patient.Sex = sex
        set patient.RecordNumber = ##class(LabStudy.Sequence).NewRecordNumber()

        $$$ThrowOnError(patient.%Save())

        if $ISOBJECT(examList) {
            set iterator = examList.%GetIterator()

            while iterator.%GetNext(.index, .item) {
                if 'item.%IsDefined("testCode") {
                    throw ##class(%Exception.General).%New(
                        "ExameSemCodigo", 5010, ,
                        "o exame na posicao "_index_" nao tem testCode")
                }

                set exam = ##class(LabStudy.Exam).%New(item.testCode)
                set exam.Unit = item.%Get("unit", "")
                set exam.Patient = patient

                if item.%IsDefined("value") {
                    $$$ThrowOnError(exam.SetResult(item.value, exam.Unit))
                }

                $$$ThrowOnError(exam.%Save())
            }
        }

        tcommit

        set patientId = patient.%Id()

        do ##class(LabStudy.Log).Info("patient",
            "paciente criado: "_patient.RecordNumber,
            "id="_patientId_" exames="_$SELECT($ISOBJECT(examList): examList.%Size(), 1: 0))
    }
    catch e {
        // ONE rollback, covering every failure path
        while $TLEVEL > entryLevel {
            trollback 1
        }

        do ##class(LabStudy.ErrorHandler).Handle("patient", e,
            "CreateWithExams nome=["_name_"] nascimento=["_birthDate_"]")

        write "  cadastro cancelado: ", e.DisplayString(), !
        set patientId = ""
    }

    quit patientId
}
```

Reescreva a importação em `src/LabStudy/FileIO.cls`:

```objectscript
/// Imports patients from CSV, accumulating problems per line.
/// SIMULATION IS THE DEFAULT. Returns a composite %Status.
ClassMethod ImportPatientsCsv(path As %String, reallyImport As %Boolean = 0, Output created As %Integer) As %Status
{
    set created = 0
    set result = $$$OK

    do ##class(LabStudy.Log).Info("import", "importacao iniciada",
        "arquivo="_path_" simulacao="_'reallyImport)

    try {
        if '##class(%File).Exists(path) {
            throw ##class(%Exception.General).%New(
                "ArquivoNaoEncontrado", 5020, , path)
        }

        set file = ##class(%Stream.FileCharacter).%New()
        $$$ThrowOnError(file.LinkToFile(path))

        do file.Rewind()

        set header = $ZSTRIP(file.ReadLine(), ">", $CHAR(13))

        if $ZCONVERT(header, "L") '= $ZCONVERT(..#PATIENTHEADER, "L") {
            throw ##class(%Exception.General).%New(
                "CabecalhoInvalido", 5021, ,
                "esperado ["_..#PATIENTHEADER_"] recebido ["_header_"]")
        }

        write "  ", $SELECT(reallyImport: "IMPORTANDO", 1: "simulacao"), " de ", path, !

        set line = 1, skipped = 0, failed = 0

        while 'file.AtEnd {
            set line = line + 1
            set raw = $ZSTRIP(file.ReadLine(), ">", $CHAR(13))
            continue:$ZSTRIP(raw, "<>W")=""

            // each line is protected on its own: one bad line
            // must not abort the whole file
            try {
                set row = ##class(LabStudy.ListUtil).FromCsv(raw)

                set rec   = $ZCONVERT(##class(LabStudy.Text).Clean($LISTGET(row, 1)), "U")
                set name  = ##class(LabStudy.Text).Clean($LISTGET(row, 2))
                set birth = ##class(LabStudy.Text).Clean($LISTGET(row, 3))
                set sex   = $ZCONVERT(##class(LabStudy.Text).Clean($LISTGET(row, 4)), "U")
                set city  = ##class(LabStudy.Text).Clean($LISTGET(row, 5))
                set email = ##class(LabStudy.Text).Clean($LISTGET(row, 6))

                // all problems of this line at once
                set lineSc = ##class(LabStudy.Patient).ValidateAll(name, birth, rec, sex)

                if $$$ISERR(lineSc) {
                    set failed = failed + 1
                    write "    linha ", line, ": ",
                          ##class(LabStudy.ErrorHandler).Summarize(lineSc), !

                    set result = $SYSTEM.Status.AppendStatus(result,
                        $$$ERROR($$$GeneralError, "linha "_line_": "_
                            ##class(LabStudy.ErrorHandler).Summarize(lineSc)))
                    continue
                }

                if ##class(LabStudy.Patient).RecordIdxExists(rec) {
                    set skipped = skipped + 1
                    continue
                }

                if 'reallyImport {
                    write "    linha ", line, ": criaria ", rec, " - ", name, !
                    set created = created + 1
                    continue
                }

                set p = ##class(LabStudy.Patient).%New()
                set p.RecordNumber = rec
                set p.Name = name
                set p.BirthDate = ##class(LabStudy.DateTime).Parse(birth, 3)
                set p.Sex = sex
                set p.Email = email
                set:city'="" p.Address.City = city

                $$$ThrowOnError(p.%Save())
                set created = created + 1
            }
            catch lineError {
                set failed = failed + 1

                do ##class(LabStudy.Log).Warn("import",
                    "linha "_line_" falhou",
                    lineError.DisplayString()_" | conteudo=["_raw_"]")

                write "    linha ", line, ": ", lineError.DisplayString(), !

                set result = $SYSTEM.Status.AppendStatus(result,
                    $$$ERROR($$$GeneralError,
                        "linha "_line_": "_lineError.DisplayString()))
            }
        }

        write "  ------------------------------", !
        write "  linhas lidas : ", line - 1, !
        write "  criados      : ", created, !
        write "  ja existiam  : ", skipped, !
        write "  falhas       : ", failed, !

        do ##class(LabStudy.Log).Info("import", "importacao concluida",
            "criados="_created_" existentes="_skipped_" falhas="_failed)
    }
    catch e {
        // fatal errors: file missing, bad header, unreadable stream
        do ##class(LabStudy.ErrorHandler).Handle("import", e, "arquivo="_path)
        write "  importacao abortada: ", e.DisplayString(), !
        quit e.AsStatus()
    }

    quit result
}
```

Proteja a fronteira em `src/LabStudy/Menu.cls`:

```objectscript
/// Main loop. NOTHING escapes to the user unhandled.
/// Option -> method name. Pure data: consulting the table executes nothing.
ClassMethod Dispatch(option As %String) As %String
{
    quit $CASE(option,
        "1":  "OptListPatients",
        "2":  "OptFindByRecord",
        "3":  "OptPending",
        "4":  "OptDashboard",
        "5":  "OptRankings",
        "6":  "OptBenchmark",
        "7":  "OptAgeDistribution",
        "8":  "OptClassify",
        "9":  "OptByPriority",
        "10": "OptBackgroundReport",
        "11": "OptExport",
        "12": "OptImport",
        "13": "OptSystemInfo",
        "14": "OptSelfCheck",
        "15": "OptLog",
        "0":  "quit",
        :     "")
}

/// Main loop. NOTHING escapes to the user unhandled.
ClassMethod Run() As %Status
{
    for {
        do ..Draw()

        set option = ..Prompt("Opcao", "0")
        set method = ..Dispatch(option)

        if method = "" {
            write "  Opcao invalida: [", option, "]", !
            do ..Pause()
            continue
        }

        if method = "quit" {
            if ..Confirm("  Deseja realmente sair?") {
                write "  Ate logo.", !
                quit
            }
            continue
        }

        // the boundary catches everything, including programming errors
        try {
            do $CLASSMETHOD($CLASSNAME(), method)
        }
        catch e {
            do ##class(LabStudy.ErrorHandler).Handle("menu", e,
                "opcao=["_option_"] metodo=["_method_"]")

            write !
            write "  Ocorreu um erro inesperado nesta operacao.", !
            write "  O problema foi registrado e pode ser consultado no log.", !
            write "  Detalhe tecnico: ", e.Name, !
            write !
        }

        do ..Pause()
    }

    quit $$$OK
}

ClassMethod OptSelfCheck() As %Status
{
    do ##class(LabStudy.Diagnostics).SelfCheck()
    quit $$$OK
}

ClassMethod OptLog() As %Status
{
    set n = ..Prompt("Quantas entradas", "20")
    set level = $ZCONVERT(..Prompt("Nivel minimo", "INFO"), "U")

    do ##class(LabStudy.Log).Show(n, level)
    quit $$$OK
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "2.2";
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.ErrorHandler).Print(##class(LabStudy.Patient).ValidateAll("", "31/02/2026", "REG-1", "X"))
  4 problema(s):
    1. ERROR #5001: nome e obrigatorio
    2. ERROR #5001: data invalida (use AAAA-MM-DD): [31/02/2026]
    3. ERROR #5001: numero de registro fora do padrao REG-000000: [REG-1]
    4. ERROR #5001: sexo deve ser M ou F: [X]

LABSTUDY>SET id = ##class(LabStudy.Patient).CreateWithExams("", "1990-05-17", "F", [])
  cadastro cancelado: ERROR #5001: nome e obrigatorio

LABSTUDY>WRITE "[", id, "]", !
[]

LABSTUDY>SET id = ##class(LabStudy.Patient).CreateWithExams("Ana Teste","1990-05-17","F",[{"testCode":"GLU","value":92,"unit":"mg/dL"},{"value":10}])
  cadastro cancelado: ExameSemCodigo

LABSTUDY>WRITE ##class(LabStudy.Patient).%ExistsId(id), !
0

LABSTUDY>SET sc = ##class(LabStudy.ErrorHandler).Guard("teste", "LabStudy.Demo.Er1", "Provoke", "divide")
LABSTUDY>WRITE $$$ISERR(sc), " -> ", $SYSTEM.Status.GetErrorText(sc), !
1 -> <DIVIDE>zProvoke+2^LabStudy.Demo.Er1.1

LABSTUDY>DO ##class(LabStudy.Log).Show(5, "WARN")
quando               nivel  area         mensagem
------------------------------------------------------------------------------------
2026-08-19 18:04:22  ERROR  teste        <DIVIDE>
2026-08-19 18:03:55  ERROR  patient      ExameSemCodigo
2026-08-19 18:03:41  ERROR  patient      ERROR #5001: nome e obrigatorio
------------------------------------------------------------------------------------
  3 entradas (nivel atual: DEBUG)
```

**Por que cada decisão:**

- **`ValidateAll` devolveu os quatro problemas de uma vez.** O usuário corrige tudo numa passada. Compare com validar campo a campo e parar no primeiro: quatro tentativas.
- **A validação acontece ANTES do `TSTART`.** Não faz sentido abrir uma transação para descobrir, na primeira gravação, que o nome está vazio. Validar cedo economiza travas, journal e rollback.
- **`CreateWithExams` foi de quatro blocos de rollback para um.** Compare com a versão do Capítulo 5: eram quatro `while $TLEVEL > entryLevel { trollback 1 }` idênticos, um por caminho de saída. Agora há um, e ele cobre **inclusive** os erros que ninguém previu.
- **O segundo teste provou a atomicidade:** o paciente e o primeiro exame chegaram a ser gravados, e o segundo exame sem código derrubou tudo. `%ExistsId` devolveu `0`. **Nada pela metade.**
- **`$$$ThrowOnError` eliminou os `if $$$ISERR(sc)` repetidos.** Cada gravação passou de quatro linhas para uma, sem perder o tratamento — porque o `catch` cobre todas.
- **Na importação, há DOIS `try` aninhados**, e isso é deliberado: o externo trata erros **fatais** (arquivo ausente, cabeçalho errado) que impedem qualquer processamento; o interno trata erros **de linha**, que não devem abortar o arquivo. **A granularidade do `try` é uma decisão de projeto**, e escolhê-la errado produz ou um arquivo abortado por uma linha ruim, ou um erro fatal ignorado.
- **A importação devolve um `%Status` composto** com uma entrada por linha problemática. Quem chamou pode exibir a lista completa com `Print()`.
- **O menu captura tudo e mostra uma mensagem digna ao usuário**, com o detalhe técnico reduzido ao nome do erro. O rastro completo — pilha, dados, contexto — foi para o log. Isso é o Capítulo 7 aplicado a mensagens de erro: **discrição para o usuário, detalhe completo no registro**.
- **A tabela de despacho devolve o nome do método, e o `try` envolve apenas a chamada.** Além do motivo dado no Capítulo 17 (uma tabela de dados puros não depende do curto-circuito do `$CASE`), aqui há um ganho específico de tratamento de erros: **a opção inválida é resolvida antes do `try`**, então o bloco protegido contém exatamente uma coisa — a execução da opção escolhida. Um `try` que envolve validação, despacho e execução ao mesmo tempo dificulta distinguir "o usuário digitou errado" de "a operação falhou".
- **O contexto registrado inclui a opção e o método.** `opcao=[12] metodo=[OptImport]` diz, numa linha de log, exatamente o que o usuário estava tentando fazer.
- **O menu continua funcionando após o erro.** Um erro numa opção não encerra a sessão — o `catch` está **dentro** do laço.
- **`Guard` transforma qualquer método numa chamada segura**, devolvendo `%Status` e registrando. É a ferramenta para proteger fronteiras sem escrever `try`/`catch` em cinquenta lugares.
- **`IsTransient` usa `RETURN` dentro do laço**, e não `QUIT` seguido de inspeção da variável de controle. A segunda forma funciona, mas depende de saber se o laço terminou por parada ou por esgotamento — uma distinção que o Capítulo 17 recomenda expressamente não usar. Aqui a resposta sai no ponto em que a decisão é tomada, e o `quit 0` final cobre o caso de nenhuma correspondência.
- **A lista de nomes transitórios é um `Parameter`, não uma cadeia de `if`.** Ajustar o que vale a pena repetir num ambiente específico é editar uma linha. E vale insistir: **essa lista é uma decisão a ser combinada com quem conhece o ambiente**, não uma verdade do produto.

---

## 9. Quiz do capítulo

**Q1.** Qual é o pior tratamento de erro possível?

- A) Registrar e relançar.
- B) Um `catch` vazio, que engole a exceção.
- C) Devolver `%Status` de erro.
- D) Não usar `try`/`catch`.

---

**Q2.** O que acontece com o restante do bloco `try` depois de um erro?

- A) Continua executando.
- B) Não executa: a execução salta para o `catch`.
- C) Executa apenas as linhas sem risco.
- D) Depende da configuração.

---

**Q3.** Existe bloco `finally` no ObjectScript?

- A) Sim.
- B) Não: escreva a limpeza depois do `try`/`catch`, uma vez só.
- C) Sim, mas só em métodos de classe.
- D) Sim, chamado `ensure`.

---

**Q4.** Qual propriedade da exceção traz o método e o deslocamento onde o erro ocorreu?

- A) `Name`
- B) `Code`
- C) `Location`
- D) `Data`

---

**Q5.** O que faz `THROW` sem argumento dentro de um `catch`?

- A) Gera um erro genérico.
- B) Relança a exceção que está sendo tratada, preservando-a.
- C) Encerra o método.
- D) É erro de sintaxe.

---

**Q6.** O que faz `$$$ThrowOnError(sc)`?

- A) Sempre lança uma exceção.
- B) Lança uma exceção se o `%Status` for de erro.
- C) Converte exceção em `%Status`.
- D) Registra o erro no log.

---

**Q7.** Como converter uma exceção em `%Status`?

- A) `e.AsStatus()`
- B) `$$$ERROR(e)`
- C) `$SYSTEM.Status.GetErrorText(e)`
- D) Não é possível.

---

**Q8.** Como combinar vários erros de validação num único `%Status`?

- A) Concatenando os textos.
- B) Com `$SYSTEM.Status.AppendStatus(sc1, sc2)`.
- C) Devolvendo apenas o primeiro.
- D) Com `$LISTBUILD`.

---

**Q9.** Como separar um `%Status` composto em mensagens individuais?

- A) `$PIECE`
- B) `$SYSTEM.Status.DecomposeStatus(sc, .array)`
- C) `$LISTFROMSTRING`
- D) `$SYSTEM.Status.GetErrorText`

---

**Q10.** Onde um método utilitário sem contexto de negócio deve tratar um erro?

- A) Ali mesmo, com um valor padrão.
- B) Deve **propagar**: ele não tem informação para decidir.
- C) Registrando e ignorando.
- D) Encerrando o processo.

---

**Q11.** No padrão de transação com `try`/`catch`, onde fica o rollback?

- A) Em cada caminho de saída do `try`.
- B) No `catch`, uma vez só, usando o `entryLevel`.
- C) Depois do `tcommit`.
- D) Não é necessário.

---

**Q12.** Por que usar espera **crescente** entre tentativas?

- A) Para o código ficar mais lento.
- B) Para que muitos processos que falharam juntos não retornem todos no mesmo instante.
- C) Porque o IRIS exige.
- D) Para economizar memória.

---

**Q13.** Vale a pena repetir um `<UNDEFINED>`?

- A) Sim, sempre.
- B) Não: é erro permanente, vai falhar igual em todas as tentativas.
- C) Sim, com espera longa.
- D) Depende do namespace.

---

**Q14.** Ao decidir o tratamento no `catch`, é melhor comparar o quê?

- A) O texto da mensagem.
- B) O `Name` da exceção, que é um contrato estável.
- C) O `Code` numérico apenas.
- D) O comprimento da mensagem.

---

**Q15.** O que deve chegar ao usuário quando um erro inesperado ocorre?

- A) O texto completo do erro, com pilha e localização.
- B) Uma mensagem digna, com o detalhe completo registrado no log.
- C) Nada; a tela deve travar.
- D) O código-fonte da linha que falhou.

---

**Q16.** Qual é a forma legada de tratamento de erros?

- A) `try`/`catch`
- B) `$ZTRAP` com um rótulo
- C) `%Status`
- D) `$ECODE`

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** engolir a exceção é pior que não tratar, porque o sistema continua como se tudo estivesse bem e os dados ficam inconsistentes sem aviso.
- **A e C estão erradas:** são tratamentos legítimos.
- **D está errada:** sem tratamento, ao menos o erro aparece e alguém o corrige.

**Q2 — Resposta: B.**
- **B está certa:** a execução salta do ponto do erro direto para o `catch`.
- **A, C e D estão erradas:** nenhuma linha após o erro é executada dentro do `try`.

**Q3 — Resposta: B.**
- **B está certa:** não há `finally`. A limpeza fica depois do `try`/`catch`, escrita uma vez.
- **A, C e D estão erradas:** a construção não existe na linguagem.

**Q4 — Resposta: C.**
- **C está certa:** `Location` traz `Metodo+N^Classe`, o que permite achar a linha exata.
- **A está errada:** `Name` é o nome do erro.
- **B está errada:** `Code` é o código numérico.
- **D está errada:** `Data` traz informação adicional.

**Q5 — Resposta: B.**
- **B está certa:** relança a exceção atual, intacta, para que um nível superior decida.
- **A está errada:** não cria erro novo — e isso é a vantagem, porque preserva a informação.
- **C está errada:** encerrar seria `quit` ou `return`.
- **D está errada:** é sintaxe válida e idiomática.

**Q6 — Resposta: B.**
- **B está certa:** avalia o `%Status` e lança apenas se for de erro.
- **A está errada:** com status de sucesso, nada acontece.
- **C está errada:** isso é `AsStatus()`.
- **D está errada:** ela não registra nada.

**Q7 — Resposta: A.**
- **A está certa:** `AsStatus()` converte a exceção em `%Status`, preservando a informação.
- **B, C e D estão erradas:** não fazem essa conversão.

**Q8 — Resposta: B.**
- **B está certa:** `AppendStatus` combina status, e o resultado é de erro se qualquer parte for.
- **A está errada:** perde a estrutura e impede a decomposição posterior.
- **C está errada:** obriga o usuário a corrigir um problema por vez.
- **D está errada:** listas não são `%Status`.

**Q9 — Resposta: B.**
- **B está certa:** `DecomposeStatus` preenche um array com cada mensagem.
- **A e C estão erradas:** operam sobre texto, não sobre a estrutura do status.
- **D está errada:** devolve um texto único.

**Q10 — Resposta: B.**
- **B está certa:** quem não tem contexto para decidir deve propagar para quem tem.
- **A está errada:** escolher um padrão sem contexto esconde o problema.
- **C está errada:** ignorar é engolir.
- **D está errada:** desproporcional.

**Q11 — Resposta: B.**
- **B está certa:** um `catch`, um rollback, guardando `entryLevel` para não derrubar a transação do chamador.
- **A está errada:** era o padrão anterior, com repetição desnecessária e risco de esquecer um caminho.
- **C está errada:** depois do commit não há o que desfazer.
- **D está errada:** sem rollback, a transação fica aberta.

**Q12 — Resposta: B.**
- **B está certa:** espera crescente espalha as tentativas no tempo e evita agravar a causa da falha.
- **A, C e D estão erradas:** não descrevem a motivação.

**Q13 — Resposta: B.**
- **B está certa:** um erro de programação é determinístico; repetir só desperdiça tempo.
- **A e C estão erradas:** o resultado será idêntico.
- **D está errada:** não depende do namespace.

**Q14 — Resposta: B.**
- **B está certa:** o `Name` é estável; o texto da mensagem pode ser reescrito e quebrar o tratamento.
- **A está errada:** comparar texto de mensagem é frágil.
- **C está errada:** o código pode ser reaproveitado entre erros diferentes.
- **D está errada:** sem sentido.

**Q15 — Resposta: B.**
- **B está certa:** mensagem digna para o usuário, detalhe completo no log — o mesmo princípio do Capítulo 7.
- **A está errada:** expõe estrutura interna e não ajuda quem usa o sistema.
- **C está errada:** travar é falha de desenho.
- **D está errada:** inaceitável.

**Q16 — Resposta: B.**
- **B está certa:** `$ZTRAP` recebe o nome de um rótulo para onde a execução salta em caso de erro.
- **A e C estão erradas:** são as formas atuais.
- **D está errada:** `$ECODE` é uma variável de leitura.

---

## 10. Resumo relâmpago

1. Dois mundos: **`%Status`** (erro como valor devolvido) e **exceção** (erro como interrupção). Converta na **fronteira das camadas**.
2. **`%Status`** para erros esperados de negócio; **exceção** para condições excepcionais e erros de programação.
3. **`catch` vazio é o pior tratamento possível.** Sempre: resolver, registrar ou propagar.
4. Após um erro, **o restante do `try` não executa**.
5. **Não existe `finally`.** Escreva a limpeza depois do `try`/`catch`, uma vez só.
6. Objeto de exceção: **`Name`**, **`Code`**, **`Location`**, **`Data`**, **`DisplayString()`**, **`AsStatus()`**.
7. **`Location`** é a informação mais valiosa na investigação.
8. Classes: `%Exception.SystemException` (erros do sistema), `%Exception.StatusException` (envolve `%Status`), `%Exception.General` (próprias).
9. **`THROW` sem argumento relança a exceção atual**, preservando-a.
10. **`$$$ThrowOnError(sc)`** converte `%Status` de erro em exceção; **`AsStatus()`** faz o inverso.
11. **`AppendStatus`** combina erros; **`DecomposeStatus`** os separa. Entregue **todos** os problemas de validação de uma vez.
12. **Trate no nível que tem informação para decidir.** Utilitários propagam; serviços decidem; a fronteira captura tudo.
13. **Nenhum erro deve escapar da fronteira.** Mensagem digna ao usuário, detalhe completo no log.
14. Padrão de transação: `entryLevel` antes do `tstart`, **um** rollback no `catch`, `AsStatus()` para propagar.
15. **Repetição**: número máximo de tentativas, **espera crescente**, e **só para erros transitórios**.
16. **Decida pelo `Name`, não pelo texto da mensagem.** Nomes são contratos; textos são para humanos.
17. **Preserve a exceção original.** Substituí-la por um erro genérico destrói a informação de diagnóstico.
18. **Registre com contexto**: o que, onde, com quais dados, em que contexto — **sem dados sensíveis**.
19. **Valide antes de abrir a transação.**
20. **`$ZTRAP`** é o tratamento legado. Reconheça; não escreva. Nunca misture com `try`/`catch` no mesmo método.

---

## 11. Cartões de memorização

**Frente:** Quais são os dois mundos de sinalização de erro no ObjectScript?
**Verso:** `%Status` (erro devolvido como valor) e exceção (erro que interrompe). Converta na fronteira das camadas.

**Frente:** Qual é o pior tratamento de erro?
**Verso:** `catch` vazio. O sistema continua como se estivesse tudo bem, com dados inconsistentes.

**Frente:** O que um `catch` legítimo faz?
**Verso:** Pelo menos uma de três coisas: resolve, registra, ou propaga.

**Frente:** O restante do `try` executa após um erro?
**Verso:** Não. A execução salta direto para o `catch`.

**Frente:** Existe `finally` no ObjectScript?
**Verso:** Não. Escreva a limpeza depois do `try`/`catch`, uma vez só.

**Frente:** Quais são os membros do objeto de exceção?
**Verso:** `Name`, `Code`, `Location`, `Data`, `DisplayString()` e `AsStatus()`.

**Frente:** Qual propriedade da exceção mais ajuda na investigação?
**Verso:** `Location` — traz `Metodo+N^Classe`, que localiza a linha exata.

**Frente:** O que faz `THROW` sem argumento?
**Verso:** Relança a exceção atual, intacta, para que um nível superior decida.

**Frente:** O que faz `$$$ThrowOnError(sc)`?
**Verso:** Lança uma exceção se o `%Status` for de erro. É a ponte de `%Status` para exceção.

**Frente:** Como converter exceção em `%Status`?
**Verso:** `e.AsStatus()`.

**Frente:** Como acumular vários erros de validação?
**Verso:** `$SYSTEM.Status.AppendStatus(sc1, sc2)`, e depois `DecomposeStatus` para listar.

**Frente:** Onde tratar um erro?
**Verso:** No nível que tem informação para decidir. Utilitários propagam; serviços decidem; a fronteira captura tudo.

**Frente:** O que o usuário deve ver num erro inesperado?
**Verso:** Uma mensagem digna. O detalhe técnico — pilha, dados, contexto — vai para o log.

**Frente:** Qual o padrão de transação com `try`/`catch`?
**Verso:** `entryLevel = $TLEVEL` antes do `tstart`; **um** rollback no `catch` com `while $TLEVEL > entryLevel { trollback 1 }`.

**Frente:** Quais são as três regras da repetição?
**Verso:** Número máximo de tentativas, espera crescente, e só para erros transitórios.

**Frente:** Vale repetir um `<UNDEFINED>`?
**Verso:** Não. É erro permanente: falhará igual em todas as tentativas.

**Frente:** No `catch`, decidir pelo texto da mensagem ou pelo `Name`?
**Verso:** Pelo `Name`. Nomes são contratos estáveis; textos de mensagem podem ser reescritos.

**Frente:** Por que não substituir a exceção por um erro genérico?
**Verso:** Perde `Name`, `Location` e `Data` — exatamente o que permitiria investigar depois.

**Frente:** Validar antes ou depois do `TSTART`?
**Verso:** Antes. Não faz sentido abrir transação para descobrir que um campo obrigatório está vazio.

**Frente:** Qual o tratamento de erros legado?
**Verso:** `$ZTRAP` com o nome de um rótulo. Reconheça em código antigo; não escreva, e nunca misture com `try`/`catch`.

---

Digite CONTINUAR para o próximo capítulo.
