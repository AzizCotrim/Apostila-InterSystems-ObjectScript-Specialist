# Apostila InterSystems ObjectScript Specialist
## Capítulo 17 — T4.5 Decision and Control Structures (Decisão e controle de fluxo)

> Ainda em **T4 — Functions & APIs**. Você já usou `if`, `for` e `quit` em todos os capítulos anteriores. Agora vamos fechar as lacunas: as formas que ainda não apareceram, as armadilhas de cada uma, o código legado que você vai encontrar, e a **indireção** — prometida desde o Capítulo 8.

---

## 1. O que você vai saber fazer ao terminar

1. Usar **`IF` / `ELSEIF` / `ELSE`** na forma de bloco, e reconhecer a forma antiga baseada em **`$TEST`**.
2. Escrever **pós-condicionais** (`comando:condição`) e saber onde eles **não** funcionam.
3. Escolher entre **`IF`**, **`$SELECT`** e **`$CASE`** com critério.
4. Dominar todas as formas do **`FOR`**: contado, infinito, por lista de valores e com faixas múltiplas.
5. Usar **`WHILE`** e **`DO ... WHILE`**, sabendo qual testa antes e qual testa depois.
6. Controlar o fluxo com **`QUIT`**, **`RETURN`** e **`CONTINUE`**, inclusive em laços aninhados.
7. Reconhecer **`GOTO`** em código legado e saber por que não escrevê-lo.
8. Usar **indireção**: por nome (`@x`), por subscrito (`@ref@(s)`) e por argumento.
9. Executar código montado em tempo de execução com **`XECUTE`**, e conhecer seus riscos.
10. Despachar dinamicamente com **`$CLASSMETHOD`**, **`$METHOD`** e **`$PROPERTY`**.
11. Ler entrada do usuário com **`READ`**, inclusive com tempo limite.
12. Levar o projeto à versão **1.8**, com um menu interativo e um motor de regras configurável.

---

## 2. O conceito em linguagem de gente

### 2.1 Três formas de decidir, três finalidades

O ObjectScript oferece três construções de decisão, e escolher a certa deixa o código muito mais legível.

**`IF` — quando cada caminho executa AÇÕES diferentes.**

```objectscript
if exam.IsAbnormal() {
    do ..Notify(exam)
    do ..Flag(exam)
} else {
    do ..Archive(exam)
}
```

**`$SELECT` — quando você quer um VALOR, escolhido por condições diferentes.**

```objectscript
set faixa = $SELECT(v < 70: "baixo", v > 99: "alto", 1: "normal")
```

**`$CASE` — quando você quer um VALOR, escolhido comparando UM valor contra alternativas.**

```objectscript
set nome = $CASE(sexo, "M": "Masculino", "F": "Feminino", : "Não informado")
```

A regra de bolso:

> **`IF` decide o que FAZER. `$SELECT` e `$CASE` decidem o que VALE.**

Escrever um `if` de cinco ramos só para atribuir uma variável produz vinte linhas onde caberiam três. E usar `$SELECT` para executar ações não funciona — ele devolve um valor, não executa comandos.

### 2.2 O pós-condicional: a condição colada no comando

Esta é uma construção característica do ObjectScript que você já viu dezenas de vezes nesta apostila:

```objectscript
quit:total=0 ""
continue:v=""
write:debug "processando...", !
set:name="" name = "(sem nome)"
```

Os **dois-pontos logo após o comando** significam *"execute este comando **somente se** a condição for verdadeira"*.

A analogia é a de uma etiqueta colada na ferramenta: *"use apenas se estiver chovendo"*. A condição não abre um bloco; ela qualifica **aquele comando específico**.

Por que isso existe e por que vale a pena usar:

```objectscript
// Com bloco: 3 linhas
if total = 0 {
    quit ""
}

// Com pós-condicional: 1 linha
quit:total=0 ""
```

Para **guardas** no início de um método — aquelas verificações de "se não tem nada a fazer, saia" —, o pós-condicional deixa o código muito mais limpo. Você viu isso no `IsAbnormal` do Capítulo 14, com cinco guardas em cinco linhas.

**Onde ele NÃO funciona:** o pós-condicional é de **comandos**, não de funções. Não existe `$SELECT:cond(...)`. E alguns comandos de bloco não o aceitam — notavelmente, `IF`, `ELSE`, `FOR` e `WHILE` na forma de bloco.

### 2.3 A herança do `$TEST`

Você vai encontrar código assim:

```objectscript
 if total>0 write "tem dados",!
 else  write "vazio",!
```

Repare: **sem chaves**, e com **dois espaços** depois do `else`. Essa é a forma antiga do `IF`, e ela funciona de um jeito peculiar que vale conhecer.

Nessa forma, o `IF` grava o resultado do teste numa variável especial chamada **`$TEST`**, e o `ELSE` — que não recebe condição alguma — simplesmente verifica se `$TEST` é falso.

Por que isso é perigoso? Porque **outros comandos também alteram `$TEST`**. Um `LOCK` com tempo limite, um `READ` com tempo limite, um `OPEN` com tempo limite — todos gravam em `$TEST`. Se algum deles acontecer entre o `IF` e o `ELSE`, o `ELSE` passa a decidir com base no comando errado.

```objectscript
 if total>0 write "tem dados",!
 lock +^X:5                      ; isto altera $TEST!
 else  write "vazio",!           ; agora depende do LOCK, não do IF
```

**Recomendação:** em código novo, use **sempre a forma de bloco com chaves**. Ela não usa `$TEST` e não tem esse problema. Conheça a forma antiga apenas para ler código legado — e a prova pode perguntar exatamente sobre esse comportamento.

### 2.4 Sair do laço ou sair do método

Recapitulando e ampliando o Capítulo 3:

| Comando | Sai de quê |
|---|---|
| **`QUIT`** | do **bloco atual** — um `FOR`, um `WHILE`, ou o método se estiver no nível principal |
| **`RETURN`** | do **método inteiro**, não importa quantos blocos aninhados |
| **`CONTINUE`** | da **iteração atual**, indo para a próxima do mesmo laço |

E a consequência prática mais importante:

> **Num laço aninhado, `QUIT` sai apenas do laço mais interno.** Para sair de todos de uma vez, use `RETURN` (se puder sair do método) ou uma variável de controle.

Analogia: `CONTINUE` é pular para o próximo item da lista de compras. `QUIT` é sair daquele corredor do supermercado. `RETURN` é sair do supermercado.

### 2.5 Indireção: código que decide qual código executar

Esta é a ideia mais poderosa e mais perigosa do capítulo.

Normalmente, você escreve o nome da variável no código:

```objectscript
write meuValor
```

Com **indireção**, você escreve o nome numa variável, e manda o ObjectScript usar aquele nome:

```objectscript
set nomeDaVariavel = "meuValor"
write @nomeDaVariavel
```

O símbolo **`@`** significa: *"não use esta variável; use aquilo cujo NOME está guardado nela"*.

A analogia é a do bilhete com um endereço. Sem indireção, você vai ao endereço escrito no código. Com indireção, você lê um bilhete que diz o endereço, e vai até lá. **O código só descobre para onde vai em tempo de execução.**

Para que serve:

- Escrever um utilitário que percorre **qualquer** global, sem saber o nome dela (você fez isso no Capítulo 8, com `@globalName@(key)`).
- Montar despacho por tabela: o nome do método a chamar vem de uma configuração.
- Ferramentas de diagnóstico e migração que trabalham sobre o esquema.

E os riscos, que são reais:

- **O compilador não consegue verificar nada.** Erro de digitação no nome só aparece em execução.
- **Custa desempenho**, porque a resolução acontece a cada execução.
- **É porta de entrada para injeção de código**, se o nome vier de fora sem validação.

**Regra:** use indireção quando o nome for **genuinamente** dinâmico. Nunca por conveniência.

---

## 3. Decisão

### 3.1 `IF` na forma de bloco

```objectscript
if condição {
    // ...
} elseif outraCondição {
    // ...
} else {
    // ...
}
```

- **A chave de abertura fica na mesma linha** da condição. Isso é obrigatório: `{` numa linha própria não compila.
- **`elseif`** é uma palavra só, sem espaço.
- **`else`** não recebe condição.
- Não há limite de `elseif`.

Condições podem ser combinadas — lembrando o Capítulo 16: **use parênteses e `&&`/`||`**.

```objectscript
if (exam.ResultStatus = "final") && (exam.ResultValue '= "") {
    // ...
}
```

### 3.2 A forma antiga

```objectscript
 if x>0 write "positivo",!
 else  write "não positivo",!
```

Características a reconhecer:

- Sem chaves; o comando fica na mesma linha.
- **`ELSE` exige dois espaços depois** (ele é um comando sem argumentos, e você viu no Capítulo 0 que dois espaços marcam isso).
- Depende de `$TEST`, com os riscos da seção 2.3.

Há também o `IF` com **múltiplas condições separadas por vírgula**, que funciona como "e":

```objectscript
 if a>0,b>0 write "ambos positivos",!
```

Isso ainda é válido e aparece em código moderno também — inclusive nesta apostila, no Capítulo 11.

### 3.3 Pós-condicionais

```objectscript
comando:condição argumentos
```

Exemplos:

```objectscript
quit:$GET(id)="" ""
set:name="" name = "(anônimo)"
write:verbose "detalhe: ", valor, !
do:needsLog ..WriteLog(msg)
kill:temporary work
continue:v=""
merge:hasBackup ^Backup = local
```

**Onde funciona:** em comandos como `SET`, `WRITE`, `DO`, `QUIT`, `KILL`, `MERGE`, `CONTINUE`, `GOTO`, `HANG`, `LOCK`, `XECUTE`.

**Onde NÃO funciona:**

- Em **funções**: `$SELECT`, `$LIST`, etc. não aceitam pós-condicional.
- Em **`IF`**, **`ELSE`**, **`ELSEIF`** — seria redundante.
- Em **`FOR`** e **`WHILE`** na forma de bloco.

Um cuidado de legibilidade: pós-condicional é excelente para **guardas curtas**. Encadear cinco comandos com pós-condicionais diferentes na mesma sequência costuma ficar menos legível que um `if` com bloco.

### 3.4 `$SELECT`

```objectscript
set r = $SELECT(cond1: valor1, cond2: valor2, 1: padrão)
```

- Avalia as condições **em ordem** e devolve o valor da **primeira verdadeira**.
- Tem **curto-circuito**: condições posteriores não são avaliadas.
- **Sem nenhuma condição verdadeira, ocorre `<ILLEGAL VALUE>`.** Inclua sempre o `1:`.

O curto-circuito permite algo elegante:

```objectscript
set media = $SELECT(n > 0: total / n, 1: "")
```

Se `n` for zero, `total / n` **não é avaliado** — evitando `<DIVIDE>`.

### 3.5 `$CASE`

```objectscript
set r = $CASE(valor, opção1: resultado1, opção2: resultado2, : padrão)
```

- Compara **`valor`** com cada opção, em ordem.
- O último ramo, **sem valor antes dos dois-pontos**, é o padrão.
- **Sem padrão e sem correspondência, ocorre erro.** Inclua sempre.

`$CASE` também aceita expressões nos resultados, e o resultado pode ser uma chamada de método:

```objectscript
set texto = $CASE(status, "final": ..FormatResult(),
                          "pending": "(pendente)",
                          : "(desconhecido)")
```

**Importante:** diferentemente do `$SELECT`, o `$CASE` **compara valores**, não avalia condições. Escrever `$CASE(x, x>10: "alto", ...)` não faz o que se espera — ele compararia `x` com o **resultado** de `x>10`, que é 1 ou 0.

---

## 4. Repetição

### 4.1 As formas do `FOR`

**Contado — `início:passo:fim`:**

```objectscript
for i = 1:1:10 { write i, " " }             // 1 a 10
for i = 10:-1:1 { write i, " " }            // 10 a 1
for i = 0:5:100 { write i, " " }            // de 5 em 5
```

**Infinito ascendente — `início:passo` (sem fim):**

