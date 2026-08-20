# Apostila InterSystems ObjectScript Specialist
## Capítulo 22 — T5.3 Recognizes and Resolves Common Errors (Reconhecendo e resolvendo erros comuns)

> Último tópico do domínio **T5** e último capítulo da apostila. Aqui você monta o catálogo dos erros que realmente aparecem, aprende a passar do sintoma à causa em segundos, e fecha o projeto **LabStudy** na versão **3.0**. No fim, há um roteiro de revisão para a prova.

---

## 1. O que você vai saber fazer ao terminar

1. Distinguir erro de **compilação** de erro de **execução**.
2. Reconhecer os **erros de execução** mais comuns pelo nome e ir direto à causa.
3. Interpretar **`SQLCODE`** e as mensagens de **`%Status`**.
4. Aplicar uma **árvore de decisão** que vai do sintoma à causa.
5. Corrigir cada erro **e prevenir** que ele volte.
6. Reconhecer erros que **não** geram mensagem — os silenciosos.
7. Fechar o projeto na versão **3.0**, com um catálogo de erros executável.
8. Revisar toda a apostila com um roteiro por domínio.

---

## 2. O conceito em linguagem de gente

### 2.1 Compilação e execução: dois momentos, dois tipos de erro

**Erro de compilação** acontece quando o IRIS transforma a sua classe em código executável. Ele impede a compilação e você o vê imediatamente.

Causas típicas: sintaxe errada, chave de bloco em linha própria, classe ou propriedade referenciada que não existe, tipo de dado inválido.

**Erro de execução** acontece com o código já compilado, quando ele roda com dados reais. Ele só aparece **quando aquele caminho é percorrido**.

Causas típicas: variável não definida, objeto vazio, divisão por zero, valor fora de faixa.

A diferença prática é enorme:

> **O erro de compilação é seu amigo:** ele aparece na hora, aponta a linha, e ninguém sofre.
> **O erro de execução é seu problema:** ele pode ficar escondido meses num caminho raro, até acontecer com um cliente.

E daí decorre uma orientação de projeto que atravessou toda esta apostila: **empurre erros para a compilação sempre que possível**. É por isso que `..Metodo()` é melhor que `XECUTE`, que propriedades tipadas são melhores que arrays soltos, e que despacho dinâmico deve ser exceção.

### 2.2 A anatomia de uma mensagem de erro

```
<UNDEFINED>zProcessar+12^LabStudy.Import.1 *codigo
```

Quatro informações, todas úteis:

| Parte | Significa |
|---|---|
| `<UNDEFINED>` | **que tipo** de erro |
| `zProcessar` | em **qual método** (o `z` é prefixo do compilador) |
| `+12` | **quantas linhas** depois do início do método |
| `^LabStudy.Import.1` | em **qual rotina** gerada, ou seja, de qual classe |
| `*codigo` | **qual elemento** causou (quando o erro traz essa informação) |

O asterisco no fim é ouro: em `<UNDEFINED>`, ele diz **qual variável** faltava. Em `<PROPERTY DOES NOT EXIST>`, qual propriedade. **Leia sempre até o fim da mensagem.**

### 2.3 Do sintoma à causa: a árvore de decisão

Diante de um problema, três perguntas resolvem a maior parte dos casos:

**Pergunta 1: houve mensagem de erro?**

- **Sim** → vá ao catálogo da seção 3. O nome do erro já aponta a família de causas.
- **Não, mas o resultado está errado** → é um **erro silencioso**. Vá à seção 5.
- **Não, mas está lento** → é desempenho, Capítulo 12.
- **Não, e nada acontece** → é travamento: veja processos e travas, Capítulos 5 e 20.

**Pergunta 2: quando começou?**

- **Sempre foi assim** → provavelmente lógica ou modelagem.
- **Começou depois de uma mudança** → compare com o que mudou. Se houve mudança de esquema, Capítulo 11.
- **Acontece só com alguns dados** → é dado, não código. Vá ver **quais** dados.
- **Acontece só sob carga** → concorrência ou desempenho, Capítulos 5 e 12.

**Pergunta 3: acontece sempre com o mesmo dado?**

- **Sim** → você tem um caso reproduzível. Metade do trabalho está feita.
- **Não, é aleatório** → concorrência, ordem de execução, ou dependência de horário.

Essas três perguntas custam trinta segundos e economizam horas.

---

## 3. O catálogo dos erros de execução

Cada entrada segue o mesmo formato: **o que significa**, **causas típicas**, **como confirmar**, **como corrigir**, **como prevenir**.

### 3.1 `<UNDEFINED>` — variável não existe

**Significa:** você leu uma variável, propriedade ou nó que nunca recebeu valor.

**Causas típicas:**
- Ler um nó de array ou global que não existe.
- Erro de digitação no nome da variável.
- Variável definida só dentro de um `if` que não executou.
- Parâmetro `Output` que o método chamado não preencheu.
- Valor devolvido por `$LISTNEXT` num elemento sem valor.

**Como confirmar:** a mensagem traz o nome após o asterisco. `ZWRITE` mostra o que existe.

**Como corrigir:** use **`$GET`** ou teste com **`$DATA`**.

**Como prevenir:** em código que lê arrays e globais, `$GET` como padrão. Inicialize variáveis no início do método.

### 3.2 `<INVALID OREF>` — referência de objeto inválida

**Significa:** você acessou uma propriedade ou método de algo que não é um objeto.

**Causas típicas:**
- `%OpenId()` devolveu vazio porque o ID não existe.
- Propriedade de relacionamento não preenchida.
- Objeto de uma cadeia vazia: `exam.Patient.Address.City` com `Patient` vazio.
- Método que devolveu `""` em vez de objeto.

**Como confirmar:** `$ISOBJECT(x)` devolve `0`.

**Como corrigir:** teste antes de usar.

```objectscript
set p = ##class(LabStudy.Patient).%OpenId(id)
quit:'$ISOBJECT(p) $$$ERROR($$$GeneralError, "paciente nao encontrado: "_id)
```

**Como prevenir:** **sempre** verifique o retorno de `%OpenId`. Em cadeias, use `&&` com curto-circuito (Capítulo 16):

```objectscript
if $ISOBJECT(exam.Patient) && (exam.Patient.Address.City '= "") { ... }
```

### 3.3 `<SUBSCRIPT>` — subscrito inválido

**Significa:** um subscrito é longo demais, a referência tem subscritos demais, ou a referência está malformada.

**Causas típicas:**
- Usar um texto muito longo como subscrito.
- Concatenar valores para formar um subscrito e um deles vir enorme.
- Referência montada por indireção com formato errado.

**Como corrigir:** limite o tamanho do subscrito — por exemplo, com `$EXTRACT(valor, 1, 200)`, ou use um hash do valor como chave.

**Como prevenir:** nunca use texto livre de tamanho indeterminado como subscrito.

### 3.4 `<DIVIDE>` — divisão por zero

**Significa:** divisão ou módulo por zero.

**Causa quase sempre a mesma:** um contador que ficou em zero porque o laço não encontrou nada.

**Como corrigir:**

```objectscript
set media = $SELECT(n > 0: total / n, 1: "")
```

**Como prevenir:** toda divisão cujo divisor seja calculado precisa de guarda. E lembre do Capítulo 10: quando não há dados, o resultado é **vazio**, não zero.

### 3.5 `<ILLEGAL VALUE>` — valor inaceitável

**Significa:** uma função recebeu um valor que não pode processar.

**Causas típicas:**
- `$SELECT` sem nenhuma condição verdadeira (falta o ramo `1:`).
- `$CASE` sem correspondência e sem ramo padrão.
- `$ZDATEH` com texto que não é data.
- `$ZTIMEH` com hora inválida.

**Como corrigir:** acrescente o ramo padrão; valide antes de converter.

**Como prevenir:** **todo `$SELECT` termina com `1:`** e **todo `$CASE` tem ramo padrão**. Conversões de data e hora sempre dentro de `try`/`catch`.

### 3.6 `<LIST>` — lista inválida ou posição inexistente

**Significa:** você usou `$LIST` numa posição que não existe, ou passou algo que não é uma lista.

**Como corrigir:** use **`$LISTGET`**, que devolve vazio em vez de erro. Valide a estrutura com `$LISTVALID`.

**Como prevenir:** `$LISTGET` como padrão fora de laços controlados; `$LISTVALID` em dados vindos de fora.

Lembre da assimetria do Capítulo 15: **`$EXTRACT` fora de faixa devolve vazio; `$LIST` gera erro.**

### 3.7 `<MAXSTRING>` — string longa demais

**Significa:** uma string ultrapassou o limite máximo.

**Causas típicas:**
- Concatenar num laço com muitas iterações.
- Ler um arquivo inteiro para uma variável.
- Montar um relatório inteiro numa string.

**Como corrigir:** use **stream** (Capítulo 4).

**Como prevenir:** conteúdo de tamanho indeterminado **sempre** em stream. Nunca em string.

### 3.8 `<STORE>` — memória do processo esgotada

**Significa:** o processo consumiu toda a memória disponível.

**Causas típicas:**
- Acumular estruturas grandes em variáveis locais.
- Recursão sem condição de parada.
- Carregar milhares de objetos sem liberá-los.

**Como corrigir:** trocar variável local por **PPG** (`^||`), que não consome memória do processo (Capítulo 8).

**Como prevenir:** processe em **lotes** (Capítulo 12); acompanhe `$STORAGE` em rotinas pesadas.

### 3.9 `<SYNTAX>` — código inválido

**Significa:** código malformado em tempo de execução.

**Causa quase sempre:** **`XECUTE`** com uma string que não é código válido, ou indireção com conteúdo inesperado.

**Como corrigir:** validar a string antes de executar.

**Como prevenir:** evitar `XECUTE`; preferir `$CLASSMETHOD` (Capítulo 17).

### 3.10 `<CLASS DOES NOT EXIST>`, `<METHOD DOES NOT EXIST>`, `<PROPERTY DOES NOT EXIST>`

**Significam:** o nome referenciado não existe ou a classe não está compilada.