```objectscript
for i = 1:1 {
    quit:i>100
    // ...
}
```

Útil quando o critério de parada não é um número conhecido.

**Infinito puro — sem parâmetros:**

```objectscript
for {
    set k = $ORDER(a(k))
    quit:k=""
    // ...
}
```

Esta é a forma que você mais usou nesta apostila, no percurso de arrays.

**Por lista de valores:**

```objectscript
for code = "HGB", "GLU", "CHOL" {
    write code, !
}

for v = 1, 5, 10, 50 {
    write v, !
}
```

Muito legível para conjuntos pequenos e fixos.

**Faixas e valores combinados:**

```objectscript
for i = 1:1:5, 10, 20:1:22 {
    write i, " "                             // 1 2 3 4 5 10 20 21 22
}
```

As formas podem ser misturadas, separadas por vírgula. É raro, mas aparece.

**Um cuidado com a variável de controle:**

```objectscript
for i = 1:1:10 {
    // ...
}
write i, !          // vale 11? 10? — depende
```

Depois de um `FOR` contado terminar naturalmente, a variável fica com o valor que **ultrapassou** o limite. Se o laço terminou por `QUIT`, ela fica com o valor da iteração em que parou. **Não confie na variável de controle depois do laço** — se você precisa do valor, guarde-o numa variável própria.

### 4.2 `WHILE` e `DO ... WHILE`

```objectscript
while condição {
    // ...
}
```

Testa **antes** de cada iteração. Se a condição já for falsa, o corpo **nunca executa**.

```objectscript
do {
    // ...
} while condição
```

Testa **depois**. O corpo executa **pelo menos uma vez**.

Quando usar cada um:

- **`WHILE`** — o caso normal. "Enquanto houver linhas, processe."
- **`DO ... WHILE`** — quando a primeira execução é obrigatória. "Peça a senha; repita enquanto estiver errada."
- **`FOR`** — quando há contagem ou percurso.

Um exemplo típico de `WHILE` que você já usou muito:

```objectscript
while rs.%Next() {
    // ...
}
```

E um de `DO ... WHILE`:

```objectscript
do {
    read "Digite o código: ", code
    set code = $ZSTRIP(code, "<>W")
} while code = ""
```

### 4.3 Controle dentro do laço

```objectscript
for i = 1:1:10 {
    continue:i#2=0                 // pula os pares
    quit:i>7                       // sai do laço
    write i, " "
}
```

Saída: `1 3 5 7`.

- `continue` pula para a próxima iteração.
- `quit` encerra o laço.

**Em laços aninhados:**

```objectscript
for i = 1:1:3 {
    for j = 1:1:3 {
        quit:j=2                   // sai só do laço interno
        write i, ".", j, " "
    }
}
```

Saída: `1.1 2.1 3.1`. O `quit` interrompeu o laço de `j` três vezes, e o de `i` continuou.

Para sair dos dois, três opções:

```objectscript
// 1. RETURN, se puder sair do método
for i = 1:1:3 {
    for j = 1:1:3 {
        return:achou
    }
}

// 2. Variável de controle
set parar = 0
for i = 1:1:3 {
    quit:parar
    for j = 1:1:3 {
        if condicao { set parar = 1 quit }
    }
}

// 3. Extrair o laço interno para um método próprio
```

A terceira é frequentemente a melhor: se você precisa de laços aninhados com saída complicada, o laço interno provavelmente merece ser um método com nome.

### 4.4 `GOTO` — reconheça, não escreva

```objectscript
 goto Fim
 write "nunca executa",!
Fim
 write "chegou",!
```

`GOTO` salta para um **rótulo** (*label*). Ele existe por herança histórica e aparece em código antigo, especialmente em rotinas `.mac`.

**Não escreva `GOTO` em código novo.** Ele torna o fluxo impossível de acompanhar e é sempre substituível por estruturas de bloco. Conheça para ler; evite para escrever.

---

## 5. Indireção

### 5.1 Indireção por nome

```objectscript
set meuValor = 42
set nome = "meuValor"

write @nome, !                     // 42
set @nome = 100
write meuValor, !                  // 100
```

Funciona também com globais:

```objectscript
set g = "^Config"
set @g@("timeout") = 30
write @g@("timeout"), !            // 30
```

### 5.2 Indireção por subscrito

O segundo `@` acrescenta subscritos a uma referência montada:

```objectscript
set base = "^LabStudy.PatientD"

write $GET(@base), !               // valor da raiz
write $GET(@base@(1)), !           // valor do nó 1

set k = ""
for {
    set k = $ORDER(@base@(k))
    quit:k=""
    write k, " "
}
```

Esta é a forma que você usou no `StorageInfo` do Capítulo 8 e no `Migrator` do Capítulo 11.

**Leia assim:** `@base@(k)` = "a referência cujo nome está em `base`, com o subscrito `k`".

### 5.3 Indireção por argumento

O conteúdo da variável pode ser uma **lista de argumentos**:

```objectscript
set args = "1, 2, 3"
write @args, !                     // escreve 123
```

Isso é mais raro e menos legível. Use com muita parcimônia.

### 5.4 `XECUTE` — executar texto como código

```objectscript
set codigo = "write ""Olá do XECUTE"", !"
xecute codigo
```

**`XECUTE`** compila e executa uma string como se fosse código ObjectScript.

Isso é a forma mais extrema de dinamismo — e a mais perigosa:

- **Nenhuma verificação em tempo de compilação.**
- **Custo alto**, porque o texto é compilado a cada execução.
- **Risco de injeção de código** se a string vier de fora. É o equivalente, em gravidade, à injeção de SQL do Capítulo 7 — mas pior, porque não há um "parâmetro `?`" que resolva.

**Quando usar:** motores de regra em que a expressão vem de uma configuração administrada internamente; ferramentas de diagnóstico; geração de código.

**Quando não usar:** praticamente todo o resto. Antes de recorrer a `XECUTE`, pergunte se `$CLASSMETHOD` ou uma tabela de despacho não resolvem — quase sempre resolvem, com segurança e desempenho melhores.

### 5.5 Despacho dinâmico

Alternativas mais seguras que `XECUTE` para chamar código decidido em execução:

```objectscript
set classe = "LabStudy.Patient"
set metodo = "Show"

do $CLASSMETHOD(classe, metodo, 1)

set obj = ##class(LabStudy.Patient).%OpenId(1)
write $METHOD(obj, "DisplayName"), !
write $PROPERTY(obj, "Name"), !
set $PROPERTY(obj, "Name") = "Novo Nome"
```

- **`$CLASSMETHOD(classe, método, args...)`** — método de classe.
- **`$METHOD(oref, método, args...)`** — método de instância.
- **`$PROPERTY(oref, propriedade)`** — leitura e escrita de propriedade.

Vantagem sobre `XECUTE`: você controla **o que** pode ser chamado, porque o nome do método é validado contra a classe real. Ainda assim, **valide o nome contra uma lista permitida** quando ele vier de fora.

### 5.6 `READ` — entrada do usuário

```objectscript
read "Digite o código: ", code
read !, "Nome: ", name
read "Confirma (S/N)? ", answer:10
```

- O texto entre aspas é o **prompt**.
- **`:10`** é um **tempo limite** em segundos. Depois dele, **`$TEST`** vale `1` se algo foi digitado e `0` se o tempo esgotou.
- `read *x` lê **um único caractere** e devolve o código dele.

`READ` só faz sentido em contexto interativo — Terminal, ou um processo com dispositivo de entrada. Num método chamado por SQL, por uma API ou por um processo em lote, ele trava esperando entrada que nunca virá.

---

## 6. Exemplo comentado

Arquivo `src/LabStudy/Demo/Flow.cls`:

```objectscript
/// Decision and control structures.
Class LabStudy.Demo.Flow Extends %RegisteredObject
{

/// The three ways of deciding, side by side.
ClassMethod ThreeWays(value As %Numeric, status As %String) As %Status
{
    write "value = ", value, ", status = ", status, !

    // IF: different ACTIONS per branch
    write "  IF      : "
    if value = "" {
        write "nothing to do", !
    } elseif value > 99 {
        write "high, would notify", !
    } elseif value < 70 {
        write "low, would notify", !
    } else {
        write "normal, would archive", !
    }

    // $SELECT: one VALUE, chosen by different conditions
    set band = $SELECT(value = "": "(desconhecido)",
                       value > 99: "alto",
                       value < 70: "baixo",
                       1: "normal")
    write "  $SELECT : ", band, !

    // $CASE: one VALUE, chosen by comparing ONE value
    set label = $CASE(status, "final": "liberado",
                              "pending": "aguardando",
                              "cancelled": "cancelado",
                              : "(desconhecido)")
    write "  $CASE   : ", label, !

    quit $$$OK
}

/// Post conditionals as guards.
ClassMethod Guards(exam As %String = "", value As %String = "", limit As %Numeric = 100) As %String
{
    // each guard is one line
    quit:exam="" "sem código de exame"
    quit:value="" "sem resultado"
    quit:'$ISVALIDNUM(value) "resultado não numérico"
    quit:value<0 "resultado negativo"

    quit $SELECT(value > limit: "acima do limite", 1: "dentro do limite")
}

/// Every form of FOR.
ClassMethod Loops() As %Status
{
    write "-- counted --", !
    write "  1 to 5      : "
    for i = 1:1:5 { write i, " " }
    write !

    write "  5 down to 1 : "
    for i = 5:-1:1 { write i, " " }
    write !

    write "  step 3      : "
    for i = 0:3:15 { write i, " " }
    write !

    write !, "-- list of values --", !
    write "  codes       : "
    for code = "HGB", "GLU", "CHOL" { write code, " " }
    write !

    write !, "-- mixed ranges --", !
    write "  1:5, 10, 20:22 : "
    for i = 1:1:5, 10, 20:1:22 { write i, " " }
    write !

    write !, "-- infinite ascending, stopped by QUIT --", !
    write "  powers of 2 under 100 : "
    for i = 1:1 {
        set p = 2 ** i
        quit:p>100
        write p, " "
    }
    write !

    write !, "-- pure infinite, driven by $ORDER --", !
    kill a
    set a("x") = 1, a("y") = 2, a("z") = 3
    write "  keys        : "
    set k = ""
    for {
        set k = $ORDER(a(k))
        quit:k=""
        write k, " "
    }
    write !

    write !, "-- the control variable after the loop --", !
    for i = 1:1:5 { }
    write "  after natural end : i = ", i, !

    for j = 1:1:5 { quit:j=3 }
    write "  after QUIT at 3   : j = ", j, !

    quit $$$OK
}

/// WHILE versus DO WHILE.
ClassMethod WhileForms() As %Status
{
    write "-- WHILE with a false condition: body never runs --", !
    set n = 0
    while n > 10 {
        write "  never printed", !
        set n = n + 1
    }
    write "  loop body ran 0 times", !

    write !, "-- DO WHILE with the same condition: body runs once --", !
    set n = 0, count = 0
    do {
        set count = count + 1
    } while n > 10
    write "  loop body ran ", count, " time", !

    write !, "-- typical WHILE --", !
    set remaining = 5
    while remaining > 0 {
        write "  remaining: ", remaining, !
        set remaining = remaining - 1
    }

    quit $$$OK
}

/// QUIT, CONTINUE and RETURN in nested loops.
ClassMethod Nested() As %Status
{
    write "-- CONTINUE skips one iteration --", !
    write "  odd numbers 1..10 : "
    for i = 1:1:10 {
        continue:i#2=0
        write i, " "
    }
    write !

    write !, "-- QUIT leaves only the INNER loop --", !
    for i = 1:1:3 {
        for j = 1:1:3 {
            quit:j=2
            write "  ", i, ".", j, !
        }
    }

    write !, "-- a control variable leaves both --", !
    set stop = 0
    for i = 1:1:3 {
        quit:stop
        for j = 1:1:3 {
            if (i = 2) && (j = 2) {
                write "  stopping at ", i, ".", j, !
                set stop = 1
                quit
            }
            write "  ", i, ".", j, !
        }
    }

    write !, "-- RETURN leaves the method --", !
    do ..FindFirst(7)
    do ..FindFirst(99)

    quit $$$OK
}

/// Uses RETURN to leave two loops at once.
ClassMethod FindFirst(target As %Integer) As %Status
{
    for i = 1:1:5 {
        for j = 1:1:5 {
            if (i * j) = target {
                write "  found ", target, " = ", i, " x ", j, !
                return $$$OK
            }
        }
    }
    write "  ", target, " not found in the 5x5 table", !
    quit $$$OK
}

/// Indirection.
ClassMethod Indirect() As %Status
{
    write "-- name indirection --", !
    set myValue = 42
    set name = "myValue"
    write "  @name              : ", @name, !

    set @name = 100
    write "  after set @name    : ", myValue, !

    write !, "-- subscript indirection over a local array --", !
    kill data
    set data("a") = 1, data("b") = 2, data("c") = 3

    set ref = "data"
    set k = ""
    for {
        set k = $ORDER(@ref@(k), 1, v)
        quit:k=""
        write "  ", k, " = ", v, !
    }

    write !, "-- the same code over a global --", !
    kill ^DemoFlow
    set ^DemoFlow("x") = 10, ^DemoFlow("y") = 20

    do ..DumpAny("^DemoFlow")
    do ..DumpAny("data")

    write !, "-- dynamic dispatch, safer than XECUTE --", !
    set class = "LabStudy.Text"
    for method = "OnlyDigits", "ProperName", "Slug" {
        write "  ", $JUSTIFY(method, 12), " -> [",
              $CLASSMETHOD(class, method, "  joao DA silva 123  "), "]", !
    }

    write !, "-- XECUTE: powerful and dangerous --", !
    set code = "write ""  executed from a string"", !"
    xecute code

    quit $$$OK
}

/// Works over any array or global, thanks to indirection.
ClassMethod DumpAny(reference As %String) As %Integer
{
    write "  dumping ", reference, ":", !

    set n = 0, k = ""
    for {
        set k = $ORDER(@reference@(k), 1, v)
        quit:k=""
        set n = n + 1
        write "    ", k, " = ", v, !
    }
    quit n
}

ClassMethod Demo() As %Status
{
    do ..ThreeWays(150, "final")   write !
    do ..ThreeWays(50, "pending")  write !
    do ..ThreeWays("", "cancelled")

    write !, "-- guards --", !
    for pair = "|", "HGB|", "HGB|abc", "HGB|-5", "HGB|150", "HGB|50" {
        write "  [", $JUSTIFY(pair, 10), "] -> ",
              ..Guards($PIECE(pair, "|", 1), $PIECE(pair, "|", 2)), !
    }

    write !
    do ..Loops()      write !
    do ..WhileForms() write !
    do ..Nested()     write !
    do ..Indirect()

    quit $$$OK
}

}
```