**Causas típicas:**
- Erro de digitação num nome usado por indireção ou `$CLASSMETHOD`.
- Classe não compilada, ou compilada com erro.
- Classe removida e ainda referenciada.
- Propriedade renomeada em um lugar e não em outro.

**Como confirmar:**

```objectscript
write ##class(%Dictionary.CompiledClass).%ExistsId("LabStudy.Patient"), !
write ##class(%Dictionary.CompiledMethod).%ExistsId("LabStudy.Patient||Show"), !
```

**Como corrigir:** recompilar o pacote inteiro (`$SYSTEM.OBJ.CompilePackage`).

**Como prevenir:** validar nomes antes de despachar dinamicamente (Capítulo 17); recompilar dependentes após alterar uma superclasse (Capítulo 11).

### 3.11 `<NOROUTINE>` — rotina não existe

**Significa:** você chamou uma rotina inexistente.

**Causas típicas:** classe não compilada; nome errado em `DO ^Rotina`; namespace errado.

**Como corrigir:** confirme o namespace com `$NAMESPACE` e recompile.

### 3.12 `<PROTECT>` — sem permissão

**Significa:** o usuário não tem privilégio sobre o recurso.

**Causas típicas:** falta de privilégio sobre a base de dados; acesso a global protegida; operação exigindo papel que o usuário não tem.

**Como confirmar:** `WRITE $USERNAME`, `WRITE $ROLES`, e `$SYSTEM.Security.Check(recurso, permissao)`.

**Como corrigir:** conceder o privilégio ao **papel** correto (nunca ao usuário individualmente — Capítulo 7).

**Como prevenir:** verificar privilégios explicitamente na operação, não apenas na tela.

### 3.13 `<NAKED>` — referência nua sem contexto

**Significa:** foi usada uma referência nua (`^(x)`) sem que uma referência completa a tenha precedido.

**Como corrigir:** escrever a referência completa.

**Como prevenir:** **não use referência nua em código novo** (Capítulo 13).

### 3.14 `<FRAMESTACK>` — pilha esgotada

**Significa:** chamadas aninhadas demais.

**Causa quase sempre:** **recursão sem condição de parada**.

**Como corrigir:** revisar a condição de parada; converter a recursão em laço quando possível.

### 3.15 Erros de trava e transação

| Erro | Significa | O que fazer |
|---|---|---|
| trava não obtida | outro processo segura o recurso | usar tempo limite e conferir `$TEST`; revisar a ordem de aquisição (Capítulo 5) |
| `<ROLLFAILED>` | o rollback não pôde ser concluído | investigar imediatamente: pode indicar problema de integridade |
| transação aberta | `TSTART` sem `TCOMMIT` nem `TROLLBACK` | conferir `$TLEVEL`; usar o padrão `try`/`catch` do Capítulo 21 |

Uma transação esquecida aberta é uma das causas mais comuns de "o sistema travou": ela segura travas indefinidamente e faz o journal crescer.

### 3.16 Erros de base de dados e disco

| Erro | Significa |
|---|---|
| `<DATABASE>` | problema de acesso à base |
| `<FILEFULL>` | a base atingiu o limite ou o disco encheu |
| `<DIRECTORY>` | caminho de base inválido |

São erros de **infraestrutura**, não de código. A ação é acionar o administrador, verificar o `messages.log` e o espaço em disco. Aparecem na prova para você saber **que não são culpa do seu código**.

---

## 4. Erros de SQL e de `%Status`

### 4.1 `SQLCODE`

Recapitulando o Capítulo 9:

| Valor | Significa |
|---|---|
| **`0`** | sucesso |
| **`100`** | **não há (mais) linhas** — **não é erro** |
| **negativo** | erro; a descrição está em **`%msg`** |

O erro número um com `SQLCODE` é tratar `100` como falha. Um `SELECT` que não encontra nada **funcionou**; ele apenas não achou nada. Isso é resultado, não erro (Capítulo 10).

Os códigos negativos mais comuns na prática envolvem: tabela ou coluna inexistente, violação de restrição de validação, violação de unicidade, e erro de sintaxe. **O número exato varia e não vale a pena decorar** — o que vale é o método:

```objectscript
&sql(INSERT INTO LabStudy.PATIENT (Name) VALUES (:nome))

if SQLCODE < 0 {
    write "SQLCODE: ", SQLCODE, !
    write "mensagem: ", $GET(%msg), !
}
```

**Sempre imprima `%msg`.** Ela contém a descrição completa, que é o que resolve o problema. E lembre do `NEW SQLCODE, %msg` no início do método, para não contaminar nem ser contaminado.

Com `%SQL.Statement`, os equivalentes são **`rs.%SQLCODE`** e **`rs.%Message`**.

### 4.2 Mensagens de `%Status`

```
ERROR #5001: mensagem personalizada
ERROR #7207: Datatype value 'xxx' length longer than MAXLEN allowed of 30
```

O número identifica a mensagem no catálogo do sistema; o texto explica.

- **`#5001`** é o erro geral, produzido por `$$$ERROR($$$GeneralError, "texto")`. É o que você usa nas suas próprias validações.
- Os demais vêm do sistema e descrevem violações de tipo, de obrigatoriedade, de unicidade e de relacionamento.

**Sempre leia o texto**, não apenas o número. E, ao criar mensagens próprias, **inclua o valor recebido**:

```objectscript
// Ruim
quit $$$ERROR($$$GeneralError, "numero de registro invalido")

// Bom
quit $$$ERROR($$$GeneralError, "numero de registro fora do padrao REG-000000: ["_rec_"]")
```

A diferença aparece três meses depois, quando alguém precisa entender por que uma importação falhou.

---

## 5. Os erros silenciosos

Estes são os piores, porque **não produzem mensagem alguma**. Toda a apostila os encontrou; aqui está o catálogo reunido.

| Sintoma | Causa provável | Capítulo |
|---|---|---|
| média menor que o esperado | vazio somado como zero e contado no divisor | 10 |
| consulta devolve menos linhas que deveria | índice criado sem `%BuildIndices()` | 12 |
| relatório com menos linhas que os dados | índice invertido sem o ID como segundo subscrito | 13 |
| campos deslocados num registro | separador presente no conteúdo | 14 |
| `"A10"` antes de `"A7"` | ordenação de texto sem preenchimento de zeros | 13 e 15 |
| conta com resultado estranho | falta de parênteses (avaliação da esquerda para a direita) | 16 |
| idade errada | subtração de anos sem verificar o aniversário | 16 |
| propriedade nova com dados antigos | posição de armazenamento reaproveitada após remoção | 11 |
| linha antiga que não salva mais | regra apertada sem varrer o legado | 11 |
| comparação de texto falha | caractere invisível | 20 |
| busca não encontra o que existe | função aplicada à coluna no `WHERE` | 12 |
| dados pela metade sem aviso | `catch` vazio | 21 |
| resultado desatualizado | cache sem invalidação | 12 |
| dois registros viram um | `a(10)` e `a("10")` são o mesmo nó | 13 |
| relatório correto em teste, lento em produção | N+1 invisível com poucos dados | 12 |

**A defesa contra erros silenciosos não é o tratamento de erros — é a verificação.** Contar antes e depois, comparar duas implementações independentes, e rodar um `SelfCheck` (Capítulo 20). Foi por isso que quase todos os exercícios desta apostila pediram para **verificar que duas versões concordam** antes de comparar tempos ou aceitar um resultado.

---

## 6. Exemplo comentado

Arquivo `src/LabStudy/Demo/Catalog.cls`:

```objectscript
/// An executable catalogue of common errors.
/// Each entry reproduces the error, explains it and shows the fix.
Class LabStudy.Demo.Catalog Extends %RegisteredObject
{

/// Reproduces one error and returns its name.
ClassMethod Reproduce(kind As %String) As %String
{
    set name = "(nao falhou)"

    try {
        do ..Broken(kind)
    }
    catch e {
        set name = e.Name
    }
    quit name
}

/// The broken versions.
ClassMethod Broken(kind As %String) As %Status [ Private ]
{
    if kind = "undefined" {
        kill dados
        write dados("inexistente")
    }
    elseif kind = "oref" {
        set p = ##class(LabStudy.Patient).%OpenId(999999)
        write p.Name
    }
    elseif kind = "divide" {
        set total = 0, n = 0
        write total / n
    }
    elseif kind = "select" {
        set v = 150
        write $SELECT(v < 70: "baixo", v > 200: "alto")
    }
    elseif kind = "list" {
        set row = $LISTBUILD("HGB", 13.5)
        write $LIST(row, 5)
    }
    elseif kind = "date" {
        write $ZDATEH("17/05/1990", 3)
    }
    elseif kind = "method" {
        do $CLASSMETHOD("LabStudy.Patient", "MetodoQueNaoExiste")
    }
    elseif kind = "class" {
        do $CLASSMETHOD("LabStudy.NaoExiste", "Qualquer")
    }
    elseif kind = "syntax" {
        xecute "isto nao e codigo"
    }
    else {
        set x = 1 / 0
    }
    quit $$$OK
}

/// The fixed versions. None of them raises an error.
ClassMethod Fixed(kind As %String) As %String [ Private ]
{
    if kind = "undefined" {
        kill dados
        quit "[" _ $GET(dados("inexistente"), "(sem valor)") _ "]"
    }
    elseif kind = "oref" {
        set p = ##class(LabStudy.Patient).%OpenId(999999)
        quit $SELECT($ISOBJECT(p): p.Name, 1: "(paciente nao encontrado)")
    }
    elseif kind = "divide" {
        set total = 0, n = 0
        quit "[" _ $SELECT(n > 0: total / n, 1: "") _ "]  (vazio, nao zero)"
    }
    elseif kind = "select" {
        set v = 150
        quit $SELECT(v < 70: "baixo", v > 200: "alto", 1: "normal")
    }
    elseif kind = "list" {
        set row = $LISTBUILD("HGB", 13.5)
        quit "[" _ $LISTGET(row, 5) _ "]  (vazio, sem erro)"
    }
    elseif kind = "date" {
        set d = ##class(LabStudy.DateTime).Parse("17/05/1990", 4)
        quit $SELECT(d '= "": ##class(LabStudy.DateTime).Format(d, 3), 1: "(data invalida)")
    }
    elseif kind = "method" {
        set exists = ##class(%Dictionary.CompiledMethod).%ExistsId(
            "LabStudy.Patient||MetodoQueNaoExiste")
        quit $SELECT(exists: "existe", 1: "(metodo nao existe, recusado antes de chamar)")
    }
    elseif kind = "class" {
        set exists = ##class(%Dictionary.CompiledClass).%ExistsId("LabStudy.NaoExiste")
        quit $SELECT(exists: "existe", 1: "(classe nao existe, recusada antes de chamar)")
    }
    elseif kind = "syntax" {
        quit "(nao usar XECUTE: usar $CLASSMETHOD com lista de permitidos)"
    }
    else {
        quit "(protegido)"
    }
}

/// Explains each error in one line.
ClassMethod Explain(kind As %String) As %String [ Private ]
{
    quit $CASE(kind,
        "undefined": "leitura de variavel/no inexistente -> use $GET",
        "oref":      "%OpenId devolveu vazio -> teste $ISOBJECT",
        "divide":    "divisor zero -> guarde com $SELECT",
        "select":    "$SELECT sem ramo verdadeiro -> acrescente 1:",
        "list":      "$LIST fora de faixa -> use $LISTGET",
        "date":      "formato de data errado -> valide e use try/catch",
        "method":    "metodo inexistente -> valide no dicionario antes",
        "class":     "classe inexistente ou nao compilada -> recompile",
        "syntax":    "XECUTE com texto invalido -> evite XECUTE",
        :            "divisao por zero")
}

/// The full catalogue.
ClassMethod Show() As %Status
{
    set kinds = $LISTBUILD("undefined", "oref", "divide", "select", "list",
                           "date", "method", "class", "syntax")

    write "=== catalogo de erros comuns ===", !, !

    set ptr = 0
    while $LISTNEXT(kinds, ptr, kind) {
        write "-- ", kind, " --", !
        write "  erro produzido : ", ..Reproduce(kind), !
        write "  causa          : ", ..Explain(kind), !
        write "  versao corrigida: ", ..Fixed(kind), !
        write !
    }

    quit $$$OK
}

/// Silent errors: no message, wrong result.
ClassMethod Silent() As %Status
{
    write "=== erros silenciosos ===", !

    write !, "-- 1. vazio somado como zero --", !
    kill v
    set v(1) = 10, v(2) = 20, v(3) = "", v(4) = 30

    set totalErrado = 0, nErrado = 0, k = ""
    for {
        set k = $ORDER(v(k), 1, val)
        quit:k=""
        set nErrado = nErrado + 1
        set totalErrado = totalErrado + val
    }

    set totalCerto = 0, nCerto = 0, k = ""
    for {
        set k = $ORDER(v(k), 1, val)
        quit:k=""
        continue:val=""
        set nCerto = nCerto + 1
        set totalCerto = totalCerto + val
    }

    write "  errado : soma ", totalErrado, " em ", nErrado, " -> media ", totalErrado / nErrado, !
    write "  certo  : soma ", totalCerto, " em ", nCerto, " -> media ", totalCerto / nCerto, !

    write !, "-- 2. indice invertido sem o id --", !
    kill dados, semId, comId
    set dados(1) = 92, dados(2) = 92, dados(3) = 92, dados(4) = 150

    set k = ""
    for {
        set k = $ORDER(dados(k), 1, val)
        quit:k=""
        set semId(val) = k
        set comId(val, k) = ""
    }

    set nSem = 0, x = ""
    for { set x = $ORDER(semId(x)) quit:x=""  set nSem = nSem + 1 }

    set nCom = 0, x = ""
    for {
        set x = $ORDER(comId(x)) quit:x=""
        set y = ""
        for { set y = $ORDER(comId(x, y)) quit:y=""  set nCom = nCom + 1 }
    }

    write "  registros originais : 4", !
    write "  recuperados sem id  : ", nSem, "   <-- perdeu ", 4 - nSem, !
    write "  recuperados com id  : ", nCom, !

    write !, "-- 3. avaliacao da esquerda para a direita --", !
    write "  2 + 3 * 4       = ", 2 + 3 * 4, "   (esperado 14)", !
    write "  2 + (3 * 4)     = ", 2 + (3 * 4), !

    write !, "-- 4. ordenacao de texto --", !
    kill codes
    set codes("A7") = "", codes("A10") = "", codes("A2") = ""
    write "  ordem obtida    : "
    set k = "" for { set k = $ORDER(codes(k)) quit:k=""  write k, " " }
    write !

    kill padded
    for c = "A7", "A10", "A2" {
        set padded("A"_$TRANSLATE($JUSTIFY($EXTRACT(c, 2, *), 4), " ", "0")) = c
    }
    write "  com preenchimento: "
    set k = "" for { set k = $ORDER(padded(k), 1, orig) quit:k=""  write orig, " " }
    write !

    write !, "-- 5. o mesmo no --", !
    kill nos
    set nos(10) = "primeiro"
    set nos("10") = "segundo"
    write "  gravei dois valores, existem: "
    set n = 0, k = "" for { set k = $ORDER(nos(k)) quit:k=""  set n = n + 1 }
    write n, " no(s)", !
    write "  conteudo: ", nos(10), "   <-- o primeiro foi sobrescrito", !

    quit $$$OK
}

/// The three questions that lead from symptom to cause.
ClassMethod DecisionTree() As %Status
{
    write "=== do sintoma a causa ===", !, !

    write "1. Houve mensagem de erro?", !
    write "   sim ................. va ao catalogo pelo nome do erro", !
    write "   nao, resultado errado  erro silencioso: verifique contagens", !
    write "   nao, esta lento ..... desempenho: plano, N+1, indices", !
    write "   nao, nada acontece .. travamento: processos e travas", !, !

    write "2. Quando comecou?", !
    write "   sempre foi assim .... logica ou modelagem", !
    write "   apos uma mudanca .... compare com o que mudou", !
    write "   so com alguns dados . e dado: descubra QUAIS", !
    write "   so sob carga ........ concorrencia ou desempenho", !, !

    write "3. Acontece com o mesmo dado?", !
    write "   sim ................. caso reproduzivel: metade do trabalho feita", !
    write "   nao ................. concorrencia, ordem ou horario", !, !

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..DecisionTree()
    do ..Show()
    do ..Silent()
    quit $$$OK
}

}
```

### 6.1 Executando

```
LABSTUDY>DO ##class(LabStudy.Demo.Catalog).Show()
=== catalogo de erros comuns ===

-- undefined --
  erro produzido : <UNDEFINED>
  causa          : leitura de variavel/no inexistente -> use $GET
  versao corrigida: [(sem valor)]

-- oref --
  erro produzido : <INVALID OREF>
  causa          : %OpenId devolveu vazio -> teste $ISOBJECT
  versao corrigida: (paciente nao encontrado)

-- divide --
  erro produzido : <DIVIDE>
  causa          : divisor zero -> guarde com $SELECT
  versao corrigida: []  (vazio, nao zero)

-- select --
  erro produzido : <ILLEGAL VALUE>
  causa          : $SELECT sem ramo verdadeiro -> acrescente 1:
  versao corrigida: normal

-- list --
  erro produzido : <LIST>
  causa          : $LIST fora de faixa -> use $LISTGET
  versao corrigida: []  (vazio, sem erro)

-- date --
  erro produzido : <ILLEGAL VALUE>
  causa          : formato de data errado -> valide e use try/catch
  versao corrigida: 1990-05-17

-- method --
  erro produzido : <METHOD DOES NOT EXIST>
  causa          : metodo inexistente -> valide no dicionario antes
  versao corrigida: (metodo nao existe, recusado antes de chamar)

-- class --
  erro produzido : <CLASS DOES NOT EXIST>
  causa          : classe inexistente ou nao compilada -> recompile
  versao corrigida: (classe nao existe, recusada antes de chamar)

-- syntax --
  erro produzido : <SYNTAX>
  causa          : XECUTE com texto invalido -> evite XECUTE
  versao corrigida: (nao usar XECUTE: usar $CLASSMETHOD com lista de permitidos)
```

O que observar:

- **O `date` produziu `<ILLEGAL VALUE>`, o mesmo do `select`.** Dois problemas completamente diferentes com o mesmo nome de erro. **É por isso que a localização importa mais que o nome** — `Location` diz qual linha, e a linha diz qual problema.
- **A versão corrigida de `date` funcionou** porque usou o formato **4** (dia/mês/ano) em vez do 3. O texto `"17/05/1990"` é uma data válida — no formato errado. **Nem todo erro de conversão é dado ruim; às vezes é formato errado.**
- **As correções de `method` e `class` não tratam o erro: elas o evitam.** Consultar o dicionário antes de despachar transforma um erro de execução numa decisão de programa. **Prevenir é sempre melhor que capturar.**
- **A correção de `divide` devolve vazio, não zero.** É a lição do Capítulo 10 aparecendo no tratamento de erro: `<DIVIDE>` costuma indicar "não havia dados", e a resposta correta para isso é "desconhecido".

```
LABSTUDY>DO ##class(LabStudy.Demo.Catalog).Silent()
=== erros silenciosos ===

-- 1. vazio somado como zero --
  errado : soma 60 em 4 -> media 15
  certo  : soma 60 em 3 -> media 20

-- 2. indice invertido sem o id --
  registros originais : 4
  recuperados sem id  : 2   <-- perdeu 2
  recuperados com id  : 4

-- 3. avaliacao da esquerda para a direita --
  2 + 3 * 4       = 20   (esperado 14)
  2 + (3 * 4)     = 14

-- 4. ordenacao de texto --
  ordem obtida    : A10 A2 A7
  com preenchimento: A2 A7 A10

-- 5. o mesmo no --
  gravei dois valores, existem: 1 no(s)
  conteudo: segundo   <-- o primeiro foi sobrescrito
```

**Esta é a tela mais importante do capítulo, e talvez da apostila.**

Cinco erros. **Nenhuma mensagem.** Nenhuma exceção. Nenhum `%Status` de erro. O programa rodou até o fim e entregou resultados — todos errados.

- **A média deu 15 em vez de 20**, um erro de 25% num resultado clínico.
- **Dois de quatro registros desapareceram** de um relatório.
- **Uma conta deu 20 em vez de 14.**
- **Códigos ordenados na sequência errada.**
- **Um dado sobrescreveu outro** e ninguém soube.

Contra isso, `try`/`catch` não serve para nada. O que serve é: **contar antes e depois, comparar implementações independentes, e verificar com um `SelfCheck`.**

---

## 7. Pegadinhas e erros comuns

**1) Tratar `SQLCODE = 100` como erro.**
É "não há linhas" — resultado, não falha.

**2) Ler o número do erro e ignorar o texto.**
O texto contém a informação que resolve.

**3) Criar mensagens de erro sem o valor recebido.**
"Número inválido" não permite investigar. "Número inválido: [REG-1]" permite.

**4) Confundir `<UNDEFINED>` com `<INVALID OREF>`.**
O primeiro é variável ou nó inexistente; o segundo é acesso a propriedade de algo que não é objeto.

**5) Não testar o retorno de `%OpenId`.**
Causa número um de `<INVALID OREF>`.

**6) `$SELECT` sem o ramo `1:`.**
Causa número um de `<ILLEGAL VALUE>`.

**7) Usar `$LIST` onde cabia `$LISTGET`.**
`$EXTRACT` fora de faixa devolve vazio; `$LIST` gera erro. A assimetria engana.

**8) Concatenar num laço grande.**
`<MAXSTRING>`. Use stream.

**9) Acumular estruturas grandes em variável local.**
`<STORE>`. Use PPG.

**10) Recursão sem condição de parada.**
`<FRAMESTACK>`.

**11) Referência nua em código novo.**
`<NAKED>` e ilegibilidade.

**12) Achar que erro de infraestrutura é bug de código.**
`<FILEFULL>` e `<DATABASE>` são para o administrador.

**13) Deixar transação aberta.**
Trava tudo e faz o journal crescer.

**14) Confiar apenas em tratamento de erro.**
Os piores problemas **não geram erro**. Verifique contagens e resultados.

**15) Não recompilar dependentes após alterar uma superclasse.**
Erros estranhos e intermitentes que somem após recompilação geral.

**16) Corrigir o sintoma sem entender a causa.**
Ele volta, em outro lugar, mais difícil de achar.

---

## 8. MÃO NA MASSA

---

### Exercício 22.1 — Do sintoma à causa

**a) Enunciado:** Para cada situação abaixo, **antes de olhar a solução**, escreva: (1) qual erro provavelmente ocorre ou qual é o sintoma, (2) qual a causa, (3) como corrigir, (4) como prevenir.

1. Um relatório funciona no ambiente de teste e falha em produção com `<INVALID OREF>`, sempre no mesmo ponto.
2. Uma importação processa 900 de 1000 linhas e para, sem mensagem no console.
3. A média de resultados aparece 20% menor do que a conferência manual.
4. Uma consulta que sempre foi rápida passou a demorar minutos, depois de uma manutenção.
5. Um método que funciona no Terminal falha quando chamado por um `JOB`.
6. Um paciente cadastrado não aparece na busca por nome, mas existe na tabela.
7. Ao salvar um registro antigo sem alterar nada, aparece um erro de validação.
8. Dois usuários salvam o mesmo paciente e um perde as alterações.

Depois, escreva `ClassMethod Investigate(caso)` que, para cada caso, imprima as **perguntas** que você faria e os **comandos** que executaria.

**b) Dica:** Use a árvore de decisão da seção 2.3.

**c) Como testar:** Cada caso deve ter uma causa distinta e uma verificação concreta.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

| # | Sintoma | Causa provável | Correção | Prevenção |
|---|---|---|---|---|
| 1 | `<INVALID OREF>` só em produção | dado com relacionamento vazio, que não existe em teste | testar `$ISOBJECT` antes de acessar | verificar sempre o retorno de `%OpenId` e propriedades de relacionamento |
| 2 | para na linha 900, sem mensagem | erro capturado e engolido, ou processo em segundo plano cujo erro não foi registrado | registrar o erro por linha e continuar | `try` por linha, `catch` que registra |
| 3 | média 20% menor | vazios somados como zero e contados no divisor | ignorar vazios na contagem | `continue:v=""` em toda agregação |
| 4 | consulta ficou lenta após manutenção | índice removido ou não reconstruído; estatísticas desatualizadas; cache de consultas com plano antigo | `%BuildIndices()`, coletar estatísticas, purgar cache | incluir esses passos no procedimento de manutenção |
| 5 | funciona no Terminal, falha no `JOB` | dependência de variável local, PPG ou dispositivo que o outro processo não tem | passar tudo por argumento ou global | escrever métodos de segundo plano autossuficientes |
| 6 | existe na tabela, não aparece na busca | índice de nome não populado, ou busca com função na coluna | `%BuildIndices()`; comparar a coluna nua | reconstruir índices ao criá-los; nunca aplicar função à coluna no `WHERE` |
| 7 | registro antigo não salva mais | regra apertada depois que o dado foi gravado (validação preguiçosa) | varrer o legado com `%ValidateObject` | ao apertar regras, medir e corrigir o legado antes |
| 8 | um usuário perde as alterações | atualização perdida: ambos abriram, ambos salvaram | usar concorrência adequada no `%OpenId` ou controle de versão | definir a política de concorrência no desenho |

E a classe de investigação:

`src/LabStudy/Demo/Investigate.cls`:

```objectscript
/// From symptom to cause: what to ask and what to run.
Class LabStudy.Demo.Investigate Extends %RegisteredObject
{

ClassMethod Case(n As %Integer) As %Status
{
    write "=== caso ", n, " ===", !

    if n = 1 {
        write "SINTOMA: <INVALID OREF> so em producao, sempre no mesmo ponto", !, !
        write "PERGUNTAS:", !
        write "  - qual registro especifico causa? (o dado difere, o codigo nao)", !
        write "  - que propriedade esta vazia naquele registro?", !, !
        write "COMANDOS:", !
        write "  ZWRITE ^ERRORS  (o log de erro guarda as variaveis do momento)", !
        write "  SELECT COUNT(*) FROM LabStudy.EXAM WHERE Patient IS NULL", !
    }
    elseif n = 2 {
        write "SINTOMA: importacao para na linha 900 sem mensagem", !, !
        write "PERGUNTAS:", !
        write "  - ha algum catch vazio no caminho?", !
        write "  - a linha 900 tem algo diferente das outras?", !, !
        write "COMANDOS:", !
        write "  DO ##class(LabStudy.Log).Show(50, ""WARN"")", !
        write "  (e inspecionar a linha 900 do arquivo com ZZDUMP)", !
    }
    elseif n = 3 {
        write "SINTOMA: media 20% menor que a conferencia manual", !, !
        write "PERGUNTAS:", !
        write "  - quantos registros a conta considerou?", !
        write "  - ha valores vazios entre eles?", !, !
        write "COMANDOS:", !
        write "  SELECT COUNT(*), COUNT(ResultValue) FROM LabStudy.EXAM", !
        write "  (se os dois numeros diferem, ha nulos: COUNT(coluna) os ignora)", !
    }
    elseif n = 4 {
        write "SINTOMA: consulta ficou lenta apos manutencao", !, !
        write "PERGUNTAS:", !
        write "  - os indices foram reconstruidos?", !
        write "  - as estatisticas foram coletadas?", !
        write "  - o cache de consultas foi purgado?", !, !
        write "COMANDOS:", !
        write "  DO $SYSTEM.SQL.Explain(""SELECT ..."")", !
        write "  DO ##class(LabStudy.Patient).%ValidateIndices()", !
        write "  DO ##class(LabStudy.Schema).Optimize()", !
    }
    elseif n = 5 {
        write "SINTOMA: funciona no Terminal, falha no JOB", !, !
        write "PERGUNTAS:", !
        write "  - o metodo usa variavel local ou PPG definida por quem chamou?", !
        write "  - ele escreve na tela?", !
        write "  - ele depende do usuario ou dos papeis do chamador?", !, !
        write "COMANDOS:", !
        write "  (dentro do metodo) SET ^Debug($JOB,""user"") = $USERNAME", !
        write "  ZWRITE ^Debug", !
    }
    elseif n = 6 {
        write "SINTOMA: registro existe na tabela mas nao aparece na busca", !, !
        write "PERGUNTAS:", !
        write "  - o indice usado pela busca esta populado?", !
        write "  - a consulta aplica funcao a coluna indexada?", !, !
        write "COMANDOS:", !
        write "  ZWRITE ^LabStudy.PatientI(""NameIdx"")", !
        write "  DO $SYSTEM.SQL.Explain(""SELECT ... WHERE Name ..."")", !
        write "  DO ##class(LabStudy.Patient).%BuildIndices($LISTBUILD(""NameIdx""))", !
    }
    elseif n = 7 {
        write "SINTOMA: registro antigo nao salva mais, sem ter sido alterado", !, !
        write "PERGUNTAS:", !
        write "  - alguma regra foi apertada recentemente?", !
        write "  - quantos registros legados violam a regra nova?", !, !
        write "COMANDOS:", !
        write "  DO ##class(LabStudy.Diagnostics).SelfCheck()", !
        write "  (e uma varredura com %ValidateObject sobre a extensao)", !
    }
    else {
        write "SINTOMA: um usuario perde as alteracoes do outro", !, !
        write "PERGUNTAS:", !
        write "  - qual o nivel de concorrencia usado no %OpenId?", !
        write "  - ha controle de versao do registro?", !, !
        write "COMANDOS:", !
        write "  (Portal) System Operation > Locks", !
        write "  DO ##class(LabStudy.AuditEntry).PrintFor(""PATIENT"", id)", !
    }

    write !
    quit $$$OK
}

ClassMethod All() As %Status
{
    for i = 1:1:8 {
        do ..Case(i)
    }
    quit $$$OK
}

}
```