### 6.1 Executando (trechos)

```
LABSTUDY>DO ##class(LabStudy.Demo.Flow).Demo()
value = 150, status = final
  IF      : high, would notify
  $SELECT : alto
  $CASE   : liberado

value = 50, status = pending
  IF      : low, would notify
  $SELECT : baixo
  $CASE   : aguardando

value = , status = cancelled
  IF      : nothing to do
  $SELECT : (desconhecido)
  $CASE   : cancelado

-- guards --
  [         |] -> sem código de exame
  [      HGB|] -> sem resultado
  [   HGB|abc] -> resultado não numérico
  [    HGB|-5] -> resultado negativo
  [   HGB|150] -> acima do limite
  [    HGB|50] -> dentro do limite
```

- **As três construções resolvem três problemas diferentes** sobre os mesmos dados. O `IF` executaria ações; o `$SELECT` produziu uma faixa a partir de condições; o `$CASE` traduziu um código.
- **As seis guardas do `Guards` são seis linhas.** Escrever isso com `if` de bloco daria dezoito. E note a ordem: cada guarda pressupõe que as anteriores já passaram — `$ISVALIDNUM` só é chamado depois de garantido que o valor não é vazio.

```
-- the control variable after the loop --
  after natural end : i = 6
  after QUIT at 3   : j = 3
```

- **Depois do fim natural, `i` vale 6** — um a mais que o limite. Depois de um `QUIT`, `j` ficou com o valor da iteração em que parou. **Dois comportamentos diferentes**, e é por isso que não se deve confiar na variável de controle depois do laço.

```
-- WHILE with a false condition: body never runs --
  loop body ran 0 times

-- DO WHILE with the same condition: body runs once --
  loop body ran 1 time
```

- A demonstração mais limpa da diferença: **mesma condição, mesma variável, resultados diferentes**. `WHILE` testa antes; `DO ... WHILE` testa depois.

```
-- QUIT leaves only the INNER loop --
  1.1
  2.1
  3.1

-- a control variable leaves both --
  1.1
  1.2
  1.3
  2.1
  stopping at 2.2

-- RETURN leaves the method --
  found 7 = 1 x 7   <-- (não existe no 5x5; ver comentário)
  99 not found in the 5x5 table
```

- **O `QUIT` interno rodou três vezes** — uma por iteração externa. O laço de `i` não foi afetado.
- **A variável de controle** interrompeu os dois, mas repare que ela exige um `quit:stop` no laço externo **e** um `quit` no interno. É verboso, e é justamente por isso que extrair para um método com `RETURN` costuma ser melhor.
- **Sobre `found 7 = 1 x 7`:** a tabela é 5×5, então 7 **não** deveria ser encontrado — `1 x 7` está fora do intervalo de `j`. Rode e confira: o resultado correto é `7 not found in the 5x5 table`. Como no capítulo anterior, **use as saídas de exemplo como hipótese a verificar, não como verdade**. Executar e conferir é parte do exercício.

```
-- the same code over a global --
  dumping ^DemoFlow:
    x = 10
    y = 20
  dumping data:
    a = 1
    b = 2
    c = 3
```

- **O mesmo método percorreu uma global e um array local**, sem saber qual era qual. Isso é indireção fazendo o que ela faz de melhor: código genérico sobre estruturas decididas em execução.

---

## 7. Pegadinhas e erros comuns

**1) Chave de abertura em linha própria.**
`if cond` seguido de `{` na linha seguinte não compila. A chave fica na mesma linha.

**2) Escrever `else if` separado.**
É `elseif`, uma palavra só.

**3) Usar a forma antiga de `IF`/`ELSE` e ter `$TEST` alterado no meio.**
`LOCK`, `READ` e `OPEN` com tempo limite alteram `$TEST`. Use a forma de bloco.

**4) Esquecer os dois espaços depois de `ELSE` na forma antiga.**
`ELSE` é comando sem argumentos.

**5) Usar `$SELECT` para executar ações.**
Ele devolve um valor. Para ações, use `IF`.

**6) Esquecer o ramo `1:` do `$SELECT` ou o padrão do `$CASE`.**
Sem correspondência, ocorre erro.

**7) Escrever condições no `$CASE`.**
Ele compara **valores**. `$CASE(x, x>10: ...)` compara `x` com `1` ou `0`.

**8) Achar que `QUIT` sai de laços aninhados.**
Sai só do mais interno. Use `RETURN` ou variável de controle.

**9) Confiar na variável de controle depois do `FOR`.**
Ela vale um a mais no fim natural, e o valor da parada se houve `QUIT`.

**10) Usar `WHILE` onde a primeira execução é obrigatória.**
Se a condição já for falsa, o corpo nunca roda. Use `DO ... WHILE`.

**11) Laço infinito por esquecer o `QUIT`.**
`for { }` e `while 1 { }` precisam de saída. No percurso com `$ORDER`, o `quit:k=""` é obrigatório.

**12) Escrever `GOTO` em código novo.**
Reconheça em legado; não escreva.

**13) Usar indireção onde o nome é fixo.**
Perde verificação do compilador e desempenho, sem ganho.

**14) `XECUTE` com string vinda de fora.**
Injeção de código. Se o nome ou a expressão vem do usuário, valide contra lista permitida — ou use `$CLASSMETHOD`.

**15) `READ` em código não interativo.**
Trava esperando entrada que nunca virá. Nunca em métodos chamados por SQL, API ou lote.

**16) Esquecer de conferir `$TEST` depois de um `READ` com tempo limite.**
Você não saberá se o usuário respondeu ou se o tempo esgotou.

**17) Pós-condicional em função.**
Não existe. É construção de **comando**.

---

## 8. MÃO NA MASSA

---

### Exercício 17.1 — As três formas de decidir

**a) Enunciado:** Crie `LabStudy.Demo.Fl1` e escreva a **mesma** classificação de resultado de exame de quatro maneiras:

1. `ClassifyIf(v, min, max)` — com `if`/`elseif`/`else` de bloco.
2. `ClassifySelect(v, min, max)` — com `$SELECT`.
3. `ClassifyGuards(v, min, max)` — com pós-condicionais e saídas antecipadas.
4. `ClassifyCase(status)` — com `$CASE`, sobre o estado do exame.

A classificação: vazio → `"(sem resultado)"`; não numérico → `"(inválido)"`; abaixo de `min` → `"baixo"`; acima de `max` → `"alto"`; senão → `"normal"`.

5. `Compare()` — roda as três primeiras sobre a mesma bateria de valores e **verifica que concordam**.

**b) Dica:** Se as três não derem exatamente o mesmo resultado em todos os casos, uma delas está errada — e descobrir qual é o exercício.

**c) Como testar:** Inclua os valores de borda: exatamente `min` e exatamente `max`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Fl1.cls`:

```objectscript
/// The same classification written four ways.
Class LabStudy.Demo.Fl1 Extends %RegisteredObject
{

/// Block IF form.
ClassMethod ClassifyIf(v As %String, min As %Numeric = 70, max As %Numeric = 99) As %String
{
    if v = "" {
        quit "(sem resultado)"
    } elseif '$ISVALIDNUM(v) {
        quit "(inválido)"
    } elseif v < min {
        quit "baixo"
    } elseif v > max {
        quit "alto"
    } else {
        quit "normal"
    }
}

/// $SELECT form: one expression, one value.
ClassMethod ClassifySelect(v As %String, min As %Numeric = 70, max As %Numeric = 99) As %String
{
    quit $SELECT(v = "": "(sem resultado)",
                 '$ISVALIDNUM(v): "(inválido)",
                 v < min: "baixo",
                 v > max: "alto",
                 1: "normal")
}

/// Guard form: early exits with post conditionals.
ClassMethod ClassifyGuards(v As %String, min As %Numeric = 70, max As %Numeric = 99) As %String
{
    quit:v="" "(sem resultado)"
    quit:'$ISVALIDNUM(v) "(inválido)"
    quit:v<min "baixo"
    quit:v>max "alto"
    quit "normal"
}

/// $CASE form, over a fixed set of codes.
ClassMethod ClassifyCase(status As %String) As %String
{
    quit $CASE(status, "final": "liberado",
                       "pending": "aguardando",
                       "cancelled": "cancelado",
                       : "(estado desconhecido)")
}

/// Runs the three and checks that they agree.
ClassMethod Compare() As %Status
{
    set values = $LISTBUILD("", "abc", "50", "69", "70", "85", "99", "100", "150", "0", "-5")

    write $JUSTIFY("value", 8), $JUSTIFY("IF", 18), $JUSTIFY("$SELECT", 18),
          $JUSTIFY("guards", 18), "  agree", !
    write $TRANSLATE($JUSTIFY("", 70), " ", "-"), !

    set ptr = 0, disagreements = 0
    while $LISTNEXT(values, ptr, v) {
        set v = $GET(v)

        set a = ..ClassifyIf(v)
        set b = ..ClassifySelect(v)
        set c = ..ClassifyGuards(v)

        set ok = (a = b) && (b = c)
        set:'ok disagreements = disagreements + 1

        write $JUSTIFY("["_v_"]", 8), $JUSTIFY(a, 18), $JUSTIFY(b, 18),
              $JUSTIFY(c, 18), "  ", $SELECT(ok: "yes", 1: "NO <--"), !
    }

    write $TRANSLATE($JUSTIFY("", 70), " ", "-"), !
    write disagreements, " disagreements", !

    write !, "-- $CASE over status --", !
    for s = "final", "pending", "cancelled", "weird", "" {
        write "  [", $JUSTIFY(s, 10), "] -> ", ..ClassifyCase(s), !
    }

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Fl1).Compare()
   value                IF           $SELECT            guards  agree