**Por que este exercício importa:**

- **Nenhum dos oito casos se resolve lendo o código primeiro.** Todos se resolvem com uma pergunta e um comando que revela o estado real.
- **Os casos 1, 5 e 6 têm sintomas parecidos** ("funciona aqui, não funciona lá") e causas completamente diferentes: dado, contexto de processo e índice. A árvore de decisão separa os três pela pergunta *"quando começou?"* e *"acontece com qualquer dado?"*.
- **O caso 3 tem um comando que resolve sozinho:** `COUNT(*)` contra `COUNT(coluna)`. Se os números diferem, há nulos — e a média está sendo calculada errado. **Uma consulta de dez segundos substitui uma hora de leitura de código.**
- **O caso 7 é a validação preguiçosa do Capítulo 11**, que só se manifesta quando alguém toca num registro antigo. É o exemplo perfeito de erro que aparece semanas depois da mudança que o causou.
- **O caso 2 é o `catch` vazio do Capítulo 21.** O sintoma "para sem mensagem" é a assinatura dele.

---

### Exercício 22.2 — PROJETO CONTÍNUO: LabStudy 3.0

**a) Enunciado:** Feche o projeto:

1. Crie `LabStudy.ErrorCatalog` — a versão de produção do catálogo:
   - `Explain(nomeDoErro)` — devolve causa provável, correção e prevenção para um nome de erro;
   - `Diagnose(exception)` — recebe uma exceção e devolve um diagnóstico com causa e sugestão;
   - `Suggest(status)` — o mesmo para um `%Status`;
   - integre ao `LabStudy.ErrorHandler.Handle()`, para que todo erro registrado já venha com a sugestão.
2. Crie `LabStudy.Doctor` — verificação completa de saúde:
   - executa o `SelfCheck` do Capítulo 20;
   - confere a consistência dos índices;
   - confere a versão do esquema;
   - confere se há transações abertas no processo;
   - confere se há erros recentes no log;
   - confere espaço e arquivos de exportação;
   - devolve uma nota de 0 a 100 e uma lista de recomendações.
3. Amplie `LabStudy.App`:
   - `Parameter VERSION = "3.0"`;
   - `ClassMethod Doctor()` como atalho;
   - `ClassMethod Tour()` que demonstra o sistema inteiro em sequência.
4. Acrescente as opções finais ao menu.