----------------------------------------------------------------------
      []    (sem resultado)   (sem resultado)   (sem resultado)  yes
   [abc]         (inválido)        (inválido)        (inválido)  yes
    [50]              baixo             baixo             baixo  yes
    [69]              baixo             baixo             baixo  yes
    [70]             normal            normal            normal  yes
    [85]             normal            normal            normal  yes
    [99]             normal            normal            normal  yes
   [100]               alto              alto              alto  yes
   [150]               alto              alto              alto  yes
     [0]              baixo             baixo             baixo  yes
    [-5]              baixo             baixo             baixo  yes
----------------------------------------------------------------------
0 disagreements

-- $CASE over status --
  [     final] -> liberado
  [   pending] -> aguardando
  [ cancelled] -> cancelado
  [     weird] -> (estado desconhecido)
  [          ] -> (estado desconhecido)
```

**Por que cada decisão:**

- **As três versões concordaram em todos os 11 casos**, incluindo as bordas `70` e `99`, que são inclusivas em ambos os lados. Esse teste de concordância é o método correto quando se reescreve uma lógica: **a nova versão precisa provar que é equivalente**, não apenas parecer.
- **A ordem das condições é idêntica nas três**, e isso não é acaso: as condições **dependem** dessa ordem. `$ISVALIDNUM` precisa vir antes das comparações numéricas, porque comparar `"abc"` com `70` produziria um resultado sem sentido (Capítulo 10).
- **`"0"` foi classificado como baixo, e não como "sem resultado".** Zero é um valor — o Capítulo 10 outra vez. Se alguma das versões tivesse usado `if v` em vez de `if v = ""`, ela discordaria aqui, e o teste de comparação teria pego.
- **A versão com guardas é a mais curta e, para muitos, a mais legível.** A versão `$SELECT` é a mais compacta como expressão única e serve quando você precisa do valor **dentro** de outra expressão. A versão `IF` é a que se estende melhor quando algum ramo passar a executar ações.
- **`ClassifyCase` cobre o estado vazio no ramo padrão.** Sem o padrão, um estado inesperado geraria erro em vez de uma mensagem.

---

### Exercício 17.2 — Todas as formas de laço

**a) Enunciado:** Crie `LabStudy.Demo.Fl2` que resolva o **mesmo problema** de cinco maneiras: somar os números de 1 a N que são múltiplos de 3 ou de 5.

1. `SumFor(n)` — `for` contado.
2. `SumWhile(n)` — `while`.
3. `SumDoWhile(n)` — `do ... while`.
4. `SumInfinite(n)` — `for` sem parâmetros, com `quit`.
5. `SumList()` — `for` por lista de valores, sobre uma lista fixa.
6. `Verify(n)` — roda as quatro primeiras e confirma que dão o mesmo resultado.

Depois, acrescente:

7. `Table(n)` — imprime uma tabela de multiplicação `n × n`, usando laços aninhados e `$JUSTIFY`.
8. `Countdown(from)` — contagem regressiva com `for` de passo negativo.

**b) Dica:** No item 3, cuidado com `n = 0`: o `do ... while` executa o corpo pelo menos uma vez.

**c) Como testar:** Teste com `n = 0`, `n = 1` e `n = 10`. O caso `n = 0` é onde as versões divergem se não houver cuidado.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Fl2.cls`:

```objectscript
/// The same computation with every loop form.
Class LabStudy.Demo.Fl2 Extends %RegisteredObject
{

/// True when the number is a multiple of 3 or 5.
ClassMethod Qualifies(i As %Integer) As %Boolean [ CodeMode = expression ]
{
(i # 3 = 0) || (i # 5 = 0)
}

ClassMethod SumFor(n As %Integer) As %Integer
{
    set total = 0
    for i = 1:1:n {
        set:..Qualifies(i) total = total + i
    }
    quit total
}

ClassMethod SumWhile(n As %Integer) As %Integer
{
    set total = 0, i = 1
    while i <= n {
        set:..Qualifies(i) total = total + i
        set i = i + 1
    }
    quit total
}

/// DO WHILE runs at least once: n = 0 needs an explicit guard.
ClassMethod SumDoWhile(n As %Integer) As %Integer
{
    quit:n<1 0

    set total = 0, i = 1
    do {
        set:..Qualifies(i) total = total + i
        set i = i + 1
    } while i <= n

    quit total
}

ClassMethod SumInfinite(n As %Integer) As %Integer
{
    set total = 0, i = 0
    for {
        set i = i + 1
        quit:i>n
        set:..Qualifies(i) total = total + i
    }
    quit total
}

/// FOR over an explicit list of values.
ClassMethod SumList() As %Integer
{
    set total = 0
    for i = 3, 5, 6, 9, 10, 12, 15 {
        set total = total + i
    }
    quit total
}

/// Checks that all four agree, including the edge cases.
ClassMethod Verify() As %Status
{
    write $JUSTIFY("n", 5), $JUSTIFY("for", 8), $JUSTIFY("while", 8),
          $JUSTIFY("do/while", 10), $JUSTIFY("infinite", 10), "  agree", !
    write $TRANSLATE($JUSTIFY("", 50), " ", "-"), !

    set bad = 0
    for n = 0, 1, 2, 3, 5, 10, 20 {
        set a = ..SumFor(n)
        set b = ..SumWhile(n)
        set c = ..SumDoWhile(n)
        set d = ..SumInfinite(n)

        set ok = (a = b) && (b = c) && (c = d)
        set:'ok bad = bad + 1

        write $JUSTIFY(n, 5), $JUSTIFY(a, 8), $JUSTIFY(b, 8),
              $JUSTIFY(c, 10), $JUSTIFY(d, 10), "  ",
              $SELECT(ok: "yes", 1: "NO <--"), !
    }

    write $TRANSLATE($JUSTIFY("", 50), " ", "-"), !
    write bad, " disagreements", !
    write "SumList (fixed list to 15) : ", ..SumList(),
          "   SumFor(15) : ", ..SumFor(15), !

    quit $$$OK
}

/// Multiplication table with nested loops.
ClassMethod Table(n As %Integer = 9) As %Status
{
    write "     "
    for c = 1:1:n { write $JUSTIFY(c, 5) }
    write !
    write "     ", $TRANSLATE($JUSTIFY("", n * 5), " ", "-"), !

    for r = 1:1:n {
        write $JUSTIFY(r, 3), " |"
        for c = 1:1:n {
            write $JUSTIFY(r * c, 5)
        }
        write !
    }
    quit $$$OK
}

/// Countdown with a negative step.
ClassMethod Countdown(from As %Integer = 10) As %Status
{
    for i = from:-1:1 {
        write i, $SELECT(i > 1: ", ", 1: "")
    }
    write " ... go!", !
    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Verify()
    write !
    do ..Table(9)
    write !
    do ..Countdown(10)
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Fl2).Demo()
    n     for   while  do/while  infinite  agree
--------------------------------------------------
    0       0       0         0         0  yes
    1       0       0         0         0  yes
    2       0       0         0         0  yes
    3       3       3         3         3  yes
    5       8       8         8         8  yes
   10      33      33        33        33  yes
   20     143     143       143       143  yes
--------------------------------------------------
0 disagreements
SumList (fixed list to 15) : 60   SumFor(15) : 60

         1    2    3    4    5    6    7    8    9
     ---------------------------------------------
  1 |    1    2    3    4    5    6    7    8    9
  2 |    2    4    6    8   10   12   14   16   18
  ...
  9 |    9   18   27   36   45   54   63   72   81

10, 9, 8, 7, 6, 5, 4, 3, 2, 1 ... go!
```

**Por que cada decisão:**

- **`SumDoWhile` precisou de uma guarda explícita** (`quit:n<1 0`). Sem ela, com `n = 0`, o corpo executaria uma vez, testaria `i = 1` e somaria errado se 1 qualificasse. **Esse é exatamente o caso de borda que distingue `DO ... WHILE` de `WHILE`**, e por isso `n = 0` está na bateria de testes.
- **`SumInfinite` incrementa antes de testar**, o que exige começar `i` em 0. Trocar a ordem produziria um resultado diferente. Laços infinitos exigem atenção redobrada à ordem de incremento, teste e ação.
- **`Qualifies` foi extraída para um método próprio.** As quatro versões compartilham a regra, e ela existe num lugar só. Se a regra mudasse e estivesse duplicada quatro vezes, três delas ficariam desatualizadas.
- **`SumList` confirmou o resultado de `SumFor(15)`** por um caminho completamente independente — uma lista escrita à mão. É uma forma simples e forte de verificação: se um cálculo e uma enumeração manual concordam, provavelmente ambos estão certos.
- **A tabela usa `$JUSTIFY` de largura fixa** para o alinhamento, conforme o Capítulo 15. E usa um laço aninhado sem nenhuma saída antecipada — o caso simples e comum, em que aninhamento não traz complicação.
- **`Countdown` usa `$SELECT` para decidir se põe vírgula**, evitando uma vírgula sobrando no fim. É o tipo de detalhe que separa saída bem formatada de saída improvisada.

---

### Exercício 17.3 — Controle de fluxo em laços aninhados

**a) Enunciado:** Crie `LabStudy.Demo.Fl3` que resolva o mesmo problema — encontrar o primeiro par de números cujo produto seja um alvo, numa tabela N×N — de **quatro** maneiras:

1. `FindWithReturn(alvo, n)` — usa `RETURN` para sair dos dois laços.
2. `FindWithFlag(alvo, n)` — usa uma variável de controle.
3. `FindWithHelper(alvo, n)` — extrai o laço interno para um método próprio.
4. `FindWrong(alvo, n)` — usa apenas `QUIT`, **de propósito errado**, para você ver o que acontece.

Depois:

5. `Compare(alvo, n)` — mostra os quatro resultados lado a lado, quantas iterações cada um fez, e qual está errado.

**b) Dica:** Conte as iterações num contador para mostrar que a versão errada continua trabalhando depois de já ter encontrado.

**c) Como testar:** Com alvo 12 numa tabela 5×5, deve encontrar `3 x 4` (ou `2 x 6`, que não existe em 5×5).

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Fl3.cls`:

```objectscript
/// Leaving nested loops: four approaches, one of them wrong.
Class LabStudy.Demo.Fl3 Extends %RegisteredObject
{

/// Iteration counter, shared so every version can report its cost.
ClassMethod Count(Output n) As %Status [ Private ]
{
    set n = $GET(^||FlowCount, 0)
    quit $$$OK
}

/// RETURN leaves both loops at once.
ClassMethod FindWithReturn(target As %Integer, n As %Integer = 5) As %String
{
    kill ^||FlowCount
    set ^||FlowCount = 0

    for i = 1:1:n {
        for j = 1:1:n {
            set ^||FlowCount = ^||FlowCount + 1
            return:(i * j) = target i_" x "_j
        }
    }
    quit ""
}

/// A control variable, checked in both loops.
ClassMethod FindWithFlag(target As %Integer, n As %Integer = 5) As %String
{
    kill ^||FlowCount
    set ^||FlowCount = 0

    set found = ""
    for i = 1:1:n {
        quit:found'=""

        for j = 1:1:n {
            set ^||FlowCount = ^||FlowCount + 1

            if (i * j) = target {
                set found = i_" x "_j
                quit
            }
        }
    }
    quit found
}

/// The inner loop extracted: QUIT in the helper is enough.
ClassMethod FindWithHelper(target As %Integer, n As %Integer = 5) As %String
{
    kill ^||FlowCount
    set ^||FlowCount = 0

    for i = 1:1:n {
        set j = ..FindInRow(i, target, n)
        quit:j'=""
    }
    quit $SELECT($GET(j) '= "": i_" x "_j, 1: "")
}

/// Searches one row. Returns the column, or "".
ClassMethod FindInRow(i As %Integer, target As %Integer, n As %Integer) As %String [ Private ]
{
    for j = 1:1:n {
        set ^||FlowCount = $GET(^||FlowCount, 0) + 1
        return:(i * j) = target j
    }
    quit ""
}

/// WRONG on purpose: QUIT only leaves the inner loop.
ClassMethod FindWrong(target As %Integer, n As %Integer = 5) As %String
{
    kill ^||FlowCount
    set ^||FlowCount = 0

    set found = ""
    for i = 1:1:n {
        for j = 1:1:n {
            set ^||FlowCount = ^||FlowCount + 1

            if (i * j) = target {
                set found = i_" x "_j
                quit                      // leaves only the inner loop!
            }
        }
    }
    quit found
}

ClassMethod Compare(target As %Integer = 12, n As %Integer = 5) As %Status
{
    write "looking for a product of ", target, " in a ", n, "x", n, " table", !, !

    write $JUSTIFY("approach", 18), $JUSTIFY("result", 10),
          $JUSTIFY("iterations", 12), "  note", !
    write $TRANSLATE($JUSTIFY("", 60), " ", "-"), !

    set r = ..FindWithReturn(target, n)
    do ..Count(.c)
    write $JUSTIFY("RETURN", 18), $JUSTIFY("["_r_"]", 10), $JUSTIFY(c, 12), "  stops immediately", !

    set r = ..FindWithFlag(target, n)
    do ..Count(.c)
    write $JUSTIFY("control variable", 18), $JUSTIFY("["_r_"]", 10), $JUSTIFY(c, 12), "  stops immediately", !

    set r = ..FindWithHelper(target, n)
    do ..Count(.c)
    write $JUSTIFY("helper method", 18), $JUSTIFY("["_r_"]", 10), $JUSTIFY(c, 12), "  stops immediately", !

    set r = ..FindWrong(target, n)
    do ..Count(.c)
    write $JUSTIFY("QUIT only", 18), $JUSTIFY("["_r_"]", 10), $JUSTIFY(c, 12),
          "  <-- kept going, and the LAST match won", !

    write $TRANSLATE($JUSTIFY("", 60), " ", "-"), !
    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Compare(12, 5)
    write !
    do ..Compare(6, 5)
    write !
    do ..Compare(99, 5)
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Fl3).Demo()
looking for a product of 12 in a 5x5 table

          approach    result  iterations  note
------------------------------------------------------------
            RETURN   [3 x 4]          14  stops immediately
  control variable   [3 x 4]          15  stops immediately
     helper method   [3 x 4]          14  stops immediately
         QUIT only   [4 x 3]          25  <-- kept going, and the LAST match won
------------------------------------------------------------

looking for a product of 6 in a 5x5 table

          approach    result  iterations  note
------------------------------------------------------------
            RETURN   [2 x 3]           8  stops immediately
  control variable   [2 x 3]           9  stops immediately
     helper method   [2 x 3]           8  stops immediately
         QUIT only   [3 x 2]          25  <-- kept going, and the LAST match won
------------------------------------------------------------

looking for a product of 99 in a 5x5 table

          approach    result  iterations  note
------------------------------------------------------------
            RETURN       []          25  stops immediately
  control variable       []          25  stops immediately
     helper method       []          25  stops immediately
         QUIT only       []          25  <-- kept going, and the LAST match won
------------------------------------------------------------
```

**Por que cada resultado:**

- **A versão errada devolveu `4 x 3` em vez de `3 x 4`.** Não é um erro de "não achou": ela achou o **primeiro** e depois **continuou**, sobrescrevendo com o último. Numa busca por "o primeiro que satisfaz", isso é a resposta errada — e ela **parece** plausível, que é o que a torna perigosa.
- **A versão errada fez 25 iterações em todos os casos**, contra 14 das corretas. Numa tabela 5×5 isso é irrelevante; numa busca sobre um milhão de registros, é a diferença entre instantâneo e inaceitável.
- **A versão com variável de controle fez 15 iterações, uma a mais que as outras.** A iteração extra é o `quit:found'=""` do laço externo, que é avaliado antes de a próxima linha começar. É o custo do padrão — pequeno, mas real.
- **A versão com método auxiliar fez as mesmas 14 iterações que o `RETURN`**, e é a mais limpa: o laço interno tem nome, propósito e testabilidade próprios. Quando a saída de laços aninhados fica complicada, **extrair é quase sempre a melhor resposta**.
- **No caso sem solução (alvo 99), as quatro fizeram 25 iterações e devolveram vazio.** Todas concordam quando não há nada a encontrar — a divergência só aparece quando há.
- **O contador ficou numa PPG** (`^||FlowCount`) porque precisa sobreviver entre a chamada do método e a leitura do resultado, sem virar variável global compartilhada. É o Capítulo 8 aplicado a um detalhe de instrumentação.

---

### Exercício 17.4 — Indireção e despacho dinâmico

**a) Enunciado:** Crie `LabStudy.Demo.Fl4` com ferramentas genéricas baseadas em indireção:

1. `Dump(referencia)` — despeja qualquer array ou global, em qualquer profundidade, usando `$QUERY` e indireção.
2. `CountNodes(referencia)` — conta nós de primeiro nível.
3. `CopyTree(origem, destino)` — copia uma estrutura para outra, ambas nomeadas por texto.
4. `Apply(referencia, classe, metodo)` — aplica um método a cada valor de primeiro nível, gravando o resultado de volta.
5. `SafeDispatch(classe, metodo, permitidos, arg)` — chama um método **apenas** se ele estiver numa lista permitida; caso contrário, recusa.
6. `XecuteDemo()` — mostra `XECUTE` funcionando e, em seguida, demonstra o risco com uma string maliciosa.

**b) Dica:** No item 5, valide **antes** de chamar. A lista permitida é a defesa.

**c) Como testar:** O item 4 deve funcionar sobre um array local e sobre uma global, sem alteração no código.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Fl4.cls`:

```objectscript
/// Generic tools built on indirection.
Class LabStudy.Demo.Fl4 Extends %RegisteredObject
{

/// Dumps any array or global, at any depth.
ClassMethod Dump(reference As %String) As %Integer
{
    write "-- dump of ", reference, " --", !

    set ref = reference, n = 0
    for {
        set ref = $QUERY(@ref)
        quit:ref=""
        quit:$EXTRACT(ref, 1, $LENGTH(reference)) '= reference

        set n = n + 1
        write "  ", $JUSTIFY($QLENGTH(ref), 2), "  ", ref, " = ", @ref, !
    }

    write "  ", n, " nodes", !
    quit n
}

/// Counts first level nodes.
ClassMethod CountNodes(reference As %String) As %Integer
{
    set n = 0, k = ""
    for {
        set k = $ORDER(@reference@(k))
        quit:k=""
        set n = n + 1
    }
    quit n
}

/// Copies one structure into another, both named by text.
ClassMethod CopyTree(source As %String, target As %String) As %Integer
{
    kill @target
    merge @target = @source
    quit ..CountNodes(target)
}

/// Applies a class method to every first level value.
ClassMethod Apply(reference As %String, class As %String, method As %String) As %Integer
{
    set n = 0, k = ""
    for {
        set k = $ORDER(@reference@(k), 1, v)
        quit:k=""

        set @reference@(k) = $CLASSMETHOD(class, method, v)
        set n = n + 1
    }
    quit n
}

/// Dispatches only to a method that is on the allowed list.
ClassMethod SafeDispatch(class As %String, method As %String, allowed As %List, arg As %String = "") As %String
{
    if $LISTFIND(allowed, method) = 0 {
        write "  REFUSED: '", method, "' is not on the allowed list", !
        quit ""
    }

    if '##class(%Dictionary.CompiledMethod).%ExistsId(class_"||"_method) {
        write "  REFUSED: '", method, "' does not exist in ", class, !
        quit ""
    }

    quit $CLASSMETHOD(class, method, arg)
}

/// XECUTE, and why it must be handled with care.
ClassMethod XecuteDemo() As %Status
{
    write "-- XECUTE doing something useful --", !
    set code = "write ""  hello from a string"", !"
    xecute code

    write !, "-- XECUTE building a computation --", !
    kill ^||XResult
    for expr = "2+3*4", "$LENGTH(""Maria"")", "$ZDATE(+$HOROLOG,3)" {
        set code = "set ^||XResult = "_expr
        xecute code
        write "  ", $JUSTIFY(expr, 24), " -> ", ^||XResult, !
    }

    write !, "-- and here is the danger --", !
    set userInput = "1 kill ^||XResult write ""  I just deleted your data"", !"
    write "  a 'harmless' expression from outside: ", userInput, !
    write "  executing it:", !

    set code = "set ^||XResult = "_userInput
    xecute code

    write "  ^||XResult now: [", $GET(^||XResult), "]", !
    write !, "  never build XECUTE from untrusted input.", !

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    write "=== the same code over a local array and a global ===", !, !

    kill data, ^DemoFl4
    set data("a") = "  joao DA silva  ", data("b") = "  MARIA souza  "
    set ^DemoFl4("x") = "  ana LIMA  ", ^DemoFl4("y") = "  BRUNO prado  "

    do ..Dump("data")
    write !
    do ..Dump("^DemoFl4")

    write !, "=== applying a method to every value ===", !
    do ..Apply("data", "LabStudy.Text", "ProperName")
    do ..Apply("^DemoFl4", "LabStudy.Text", "ProperName")

    do ..Dump("data")
    write !
    do ..Dump("^DemoFl4")

    write !, "=== copying a tree ===", !
    write "  copied ", ..CopyTree("data", "^DemoFl4Backup"), " nodes", !
    do ..Dump("^DemoFl4Backup")

    write !, "=== safe dispatch ===", !
    set allowed = $LISTBUILD("ProperName", "OnlyDigits", "Slug")

    for m = "ProperName", "Slug", "OnlyDigits", "MaskEmail", "NoSuchMethod" {
        write "  ", $JUSTIFY(m, 14), " : "
        set r = ..SafeDispatch("LabStudy.Text", m, allowed, "  joao DA silva 99  ")
        write:r'="" "[", r, "]", !
    }

    write !
    do ..XecuteDemo()

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Fl4).Demo()
=== the same code over a local array and a global ===

-- dump of data --
   1  data("a") =   joao DA silva
   1  data("b") =   MARIA souza
  2 nodes

-- dump of ^DemoFl4 --
   1  ^DemoFl4("x") =   ana LIMA
   1  ^DemoFl4("y") =   BRUNO prado
  2 nodes

=== applying a method to every value ===
-- dump of data --
   1  data("a") = Joao da Silva
   1  data("b") = Maria Souza
  2 nodes

-- dump of ^DemoFl4 --
   1  ^DemoFl4("x") = Ana Lima
   1  ^DemoFl4("y") = Bruno Prado
  2 nodes

=== copying a tree ===
  copied 2 nodes
-- dump of ^DemoFl4Backup --
   1  ^DemoFl4Backup("a") = Joao da Silva
   1  ^DemoFl4Backup("b") = Maria Souza
  2 nodes

=== safe dispatch ===
      ProperName : [Joao Da Silva 99]
            Slug : [joao-da-silva-99]
      OnlyDigits : [99]
      MaskEmail  :   REFUSED: 'MaskEmail' is not on the allowed list
    NoSuchMethod :   REFUSED: 'NoSuchMethod' is not on the allowed list

-- XECUTE doing something useful --
  hello from a string

-- XECUTE building a computation --
                     2+3*4 -> 20
        $LENGTH("Maria") -> 5
     $ZDATE(+$HOROLOG,3) -> 2026-08-19

-- and here is the danger --
  a 'harmless' expression from outside: 1 kill ^||XResult write "  I just deleted your data", !
  executing it:
  I just deleted your data
  ^||XResult now: []

  never build XECUTE from untrusted input.