**b) Dica:** Na nota, atribua pesos: problemas de integridade valem mais que arquivos ausentes.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Doctor).Run()
LABSTUDY>DO ##class(LabStudy.App).Tour()
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/ErrorCatalog.cls`:

```objectscript
/// Turns an error name into a diagnosis: cause, fix and prevention.
/// Used by ErrorHandler so every logged error already carries a suggestion.
Class LabStudy.ErrorCatalog Extends %RegisteredObject
{

/// Returns $LB(causa, correcao, prevencao) for a known error name.
ClassMethod Explain(errorName As %String) As %List
{
    set name = $ZCONVERT(errorName, "U")

    if name["UNDEFINED" {
        quit $LISTBUILD(
            "leitura de variavel, propriedade ou no que nunca recebeu valor",
            "use $GET(x) ou teste com $DATA(x) antes de ler",
            "adote $GET como padrao ao ler arrays e globais")
    }
    if name["INVALID OREF" {
        quit $LISTBUILD(
            "acesso a propriedade ou metodo de algo que nao e um objeto",
            "teste $ISOBJECT(x) antes de usar; verifique o retorno de %OpenId",
            "sempre confira %OpenId; em cadeias use && com curto-circuito")
    }
    if name["DIVIDE" {
        quit $LISTBUILD(
            "divisao ou modulo por zero, normalmente um contador que ficou em zero",
            "proteja com $SELECT(n > 0: total / n, 1: """")",
            "sem dados o resultado e VAZIO, nao zero")
    }
    if name["ILLEGAL VALUE" {
        quit $LISTBUILD(
            "$SELECT sem ramo verdadeiro, $CASE sem padrao, ou conversao de data/hora invalida",
            "acrescente o ramo 1: ao $SELECT e o ramo padrao ao $CASE; valide datas antes",
            "todo $SELECT termina com 1:; conversoes de data dentro de try/catch")
    }
    if name["LIST" {
        quit $LISTBUILD(
            "$LIST em posicao inexistente, ou valor que nao e uma lista",
            "use $LISTGET, que devolve vazio; valide com $LISTVALID",
            "$LISTGET como padrao; $LISTVALID em dados vindos de fora")
    }
    if name["MAXSTRING" {
        quit $LISTBUILD(
            "string ultrapassou o limite, normalmente por concatenacao em laco",
            "use %Stream.GlobalCharacter ou %Stream.FileCharacter",
            "conteudo de tamanho indeterminado sempre em stream")
    }
    if name["STORE" {
        quit $LISTBUILD(
            "memoria do processo esgotada por acumulo em variaveis locais ou recursao",
            "troque variavel local por global privada do processo (^||)",
            "processe em lotes e acompanhe $STORAGE em rotinas pesadas")
    }
    if name["SUBSCRIPT" {
        quit $LISTBUILD(
            "subscrito longo demais ou referencia malformada",
            "limite o tamanho do subscrito ou use um valor derivado como chave",
            "nunca use texto livre de tamanho indeterminado como subscrito")
    }
    if (name["METHOD DOES NOT EXIST") || (name["CLASS DOES NOT EXIST")
       || (name["PROPERTY DOES NOT EXIST") {
        quit $LISTBUILD(
            "nome inexistente ou classe nao compilada",
            "recompile o pacote e confira o nome no %Dictionary",
            "valide nomes antes de despachar dinamicamente")
    }
    if name["SYNTAX" {
        quit $LISTBUILD(
            "codigo invalido em XECUTE ou indireção",
            "valide a string antes de executar",
            "evite XECUTE: prefira $CLASSMETHOD com lista de permitidos")
    }
    if name["PROTECT" {
        quit $LISTBUILD(
            "usuario sem privilegio sobre o recurso",
            "conceda o privilegio ao papel adequado",
            "verifique privilegios na operacao, nao apenas na tela")
    }
    if name["FRAMESTACK" {
        quit $LISTBUILD(
            "chamadas aninhadas demais, quase sempre recursao sem parada",
            "revise a condicao de parada da recursao",
            "prefira laco a recursao quando a profundidade for indeterminada")
    }
    if name["NAKED" {
        quit $LISTBUILD(
            "referencia nua sem uma referencia completa anterior",
            "escreva a referencia completa",
            "nao use referencia nua em codigo novo")
    }
    if (name["FILEFULL") || (name["DATABASE") || (name["DIRECTORY") {
        quit $LISTBUILD(
            "problema de infraestrutura: base cheia, disco cheio ou caminho invalido",
            "acione o administrador; verifique o messages.log e o espaco em disco",
            "monitore o crescimento das bases")
    }

    quit $LISTBUILD(
        "(erro nao catalogado)",
        "leia a mensagem completa e a localizacao (Metodo+N^Classe)",
        "acrescente este erro ao catalogo quando entender a causa")
}

/// Diagnoses an exception.
ClassMethod Diagnose(exception As %Exception.AbstractException) As %String
{
    set info = ..Explain(exception.Name)

    quit "causa: "_$LIST(info, 1)
         _" | corrija: "_$LIST(info, 2)
         _" | previna: "_$LIST(info, 3)
}

/// Prints a full diagnosis.
ClassMethod Print(errorName As %String) As %Status
{
    set info = ..Explain(errorName)

    write "  erro     : ", errorName, !
    write "  causa    : ", $LIST(info, 1), !
    write "  corrija  : ", $LIST(info, 2), !
    write "  previna  : ", $LIST(info, 3), !
    quit $$$OK
}

/// Suggestion for a %Status.
ClassMethod Suggest(sc As %Status) As %String
{
    quit:$$$ISOK(sc) ""

    set text = $SYSTEM.Status.GetErrorText(sc)

    if text["MAXLEN" {
        quit "o valor excede o comprimento maximo da propriedade: verifique MAXLEN"
    }
    if text["required" {
        quit "propriedade obrigatoria vazia: preencha ou reveja a regra"
    }
    if text["UNIQUE" {
        quit "violacao de indice unico: ja existe um registro com esse valor"
    }
    if text["VALUELIST" {
        quit "valor fora da lista permitida: verifique VALUELIST da propriedade"
    }

    quit "leia a mensagem completa: "_##class(LabStudy.Text).Abbreviate(text, 80)
}

}
```

Integre em `src/LabStudy/ErrorHandler.cls`:

```objectscript
ClassMethod Handle(area As %String, exception As %Exception.AbstractException, context As %String = "") As %Status
{
    set detail = exception.DisplayString()
    set:exception.Location'="" detail = detail_" | local="_exception.Location
    set:exception.Data'="" detail = detail_" | dados="_exception.Data
    set:context'="" detail = detail_" | "_context

    // every logged error already carries its diagnosis
    set detail = detail_" | "_##class(LabStudy.ErrorCatalog).Diagnose(exception)
    set detail = detail_" | "_##class(LabStudy.Diagnostics).Context()

    do ##class(LabStudy.Log).Error(area, exception.Name, detail)

    quit exception.AsStatus()
}
```

`src/LabStudy/Doctor.cls`:

```objectscript
/// Full health check of the LabStudy system, with a score and recommendations.
Class LabStudy.Doctor Extends %RegisteredObject
{

/// Weights: integrity matters more than housekeeping.
Parameter WEIGHTINTEGRITY = 40;

Parameter WEIGHTSCHEMA = 20;

Parameter WEIGHTINDEXES = 20;

Parameter WEIGHTOPERATION = 20;

ClassMethod Section(title As %String) As %Status [ Private ]
{
    write !
    do ##class(LabStudy.Formatter).Line(56, "-")
    write "  ", title, !
    do ##class(LabStudy.Formatter).Line(56, "-")
    quit $$$OK
}

ClassMethod Item(label As %String, ok As %Boolean, detail As %String = "") As %Status [ Private ]
{
    do ##class(LabStudy.Formatter).Row(
        $LISTBUILD(label, $SELECT(ok: "OK", 1: "ATENCAO"), detail),
        $LISTBUILD(28, 10, 18),
        $LISTBUILD("L", "C", "L"))
    quit $$$OK
}

/// Counts rows returned by a check query.
ClassMethod Count(sql As %String) As %Integer [ Private ]
{
    set rs = ##class(%SQL.Statement).%ExecDirect(, sql)
    quit:rs.%SQLCODE<0 -1
    quit:'rs.%Next() 0
    quit +rs.%GetData(1)
}

ClassMethod Run() As %Integer
{
    set recommendations = ""
    set score = 0

    write !
    do ##class(LabStudy.Formatter).Line(56, "=")
    write "  ", ##class(LabStudy.Text).Pad(
        "LabStudy Doctor "_##class(LabStudy.App).#VERSION, 52, "C"), !
    do ##class(LabStudy.Formatter).Line(56, "=")

    // ---- integrity ----
    do ..Section("integridade dos dados")

    set problems = 0

    set n = ..Count("SELECT COUNT(*) FROM LabStudy.PATIENT "
                    _"WHERE RecordNumber IS NULL OR RecordNumber = ''")
    do ..Item("pacientes sem registro", n = 0, $SELECT(n: n_" casos", 1: ""))
    set problems = problems + (n > 0)

    set n = ..Count("SELECT COUNT(*) FROM (SELECT RecordNumber FROM LabStudy.PATIENT "
                    _"GROUP BY RecordNumber HAVING COUNT(*) > 1)")
    do ..Item("registros duplicados", n = 0, $SELECT(n: n_" casos", 1: ""))
    set problems = problems + (n > 0)

    set n = ..Count("SELECT COUNT(*) FROM LabStudy.EXAM WHERE Patient IS NULL")
    do ..Item("exames sem paciente", n = 0, $SELECT(n: n_" casos", 1: ""))
    set problems = problems + (n > 0)

    set n = ..Count("SELECT COUNT(*) FROM LabStudy.EXAM "
                    _"WHERE ResultStatus = 'final' AND ResultValue IS NULL")
    do ..Item("finais sem valor", n = 0, $SELECT(n: n_" casos", 1: ""))
    set problems = problems + (n > 0)

    set n = ..Count("SELECT COUNT(*) FROM LabStudy.PATIENT WHERE BirthDate > "
                    _##class(LabStudy.DateTime).Today())
    do ..Item("nascimento no futuro", n = 0, $SELECT(n: n_" casos", 1: ""))
    set problems = problems + (n > 0)

    if problems = 0 {
        set score = score + ..#WEIGHTINTEGRITY
    } else {
        set recommendations = recommendations_$LISTBUILD(
            "corrigir "_problems_" problema(s) de integridade (veja o SelfCheck)")
    }

    // ---- schema ----
    do ..Section("esquema")

    set current = ##class(LabStudy.Schema).Version()
    set latest = ##class(LabStudy.Schema).#LATEST
    set schemaOk = (current = latest)

    do ..Item("versao do esquema", schemaOk, current_" de "_latest)

    if schemaOk {
        set score = score + ..#WEIGHTSCHEMA
    } else {
        set recommendations = recommendations_$LISTBUILD(
            "executar ##class(LabStudy.Schema).Upgrade()")
    }

    // ---- indexes ----
    do ..Section("indices")

    set indexOk = 1

    for class = "LabStudy.Patient", "LabStudy.Exam" {
        set sc = $$$OK

        try {
            set sc = $CLASSMETHOD(class, "%ValidateIndices")
        }
        catch e {
            set sc = e.AsStatus()
        }

        set ok = $$$ISOK(sc)
        set:'ok indexOk = 0

        do ..Item($PIECE(class, ".", 2), ok,
                  $SELECT(ok: "", 1: ##class(LabStudy.Text).Abbreviate(
                      $SYSTEM.Status.GetErrorText(sc), 16)))
    }

    if indexOk {
        set score = score + ..#WEIGHTINDEXES
    } else {
        set recommendations = recommendations_$LISTBUILD(
            "reconstruir indices com ##class(LabStudy.Schema).Optimize()")
    }

    // ---- operation ----
    do ..Section("operacao")

    set operationOk = 1

    set txOpen = ($TLEVEL > 0)
    do ..Item("transacoes abertas", 'txOpen, $SELECT(txOpen: "$TLEVEL="_$TLEVEL, 1: ""))
    set:txOpen operationOk = 0

    set n = ..Count("SELECT COUNT(*) FROM LabStudy.EXAM WHERE ResultStatus = 'pending'")
    do ..Item("exames pendentes", n < 50, n_" pendentes")
    set:n>=50 operationOk = 0

    // recent errors in the log
    set errors = 0, seq = "", limit = ##class(LabStudy.DateTime).Today() - 1
    for {
        set seq = $ORDER(^LabStudyLog("byLevel", "ERROR", seq), -1)
        quit:seq=""
        quit:errors>20
        quit:$GET(^LabStudyLog("entry", seq, "day")) < limit
        set errors = errors + 1
    }

    do ..Item("erros nas ultimas 24h", errors = 0, $SELECT(errors: errors_" erros", 1: ""))
    set:errors>0 operationOk = 0

    set dirOk = ##class(%File).DirectoryExists(##class(LabStudy.FileIO).#DEFAULTDIR)
    do ..Item("diretorio de exportacao", dirOk,
              $SELECT(dirOk: "", 1: "nao existe"))

    if operationOk {
        set score = score + ..#WEIGHTOPERATION
    } else {
        set:txOpen recommendations = recommendations_$LISTBUILD(
            "encerrar a transacao aberta neste processo")
        set:errors recommendations = recommendations_$LISTBUILD(
            "revisar os erros recentes: ##class(LabStudy.Log).Show(20, ""ERROR"")")
        set:n>=50 recommendations = recommendations_$LISTBUILD(
            "verificar o acumulo de exames pendentes")
    }

    // ---- score ----
    write !
    do ##class(LabStudy.Formatter).Line(56, "=")
    write "  nota: ", score, " de 100", !
    write "  situacao: ", $CASE(score \ 25,
        4: "excelente",
        3: "boa, com pontos de atencao",
        2: "requer atencao",
        1: "requer acao",
        : "critica"), !
    do ##class(LabStudy.Formatter).Line(56, "=")

    if $LISTLENGTH(recommendations) {
        write !, "  recomendacoes:", !
        for i = 1:1:$LISTLENGTH(recommendations) {
            write "    ", i, ". ", $LIST(recommendations, i), !
        }
    } else {
        write !, "  nenhuma acao necessaria.", !
    }

    write !
    do ##class(LabStudy.Log).Info("doctor", "verificacao concluida", "nota="_score)

    quit score
}

}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "3.0";

/// Shortcut to the health check.
ClassMethod Doctor() As %Integer [ CodeMode = expression ]
{
##class(LabStudy.Doctor).Run()
}

/// Demonstrates the whole system in sequence.
ClassMethod Tour() As %Status
{
    do ..About()

    write !, ">>> 1. estado do esquema", !
    do ##class(LabStudy.Schema).Report()

    write !, ">>> 2. painel", !
    do ##class(LabStudy.Reports).Dashboard()

    write !, ">>> 3. rankings", !
    do ##class(LabStudy.RankReport).ByExamCount(3)

    write !, ">>> 4. distribuicao por idade", !
    do ##class(LabStudy.Reports).AgeDistribution()

    write !, ">>> 5. resultados fora da faixa", !
    do ##class(LabStudy.Reports).AbnormalByRange()

    write !, ">>> 6. motor de regras", !
    do ##class(LabStudy.RuleEngine).Check("GLU")

    write !, ">>> 7. desempenho", !
    do ##class(LabStudy.Reports).Benchmark()

    write !, ">>> 8. saude do sistema", !
    do ##class(LabStudy.Doctor).Run()

    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.ErrorCatalog).Print("<INVALID OREF>")
  erro     : <INVALID OREF>
  causa    : acesso a propriedade ou metodo de algo que nao e um objeto
  corrija  : teste $ISOBJECT(x) antes de usar; verifique o retorno de %OpenId
  previna  : sempre confira %OpenId; em cadeias use && com curto-circuito

LABSTUDY>DO ##class(LabStudy.Doctor).Run()

========================================================
                 LabStudy Doctor 3.0
========================================================

--------------------------------------------------------
  integridade dos dados
--------------------------------------------------------
pacientes sem registro          OK
registros duplicados            OK
exames sem paciente             OK
finais sem valor                OK
nascimento no futuro            OK

--------------------------------------------------------
  esquema
--------------------------------------------------------
versao do esquema               OK        5 de 5

--------------------------------------------------------
  indices
--------------------------------------------------------
Patient                         OK
Exam                            OK

--------------------------------------------------------
  operacao
--------------------------------------------------------
transacoes abertas              OK
exames pendentes                OK        3 pendentes
erros nas ultimas 24h        ATENCAO      7 erros
diretorio de exportacao         OK

========================================================
  nota: 80 de 100
  situacao: boa, com pontos de atencao
========================================================

  recomendacoes:
    1. revisar os erros recentes: ##class(LabStudy.Log).Show(20, "ERROR")
```

**Por que cada decisão:**

- **O catálogo transforma conhecimento em código.** O que estava na sua cabeça (ou num caderno) agora está numa classe que qualquer pessoa da equipe consulta — e que o próprio sistema usa ao registrar erros.
- **Todo erro registrado já vem com o diagnóstico.** Quem abrir o log daqui a seis meses não precisa saber o que é `<INVALID OREF>`: a entrada já explica a causa provável e a prevenção. **Isso reduz o tempo de investigação de horas para minutos**, e é o tipo de investimento que só compensa se feito antes de precisar.
- **A nota do `Doctor` tem pesos deliberados.** Integridade vale 40 porque um dado corrompido é irreversível; operação vale 20 porque é transitório. **Uma nota sem pesos trataria "não existe o diretório de exportação" como igual a "há registros duplicados"** — e isso desinformaria.
- **A nota de 80 aponta exatamente o que fazer**, com o comando pronto. Um diagnóstico que não diz o próximo passo é só um relatório.
- **O `Doctor` verificou transação aberta no processo atual.** Essa é a causa clássica de "o sistema travou" que o Capítulo 5 anunciou e o Capítulo 20 ensinou a investigar.
- **O `Tour` percorre oito capacidades do sistema em sequência**, cada uma construída num capítulo diferente. É a demonstração de que as camadas se compõem: `Text`, `DateTime`, `ListUtil`, `Sorter`, `Formatter`, `Schema`, `Log`, `Diagnostics`, `ErrorHandler`, `ErrorCatalog` e `Doctor` — todas usadas juntas, cada uma existindo num lugar só.

---

## 9. Quiz do capítulo

**Q1.** Qual a diferença entre erro de compilação e erro de execução?

- A) Nenhuma.
- B) O de compilação aparece ao compilar e impede a geração do código; o de execução só aparece quando aquele caminho é percorrido com dados reais.
- C) O de execução é mais fácil de corrigir.
- D) O de compilação só acontece em rotinas.

---

**Q2.** Em `<UNDEFINED>zProcessar+12^LabStudy.Import.1 *codigo`, o que significa `*codigo`?

- A) Um erro secundário.
- B) O nome do elemento que não estava definido.
- C) O nome da classe.
- D) O número da linha.

---

**Q3.** Qual a causa mais comum de `<INVALID OREF>`?

- A) Divisão por zero.
- B) `%OpenId()` devolveu vazio e ninguém testou.
- C) String longa demais.
- D) Falta de privilégio.

---

**Q4.** Qual a causa mais comum de `<ILLEGAL VALUE>`?

- A) Objeto vazio.
- B) `$SELECT` sem o ramo `1:`, ou conversão de data inválida.
- C) Memória esgotada.
- D) Recursão infinita.

---

**Q5.** `$EXTRACT` e `$LIST` fora de faixa se comportam como?

- A) Ambos geram erro.
- B) `$EXTRACT` devolve vazio; `$LIST` gera `<LIST>`.
- C) Ambos devolvem vazio.
- D) `$EXTRACT` gera erro; `$LIST` devolve vazio.

---

**Q6.** `SQLCODE = 100` significa o quê?

- A) Erro grave.
- B) Não há (mais) linhas — não é erro.
- C) Cem linhas afetadas.
- D) Violação de restrição.

---

**Q7.** Você recebe `SQLCODE` negativo. Qual a primeira coisa a fazer?

- A) Decorar o número.
- B) Imprimir `%msg`, que traz a descrição completa.
- C) Recompilar a classe.
- D) Reiniciar o processo.

---

**Q8.** `<MAXSTRING>` indica o quê, e como se corrige?

- A) Memória do processo esgotada; use PPG.
- B) String ultrapassou o limite; use stream.
- C) Subscrito longo demais; encurte.
- D) Recursão; revise a parada.

---

**Q9.** `<STORE>` indica o quê, e como se corrige?

- A) String longa demais; use stream.
- B) Memória do processo esgotada; troque variável local por PPG e processe em lotes.
- C) Disco cheio; acione o administrador.
- D) Falta de privilégio.

---

**Q10.** `<FRAMESTACK>` quase sempre indica o quê?

- A) Disco cheio.
- B) Recursão sem condição de parada.
- C) Objeto inválido.
- D) Trava não obtida.

---

**Q11.** `<FILEFULL>` e `<DATABASE>` são problemas de quê?

- A) Do seu código.
- B) De infraestrutura: acione o administrador e verifique disco e `messages.log`.
- C) De compilação.
- D) De índice.

---

**Q12.** Qual destes **não** produz mensagem de erro?

- A) Divisão por zero.
- B) Média calculada incluindo valores vazios no divisor.
- C) `$SELECT` sem ramo verdadeiro.
- D) `$LIST` fora de faixa.

---

**Q13.** Qual a defesa contra erros silenciosos?

- A) `try`/`catch` mais abrangente.
- B) Verificação: contar antes e depois, comparar implementações independentes, rodar um `SelfCheck`.
- C) Mais mensagens de log.
- D) Aumentar a memória do processo.

---

**Q14.** Uma consulta que sempre foi rápida ficou lenta após manutenção. O que verificar primeiro?

- A) A memória do processo.
- B) Se os índices foram reconstruídos, se as estatísticas foram coletadas e se o cache de consultas foi purgado.
- C) A versão do IRIS.
- D) O log de erros.

---

**Q15.** Um método funciona no Terminal e falha quando executado por `JOB`. Causa provável?

- A) Erro de sintaxe.
- B) Dependência de variável local, PPG ou dispositivo que o outro processo não tem.
- C) Índice desatualizado.
- D) Falta de memória.

---

**Q16.** Ao criar uma mensagem de erro própria, o que incluir?

- A) Apenas a descrição do problema.
- B) A descrição **e o valor recebido**, para permitir investigar depois.
- C) O código-fonte da linha.
- D) O nome do usuário.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** o de compilação aparece imediatamente; o de execução pode ficar escondido meses num caminho raro.
- **A está errada:** a diferença é fundamental.
- **C está errada:** o de execução costuma ser mais difícil, porque depende de reproduzir o caminho.
- **D está errada:** ocorre em classes também.

**Q2 — Resposta: B.**
- **B está certa:** o elemento após o asterisco é o que faltava. **Leia sempre até o fim da mensagem.**
- **A, C e D estão erradas:** essas informações estão em outras partes da mensagem.

**Q3 — Resposta: B.**
- **B está certa:** `%OpenId` devolve vazio quando o ID não existe, e o acesso seguinte falha.
- **A, C e D estão erradas:** produzem outros erros.

**Q4 — Resposta: B.**
- **B está certa:** `$SELECT` sem ramo verdadeiro e conversões de data inválidas são as duas causas dominantes.
- **A, C e D estão erradas:** produzem `<INVALID OREF>`, `<STORE>` e `<FRAMESTACK>`.

**Q5 — Resposta: B.**
- **B está certa:** a assimetria é real e engana. `$EXTRACT` é tolerante; `$LIST` não.
- **A, C e D estão erradas:** invertem ou uniformizam o comportamento.

**Q6 — Resposta: B.**
- **B está certa:** é o fim normal de um conjunto de resultados, ou a ausência de linhas.
- **A e D estão erradas:** erros são negativos.
- **C está errada:** não é uma contagem.

**Q7 — Resposta: B.**
- **B está certa:** `%msg` traz a descrição completa, que é o que resolve.
- **A está errada:** decorar números não ajuda; eles variam.
- **C e D estão erradas:** ações sem diagnóstico.

**Q8 — Resposta: B.**
- **B está certa:** conteúdo de tamanho indeterminado vai para stream.
- **A está errada:** isso é `<STORE>`.
- **C e D estão erradas:** são outros erros.

**Q9 — Resposta: B.**
- **B está certa:** a PPG vive no banco temporário e não consome memória do processo.
- **A está errada:** isso é `<MAXSTRING>`.
- **C e D estão erradas:** são outros problemas.

**Q10 — Resposta: B.**
- **B está certa:** aninhamento excessivo, quase sempre recursão sem parada.
- **A, C e D estão erradas:** produzem outros erros.

**Q11 — Resposta: B.**
- **B está certa:** são erros de ambiente, não de código.
- **A, C e D estão erradas:** não têm relação com a causa.

**Q12 — Resposta: B.**
- **B está certa:** é um erro silencioso: o programa roda até o fim e entrega um resultado errado.
- **A, C e D estão erradas:** produzem `<DIVIDE>`, `<ILLEGAL VALUE>` e `<LIST>`.

**Q13 — Resposta: B.**
- **B está certa:** erros silenciosos não geram exceção; só verificação os revela.
- **A está errada:** não há exceção a capturar.
- **C está errada:** log registra o que aconteceu, não o que está errado no resultado.
- **D está errada:** sem relação.

**Q14 — Resposta: B.**
- **B está certa:** são as três causas clássicas de degradação após manutenção, e todas foram vistas no Capítulo 12.
- **A, C e D estão erradas:** improváveis nesse cenário.

**Q15 — Resposta: B.**
- **B está certa:** o processo em segundo plano tem memória e dispositivo próprios.
- **A está errada:** erro de sintaxe falharia nos dois.
- **C e D estão erradas:** afetariam os dois igualmente.

**Q16 — Resposta: B.**
- **B está certa:** o valor recebido é o que permite entender a falha meses depois.
- **A está errada:** insuficiente para investigar.
- **C e D estão erradas:** o primeiro é impraticável; o segundo já vem do contexto.

---

## 10. Resumo relâmpago

1. **Erro de compilação é amigo** (aparece na hora); **erro de execução é problema** (aparece com dados reais, talvez meses depois).
2. **Empurre erros para a compilação** sempre que possível: tipos, `..Metodo()`, evitar `XECUTE`.
3. A mensagem traz **tipo, método, deslocamento, classe** e, com asterisco, **o elemento culpado**. Leia até o fim.
4. **Árvore de decisão:** houve mensagem? quando começou? é sempre com o mesmo dado?
5. **`<UNDEFINED>`** → use `$GET`. **`<INVALID OREF>`** → teste `$ISOBJECT`.
6. **`<DIVIDE>`** → guarde o divisor, e devolva **vazio**, não zero.
7. **`<ILLEGAL VALUE>`** → `$SELECT` sem `1:`, `$CASE` sem padrão, data inválida.
8. **`<LIST>`** → use `$LISTGET`. Lembre: `$EXTRACT` tolera fora de faixa, `$LIST` não.
9. **`<MAXSTRING>`** → stream. **`<STORE>`** → PPG e lotes. **`<FRAMESTACK>`** → recursão sem parada.
10. **`<CLASS/METHOD/PROPERTY DOES NOT EXIST>`** → recompile e valide nomes no `%Dictionary`.
11. **`<PROTECT>`** → privilégio, concedido ao **papel**.
12. **`<FILEFULL>`, `<DATABASE>`** → infraestrutura, não código.
13. **`SQLCODE = 100` não é erro.** Negativo é erro: **leia `%msg`**.
14. **Mensagens próprias devem conter o valor recebido.**
15. **Os piores erros não geram mensagem.** A defesa é **verificação**, não tratamento.
16. **Corrija a causa, não o sintoma.** Sintoma corrigido volta em outro lugar.
17. **Transforme o conhecimento em código:** catálogo de erros, `SelfCheck`, `Doctor`.

---

## 11. Cartões de memorização

**Frente:** Diferença entre erro de compilação e de execução.
**Verso:** O de compilação aparece ao compilar e impede o código de existir. O de execução só aparece quando o caminho é percorrido com dados reais.

**Frente:** O que significa o asterisco no fim de uma mensagem de erro?
**Verso:** O elemento que causou — a variável, a propriedade, o método. Leia sempre até o fim.

**Frente:** Causa mais comum de `<UNDEFINED>`?
**Verso:** Leitura de variável ou nó inexistente. Corrija com `$GET`.

**Frente:** Causa mais comum de `<INVALID OREF>`?
**Verso:** `%OpenId` devolveu vazio e ninguém testou. Corrija com `$ISOBJECT`.

**Frente:** Causa mais comum de `<ILLEGAL VALUE>`?
**Verso:** `$SELECT` sem o ramo `1:`, ou conversão de data com formato errado.

**Frente:** `$EXTRACT` e `$LIST` fora de faixa.
**Verso:** `$EXTRACT` devolve vazio; `$LIST` gera `<LIST>`. Use `$LISTGET`.

**Frente:** `<MAXSTRING>` versus `<STORE>`.
**Verso:** `<MAXSTRING>` é string longa demais (use stream). `<STORE>` é memória do processo esgotada (use PPG e lotes).

**Frente:** `<FRAMESTACK>` indica o quê?
**Verso:** Chamadas aninhadas demais — quase sempre recursão sem condição de parada.

**Frente:** `<FILEFULL>` e `<DATABASE>` são culpa de quê?
**Verso:** Da infraestrutura. Acione o administrador; verifique disco e `messages.log`.

**Frente:** `SQLCODE = 100` significa?
**Verso:** Não há (mais) linhas. **Não é erro.**

**Frente:** Recebi `SQLCODE` negativo. O que fazer?
**Verso:** Imprimir `%msg` — a descrição completa está lá.

**Frente:** O que incluir numa mensagem de erro própria?
**Verso:** A descrição **e o valor recebido**. "Inválido: [REG-1]" permite investigar; "inválido" não.

**Frente:** Quais são as três perguntas do diagnóstico?
**Verso:** Houve mensagem? Quando começou? Acontece sempre com o mesmo dado?

**Frente:** Qual a defesa contra erros silenciosos?
**Verso:** Verificação: contar antes e depois, comparar implementações independentes, rodar um `SelfCheck`. `try`/`catch` não serve.

**Frente:** Consulta ficou lenta após manutenção. O que verificar?
**Verso:** Índices reconstruídos, estatísticas coletadas, cache de consultas purgado.

**Frente:** Funciona no Terminal e falha no `JOB`. Por quê?
**Verso:** O processo em segundo plano não enxerga variáveis locais, PPG nem o seu dispositivo.

**Frente:** Registro antigo parou de salvar sem ter sido alterado. Por quê?
**Verso:** Uma regra foi apertada depois que o dado foi gravado. Validação é preguiçosa.

**Frente:** Qual o princípio de projeto que reduz erros de execução?
**Verso:** Empurrá-los para a compilação: tipos declarados, `..Metodo()`, evitar `XECUTE` e indireção desnecessária.

---

# Encerramento: o caminho até a prova

## O que você construiu

O **LabStudy** saiu da versão 0.1 — uma classe com um método `About()` — até a **3.0**, com mais de trinta classes organizadas em camadas:

| Camada | Classes | Capítulos |
|---|---|---|
| Domínio | `Patient`, `Exam`, `UrgentExam`, `CriticalExam`, `Address`, `User` | 1–3, 18 |
| Persistência e integridade | `Sequence`, `AuditEntry`, `Staging`, `Cache` | 5–8 |
| Segurança | `Security`, `User` | 7 |
| Esquema | `Schema`, `StorageInfo` | 8, 11 |
| Consulta e relatório | `Reports`, `RankReport`, `Formatter` | 9, 13, 15 |
| Utilitários | `Text`, `DateTime`, `ListUtil`, `Sorter` | 13–16 |
| Aplicação | `Menu`, `RuleEngine`, `ExamFactory`, `FileIO`, `BackgroundReport` | 17–19 |
| Diagnóstico | `Log`, `Diagnostics`, `ErrorHandler`, `ErrorCatalog`, `Bench`, `Doctor` | 12, 20–22 |

Cada regra existe **num lugar só**. Essa foi a decisão mais repetida da apostila, e é o que permitiu corrigir o cálculo de idade — errado por vários capítulos — alterando **um** método.

## Roteiro de revisão por domínio

**T1 — Objects (23 questões).** Capítulos 1 a 4.
Revise: `%RegisteredObject` × `%Persistent` × `%SerialObject`; OREF × ID; ciclo `%New`/`%Save`/`%OpenId`; propriedades, parâmetros e os três "rostos" do valor; tipos de índice; `Relationship`; `QUIT` × `RETURN`; `ByRef`/`Output` e **o ponto na chamada**; callbacks e sua ordem; `%DynamicObject` (começa em **0**) × `list Of` (começa em 1); streams e o ciclo escrever→rebobinar→ler.

**T2 — Data Integrity (15 questões).** Capítulos 5 a 7.
Revise: `TSTART`/`TCOMMIT`/`TROLLBACK` e `$TLEVEL`; o padrão `entryLevel`; travas **consultivas**, incrementais e com tempo limite; `$INCREMENT` atômico; triggers (`{Campo*O}`, `{Campo*N}`, `%ok`, `Foreach`); trigger × callback; papéis, privilégios e `$SYSTEM.Security.Check`; hash × criptografia; sal com `GenCryptRand`; injeção de SQL.

**T3 — IRIS Features (14 questões).** Capítulos 8 a 12.
Revise: os quatro meios de armazenamento; `$DATA` (0/1/10/11); `$ORDER` × `$QUERY`; `KILL` × `ZKILL`; globais `D`/`I`/`S`; SQL embutido, `SQLCODE`, cursores, `%SQL.Statement`; os três estados do vazio e `NULL` no SQL; evolução de esquema e validação preguiçosa; N+1, índices, plano de consulta, estatísticas.

**T4 — Functions & APIs (26 questões — o maior peso).** Capítulos 13 a 19.
Revise: ordem de classificação de subscritos e número canônico; o idioma do índice invertido **com o ID como segundo subscrito**; `$LIST` × `$PIECE`; funções de texto e o operador `?`; **avaliação da esquerda para a direita**; `$HOROLOG` e os formatos de `$ZDATE`; `$SELECT` × `$CASE`; formas do `FOR`; `QUIT` × `RETURN` × `CONTINUE` em laços aninhados; indireção e `XECUTE`; polimorfismo, `##super()`, `%Extends`; `Extent` e `%ResultSet`; `%File`, streams e `%Net.HttpRequest`.

**T5 — Errors (12 questões).** Capítulos 20 a 22.
Revise: `ZWRITE`, `ZZDUMP`, `$ZERROR`, `$ECODE`, `$STACK`, `$ESTACK`; `BREAK` × `ZBREAK`; `LOG^%ETN`; `try`/`catch`, objeto de exceção, `THROW` e `$$$ThrowOnError`; `AppendStatus`/`DecomposeStatus`; o padrão transação + `try`/`catch`; o catálogo de erros deste capítulo.

## As dez coisas que mais caem e mais se erram

1. **`2 + 3 * 4` = 20.** Avaliação estritamente da esquerda para a direita.
2. **`$FIND` devolve a posição APÓS o texto encontrado.**
3. **`$SELECT` sem `1:` gera `<ILLEGAL VALUE>`.**
4. **`$LENGTH("", ";")` devolve 0**, não 1.
5. **`a(10)` e `a("10")` são o mesmo nó**; `a(7)` e `a("007")` não são.
6. **`$EXTRACT` fora de faixa devolve vazio; `$LIST` gera erro.**
7. **`QUIT` num laço aninhado sai só do laço interno.**
8. **`SQLCODE = 100` não é erro.**
9. **Zero e string vazia são falsos**; `""` numa soma vale 0, mas `"" = 0` é falso.
10. **`%DynamicArray` começa em 0**; `%List` e `list Of` começam em 1.

## Como usar esta apostila na reta final

1. **Refaça os quizzes** sem olhar o gabarito. São 22 capítulos × 16 questões — mais de 350 questões.
2. **Releia apenas os "Resumo relâmpago"** — eles condensam tudo em cerca de vinte itens por capítulo.
3. **Use os cartões** para as duas semanas finais.
4. **Execute o projeto.** Ler código não é o mesmo que rodá-lo. Os erros que você cometer digitando são exatamente os que a prova cobra.
5. **Refaça os exercícios das seções "Pegadinhas"** — elas listam os erros que os candidatos realmente cometem.

E uma observação final sobre método, que vale para a prova e para o trabalho: **em vários pontos desta apostila, saídas de exemplo divergiram do resultado real**, e eu apontei isso quando percebi — o cálculo de idade do Capítulo 16, a expressão do Capítulo 16, a busca do Capítulo 18. Isso não é um defeito a ser desculpado: é a demonstração mais honesta que posso oferecer do princípio central do domínio T5.

**Nunca confie numa saída que você não executou.** Nem a minha, nem a sua.

Boa prova.

---

*Fim da apostila. Se quiser, posso gerar um simulado completo no formato da prova (76 questões distribuídas pelos cinco domínios com os pesos reais), um resumo consolidado de todos os capítulos num único arquivo, ou aprofundar qualquer tópico específico.*