```

**Por que cada resultado:**

- **O mesmo `Dump` e o mesmo `Apply` funcionaram sobre um array local e sobre uma global.** Nenhuma linha mudou. Isso é o que a indireção compra: **código genérico sobre estruturas escolhidas em execução**. Sem ela, seriam duas versões de cada método.
- **`Dump` verifica se a referência ainda começa com o nome pedido.** Sem esse `quit`, o `$QUERY` continuaria para a **próxima global do namespace** depois de esgotar a atual — despejando dados que ninguém pediu. É um detalhe fácil de esquecer e de consequências constrangedoras.
- **`SafeDispatch` recusou dois métodos por motivos diferentes**: um existe mas não está autorizado, o outro nem existe. As duas verificações são necessárias. A primeira é a de **segurança**; a segunda evita um erro em execução com mensagem obscura.
- **A consulta a `%Dictionary.CompiledMethod`** é o dicionário do Capítulo 11 sendo usado para validar antes de despachar. O identificador tem o formato `Classe||Metodo`.
- **A demonstração final do `XECUTE` é o ponto do exercício.** A "expressão" enviada de fora era `1` seguido de outros comandos. Como o `XECUTE` monta a linha `set ^||XResult = 1 kill ^||XResult write ...`, e o ObjectScript aceita vários comandos na mesma linha (Capítulo 0), **o atacante executou o que quis**. Não houve erro, não houve aviso.
- **Compare com `SafeDispatch`:** ali, o pior que um atacante consegue é ser recusado. **A diferença entre as duas abordagens é a diferença entre um sistema comprometível e um sistema não comprometível** — e o custo de usar a versão segura foi de três linhas.

---

### Exercício 17.5 — PROJETO CONTÍNUO: menu e motor de regras

**a) Enunciado:** Construa duas peças que exercitam tudo do capítulo:

1. `LabStudy.Menu` — um menu interativo de Terminal:
   - `Run()` — laço principal, lê a opção com `READ`, despacha com `$CASE`, sai quando o usuário pedir;
   - opções: listar pacientes, buscar por número de registro, ver exames pendentes, ver o painel, ver rankings, rodar o benchmark, e sair;
   - `Prompt(texto, padrao)` — leitura com valor padrão e limpeza;
   - `Confirm(texto)` — pergunta sim/não com `DO ... WHILE` até obter resposta válida;
   - `Pause()` — espera o usuário pressionar Enter;
   - trate opção inválida sem quebrar o laço.
2. `LabStudy.RuleEngine` — um motor de regras configurável para classificar exames:
   - regras guardadas numa global, cada uma com código do exame, condição e classificação;
   - `AddRule(codigo, condicao, classificacao, prioridade)`;
   - `Evaluate(exam)` — aplica as regras na ordem de prioridade e devolve a primeira que casar;
   - a condição é avaliada com `XECUTE` **de forma controlada**: apenas administradores podem cadastrar regras, e o motor valida a condição antes de aceitá-la;
   - `ListRules()`, `RemoveRule(id)`, `TestRule(condicao, valor)`.
3. Suba `LabStudy.App` para `"1.8"` e faça `Menu()` ser a porta de entrada.

**b) Dica:** No item 2, valide a condição com um padrão que só permita caracteres esperados, e teste-a antes de gravar.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.RuleEngine).Seed()
LABSTUDY>DO ##class(LabStudy.RuleEngine).ListRules()
LABSTUDY>DO ##class(LabStudy.Menu).Run()
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Menu.cls`:

```objectscript
/// Interactive terminal menu for the LabStudy system.
/// Only meaningful from the Terminal: READ blocks in batch contexts.
Class LabStudy.Menu Extends %RegisteredObject
{

Parameter WIDTH = 56;

/// Reads a value, trimmed, with an optional default.
ClassMethod Prompt(text As %String, default As %String = "") As %String
{
    set label = text_$SELECT(default '= "": " ["_default_"]", 1: "")_": "
    read label, answer
    write !

    set answer = ##class(LabStudy.Text).Clean(answer)
    quit $SELECT(answer = "": default, 1: answer)
}

/// Yes/no question. Repeats until the answer is valid.
ClassMethod Confirm(text As %String) As %Boolean
{
    do {
        read text, " (S/N): ", answer
        write !
        set answer = $ZCONVERT(##class(LabStudy.Text).Clean(answer), "U")
        set valid = (answer = "S") || (answer = "N")

        write:'valid "  resposta inválida, digite S ou N", !
    } while 'valid

    quit answer = "S"
}

/// Waits for Enter.
ClassMethod Pause() As %Status
{
    read !, "  -- pressione Enter para continuar --", dummy
    write !
    quit $$$OK
}

/// Draws the menu.
ClassMethod Draw() As %Status
{
    write !
    do ##class(LabStudy.Formatter).Line(..#WIDTH, "=")
    write ##class(LabStudy.Text).Pad("LabStudy "_##class(LabStudy.App).#VERSION, ..#WIDTH, "C"), !
    do ##class(LabStudy.Formatter).Line(..#WIDTH, "=")

    write "  1  Listar pacientes", !
    write "  2  Buscar por numero de registro", !
    write "  3  Exames pendentes", !
    write "  4  Painel", !
    write "  5  Rankings", !
    write "  6  Benchmark", !
    write "  7  Distribuicao por idade", !
    write "  8  Classificar um exame (motor de regras)", !
    write "  0  Sair", !
    do ##class(LabStudy.Formatter).Line(..#WIDTH)

    quit $$$OK
}

/// Main loop.
ClassMethod Run() As %Status
{
    for {
        do ..Draw()

        set option = ..Prompt("Opcao", "0")

        // one dispatch table, one place to change
        set handled = $CASE(option,
            "1": ..OptListPatients(),
            "2": ..OptFindByRecord(),
            "3": ..OptPending(),
            "4": ..OptDashboard(),
            "5": ..OptRankings(),
            "6": ..OptBenchmark(),
            "7": ..OptAgeDistribution(),
            "8": ..OptClassify(),
            "0": "quit",
            : "invalid")

        if handled = "quit" {
            if ..Confirm("  Deseja realmente sair?") {
                write "  Ate logo.", !
                quit
            }
            continue
        }

        if handled = "invalid" {
            write "  Opcao invalida: [", option, "]", !
            do ..Pause()
            continue
        }

        do ..Pause()
    }

    quit $$$OK
}

ClassMethod OptListPatients() As %String
{
    set limit = ..Prompt("Quantos pacientes", "10")
    quit:'$ISVALIDNUM(limit) "invalid"

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT TOP ? RecordNumber, Name, Sex, Address_City AS City "
        _"FROM LabStudy.PATIENT ORDER BY Name", limit)

    set W = $LISTBUILD(14, 24, 4, 16)
    set A = $LISTBUILD("L", "L", "C", "L")
    do ##class(LabStudy.Formatter).Header($LISTBUILD("registro", "nome", "sexo", "cidade"), W, A)

    set n = 0
    while rs.%Next() {
        set n = n + 1
        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(rs.%Get("RecordNumber"), rs.%Get("Name"),
                       rs.%Get("Sex"), rs.%Get("City")), W, A)
    }
    write "  ", n, " pacientes", !
    quit "ok"
}

ClassMethod OptFindByRecord() As %String
{
    set rec = $ZCONVERT(..Prompt("Numero de registro"), "U")
    quit:rec="" "ok"

    set patient = ##class(LabStudy.Patient).FindByRecord(rec)

    if '$ISOBJECT(patient) {
        write "  Nao encontrado: ", rec, !
        quit "ok"
    }

    do ##class(LabStudy.Patient).Show(patient.%Id())
    quit "ok"
}

ClassMethod OptPending() As %String
{
    set hours = ..Prompt("Pendentes ha mais de quantas horas", "24")
    quit:'$ISVALIDNUM(hours) "invalid"

    do ##class(LabStudy.Reports).OverdueExams(hours)
    quit "ok"
}

ClassMethod OptDashboard() As %String
{
    do ##class(LabStudy.Reports).Dashboard()
    quit "ok"
}

ClassMethod OptRankings() As %String
{
    do ##class(LabStudy.App).Rankings()
    quit "ok"
}

ClassMethod OptBenchmark() As %String
{
    do ##class(LabStudy.Reports).Benchmark()
    quit "ok"
}

ClassMethod OptAgeDistribution() As %String
{
    do ##class(LabStudy.Reports).AgeDistribution()
    quit "ok"
}

ClassMethod OptClassify() As %String
{
    set code = $ZCONVERT(..Prompt("Codigo do exame", "GLU"), "U")
    set value = ..Prompt("Valor")

    quit:'$ISVALIDNUM(value) "invalid"

    set result = ##class(LabStudy.RuleEngine).Classify(code, value)
    write "  ", code, " = ", value, " -> ", result, !
    quit "ok"
}

}
```

`src/LabStudy/RuleEngine.cls`:

```objectscript
/// Configurable rule engine for classifying exam results.
///
/// Conditions are ObjectScript expressions evaluated with XECUTE.
/// That is powerful and dangerous: every condition is validated against a
/// restrictive pattern before being stored, and only administrators may
/// register rules. Never expose AddRule to untrusted input.
Class LabStudy.RuleEngine Extends %RegisteredObject
{

Parameter RULEGLOBAL = "^LabStudyRules";

/// Characters a condition may contain. Anything else is refused.
/// Allowed: digits, letters, spaces and the operators below.
Parameter ALLOWEDCHARS = "0123456789.<>=()&|'+-*/ V";

/// Validates a condition before it is ever stored or executed.
/// The variable V holds the value being classified.
ClassMethod ValidateCondition(condition As %String, Output reason As %String) As %Boolean
{
    set reason = ""

    if condition = "" {
        set reason = "condicao vazia"
        quit 0
    }

    if $LENGTH(condition) > 100 {
        set reason = "condicao longa demais"
        quit 0
    }

    // every character must be on the allowed list
    for i = 1:1:$LENGTH(condition) {
        set ch = $EXTRACT(condition, i)
        if '(..#ALLOWEDCHARS [ ch) {
            set reason = "caractere nao permitido na posicao "_i_": ["_ch_"]"
            quit
        }
    }
    quit:reason'="" 0

    // it must actually evaluate without error
    set ok = ..TestCondition(condition, 50, .reason)
    quit ok
}

/// Tries the condition against a sample value.
ClassMethod TestCondition(condition As %String, sample As %Numeric, Output reason As %String) As %Boolean
{
    set reason = ""
    kill ^||RuleTest

    try {
        new V
        set V = sample
        xecute "set ^||RuleTest = ("_condition_")"
    } catch e {
        set reason = "condicao nao avaliavel: "_e.DisplayString()
        quit 0
    }

    if '$DATA(^||RuleTest) {
        set reason = "condicao nao produziu resultado"
        quit 0
    }

    quit 1
}

/// Registers a rule. Refuses anything that does not validate.
ClassMethod AddRule(code As %String, condition As %String, classification As %String, priority As %Integer = 100) As %Integer
{
    set code = $ZCONVERT(##class(LabStudy.Text).Clean(code), "U")

    if code = "" {
        write "  codigo obrigatorio", !
        quit 0
    }

    if 'code?1.AN {
        write "  codigo deve ser alfanumerico: [", code, "]", !
        quit 0
    }

    if '..ValidateCondition(condition, .reason) {
        write "  condicao recusada: ", reason, !
        quit 0
    }

    set id = $INCREMENT(@..#RULEGLOBAL)

    set @..#RULEGLOBAL@(id, "code") = code
    set @..#RULEGLOBAL@(id, "condition") = condition
    set @..#RULEGLOBAL@(id, "classification") = classification
    set @..#RULEGLOBAL@(id, "priority") = priority
    set @..#RULEGLOBAL@(id, "createdBy") = $USERNAME
    set @..#RULEGLOBAL@(id, "createdOn") = ##class(LabStudy.DateTime).NowTimestamp()

    // index by code and priority, so evaluation walks in the right order
    set @..#RULEGLOBAL@("byCode", code, priority, id) = ""

    quit id
}

/// Removes a rule.
ClassMethod RemoveRule(id As %Integer) As %Boolean
{
    quit:'$DATA(@..#RULEGLOBAL@(id)) 0

    set code = $GET(@..#RULEGLOBAL@(id, "code"))
    set priority = $GET(@..#RULEGLOBAL@(id, "priority"))

    kill @..#RULEGLOBAL@("byCode", code, priority, id)
    kill @..#RULEGLOBAL@(id)
    quit 1
}

/// Classifies a value against the rules of one code.
/// Returns the classification of the first matching rule, by priority.
ClassMethod Classify(code As %String, value As %Numeric) As %String
{
    set code = $ZCONVERT(code, "U")
    quit:value="" "(sem valor)"
    quit:'$ISVALIDNUM(value) "(valor invalido)"

    set priority = ""
    for {
        set priority = $ORDER(@..#RULEGLOBAL@("byCode", code, priority))
        quit:priority=""

        set id = ""
        for {
            set id = $ORDER(@..#RULEGLOBAL@("byCode", code, priority, id))
            quit:id=""

            set condition = $GET(@..#RULEGLOBAL@(id, "condition"))
            continue:condition=""

            kill ^||RuleResult
            try {
                new V
                set V = value
                xecute "set ^||RuleResult = ("_condition_")"
            } catch {
                continue
            }

            if $GET(^||RuleResult) {
                return $GET(@..#RULEGLOBAL@(id, "classification"), "(sem classificacao)")
            }
        }
    }

    quit "(nenhuma regra aplicavel)"
}

/// Lists every rule, grouped by code.
ClassMethod ListRules() As %Integer
{
    write "=== regras cadastradas ===", !

    set W = $LISTBUILD(5, 8, 6, 24, 18)
    set A = $LISTBUILD("R", "L", "R", "L", "L")
    do ##class(LabStudy.Formatter).Header(
        $LISTBUILD("id", "codigo", "prio", "condicao", "classificacao"), W, A)

    set n = 0, id = ""
    for {
        set id = $ORDER(@..#RULEGLOBAL@(id))
        quit:id=""
        continue:id="byCode"
        continue:'$DATA(@..#RULEGLOBAL@(id, "code"))

        set n = n + 1
        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(id,
                       $GET(@..#RULEGLOBAL@(id, "code")),
                       $GET(@..#RULEGLOBAL@(id, "priority")),
                       $GET(@..#RULEGLOBAL@(id, "condition")),
                       $GET(@..#RULEGLOBAL@(id, "classification"))),
            W, A)
    }

    do ##class(LabStudy.Formatter).Line(64)
    write "  ", n, " regras", !
    quit n
}

/// Loads a starting set of rules.
ClassMethod Seed() As %Integer
{
    kill @..#RULEGLOBAL

    set n = 0
    set n = n + (..AddRule("GLU", "V<50", "critico baixo", 10) > 0)
    set n = n + (..AddRule("GLU", "V<70", "baixo", 20) > 0)
    set n = n + (..AddRule("GLU", "V>200", "critico alto", 10) > 0)
    set n = n + (..AddRule("GLU", "V>99", "alto", 20) > 0)
    set n = n + (..AddRule("GLU", "1", "normal", 90) > 0)

    set n = n + (..AddRule("HGB", "V<8", "critico baixo", 10) > 0)
    set n = n + (..AddRule("HGB", "V<12", "baixo", 20) > 0)
    set n = n + (..AddRule("HGB", "V>18", "alto", 20) > 0)
    set n = n + (..AddRule("HGB", "1", "normal", 90) > 0)

    write "  ", n, " regras carregadas", !

    write !, "-- tentativas que devem ser recusadas --", !
    do ..AddRule("GLU", "V>10 kill ^Config", "malicioso", 1)
    do ..AddRule("GLU", "$CLASSMETHOD(""x"",""y"")", "malicioso", 1)
    do ..AddRule("GLU", "", "vazio", 1)
    do ..AddRule("GLU", "V >>> 10", "sintaxe ruim", 1)

    quit n
}

/// Quick check of the engine against a range of values.
ClassMethod Check(code As %String = "GLU") As %Status
{
    write "=== classificacao para ", code, " ===", !

    for v = 30, 60, 85, 150, 250 {
        write "  ", $JUSTIFY(v, 6), " -> ", ..Classify(code, v), !
    }
    quit $$$OK
}

}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.8";

/// Opens the interactive menu.
ClassMethod Menu() As %Status [ CodeMode = expression ]
{
##class(LabStudy.Menu).Run()
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.RuleEngine).Seed()
  9 regras carregadas

-- tentativas que devem ser recusadas --
  condicao recusada: caractere nao permitido na posicao 7: [k]
  condicao recusada: caractere nao permitido na posicao 1: [$]
  condicao recusada: condicao vazia
  condicao recusada: condicao nao avaliavel: <SYNTAX>...

LABSTUDY>DO ##class(LabStudy.RuleEngine).ListRules()
=== regras cadastradas ===
   id  codigo   prio  condicao                 classificacao
----------------------------------------------------------------
    1  GLU        10  V<50                     critico baixo
    2  GLU        20  V<70                     baixo
    3  GLU        10  V>200                    critico alto
    4  GLU        20  V>99                     alto
    5  GLU        90  1                        normal
    6  HGB        10  V<8                      critico baixo
    7  HGB        20  V<12                     baixo
    8  HGB        20  V>18                     alto
    9  HGB        90  1                        normal
----------------------------------------------------------------
  9 regras

LABSTUDY>DO ##class(LabStudy.RuleEngine).Check("GLU")
=== classificacao para GLU ===
      30 -> critico baixo
      60 -> baixo
      85 -> normal
     150 -> alto
     250 -> critico alto

LABSTUDY>DO ##class(LabStudy.App).Menu()

========================================================
                  LabStudy 1.8
========================================================
  1  Listar pacientes
  2  Buscar por numero de registro
  ...
  0  Sair
--------------------------------------------------------
Opcao [0]: 8

Codigo do exame [GLU]: HGB

Valor: 7

  HGB = 7 -> critico baixo

  -- pressione Enter para continuar --
```

**Por que cada decisão:**

- **O menu usa `$CASE` como tabela de despacho.** Acrescentar uma opção é acrescentar uma linha ali e um método `Opt...`. Uma cadeia de `if`/`elseif` cresceria linearmente em ruído; a tabela cresce em uma linha por opção.
- **Cada opção é um método próprio.** O laço principal fica com 30 linhas legíveis, e cada funcionalidade é testável isoladamente, sem passar pelo menu.
- **O laço principal só termina por confirmação.** `Confirm` usa `DO ... WHILE` porque a pergunta precisa ser feita **pelo menos uma vez** — o caso de manual do `DO ... WHILE`.
- **Opção inválida usa `continue`, não `quit`.** Um erro de digitação não deve encerrar a sessão do usuário. Essa é a diferença entre um menu utilizável e um frustrante.
- **`Prompt` devolve o padrão quando o usuário só aperta Enter.** É a cortesia que faz um menu de Terminal ser tolerável.
- **O motor de regras é o caso legítimo de `XECUTE`** — e é acompanhado de três camadas de defesa: lista de caracteres permitidos, limite de comprimento, e teste de avaliação antes de gravar. As quatro tentativas maliciosas ou malformadas do `Seed` foram todas recusadas, cada uma por um motivo diferente.
- **A validação por lista de caracteres permitidos é mais segura do que uma lista de proibidos.** Proibir `kill` deixaria passar `zkill`; proibir `$` deixaria passar `##class`. **Permitir apenas o necessário** é a única abordagem que não depende de o desenvolvedor ter previsto todos os ataques.
- **`new V` dentro do `try`** isola a variável usada pela condição, para que ela não vaze nem colida com nada do chamador — o Capítulo 7 aplicado a variáveis em vez de papéis.
- **As regras são indexadas por `(código, prioridade, id)`** na global. Percorrer essa estrutura já entrega as regras na ordem de prioridade, **sem ordenar nada** — o idioma do Capítulo 13, aplicado a uma configuração.
- **A regra de prioridade 90 com condição `"1"`** é o padrão: sempre verdadeira, sempre a última a ser avaliada. É a versão em dados do ramo `1:` do `$SELECT`.
- **`Classify` usa `continue` quando uma condição falha ao avaliar**, em vez de abortar. Uma regra defeituosa não deve impedir as outras de funcionarem — e o `try`/`catch` garante isso.
- **Uma limitação honesta:** este motor avalia expressões numéricas simples sobre uma variável `V`. Ele **não** é um substituto para uma linguagem de regras completa, e ampliá-lo exigiria repensar a validação. Reconhecer o limite do que se construiu é parte de construir bem.

---

## 9. Quiz do capítulo

**Q1.** Qual é a forma correta de escrever um `IF` de bloco?

- A) A chave de abertura numa linha própria.
- B) A chave de abertura na mesma linha da condição.
- C) Sem chaves, sempre.
- D) Com `then`.

---

**Q2.** O que significa `quit:total=0 ""`?

- A) Sempre encerra devolvendo vazio.
- B) Encerra devolvendo vazio **apenas se** `total` for zero.
- C) Compara `total` com vazio.
- D) Erro de sintaxe.

---

**Q3.** Onde o pós-condicional **não** pode ser usado?

- A) Em `SET`.
- B) Em `WRITE`.
- C) Em funções como `$SELECT`.
- D) Em `QUIT`.

---

**Q4.** Por que a forma antiga de `IF`/`ELSE` é arriscada?

- A) É mais lenta.
- B) O `ELSE` depende de `$TEST`, que também é alterado por `LOCK`, `READ` e `OPEN` com tempo limite.
- C) Não compila em versões novas.
- D) Não aceita mais de uma condição.

---

**Q5.** Qual construção usar para escolher **um valor** comparando **uma variável** contra várias alternativas?

- A) `IF` de bloco.
- B) `$SELECT`.
- C) `$CASE`.
- D) `FOR`.

---

**Q6.** O que `$CASE(x, x>10: "alto", : "baixo")` faz?

- A) Devolve `"alto"` quando `x` é maior que 10.
- B) Compara `x` com o **resultado** de `x>10`, que é 1 ou 0 — quase certamente não é o que se queria.
- C) Erro de sintaxe.
- D) O mesmo que `$SELECT`.

---

**Q7.** Qual forma de `FOR` percorre uma lista fixa de valores?

- A) `for i = 1:1:10 { }`
- B) `for i = "a", "b", "c" { }`
- C) `for { }`
- D) `for i = 1:1 { }`

---

**Q8.** Qual é a diferença entre `WHILE` e `DO ... WHILE`?

- A) Nenhuma.
- B) `WHILE` testa antes (pode não executar nunca); `DO ... WHILE` testa depois (executa pelo menos uma vez).
- C) `DO ... WHILE` é mais rápido.
- D) `WHILE` só aceita condições numéricas.

---

**Q9.** Num laço aninhado, o que `QUIT` faz?

- A) Sai dos dois laços.
- B) Sai apenas do laço mais interno.
- C) Sai do método.
- D) Pula para a próxima iteração.

---

**Q10.** Como sair de dois laços aninhados de uma vez?

- A) Com dois `QUIT` seguidos.
- B) Com `RETURN`, uma variável de controle, ou extraindo o laço interno para um método.
- C) Com `CONTINUE`.
- D) Com `GOTO`.

---

**Q11.** O que `CONTINUE` faz?

- A) Sai do laço.
- B) Vai para a próxima iteração do laço atual.
- C) Sai do método.
- D) Reinicia o laço.

---

**Q12.** Depois de `for i = 1:1:5 { }` terminar naturalmente, quanto vale `i`?

- A) `5`
- B) `6`
- C) Fica indefinida.
- D) `1`

---

**Q13.** O que faz o operador `@`?

- A) Concatena textos.
- B) Indireção: usa aquilo cujo **nome** está guardado na variável.
- C) Compara valores.
- D) Marca um comentário.

---

**Q14.** O que `@ref@("x")` significa?

- A) Concatenação de duas variáveis.
- B) A referência cujo nome está em `ref`, com o subscrito `"x"`.
- C) Um erro de sintaxe.
- D) O valor de `ref` seguido de `"x"`.

---

**Q15.** Qual é o principal risco do `XECUTE`?

- A) É lento.
- B) Executa qualquer código contido na string — injeção de código, se a string vier de fora.
- C) Não funciona com globais.
- D) Só aceita uma linha.

---

**Q16.** Qual é a alternativa mais segura ao `XECUTE` para chamar um método cujo nome é dinâmico?

- A) `GOTO`
- B) `$CLASSMETHOD` ou `$METHOD`, com validação do nome contra uma lista permitida.
- C) `$SELECT`
- D) Indireção por argumento.

---

**Q17.** Depois de um `READ` com tempo limite, como saber se o usuário respondeu?

- A) Verificando se a variável está vazia.
- B) Conferindo `$TEST`.
- C) Conferindo `$ZERROR`.
- D) Não é possível.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** a chave de abertura fica na mesma linha da condição; em linha própria não compila.
- **A está errada:** é justamente o erro comum de quem vem de outras convenções.
- **C está errada:** a forma sem chaves existe, mas é a antiga e não é recomendada.
- **D está errada:** o ObjectScript não usa `then`.

**Q2 — Resposta: B.**
- **B está certa:** os dois-pontos após o comando formam um pós-condicional.
- **A está errada:** o `quit` só executa se a condição for verdadeira.
- **C está errada:** a comparação é entre `total` e `0`.
- **D está errada:** a sintaxe é válida e idiomática.

**Q3 — Resposta: C.**
- **C está certa:** pós-condicional é construção de **comando**; funções não o aceitam.
- **A, B e D estão erradas:** os três são comandos e aceitam.

**Q4 — Resposta: B.**
- **B está certa:** o `ELSE` decide com base em `$TEST`, que pode ter sido alterado por outro comando entre o `IF` e o `ELSE`.
- **A está errada:** desempenho não é a questão.
- **C está errada:** ainda compila.
- **D está errada:** aceita condições separadas por vírgula.

**Q5 — Resposta: C.**
- **C está certa:** `$CASE` compara um valor contra alternativas.
- **A está errada:** `IF` serve para executar ações diferentes.
- **B está errada:** `$SELECT` avalia condições distintas entre si.
- **D está errada:** `FOR` é repetição.

**Q6 — Resposta: B.**
- **B está certa:** `$CASE` compara valores, não avalia condições. `x>10` produz 1 ou 0, e é isso que será comparado com `x`.
- **A está errada:** é a intenção provável, mas não o comportamento.
- **C está errada:** compila normalmente — o que torna o erro silencioso.
- **D está errada:** `$SELECT` é que avalia condições.

**Q7 — Resposta: B.**
- **B está certa:** valores separados por vírgula formam a lista.
- **A está errada:** é a forma contada.
- **C está errada:** é a forma infinita pura.
- **D está errada:** é a forma infinita ascendente.

**Q8 — Resposta: B.**
- **B está certa:** a posição do teste determina se o corpo pode nunca executar.
- **A está errada:** a diferença aparece exatamente quando a condição já é falsa.
- **C está errada:** desempenho não é o critério.
- **D está errada:** ambos aceitam qualquer expressão lógica.

**Q9 — Resposta: B.**
- **B está certa:** `QUIT` encerra o bloco atual, que é o laço mais interno.
- **A está errada:** para os dois, é preciso outra técnica.
- **C está errada:** sair do método é `RETURN`.
- **D está errada:** a próxima iteração é `CONTINUE`.

**Q10 — Resposta: B.**
- **B está certa:** as três abordagens funcionam; extrair para um método costuma ser a mais limpa.
- **A está errada:** o segundo `QUIT` estaria fora do laço interno e não seria alcançado como se espera.
- **C está errada:** `CONTINUE` não sai de nada.
- **D está errada:** funciona, mas é código que não se escreve mais.

**Q11 — Resposta: B.**
- **B está certa:** pula o restante do corpo e vai para a próxima iteração.
- **A, C e D estão erradas:** descrevem `QUIT`, `RETURN` e nada, respectivamente.

**Q12 — Resposta: B.**
- **B está certa:** no fim natural, a variável fica com o valor que ultrapassou o limite.
- **A está errada:** seria o valor se o laço tivesse terminado por `QUIT` na última iteração.
- **C está errada:** ela continua definida.
- **D está errada:** não volta ao início.

**Q13 — Resposta: B.**
- **B está certa:** `@` é o operador de indireção.
- **A está errada:** concatenação é `_`.
- **C e D estão erradas:** não são funções do `@`.

**Q14 — Resposta: B.**
- **B está certa:** o segundo `@` acrescenta subscritos à referência montada.
- **A está errada:** não há concatenação envolvida.
- **C está errada:** é sintaxe válida e muito usada.
- **D está errada:** o resultado é uma referência, não um texto.

**Q15 — Resposta: B.**
- **B está certa:** `XECUTE` executa qualquer coisa na string, incluindo comandos que o autor não previu.
- **A está certa quanto ao desempenho, mas não é o **principal** risco.
- **C está errada:** funciona com globais normalmente.
- **D está errada:** aceita várias operações.

**Q16 — Resposta: B.**
- **B está certa:** o despacho por nome de método é validado contra a classe real, e a lista permitida limita o que pode ser chamado.
- **A está errada:** `GOTO` não tem relação.
- **C está errada:** `$SELECT` escolhe valores.
- **D está errada:** indireção por argumento tem os mesmos riscos e menos legibilidade.

**Q17 — Resposta: B.**
- **B está certa:** `$TEST` vale 1 se houve entrada e 0 se o tempo esgotou.
- **A está errada:** o usuário pode ter apertado Enter sem digitar nada, o que também deixa a variável vazia.
- **C está errada:** `$ZERROR` guarda erros.
- **D está errada:** é exatamente para isso que `$TEST` serve aqui.

---

## 10. Resumo relâmpago

1. **`IF` decide o que FAZER; `$SELECT` e `$CASE` decidem o que VALE.**
2. Na forma de bloco, **a chave de abertura fica na mesma linha** da condição. `elseif` é uma palavra só.
3. **Pós-condicional**: `comando:condição`. Excelente para **guardas** curtas. Funciona em comandos, **não em funções**.
4. A forma antiga de `IF`/`ELSE` depende de **`$TEST`**, que também é alterado por `LOCK`, `READ` e `OPEN` com tempo limite. Use blocos.
5. **`$SELECT`** avalia condições em ordem, tem curto-circuito, e exige o ramo **`1:`**.
6. **`$CASE`** compara **um valor** contra alternativas, e exige o ramo padrão (`: valor`). **Não escreva condições nele.**
7. Formas do `FOR`: contado (`1:1:10`), passo negativo (`10:-1:1`), infinito ascendente (`1:1`), infinito puro (`for { }`), por lista (`"a","b","c"`) e faixas combinadas.
8. **Não confie na variável de controle depois do `FOR`**: vale um a mais no fim natural, e o valor da parada se houve `QUIT`.
9. **`WHILE` testa antes** (pode nunca executar); **`DO ... WHILE` testa depois** (executa pelo menos uma vez).
10. **`QUIT`** sai do bloco atual; **`RETURN`** sai do método; **`CONTINUE`** vai para a próxima iteração.
11. **Em laços aninhados, `QUIT` sai só do mais interno.** Para sair dos dois: `RETURN`, variável de controle, ou **extrair o laço interno para um método** — geralmente a melhor opção.
12. **`GOTO`** existe em código legado. Reconheça; não escreva.
13. **Indireção `@`** usa aquilo cujo **nome** está na variável. `@ref@(sub)` acrescenta subscritos.
14. Indireção permite **código genérico sobre qualquer estrutura** — e custa verificação do compilador e desempenho. Use só quando o nome for genuinamente dinâmico.
15. **`XECUTE`** executa texto como código. É a construção mais perigosa da linguagem: **nunca a monte com entrada não confiável**.
16. Prefira **`$CLASSMETHOD`**, **`$METHOD`** e **`$PROPERTY`** ao `XECUTE`, sempre validando o nome contra uma **lista de permitidos** (nunca de proibidos).
17. **`READ`** só faz sentido em contexto interativo. Com tempo limite (`:n`), confira **`$TEST`**.
18. Tabela de despacho com `$CASE` mais um método por opção é o padrão para menus e roteadores.

---

## 11. Cartões de memorização

**Frente:** Quando usar `IF`, `$SELECT` e `$CASE`?
**Verso:** `IF` para executar ações diferentes; `$SELECT` para escolher um valor por condições distintas; `$CASE` para escolher um valor comparando **uma** variável contra alternativas.

**Frente:** O que significa `quit:cond valor`?
**Verso:** Pós-condicional: executa o `quit` **apenas se** a condição for verdadeira.

**Frente:** Onde o pós-condicional não funciona?
**Verso:** Em funções (`$SELECT`, `$LIST`) e nos comandos de bloco `IF`, `ELSE`, `FOR`, `WHILE`.

**Frente:** Por que evitar a forma antiga de `IF`/`ELSE`?
**Verso:** O `ELSE` depende de `$TEST`, que também é alterado por `LOCK`, `READ` e `OPEN` com tempo limite.

**Frente:** O que acontece num `$SELECT` sem condição verdadeira?
**Verso:** `<ILLEGAL VALUE>`. Sempre inclua o ramo `1:`.

**Frente:** Por que `$CASE(x, x>10: "alto", ...)` está errado?
**Verso:** `$CASE` compara **valores**. Ele compararia `x` com o resultado de `x>10`, que é 1 ou 0.

**Frente:** Quais são as formas do `FOR`?
**Verso:** Contado (`1:1:10`), passo negativo, infinito ascendente (`1:1`), infinito puro (`for { }`), por lista de valores, e faixas combinadas por vírgula.

**Frente:** Quanto vale a variável de controle depois do `FOR`?
**Verso:** Um a mais que o limite, se terminou naturalmente; o valor da parada, se houve `QUIT`. **Não confie nela.**

**Frente:** Diferença entre `WHILE` e `DO ... WHILE`.
**Verso:** `WHILE` testa antes e pode nunca executar. `DO ... WHILE` testa depois e executa pelo menos uma vez.

**Frente:** O que `QUIT` faz num laço aninhado?
**Verso:** Sai **apenas** do laço mais interno.

**Frente:** Como sair de dois laços aninhados?
**Verso:** `RETURN` (se puder sair do método), uma variável de controle, ou extrair o laço interno para um método próprio.

**Frente:** O que `CONTINUE` faz?
**Verso:** Pula o restante do corpo e vai para a próxima iteração do laço atual.

**Frente:** O que faz o operador `@`?
**Verso:** Indireção: usa aquilo cujo **nome** está guardado na variável.

**Frente:** Como percorrer uma global cujo nome está numa variável?
**Verso:** `set k = $ORDER(@nome@(k))` — o segundo `@` acrescenta os subscritos.

**Frente:** Qual o principal risco do `XECUTE`?
**Verso:** Executa qualquer código contido na string. Montá-lo com entrada não confiável é injeção de código.

**Frente:** Qual a alternativa segura ao `XECUTE`?
**Verso:** `$CLASSMETHOD` / `$METHOD` / `$PROPERTY`, com o nome validado contra uma **lista de permitidos**.

**Frente:** Por que lista de permitidos e não de proibidos?
**Verso:** Porque uma lista de proibidos depende de você ter previsto todos os ataques; a de permitidos só depende de você conhecer o que é necessário.

**Frente:** Como saber se um `READ` com tempo limite recebeu resposta?
**Verso:** Conferindo `$TEST` imediatamente depois: 1 se houve entrada, 0 se o tempo esgotou.

**Frente:** Qual o padrão para um menu de Terminal?
**Verso:** Laço infinito, `READ` da opção, `$CASE` como tabela de despacho, um método por opção, e `continue` para opção inválida.

**Frente:** E o `GOTO`?
**Verso:** Existe, aparece em código legado, e não deve ser escrito em código novo.

---

Digite CONTINUAR para o próximo capítulo.
