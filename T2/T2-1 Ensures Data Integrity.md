# Apostila InterSystems ObjectScript Specialist
## Capítulo 5 — T2.1 Ensures Data Integrity (Transações e travas)

> Começa aqui o domínio **T2 — Basic Programming** (13 questões de 76). Este primeiro tópico trata de duas garantias diferentes e complementares: **transações**, que garantem que um conjunto de mudanças aconteça inteiro ou não aconteça, e **travas (locks)**, que garantem que dois processos não atrapalhem um ao outro.

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Explicar o que é uma **transação** e por que ela existe.
2. Usar **`TSTART`**, **`TCOMMIT`** e **`TROLLBACK`** corretamente.
3. Entender **transações aninhadas** e o comportamento de **`$TLEVEL`**.
4. Saber a diferença crucial entre **`TROLLBACK`** e **`TROLLBACK 1`**.
5. Entender o papel do **journal** e o que é e o que **não é** desfeito por um rollback.
6. Explicar o que é uma **trava** e por que ela é **consultiva** (*advisory*), não obrigatória.
7. Usar o comando **`LOCK`** nas suas formas: simples, incremental (`+` e `-`), com **timeout**, e conferindo **`$TEST`**.
8. Diferenciar trava **exclusiva** de trava **compartilhada** (`#"S"`).
9. Entender como travas e transações interagem — em especial, por que uma trava liberada dentro de uma transação só sai de verdade no fim dela.
10. Evitar **deadlock** aplicando a regra de ordenação.
11. Usar o argumento de **concorrência** do `%OpenId()` e o parâmetro `DEFAULTCONCURRENCY`.
12. Combinar transações com objetos: `%Save()`, `%DeleteId()` e o efeito de um rollback sobre eles.
13. Evoluir o projeto: cadastro atômico de paciente com exames, e um gerador de número de registro seguro contra concorrência.

---

## 2. O conceito em linguagem de gente

### 2.1 O problema que a transação resolve

Volte ao laboratório. Uma recepcionista cadastra uma paciente e, em seguida, registra os três exames que ela veio fazer. São quatro gravações:

1. a ficha da paciente;
2. o exame de hemograma;
3. o exame de glicemia;
4. o exame de colesterol.

Agora imagine que, entre a gravação 2 e a 3, falta luz. Ou o processo é morto. Ou aparece um erro de programação.

O que sobrou no armário? **Uma paciente cadastrada com um exame só.** Metade do trabalho. Um estado que nunca deveria existir: ou o atendimento inteiro aconteceu, ou nenhum pedaço dele deveria estar lá.

Uma **transação** é a forma de dizer ao banco: *"o que vem a seguir é um pacote. Ou entra tudo, ou não entra nada."*

A analogia mais fiel é a do **lápis com borracha mágica**. Você anuncia: "vou começar a escrever no armário, mas anote tudo que eu mudar". Ao final, você tem duas escolhas:

- **Confirmar (`TCOMMIT`)**: "está bom, pode valer de verdade".
- **Desfazer (`TROLLBACK`)**: "esqueça tudo, volte o armário exatamente como estava".

E se a luz acabar no meio, sem você ter confirmado nada? O IRIS, ao voltar, **desfaz sozinho** o que não foi confirmado. Essa é a garantia.

### 2.2 O caderno que torna isso possível: o journal

Como o IRIS consegue desfazer? Ele mantém um **caderno de anotações** paralelo, chamado **journal** (diário).

Toda vez que uma global é alterada dentro de uma transação, o IRIS anota no journal: *"a global tal, no subscrito tal, valia isto antes"*. Se for preciso desfazer, ele lê o caderno de trás para frente e restaura os valores antigos.

Isso tem uma consequência que a prova adora:

> **Só é desfeito o que é anotado no journal.**

Duas coisas importantes decorrem disso:

1. **Variáveis locais não são desfeitas.** Se você fez `SET total = 100` dentro de uma transação e deu rollback, `total` continua valendo 100. O rollback age sobre o **banco**, não sobre a memória do seu programa.
2. **Globais em bancos sem journaling não são desfeitas.** Existem bases configuradas sem journal (por desempenho, para dados descartáveis). Alterações nelas **não voltam atrás**.

Guarde: **rollback desfaz dados persistidos e journalizados. Não desfaz variáveis, não desfaz o que já foi escrito na tela, não desfaz e-mail enviado.**

### 2.3 O problema que a trava resolve

Agora um problema diferente, que acontece mesmo sem nenhuma falha.

Duas recepcionistas, ao mesmo tempo, precisam gerar o próximo número de registro. As duas fazem a mesma coisa:

1. leem o último número usado: **1000**;
2. somam 1: **1001**;
3. gravam 1001 como último número usado;
4. usam 1001 para o seu paciente.

Se as duas fizerem o passo 1 antes de qualquer uma fazer o passo 3, **ambas leem 1000 e ambas usam 1001**. Dois pacientes com o mesmo número de registro.

Nada falhou. Não houve erro, não houve queda de energia. O código está "certo". O problema é que duas pessoas mexeram na mesma coisa ao mesmo tempo.

A solução é a **trava**: antes de ler, a recepcionista pendura uma plaquinha na gaveta dizendo *"ocupado"*. A segunda chega, vê a plaquinha e **espera**. Quando a primeira termina e tira a plaquinha, a segunda entra, lê **1001**, e usa 1002.

### 2.4 O detalhe que muda tudo: a trava é consultiva

Esta é a ideia mais mal compreendida do capítulo, e cai na prova.

A plaquinha "ocupado" **não tranca a gaveta**. Ela só avisa.

Se um terceiro processo chegar e **não olhar** para a plaquinha, ele abre a gaveta e mexe nos dados assim mesmo. O IRIS **não impede**.

Em termos técnicos: o comando `LOCK` no IRIS é **consultivo** (*advisory*). Ele só funciona se **todos** os processos que mexem naquele dado combinarem de usar a mesma trava, com o mesmo nome.

Consequências práticas:

- A trava protege **por convenção**, não por imposição.
- O **nome** da trava é o que importa. Dois processos que travam nomes diferentes não se enxergam, mesmo mexendo no mesmo dado.
- Por isso, a convenção universal no IRIS é **travar com o mesmo nome da global** que se vai alterar. Se você vai mexer em `^Counter("patient")`, você trava `^Counter("patient")`. Assim, qualquer programador que siga a convenção — inclusive o você de daqui a dois anos — encontra a mesma plaquinha.

### 2.5 Exclusiva ou compartilhada

Duas situações diferentes pedem plaquinhas diferentes:

- **"Estou alterando, ninguém encoste"** — trava **exclusiva**. Só um processo por vez. É o padrão.
- **"Estou só lendo, outros podem ler junto, mas ninguém pode alterar"** — trava **compartilhada** (*shared*). Vários leitores convivem; um escritor tem que esperar todos saírem.

Analogia: numa sala de leitura de arquivo, dez pessoas podem consultar o mesmo documento ao mesmo tempo (compartilhada). Mas para levar o documento para a mesa de correção e rasurá-lo, é preciso que ninguém mais esteja lendo (exclusiva).

### 2.6 Deadlock: o abraço mortal

Dois processos, duas gavetas.

- O processo A trava a gaveta 1 e depois quer a gaveta 2.
- O processo B trava a gaveta 2 e depois quer a gaveta 1.

Nenhum dos dois solta o que já tem. Nenhum dos dois consegue o que falta. **Os dois esperam para sempre.** Isso é um **deadlock** (impasse).

A prevenção é simples e é a resposta esperada na prova: **sempre adquira as travas na mesma ordem em todo o sistema.** Se todo mundo pega primeiro a gaveta 1 e depois a 2, o impasse é impossível.

A segunda defesa é sempre usar **timeout**: em vez de esperar para sempre, espere no máximo 5 segundos e, se não conseguir, desista e trate o caso.

---

## 3. A sintaxe explicada

### 3.1 Os três comandos de transação

```
TSTART
TCOMMIT
TROLLBACK
```

- **`TSTART`** — inicia uma transação. A partir daqui, tudo que for alterado no banco fica "a lápis".
- **`TCOMMIT`** — confirma. As alterações passam a valer definitivamente.
- **`TROLLBACK`** — desfaz. O banco volta ao estado anterior ao `TSTART`.

Nenhum dos três recebe argumentos obrigatórios. Todos podem ser abreviados: `TS`, `TC`, `TRO`.

O esqueleto básico:

```objectscript
tstart

set ^Config("a") = 1
set ^Config("b") = 2

if algoDeuErrado {
    trollback
    quit $$$ERROR($$$GeneralError, "operation cancelled")
}

tcommit
quit $$$OK
```

### 3.2 `$TLEVEL`: em que profundidade estou

**`$TLEVEL`** é uma variável especial do sistema que diz **quantas transações estão abertas** neste processo.

- Fora de qualquer transação, `$TLEVEL` vale **0**.
- Cada `TSTART` soma 1.
- Cada `TCOMMIT` subtrai 1.

```
LABSTUDY>WRITE $TLEVEL, !
0
LABSTUDY>TSTART
LABSTUDY>WRITE $TLEVEL, !
1
LABSTUDY>TSTART
LABSTUDY>WRITE $TLEVEL, !
2
LABSTUDY>TCOMMIT
LABSTUDY>WRITE $TLEVEL, !
1
LABSTUDY>TCOMMIT
LABSTUDY>WRITE $TLEVEL, !
0
```

`$TLEVEL` é a ferramenta para escrever código defensivo: antes de dar rollback, confira se há transação aberta. Chamar `TROLLBACK` fora de uma transação é inofensivo em si, mas conferir mostra intenção clara.

### 3.3 Transações aninhadas: a regra que a prova cobra

O IRIS permite `TSTART` dentro de `TSTART`. Mas o comportamento **não** é o que a intuição sugere.

A regra é:

> **Só o `TCOMMIT` mais externo confirma de verdade.** Os `TCOMMIT` internos apenas reduzem `$TLEVEL`.

E, do outro lado:

> **`TROLLBACK` (sem argumento) desfaz TODOS os níveis de uma vez, levando `$TLEVEL` a 0.**

E existe a variante:

> **`TROLLBACK 1`** desfaz **apenas um nível**, reduzindo `$TLEVEL` em 1.

Vamos ver isso funcionando:

```
LABSTUDY>KILL ^T

LABSTUDY>TSTART
LABSTUDY>SET ^T(1) = "outer"

LABSTUDY>TSTART
LABSTUDY>SET ^T(2) = "inner"

LABSTUDY>WRITE $TLEVEL, !
2

LABSTUDY>TROLLBACK 1

LABSTUDY>WRITE $TLEVEL, !
1

LABSTUDY>ZWRITE ^T
^T(1)="outer"

LABSTUDY>TCOMMIT

LABSTUDY>ZWRITE ^T
^T(1)="outer"
```

O `TROLLBACK 1` desfez só o que a transação interna tinha feito. O nível externo continuou vivo e foi confirmado normalmente.

Agora com `TROLLBACK` sem argumento:

```
LABSTUDY>KILL ^T

LABSTUDY>TSTART
LABSTUDY>SET ^T(1) = "outer"
LABSTUDY>TSTART
LABSTUDY>SET ^T(2) = "inner"

LABSTUDY>TROLLBACK

LABSTUDY>WRITE $TLEVEL, !
0

LABSTUDY>ZWRITE ^T
LABSTUDY>
```

Tudo sumiu, dos dois níveis, e `$TLEVEL` foi direto a zero.

**Por que isso importa na prática:** um método que abre a sua própria transação pode ter sido chamado por outro que já tinha uma aberta. Se ele der `TROLLBACK` sem argumento por causa de um problema local, ele derruba a transação do chamador inteira, sem avisar. Esse é um bug clássico e sério.

O padrão defensivo é: **um método que abre transação deve registrar o nível em que entrou e só desfazer até ali.**

```objectscript
ClassMethod SafeWork() As %Status
{
    set entryLevel = $TLEVEL
    tstart

    // ... trabalho ...

    if algoDeuErrado {
        while $TLEVEL > entryLevel {
            trollback 1
        }
        quit $$$ERROR($$$GeneralError, "cancelled")
    }

    tcommit
    quit $$$OK
}
```

### 3.4 O comando `LOCK`

**Forma simples:**

```
LOCK ^Counter("patient")
```

Esta forma tem um efeito colateral importante: ela **libera todas as travas anteriores** do processo e adquire só esta. É a forma antiga e raramente é o que você quer.

**Forma incremental — a que você deve usar:**

```
LOCK +^Counter("patient")     ; adquire, mantendo as travas anteriores
LOCK -^Counter("patient")     ; libera apenas esta
```

- **`+`** soma uma trava ao conjunto do processo.
- **`-`** remove aquela trava específica.

Se você travar o mesmo nome duas vezes com `+`, precisa liberar duas vezes com `-`. O IRIS mantém um contador.

**Liberar tudo:**

```
LOCK
```

`LOCK` sozinho, sem argumento, libera **todas** as travas do processo.

**Com timeout:**

```
LOCK +^Counter("patient"):5
IF '$TEST {
    WRITE "não consegui a trava em 5 segundos", !
    QUIT
}
```

- Os **dois pontos seguidos de um número** definem o tempo máximo de espera, em segundos.
- Depois de um `LOCK` com timeout, a variável especial **`$TEST`** vale `1` se a trava foi obtida e `0` se o tempo esgotou.
- **Sem timeout, o `LOCK` espera para sempre.** Em código de produção, isso é quase sempre um erro de desenho.

Um timeout de zero (`:0`) significa "tente uma vez e desista imediatamente".

**Trava compartilhada:**

```
LOCK +^Patient(1)#"S"
```

O `#"S"` depois do nome pede uma trava **compartilhada**. Sem ele, a trava é **exclusiva**.

Existe também o modificador `#"E"`, de *escalating* (escalonamento): quando muitas travas são adquiridas sobre subscritos de uma mesma global, o IRIS pode substituí-las automaticamente por uma trava única no nível acima, para economizar recursos. O limite exato a partir do qual isso ocorre é configurável: **verificar na documentação oficial**.

**Travando vários nomes de uma vez:**

```
LOCK +(^A(1),^B(2))
```

Os parênteses agrupam. Isso adquire as duas travas de forma atômica — ou as duas, ou nenhuma —, o que ajuda a evitar deadlock.

### 3.5 A interação entre travas e transações

Aqui está um comportamento específico do IRIS que cai na prova:

> **Uma trava liberada com `LOCK -` de dentro de uma transação não é liberada na hora.** A liberação fica pendente e só acontece de fato no `TCOMMIT` ou no `TROLLBACK`.

Por quê? Porque, se a trava fosse solta no meio, outro processo poderia entrar e ver dados que ainda estão "a lápis" e podem ser desfeitos. Manter a trava até o fim da transação preserva o isolamento.

Consequência prática: dentro de uma transação, não adianta tentar "soltar cedo" para liberar concorrência. A trava vai até o fim.

E há a recíproca: quando uma transação termina — por commit ou por rollback —, as liberações pendentes acontecem.

### 3.6 Transações e objetos

`%Save()` e `%DeleteId()` participam das transações normalmente:

```objectscript
tstart

set sc = patient.%Save()
if $$$ISERR(sc) {
    trollback
    quit sc
}

set sc = exam.%Save()
if $$$ISERR(sc) {
    trollback
    quit sc
}

tcommit
quit $$$OK
```

Se o rollback acontecer:

- **Os dados voltam ao estado anterior no banco.** O objeto gravado deixa de existir em disco.
- **Os objetos na memória NÃO voltam.** A OREF continua na sua mão, com as propriedades preenchidas, e `%Id()` pode até continuar devolvendo o ID que foi atribuído e depois desfeito.

Isso confunde muita gente. Repita mentalmente: **o rollback é do banco, não da memória.** Depois de um rollback, descarte os objetos envolvidos e recomece do zero, em vez de tentar reaproveitá-los.

Um detalhe adicional: o próprio `%Save()`, internamente, abre a sua própria transação para garantir que o objeto e todos os seus índices sejam gravados de forma consistente. Isso acontece automaticamente e se aninha na sua transação, se houver uma.

### 3.7 O argumento de concorrência do `%OpenId()`

Você viu no Capítulo 1 que `%OpenId()` aceita um segundo argumento:

```objectscript
set patient = ##class(LabStudy.Patient).%OpenId(id, 4, .sc)
```

Esse argumento é o **nível de concorrência**, um número de 0 a 4, indo do menos restritivo ao mais restritivo:

| Valor | Significado geral |
|---|---|
| **0** | Nenhuma trava. Leitura rápida, sem garantia nenhuma. |
| **1** | Trava compartilhada mantida apenas **durante** a leitura do objeto, e liberada em seguida. |
| **2** | Trava compartilhada **mantida** enquanto o objeto estiver aberto. |
| **3** | Trava exclusiva. |
| **4** | Trava exclusiva mantida, o nível mais restritivo. |

Os nomes e o comportamento fino de cada nível variam com a versão e com o tipo de classe: **verificar na documentação oficial** quando o detalhe importar. O que a prova cobra é a ideia da escala — **0 é sem trava, valores maiores são mais restritivos** — e o fato de que o **padrão** vem do parâmetro de classe:

```objectscript
Class LabStudy.Patient Extends %Persistent
{
Parameter DEFAULTCONCURRENCY = 1;
}
```

Se você não passar nada no `%OpenId()`, é esse valor que é usado.

Regra de bolso para o dia a dia:

- Vai só **ler e mostrar**? Concorrência baixa basta.
- Vai **ler, alterar e gravar**? Use um nível que mantenha trava, ou trave manualmente, para evitar que outro processo altere o objeto entre a sua leitura e a sua gravação.

### 3.8 Transações no SQL

Quando o acesso é por SQL, os comandos mudam de nome mas a ideia é a mesma:

```sql
START TRANSACTION
INSERT INTO LabStudy.PATIENT (Name) VALUES ('Ana')
COMMIT
```

ou

```sql
ROLLBACK
```

Existe também `SAVEPOINT nome`, que marca um ponto intermediário ao qual se pode voltar sem desfazer a transação inteira — o equivalente conceitual do `TROLLBACK 1`.

E existe o **modo de autocommit**: por padrão, em muitas interfaces, cada comando SQL é uma transação própria, confirmada automaticamente. Para agrupar vários comandos, é preciso desligar o autocommit ou usar `START TRANSACTION` explicitamente. A forma de configurar isso depende da interface usada: **verificar na documentação oficial**.

### 3.9 Vendo as travas ativas

No Portal de Gerenciamento: **System Operation → Locks**. Ali você vê todas as travas do sistema, quem as detém e de que tipo são. É a primeira coisa a olhar quando um processo parece travado.

---

## 4. Exemplo comentado

Uma classe que junta transação e trava:

Arquivo `src/LabStudy/Demo/Bank.cls`:

```objectscript
/// Demonstrates transactions and locking with a simple ledger.
Class LabStudy.Demo.Bank Extends %RegisteredObject
{

/// How long to wait for a lock, in seconds.
Parameter LOCKTIMEOUT = 5;

/// Creates an account with an initial balance.
ClassMethod OpenAccount(name As %String, initial As %Numeric = 0) As %Status
{
    set ^Bank(name) = initial
    quit $$$OK
}

/// Moves money from one account to another, atomically.
ClassMethod Transfer(from As %String, to As %String, amount As %Numeric) As %Status
{
    if amount <= 0 {
        quit $$$ERROR($$$GeneralError, "Amount must be positive")
    }

    // Always lock in a fixed order, to make deadlock impossible.
    set first = from
    set second = to
    if first ] second {
        set first = to
        set second = from
    }

    lock +^Bank(first):..#LOCKTIMEOUT
    if '$TEST {
        quit $$$ERROR($$$GeneralError, "Could not lock account "_first)
    }

    lock +^Bank(second):..#LOCKTIMEOUT
    if '$TEST {
        lock -^Bank(first)
        quit $$$ERROR($$$GeneralError, "Could not lock account "_second)
    }

    set entryLevel = $TLEVEL
    tstart

    if $GET(^Bank(from)) < amount {
        while $TLEVEL > entryLevel { trollback 1 }
        lock -^Bank(first)
        lock -^Bank(second)
        quit $$$ERROR($$$GeneralError, "Insufficient funds in "_from)
    }

    set ^Bank(from) = ^Bank(from) - amount
    set ^Bank(to) = $GET(^Bank(to)) + amount
    set ^BankLog($INCREMENT(^BankLog)) = from_" -> "_to_" : "_amount

    tcommit

    lock -^Bank(first)
    lock -^Bank(second)
    quit $$$OK
}

/// Prints all accounts.
ClassMethod Report() As %Status
{
    set name = ""
    for {
        set name = $ORDER(^Bank(name))
        quit:name=""
        write name, ": ", ^Bank(name), !
    }
    quit $$$OK
}

}
```

Comentando as decisões:

- **`Parameter LOCKTIMEOUT = 5`** — o tempo de espera fica num parâmetro, num lugar só, em vez de espalhado como número mágico. Lê-se com `..#LOCKTIMEOUT`, como você aprendeu no Capítulo 0.
- **A ordenação das travas.** As duas contas são travadas sempre na mesma ordem alfabética, independentemente de quem paga e quem recebe. Se o processo A transfere de `ana` para `bruno` e o processo B transfere de `bruno` para `ana`, **os dois travam `ana` primeiro**. Deadlock impossível. O operador `]` compara textos e devolve verdadeiro se o da esquerda vier depois do da direita na ordenação — comparações de string são o Capítulo 10.
- **Timeout em toda trava.** Nenhum `LOCK` sem timeout. Se não conseguir, o método devolve erro em vez de congelar.
- **A segunda trava falhando libera a primeira.** Sair de um método deixando travas presas é um vazamento tão sério quanto vazar memória.
- **`set entryLevel = $TLEVEL` antes do `TSTART`.** Assim, se este método for chamado de dentro de outra transação, o rollback local não derruba a transação do chamador.
- **A conferência de saldo está DENTRO da transação e DEPOIS das travas.** Essa ordem não é acidental: só depois de ter a trava é que o saldo lido é confiável, porque ninguém mais pode alterá-lo enquanto seguramos a plaquinha.
- **`$INCREMENT(^BankLog)`** — essa função soma 1 a uma global e devolve o novo valor **de forma atômica**, sem precisar de trava. É a ferramenta certa para contadores. Voltaremos a ela no Capítulo 5.4 do projeto.
- **As travas são liberadas depois do `TCOMMIT`.** Como vimos na seção 3.5, dentro de uma transação a liberação ficaria pendente de qualquer forma. Liberar explicitamente depois deixa a intenção clara.

### 4.1 Usando no Terminal

```
LABSTUDY>KILL ^Bank, ^BankLog

LABSTUDY>DO ##class(LabStudy.Demo.Bank).OpenAccount("ana", 500)
LABSTUDY>DO ##class(LabStudy.Demo.Bank).OpenAccount("bruno", 100)

LABSTUDY>DO ##class(LabStudy.Demo.Bank).Report()
ana: 500
bruno: 100

LABSTUDY>WRITE $$$ISOK(##class(LabStudy.Demo.Bank).Transfer("ana","bruno",200)), !
1

LABSTUDY>DO ##class(LabStudy.Demo.Bank).Report()
ana: 300
bruno: 300

LABSTUDY>DO $SYSTEM.Status.DisplayError(##class(LabStudy.Demo.Bank).Transfer("ana","bruno",9999))
ERROR #5001: Insufficient funds in ana

LABSTUDY>DO ##class(LabStudy.Demo.Bank).Report()
ana: 300
bruno: 300

LABSTUDY>ZWRITE ^BankLog
^BankLog=1
^BankLog(1)="ana -> bruno : 200"
```

O que observar:

- A transferência inválida **não deixou rastro**: os saldos não mudaram e nada foi acrescentado ao log. Isso é o rollback funcionando.
- `^BankLog=1` (sem subscrito) é o contador mantido pelo `$INCREMENT`; `^BankLog(1)` é a entrada de fato. Uma global pode ter valor no próprio nó **e** nós filhos ao mesmo tempo — assunto do Capítulo 8.

### 4.2 Vendo o rollback com os próprios olhos

```
LABSTUDY>KILL ^Demo

LABSTUDY>SET ^Demo("before") = "existed"

LABSTUDY>TSTART

LABSTUDY>SET ^Demo("during") = "created inside transaction"
LABSTUDY>SET ^Demo("before") = "changed inside transaction"
LABSTUDY>SET localVar = "also set inside transaction"

LABSTUDY>ZWRITE ^Demo
^Demo("before")="changed inside transaction"
^Demo("during")="created inside transaction"

LABSTUDY>TROLLBACK

LABSTUDY>ZWRITE ^Demo
^Demo("before")="existed"

LABSTUDY>WRITE localVar, !
also set inside transaction
```

Três lições numa tela só:

1. A global **criada** dentro da transação sumiu.
2. A global **alterada** voltou ao valor anterior.
3. A **variável local sobreviveu intacta**. Rollback não mexe em memória.

---

## 5. Variações e detalhes

### 5.1 `$INCREMENT`: contador sem trava

```
LABSTUDY>WRITE $INCREMENT(^Seq("patient")), !
1
LABSTUDY>WRITE $INCREMENT(^Seq("patient")), !
2
LABSTUDY>WRITE $INCREMENT(^Seq("patient"), 10), !
12
```

`$INCREMENT(global)` soma 1 (ou o valor do segundo argumento) e devolve o resultado **numa única operação indivisível**. Dois processos chamando ao mesmo tempo recebem números diferentes, garantidamente.

É a solução preferida para contadores, e a razão é simples: ela é mais rápida e mais segura do que ler, somar e gravar com trava.

Um alerta que vale ouro: **`$INCREMENT` não é desfeito por rollback da mesma forma que uma atribuição comum.** O comportamento exato depende da versão e da configuração de journaling: **verificar na documentação oficial**. O importante conceitualmente é que contadores de sequência costumam ter "buracos" — números pulados — e isso é aceitável e esperado. Um número de registro não precisa ser contíguo; precisa ser único.

### 5.2 `TROLLBACK` sem transação aberta

Chamar `TROLLBACK` com `$TLEVEL = 0` não causa erro; simplesmente não faz nada. Mesmo assim, o padrão defensivo é conferir:

```objectscript
if $TLEVEL > 0 {
    trollback
}
```

Já `TCOMMIT` sem transação aberta **gera erro**. Nunca confirme o que não foi iniciado.

### 5.3 Transações e `TRY`/`CATCH`

O casamento natural das transações é com tratamento de exceções, que é o assunto do Capítulo 20. Deixo aqui o padrão, para você reconhecer quando o vir:

```objectscript
ClassMethod DoWork() As %Status
{
    set sc = $$$OK
    set entryLevel = $TLEVEL

    try {
        tstart

        // ... trabalho que pode falhar ...

        tcommit
    }
    catch exception {
        while $TLEVEL > entryLevel {
            trollback 1
        }
        set sc = exception.AsStatus()
    }

    quit sc
}
```

A vantagem: **qualquer** erro inesperado — não só os que você previu — cai no `catch` e provoca o rollback. Sem isso, um erro no meio do caminho pode deixar a transação aberta indefinidamente, segurando travas e journal.

Uma transação esquecida aberta é um problema sério em produção. O `try`/`catch` é a defesa.

### 5.4 Travando um objeto persistente

Objetos persistentes têm métodos próprios de trava, que usam a convenção correta de nomes automaticamente:

```objectscript
set sc = ##class(LabStudy.Patient).%LockId(id)
// ...
set sc = ##class(LabStudy.Patient).%UnlockId(id)
```

Há também `%LockExtent()` e `%UnlockExtent()`, que travam a classe inteira.

Vantagem sobre o `LOCK` manual: você não precisa saber o nome da global de armazenamento, e o nome usado é o mesmo que o próprio IRIS usa internamente ao gravar objetos. Prefira esses métodos quando estiver no mundo dos objetos.

### 5.5 O que uma transação **não** protege

Vale a pena listar, porque a prova gosta de opções erradas plausíveis:

- **Não protege variáveis locais.**
- **Não protege a saída na tela.** O que você escreveu com `WRITE` já foi escrito.
- **Não protege efeitos externos.** Um arquivo criado no disco, um e-mail enviado, uma chamada a outro sistema — nada disso volta atrás.
- **Não protege globais em bases sem journaling.**
- **Não substitui a trava.** Transação garante atomicidade; trava garante isolamento contra outros processos. São problemas diferentes e você costuma precisar dos dois.

Esse último ponto merece destaque: **transação e trava resolvem coisas diferentes.** Uma transação sozinha não impede que outro processo leia ou altere os mesmos dados ao mesmo tempo.

### 5.6 Quanto tempo uma transação deve durar

Regra profissional: **o mais curto possível.**

Enquanto uma transação está aberta, o IRIS acumula journal e mantém travas. Uma transação longa:

- segura travas por muito tempo, criando fila;
- faz o journal crescer;
- aumenta a chance de um rollback caro.

Nunca deixe uma transação aberta esperando entrada do usuário, resposta de outro sistema ou qualquer coisa fora do seu controle. Prepare tudo antes, abra a transação, faça as gravações, feche.

---

## 6. Pegadinhas e erros comuns

**1) Usar `TROLLBACK` sem argumento dentro de um método aninhado.**
Derruba a transação do chamador inteira. Use o padrão do `entryLevel` com `TROLLBACK 1`.

**2) Achar que `TCOMMIT` interno confirma.**
Só o commit mais externo confirma de verdade. Os internos apenas reduzem `$TLEVEL`.

**3) Achar que rollback desfaz variáveis locais.**
Não desfaz. Rollback age sobre o banco.

**4) Achar que a trava impede o acesso.**
A trava é **consultiva**. Quem não olhar a plaquinha, entra assim mesmo.

**5) Travar um nome diferente do que se vai alterar.**
Duas travas com nomes diferentes não se enxergam. Trave o nome da global que você vai mexer.

**6) `LOCK` sem timeout em produção.**
Espera indefinida. Sempre `LOCK +^X(1):5` seguido de `IF '$TEST`.

**7) Esquecer de conferir `$TEST` depois do `LOCK` com timeout.**
O código continua achando que travou, quando não travou. Silencioso e perigoso.

**8) Usar `LOCK` simples em vez de `LOCK +`.**
A forma sem `+` **libera todas as travas anteriores** do processo. Isso pode soltar travas que outra parte do código ainda precisa.

**9) Sair de um método por um caminho de erro deixando travas presas.**
Todo caminho de saída precisa liberar o que adquiriu.

**10) Esperar que `LOCK -` dentro de uma transação libere na hora.**
A liberação fica pendente até o commit ou rollback.

**11) Adquirir travas em ordens diferentes em pontos diferentes do sistema.**
Receita de deadlock. Padronize a ordem.

**12) Deixar transação aberta esperando o usuário.**
Trava e journal crescendo enquanto alguém foi tomar café.

**13) Reaproveitar objetos depois de um rollback.**
As OREFs continuam preenchidas na memória, mas o banco voltou atrás. Descarte e recomece.

**14) `TCOMMIT` sem `TSTART`.**
Gera erro. Confira `$TLEVEL` quando houver dúvida.

**15) Confiar em ler-somar-gravar para gerar sequência.**
Use `$INCREMENT`, que é atômico, ou trave corretamente.

---

## 7. MÃO NA MASSA

> **Atenção — alguns exercícios deste capítulo pedem DUAS sessões de Terminal abertas ao mesmo tempo**, para simular dois processos concorrentes. No caminho por container, abra dois terminais do sistema e rode `docker exec -it iris-study iris session IRIS` em cada um. Na instalação nativa, abra o Terminal duas vezes pelo ícone da bandeja. Em ambas, entre com `ZN "LABSTUDY"`.

---

### Exercício 5.1 — O básico da transação

**a) Enunciado:** No Terminal, sem criar classe nenhuma:

1. Limpe a global `^Test`.
2. Grave `^Test("a") = 1` **fora** de qualquer transação.
3. Abra uma transação, grave `^Test("b") = 2`, altere `^Test("a")` para `99` e crie a variável local `x = "memoria"`.
4. Mostre o estado antes de decidir.
5. Dê rollback e mostre o estado depois, incluindo a variável local.
6. Repita tudo, mas terminando com `TCOMMIT`, e compare.

**b) Dica:** Use `ZWRITE ^Test` para ver a global inteira e `WRITE $TLEVEL` para acompanhar o nível.

**c) Como testar:** Após o rollback, `^Test` deve conter apenas `("a")="1"`. A variável `x` deve sobreviver.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>KILL ^Test, x

LABSTUDY>SET ^Test("a") = 1

LABSTUDY>WRITE $TLEVEL, !
0

LABSTUDY>TSTART

LABSTUDY>WRITE $TLEVEL, !
1

LABSTUDY>SET ^Test("b") = 2
LABSTUDY>SET ^Test("a") = 99
LABSTUDY>SET x = "memoria"

LABSTUDY>ZWRITE ^Test
^Test("a")=99
^Test("b")=2

LABSTUDY>TROLLBACK

LABSTUDY>WRITE $TLEVEL, !
0

LABSTUDY>ZWRITE ^Test
^Test("a")=1

LABSTUDY>WRITE x, !
memoria

LABSTUDY>TSTART
LABSTUDY>SET ^Test("b") = 2
LABSTUDY>SET ^Test("a") = 99
LABSTUDY>TCOMMIT

LABSTUDY>ZWRITE ^Test
^Test("a")=99
^Test("b")=2
```

**Por que cada decisão:**

- **Gravar `^Test("a")` antes da transação** cria um valor "antigo" para o rollback restaurar. Sem isso, você só veria criações desaparecendo, e não veria alterações voltando.
- **A variável local no meio** é o coração do exercício. Ela prova, visualmente, o limite do rollback.
- **Repetir com `TCOMMIT`** fecha o raciocínio: o mesmo código, com um comando diferente no fim, produz o resultado oposto. Nada acontece por acaso.

---

### Exercício 5.2 — Transações aninhadas e `$TLEVEL`

**a) Enunciado:** Crie `LabStudy.Demo.Nested` com três `ClassMethod`:

1. `Outer()` — abre transação, grava `^N(1)`, chama `InnerOk()`, chama `InnerFail()`, e ao final confirma. Imprime `$TLEVEL` em cada etapa.
2. `InnerOk()` — abre transação, grava `^N(2)`, confirma.
3. `InnerFail()` — abre transação, grava `^N(3)`, e desfaz **apenas um nível** com `TROLLBACK 1`.

Rode e observe o resultado. Depois crie uma variante `InnerFailBad()` que use `TROLLBACK` sem argumento, e compare o estrago.

**b) Dica:** Imprima `$TLEVEL` no começo e no fim de cada método.

**c) Como testar:** Com `TROLLBACK 1`, o resultado final deve ter `^N(1)` e `^N(2)`, mas não `^N(3)`. Com `TROLLBACK` sem argumento, **nada** deve sobrar, e o `TCOMMIT` final deve falhar.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Nested.cls`:

```objectscript
/// Shows how nested transactions and $TLEVEL behave.
Class LabStudy.Demo.Nested Extends %RegisteredObject
{

/// Outer transaction that calls two inner ones.
ClassMethod Outer(useBadRollback As %Boolean = 0) As %Status
{
    kill ^N
    write "start   tlevel=", $TLEVEL, !

    tstart
    write "outer   tlevel=", $TLEVEL, !
    set ^N(1) = "outer"

    do ..InnerOk()

    if useBadRollback {
        do ..InnerFailBad()
    } else {
        do ..InnerFail()
    }

    write "before commit tlevel=", $TLEVEL, !

    if $TLEVEL = 0 {
        write "!! the outer transaction is gone; nothing to commit", !
        quit $$$OK
    }

    tcommit
    write "end     tlevel=", $TLEVEL, !
    quit $$$OK
}

/// Inner transaction that succeeds.
ClassMethod InnerOk() As %Status
{
    tstart
    write "  innerOk   tlevel=", $TLEVEL, !
    set ^N(2) = "inner ok"
    tcommit
    quit $$$OK
}

/// Inner transaction that cancels ONLY its own level.
ClassMethod InnerFail() As %Status
{
    set entryLevel = $TLEVEL
    tstart
    write "  innerFail tlevel=", $TLEVEL, !
    set ^N(3) = "inner fail"

    while $TLEVEL > entryLevel {
        trollback 1
    }
    write "  innerFail after rollback 1, tlevel=", $TLEVEL, !
    quit $$$OK
}

/// Inner transaction that destroys everything. Do not do this.
ClassMethod InnerFailBad() As %Status
{
    tstart
    write "  innerBad  tlevel=", $TLEVEL, !
    set ^N(3) = "inner fail"

    trollback
    write "  innerBad after full rollback, tlevel=", $TLEVEL, !
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Nested).Outer()
start   tlevel=0
outer   tlevel=1
  innerOk   tlevel=2
  innerFail tlevel=2
  innerFail after rollback 1, tlevel=1
before commit tlevel=1
end     tlevel=0

LABSTUDY>ZWRITE ^N
^N(1)="outer"
^N(2)="inner ok"

LABSTUDY>DO ##class(LabStudy.Demo.Nested).Outer(1)
start   tlevel=0
outer   tlevel=1
  innerOk   tlevel=2
  innerBad  tlevel=2
  innerBad after full rollback, tlevel=0
before commit tlevel=0
!! the outer transaction is gone; nothing to commit

LABSTUDY>ZWRITE ^N
LABSTUDY>
```

**Por que cada decisão:**

- **Imprimir `$TLEVEL` em cada ponto** transforma um conceito abstrato em algo que você vê acontecer. Note a linha `innerOk tlevel=2`: o commit dela **não** confirmou nada; apenas devolveu o nível a 1.
- **Na primeira execução, `^N(2)` sobreviveu** mesmo tendo sido gravado numa transação interna que "confirmou". Isso prova que quem decidiu foi o commit externo.
- **Na segunda execução, tudo se perdeu.** O `TROLLBACK` sem argumento, de dentro de um método aninhado, desfez também o `^N(1)` que pertencia ao chamador — e o chamador nem sabia. Repare que ele ficou com `$TLEVEL = 0` e nem pôde confirmar. Se não houvesse a conferência de `$TLEVEL`, o `TCOMMIT` teria dado erro.
- **O `while $TLEVEL > entryLevel` em vez de um único `TROLLBACK 1`** é robusto: se o método tivesse aberto dois níveis, ou se algo tivesse aberto níveis extras, ele volta exatamente até onde entrou, nem mais nem menos.

---

### Exercício 5.3 — Travas com dois processos

**a) Enunciado:** Abra **duas** sessões de Terminal, ambas em `LABSTUDY`. Chame-as de **A** e **B**.

1. Em A: adquira uma trava exclusiva incremental sobre `^Res("room1")`.
2. Em B: tente adquirir a mesma trava **sem** timeout e observe que a sessão fica presa. Interrompa com `Ctrl+C`.
3. Em B: tente com timeout de 3 segundos e confira `$TEST`.
4. Em A: libere a trava.
5. Em B: tente de novo com timeout e confirme que agora obtém.
6. Repita o teste usando travas **compartilhadas** (`#"S"`) nas duas sessões e observe que **as duas conseguem** ao mesmo tempo.

**b) Dica:** Antes e depois de cada passo, veja as travas em **System Operation → Locks** no Portal.

**c) Como testar:** No passo 3, `$TEST` deve valer `0`. No passo 5, `1`. No passo 6, as duas sessões devem obter a trava compartilhada.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

**Sessão A:**

```
LABSTUDY>LOCK +^Res("room1")

LABSTUDY>WRITE "A holds the exclusive lock", !
A holds the exclusive lock
```

**Sessão B:**

```
LABSTUDY>LOCK +^Res("room1")
   (a sessão fica parada aqui, esperando; pressione Ctrl+C)

LABSTUDY>LOCK +^Res("room1"):3

LABSTUDY>WRITE $TEST, !
0

LABSTUDY>WRITE "could not get the lock", !
could not get the lock
```

**Sessão A:**

```
LABSTUDY>LOCK -^Res("room1")

LABSTUDY>WRITE "A released", !
A released
```

**Sessão B:**

```
LABSTUDY>LOCK +^Res("room1"):3

LABSTUDY>WRITE $TEST, !
1

LABSTUDY>LOCK -^Res("room1")
```

**Agora o teste com trava compartilhada — sessão A:**

```
LABSTUDY>LOCK +^Res("room1")#"S"

LABSTUDY>WRITE "A holds a SHARED lock", !
A holds a SHARED lock
```

**Sessão B:**

```
LABSTUDY>LOCK +^Res("room1")#"S":3

LABSTUDY>WRITE $TEST, !
1

LABSTUDY>WRITE "B also holds a shared lock at the same time", !
B also holds a shared lock at the same time

LABSTUDY>LOCK +^Res("room1"):3

LABSTUDY>WRITE $TEST, !
0
```

**Por que cada decisão:**

- **O passo 2, com a sessão travando de verdade**, é desconfortável de propósito. Você precisa **sentir** o que um `LOCK` sem timeout faz num sistema real: o processo simplesmente para, sem mensagem, sem erro, para sempre. Depois disso, você nunca mais escreve um `LOCK` sem timeout.
- **`$TEST` só é confiável imediatamente depois do `LOCK` com timeout.** Outros comandos também alteram `$TEST`. Confira na linha seguinte, sempre.
- **O último bloco é a prova do modelo compartilhado:** duas travas `#"S"` convivem, mas uma tentativa **exclusiva** sobre o mesmo nome falha enquanto qualquer compartilhada existir. Leitores convivem; escritor espera.
- **Note que nada disso impediu ninguém de escrever em `^Res("room1")`.** Tente, na sessão B, fazer `SET ^Res("room1") = "x"` com a trava exclusiva de A ativa: **funciona**. Faça esse teste; é a demonstração definitiva de que a trava é consultiva.

---

### Exercício 5.4 — Objetos dentro de transação

**a) Enunciado:** Usando a classe `LabStudy.Patient` do projeto, escreva no Terminal uma sequência que:

1. Limpe os pacientes.
2. Abra uma transação.
3. Crie e grave dois pacientes válidos, guardando os IDs.
4. Confirme que os dois existem com `%ExistsId`.
5. Dê rollback.
6. Confirme que **nenhum** existe mais.
7. Mostre que as OREFs na memória **ainda** têm os dados.

**b) Dica:** Guarde as OREFs em variáveis antes do rollback, para poder inspecioná-las depois.

**c) Como testar:** Depois do rollback, `%ExistsId` deve devolver `0` para ambos, mas `p1.Name` deve continuar respondendo.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>DO ##class(LabStudy.Patient).%KillExtent()

LABSTUDY>TSTART

LABSTUDY>SET p1 = ##class(LabStudy.Patient).%New()
LABSTUDY>SET p1.Name = "Ana Souza", p1.RecordNumber = "REG-100"
LABSTUDY>SET p1.BirthDate = $ZDATEH("1985-02-10", 3)
LABSTUDY>WRITE $$$ISOK(p1.%Save()), " id=", p1.%Id(), !
1 id=1

LABSTUDY>SET p2 = ##class(LabStudy.Patient).%New()
LABSTUDY>SET p2.Name = "Bruno Lima", p2.RecordNumber = "REG-101"
LABSTUDY>SET p2.BirthDate = $ZDATEH("1978-11-03", 3)
LABSTUDY>WRITE $$$ISOK(p2.%Save()), " id=", p2.%Id(), !
1 id=2

LABSTUDY>WRITE ##class(LabStudy.Patient).%ExistsId(1), ##class(LabStudy.Patient).%ExistsId(2), !
11

LABSTUDY>TROLLBACK

LABSTUDY>WRITE ##class(LabStudy.Patient).%ExistsId(1), ##class(LabStudy.Patient).%ExistsId(2), !
00

LABSTUDY>WRITE p1.Name, " / ", p2.Name, !
Ana Souza / Bruno Lima

LABSTUDY>WRITE p1.%Id(), !
1

LABSTUDY>SET back = ##class(LabStudy.Patient).%OpenId(1)
LABSTUDY>WRITE $ISOBJECT(back), !
0
```

**Por que cada decisão:**

- **O `%ExistsId` antes do rollback** prova que os objetos estavam mesmo gravados, e não apenas na memória. Sem esse passo, alguém poderia argumentar que nunca chegaram ao disco.
- **As duas últimas verificações são o ponto do exercício.** `p1.Name` ainda responde e `p1.%Id()` ainda devolve `1` — mas `%OpenId(1)` não encontra nada. A memória e o banco divergiram.
- **Essa divergência é exatamente por que se descarta objetos após rollback.** Continuar usando `p1` daria a impressão de que ele existe. Um `p1.%Save()` depois do rollback criaria um objeto novo, provavelmente com outro ID, e ninguém entenderia por quê.

---

### Exercício 5.5 — PROJETO CONTÍNUO: cadastro atômico e número de registro seguro

**a) Enunciado:** Evolua o sistema do laboratório:

1. Crie `LabStudy.Sequence`, com:
   - `ClassMethod Next(name) As %Integer` — devolve o próximo número da sequência `name`, usando `$INCREMENT` sobre `^LabStudySeq(name)`;
   - `ClassMethod NextWithLock(name) As %Integer` — a versão "manual", com `LOCK` e timeout, que lê, soma e grava. Serve para você comparar as duas abordagens;
   - `ClassMethod Current(name) As %Integer` — apenas lê o valor atual;
   - `ClassMethod NewRecordNumber() As %String` — devolve algo como `REG-000007`, usando `Next("patient")`.
2. Em `LabStudy.Patient`, acrescente:
   - `ClassMethod CreateWithExams(name, birthDate, sex, examList) As %String` — recebe um `%DynamicArray` de exames, e **numa única transação** gera o número de registro, cria o paciente e cria todos os exames. Se qualquer passo falhar, desfaz tudo e devolve `""`.
   - O método deve seguir o padrão do `entryLevel` para não derrubar transações de fora.
3. Em `LabStudy.App`, suba para `"0.6"` e acrescente `ClassMethod SequenceReport()` que mostra o valor atual das sequências.

**b) Dica:** Para formatar o número com zeros à esquerda, use `$TRANSLATE($JUSTIFY(n, 6), " ", "0")` — `$JUSTIFY` alinha à direita num campo de 6 caracteres, preenchendo com espaços, e `$TRANSLATE` troca espaço por zero. Funções de string são o Capítulo 10.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Patient).%KillExtent()
LABSTUDY>DO ##class(LabStudy.Exam).%KillExtent()
LABSTUDY>KILL ^LabStudySeq
LABSTUDY>SET list = [{"testCode":"HGB","value":13.5,"unit":"g/dL"},{"testCode":"GLU","value":92,"unit":"mg/dL"}]
LABSTUDY>SET id = ##class(LabStudy.Patient).CreateWithExams("Maria Silva","1990-05-17","F",list)
LABSTUDY>DO ##class(LabStudy.Patient).Show(id)
LABSTUDY>SET bad = [{"value":10}]
LABSTUDY>SET id2 = ##class(LabStudy.Patient).CreateWithExams("Joao Teste","1980-01-01","M",bad)
LABSTUDY>DO ##class(LabStudy.App).Status()
```

A segunda criação deve falhar **inteira**: nenhum paciente e nenhum exame devem sobrar dela.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Sequence.cls`:

```objectscript
/// Generates sequential numbers safely under concurrency.
Class LabStudy.Sequence Extends %RegisteredObject
{

/// Seconds to wait for a lock.
Parameter LOCKTIMEOUT = 5;

/// Returns the next number of a named sequence.
/// $INCREMENT is atomic: no lock is needed.
ClassMethod Next(name As %String) As %Integer
{
    quit $INCREMENT(^LabStudySeq(name))
}

/// The manual version, kept for comparison.
/// Read, add and write is NOT safe without a lock.
ClassMethod NextWithLock(name As %String) As %Integer
{
    lock +^LabStudySeq(name):..#LOCKTIMEOUT
    if '$TEST {
        quit ""
    }

    set value = $GET(^LabStudySeq(name), 0) + 1
    set ^LabStudySeq(name) = value

    lock -^LabStudySeq(name)
    quit value
}

/// Current value without consuming a number.
ClassMethod Current(name As %String) As %Integer
{
    quit $GET(^LabStudySeq(name), 0)
}

/// Builds a formatted patient record number, e.g. REG-000007.
ClassMethod NewRecordNumber() As %String
{
    set n = ..Next("patient")
    quit "REG-"_$TRANSLATE($JUSTIFY(n, 6), " ", "0")
}

}
```

Acrescente a `src/LabStudy/Patient.cls`:

```objectscript
/// Creates a patient together with all its exams, atomically.
/// examList must be a %DynamicArray of objects with testCode, value and unit.
/// Returns the new patient id, or "" if anything failed.
ClassMethod CreateWithExams(name As %String, birthDate As %String, sex As %String, examList As %DynamicArray) As %String
{
    set entryLevel = $TLEVEL
    tstart

    set patient = ..%New()
    set patient.Name = name
    set patient.BirthDate = $ZDATEH(birthDate, 3)
    set patient.Sex = sex
    set patient.RecordNumber = ##class(LabStudy.Sequence).NewRecordNumber()

    set sc = patient.%Save()
    if $$$ISERR(sc) {
        write "Patient could not be saved:", !
        do $SYSTEM.Status.DisplayError(sc)
        while $TLEVEL > entryLevel { trollback 1 }
        quit ""
    }

    if $ISOBJECT(examList) {
        set iterator = examList.%GetIterator()
        while iterator.%GetNext(.index, .item) {

            if 'item.%IsDefined("testCode") {
                write "Exam ", index, " has no testCode. Cancelling everything.", !
                while $TLEVEL > entryLevel { trollback 1 }
                quit ""
            }

            set exam = ##class(LabStudy.Exam).%New(item.testCode)
            set exam.ResultValue = item.%Get("value", "")
            set exam.Unit = item.%Get("unit", "")
            set exam.Patient = patient

            set sc = exam.%Save()
            if $$$ISERR(sc) {
                write "Exam ", index, " could not be saved:", !
                do $SYSTEM.Status.DisplayError(sc)
                while $TLEVEL > entryLevel { trollback 1 }
                quit ""
            }
        }
    }

    tcommit
    quit patient.%Id()
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "0.6";

/// Shows the current value of the known sequences.
ClassMethod SequenceReport() As %Status
{
    write "Sequence 'patient': ", ##class(LabStudy.Sequence).Current("patient"), !
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.Patient).%KillExtent()
LABSTUDY>DO ##class(LabStudy.Exam).%KillExtent()
LABSTUDY>KILL ^LabStudySeq

LABSTUDY>SET list = [{"testCode":"HGB","value":13.5,"unit":"g/dL"},{"testCode":"GLU","value":92,"unit":"mg/dL"}]

LABSTUDY>SET id = ##class(LabStudy.Patient).CreateWithExams("Maria Silva","1990-05-17","F",list)

LABSTUDY>WRITE id, !
1

LABSTUDY>DO ##class(LabStudy.Patient).Show(id)
------------------------------
Id:      1
Name:    Maria Silva
Record:  REG-000001
Birth:   1990-05-17 (age 36)
Sex:     Female
City:
Created: 2026-08-19 15:10:44
Allergies (0):
Exams (2):
  - HGB: 13.5 g/dL
  - GLU: 92 mg/dL
------------------------------

LABSTUDY>SET bad = [{"value":10}]

LABSTUDY>SET id2 = ##class(LabStudy.Patient).CreateWithExams("Joao Teste","1980-01-01","M",bad)
Exam 0 has no testCode. Cancelling everything.

LABSTUDY>WRITE "[", id2, "]", !
[]

LABSTUDY>DO ##class(LabStudy.App).Status()
Patients:            1
Exams:               2
Patients with exams: 1

LABSTUDY>DO ##class(LabStudy.App).SequenceReport()
Sequence 'patient': 2
```

**Por que cada decisão:**

- **A transação envolve paciente e exames juntos.** Esse é o cenário exato do parágrafo de abertura do capítulo: cadastro pela metade não pode existir. O segundo teste prova que não existe.
- **Repare no `SequenceReport` no final: a sequência está em 2, mas só existe 1 paciente.** O número gerado para o "Joao Teste" foi **consumido e perdido** no rollback. Isso é intencional e correto: `$INCREMENT` garante unicidade, não continuidade. Um sistema que exige numeração sem buracos precisa de outra estratégia — e vai pagar em concorrência por isso. Saber justificar esse trade-off é conhecimento sênior.
- **`Next` com `$INCREMENT` é a versão recomendada; `NextWithLock` está lá para contraste.** Compare as duas: a segunda tem quatro linhas a mais, precisa de timeout, precisa de tratamento de falha, pode vazar trava se algo der errado no meio — e entrega a mesma garantia. Quando existe uma operação atômica pronta, use-a.
- **`NextWithLock` devolve `""` quando não consegue a trava**, em vez de devolver um número errado. Quem chama precisa conferir. Sinalizar a falha é sempre melhor que fingir sucesso.
- **`$ISOBJECT(examList)` antes de iterar** — o método deve funcionar mesmo se ninguém passar exames.
- **O padrão `entryLevel` em todos os caminhos de saída.** São quatro saídas de erro no método, e todas as quatro fazem o mesmo tratamento. Repetição chata, mas correta — e no Capítulo 20, com `try`/`catch`, você verá como reduzir isso a um único lugar.
- **O rollback está ANTES do `quit ""` em todos os caminhos.** Sair do método com transação aberta seria desastroso: a transação ficaria pendurada, segurando journal e travas, até o processo terminar.

---

## 8. Quiz do capítulo

**Q1.** Qual comando confirma definitivamente as alterações de uma transação?

- A) `TSTART`
- B) `TCOMMIT`
- C) `TROLLBACK`
- D) `LOCK`

---

**Q2.** Considere:

```
TSTART
SET ^G(1) = "a"
TSTART
SET ^G(2) = "b"
TCOMMIT
TROLLBACK
```

Qual é o estado final de `^G`?

- A) Contém `^G(1)` e `^G(2)`.
- B) Contém apenas `^G(2)`.
- C) Está vazia: o `TROLLBACK` desfez todos os níveis.
- D) Contém apenas `^G(1)`.

---

**Q3.** Qual é a diferença entre `TROLLBACK` e `TROLLBACK 1`?

- A) Nenhuma.
- B) `TROLLBACK` desfaz todos os níveis e zera `$TLEVEL`; `TROLLBACK 1` desfaz apenas um nível.
- C) `TROLLBACK 1` desfaz tudo; `TROLLBACK` desfaz um nível.
- D) `TROLLBACK 1` confirma o primeiro nível.

---

**Q4.** Dentro de uma transação você fez `SET total = 500` (variável local) e `SET ^Bank("ana") = 500` (global). Após um `TROLLBACK`, o que acontece?

- A) Ambos voltam ao valor anterior.
- B) A global volta ao valor anterior; a variável local mantém `500`.
- C) A variável local volta; a global permanece.
- D) Ambos mantêm `500`.

---

**Q5.** O que significa dizer que o `LOCK` do IRIS é **consultivo**?

- A) Que ele só funciona dentro de transações.
- B) Que ele impede fisicamente qualquer acesso ao dado.
- C) Que ele só protege se todos os processos concordarem em usar a mesma trava; quem não travar acessa o dado assim mesmo.
- D) Que ele precisa ser aprovado pelo administrador.

---

**Q6.** Qual é a diferença entre `LOCK ^A(1)` e `LOCK +^A(1)`?

- A) Nenhuma.
- B) A forma sem `+` libera todas as travas anteriores do processo antes de adquirir esta.
- C) A forma com `+` é compartilhada.
- D) A forma sem `+` tem timeout automático.

---

**Q7.** Depois de `LOCK +^X(1):5`, como saber se a trava foi obtida?

- A) Conferindo `$TEST`.
- B) Conferindo `$TLEVEL`.
- C) Conferindo `$ZERROR`.
- D) Não há como saber.

---

**Q8.** O que acontece com um `LOCK` sem timeout quando a trava já está com outro processo?

- A) Devolve erro imediatamente.
- B) Aguarda indefinidamente.
- C) Aguarda 5 segundos e desiste.
- D) Adquire a trava assim mesmo.

---

**Q9.** Duas sessões adquiriram `LOCK +^R(1)#"S"`. Uma terceira tenta `LOCK +^R(1):3`. O que acontece?

- A) Obtém a trava normalmente.
- B) Falha, porque uma trava exclusiva não convive com travas compartilhadas ativas.
- C) Converte as compartilhadas em exclusiva.
- D) Gera erro de sintaxe.

---

**Q10.** Você fez `LOCK -^A(1)` de dentro de uma transação aberta. Quando a trava é efetivamente liberada?

- A) Imediatamente.
- B) Somente quando a transação terminar, por commit ou rollback.
- C) Ao fim do processo.
- D) Nunca; `LOCK -` não funciona em transação.

---

**Q11.** Qual é a forma recomendada de evitar deadlock?

- A) Usar transações mais longas.
- B) Adquirir sempre as travas na mesma ordem em todo o sistema, e usar timeout.
- C) Nunca usar travas.
- D) Usar apenas travas compartilhadas.

---

**Q12.** Qual é a função recomendada para gerar um contador sequencial seguro sob concorrência?

- A) `SET ^Seq = ^Seq + 1`
- B) `$INCREMENT(^Seq)`
- C) `$ORDER(^Seq)`
- D) `$GET(^Seq)`

---

**Q13.** No `%OpenId(id, concurrency)`, o que significa `concurrency = 0`?

- A) Trava exclusiva máxima.
- B) Nenhuma trava é adquirida.
- C) O objeto é aberto somente leitura.
- D) O objeto é aberto dentro de uma transação.

---

**Q14.** Depois de um `TROLLBACK` que desfez o `%Save()` de um objeto, o que acontece com a OREF na memória?

- A) É automaticamente destruída.
- B) Continua existindo com as propriedades preenchidas, mesmo que o objeto não exista mais no banco.
- C) Passa a devolver erro em qualquer acesso.
- D) Volta ao estado anterior ao `%New()`.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** `TCOMMIT` confirma as alterações da transação.
- **A está errada:** `TSTART` apenas inicia.
- **C está errada:** `TROLLBACK` faz o oposto.
- **D está errada:** `LOCK` trata de concorrência, não de atomicidade.

**Q2 — Resposta: C.**
- **C está certa:** o `TCOMMIT` interno apenas reduziu `$TLEVEL` de 2 para 1, sem confirmar nada. O `TROLLBACK` seguinte, sem argumento, desfez **todos** os níveis.
- **A está errada:** nada foi confirmado, porque o commit externo nunca ocorreu.
- **B está errada:** commits internos não gravam nada de forma independente.
- **D está errada:** o rollback total apagou também o nível externo.

**Q3 — Resposta: B.**
- **B está certa:** essa é exatamente a diferença, e é o motivo de o padrão `entryLevel` existir.
- **A está errada:** o comportamento é bem diferente.
- **C está errada:** inverte os papéis.
- **D está errada:** `TROLLBACK` nunca confirma nada.

**Q4 — Resposta: B.**
- **B está certa:** o rollback age sobre o banco (globais journalizadas), não sobre a memória do processo.
- **A está errada:** variáveis locais não participam da transação.
- **C está errada:** inverte o comportamento.
- **D está errada:** a global é restaurada.

**Q5 — Resposta: C.**
- **C está certa:** a trava é um acordo entre processos. O IRIS não bloqueia fisicamente o acesso ao dado.
- **A está errada:** travas funcionam dentro e fora de transações.
- **B está errada:** justamente o contrário.
- **D está errada:** não há aprovação administrativa envolvida.

**Q6 — Resposta: B.**
- **B está certa:** a forma simples libera as travas anteriores do processo; a forma incremental preserva.
- **A está errada:** a diferença é significativa e pode causar bugs sutis.
- **C está errada:** compartilhamento se pede com `#"S"`.
- **D está errada:** nenhuma das formas tem timeout implícito.

**Q7 — Resposta: A.**
- **A está certa:** `$TEST` vale 1 se a trava foi obtida e 0 se o tempo esgotou. Confira imediatamente após o `LOCK`.
- **B está errada:** `$TLEVEL` conta transações.
- **C está errada:** `$ZERROR` guarda o último erro do processo.
- **D está errada:** é justamente para isso que `$TEST` existe.

**Q8 — Resposta: B.**
- **B está certa:** sem timeout, o processo espera indefinidamente, sem mensagem nenhuma.
- **A está errada:** não há erro; há espera.
- **C está errada:** não existe timeout padrão.
- **D está errada:** ele respeita a trava do outro processo.

**Q9 — Resposta: B.**
- **B está certa:** travas compartilhadas convivem entre si, mas bloqueiam a aquisição de uma exclusiva sobre o mesmo nome.
- **A está errada:** com timeout, `$TEST` valeria 0.
- **C está errada:** não há conversão automática nesse sentido.
- **D está errada:** a sintaxe está correta.

**Q10 — Resposta: B.**
- **B está certa:** dentro de uma transação, a liberação fica pendente até o commit ou rollback, para preservar o isolamento.
- **A está errada:** esse é justamente o comportamento que **não** ocorre.
- **C está errada:** a liberação ocorre ao fim da transação, não do processo.
- **D está errada:** o comando funciona; apenas o efeito é diferido.

**Q11 — Resposta: B.**
- **B está certa:** ordenação consistente elimina o ciclo de espera, e o timeout é a rede de segurança.
- **A está errada:** transações longas pioram a contenção.
- **C está errada:** sem travas, aparecem condições de corrida.
- **D está errada:** operações de escrita exigem exclusividade.

**Q12 — Resposta: B.**
- **B está certa:** `$INCREMENT` é atômico e devolve o novo valor numa única operação indivisível.
- **A está errada:** ler-somar-gravar é a receita clássica da condição de corrida.
- **C está errada:** `$ORDER` percorre subscritos.
- **D está errada:** `$GET` apenas lê.

**Q13 — Resposta: B.**
- **B está certa:** `0` significa abrir sem adquirir trava alguma.
- **A está errada:** os valores altos é que são mais restritivos.
- **C está errada:** o argumento controla travamento, não permissão de escrita.
- **D está errada:** não tem relação com transações.

**Q14 — Resposta: B.**
- **B está certa:** o rollback é do banco. A OREF permanece na memória com os dados, e até `%Id()` pode continuar respondendo — mas `%OpenId()` daquele ID não encontra nada.
- **A está errada:** o IRIS não destrói objetos por causa de rollback.
- **C está errada:** os acessos continuam funcionando normalmente.
- **D está errada:** as propriedades não são revertidas.

---

## 9. Resumo relâmpago

1. **Transação** garante atomicidade: tudo acontece ou nada acontece. **Trava** garante isolamento entre processos. São problemas diferentes.
2. `TSTART` abre, `TCOMMIT` confirma, `TROLLBACK` desfaz. Todos abreviáveis: `TS`, `TC`, `TRO`.
3. **`$TLEVEL`** conta as transações abertas: `0` fora de qualquer uma; `TSTART` soma 1, `TCOMMIT` subtrai 1.
4. Em transações aninhadas, **só o `TCOMMIT` mais externo confirma**. Os internos apenas reduzem `$TLEVEL`.
5. **`TROLLBACK`** desfaz **todos** os níveis e zera `$TLEVEL`. **`TROLLBACK 1`** desfaz **um** nível.
6. Método que abre transação deve guardar `entryLevel = $TLEVEL` e desfazer só até ali, para não derrubar o chamador.
7. Rollback desfaz **globais journalizadas**. Não desfaz variáveis locais, saída na tela, arquivos, e-mails, nem globais em bases sem journal.
8. Depois de um rollback, **descarte os objetos**: a memória e o banco divergiram.
9. `LOCK` é **consultivo**: protege apenas quem concorda em usar a mesma trava, com o mesmo nome. Convenção: trave o nome da global que vai alterar.
10. Use sempre a forma **incremental**: `LOCK +^X` para adquirir, `LOCK -^X` para liberar. A forma sem `+` libera todas as travas anteriores.
11. `LOCK` sozinho, sem argumento, libera **todas** as travas do processo.
12. **Sempre use timeout**: `LOCK +^X:5` e confira **`$TEST`** na linha seguinte.
13. `#"S"` pede trava **compartilhada**: vários leitores convivem, mas nenhuma exclusiva é concedida enquanto houver compartilhadas.
14. Dentro de uma transação, `LOCK -` **não libera na hora**: a liberação fica pendente até o commit ou rollback.
15. **Deadlock** se evita adquirindo as travas sempre na mesma ordem, mais timeout como rede de segurança.
16. **`$INCREMENT(^Seq)`** é atômico e é a forma correta de gerar sequências. Sequências podem ter buracos — isso é normal.
17. `%OpenId(id, concurrency)` aceita de **0** (sem trava) a **4** (mais restritivo). O padrão vem do parâmetro de classe `DEFAULTCONCURRENCY`.
18. Objetos persistentes têm `%LockId()`, `%UnlockId()`, `%LockExtent()` e `%UnlockExtent()`.
19. Transações devem ser **as mais curtas possíveis**. Nunca deixe uma aberta esperando o usuário.

---

## 10. Cartões de memorização

**Frente:** Quais são os três comandos de transação?
**Verso:** `TSTART` (abre), `TCOMMIT` (confirma), `TROLLBACK` (desfaz).

**Frente:** O que `$TLEVEL` indica?
**Verso:** Quantas transações estão abertas no processo. `0` significa nenhuma.

**Frente:** Numa transação aninhada, o que faz o `TCOMMIT` interno?
**Verso:** Apenas reduz `$TLEVEL` em 1. Só o commit mais externo confirma de verdade.

**Frente:** Diferença entre `TROLLBACK` e `TROLLBACK 1`.
**Verso:** Sem argumento desfaz **todos** os níveis e zera `$TLEVEL`. Com `1`, desfaz apenas um nível.

**Frente:** Como escrever um método com transação que não derruba a do chamador?
**Verso:** Guarde `entryLevel = $TLEVEL` antes do `TSTART` e desfaça com `while $TLEVEL > entryLevel { trollback 1 }`.

**Frente:** O rollback desfaz variáveis locais?
**Verso:** Não. Rollback age sobre globais journalizadas, não sobre a memória do processo.

**Frente:** O que o rollback **não** desfaz?
**Verso:** Variáveis locais, saída na tela, arquivos criados, mensagens enviadas e globais em bases sem journaling.

**Frente:** O que significa a trava do IRIS ser "consultiva"?
**Verso:** Ela só protege se todos os processos usarem a mesma trava. Quem não travar acessa o dado assim mesmo.

**Frente:** Qual nome usar ao travar?
**Verso:** O mesmo nome da global que se vai alterar. É a convenção que faz processos diferentes se enxergarem.

**Frente:** Diferença entre `LOCK ^A` e `LOCK +^A`.
**Verso:** A forma sem `+` libera todas as travas anteriores do processo; a incremental preserva.

**Frente:** O que faz `LOCK` sozinho, sem argumento?
**Verso:** Libera todas as travas do processo.

**Frente:** Como pedir uma trava com espera máxima de 5 segundos e saber se conseguiu?
**Verso:** `LOCK +^X(1):5` e, logo depois, conferir `$TEST` (1 = conseguiu, 0 = tempo esgotou).

**Frente:** O que acontece num `LOCK` sem timeout se a trava estiver ocupada?
**Verso:** O processo espera indefinidamente, sem mensagem alguma.

**Frente:** Como pedir uma trava compartilhada?
**Verso:** `LOCK +^X(1)#"S"`. Várias compartilhadas convivem; nenhuma exclusiva é concedida enquanto elas existirem.

**Frente:** Quando uma trava liberada dentro de uma transação sai de fato?
**Verso:** Só no `TCOMMIT` ou `TROLLBACK`. A liberação fica pendente até lá.

**Frente:** Como evitar deadlock?
**Verso:** Adquirindo as travas sempre na mesma ordem em todo o sistema, e usando timeout.

**Frente:** Qual a forma segura de gerar um contador sequencial?
**Verso:** `$INCREMENT(^Seq(nome))` — atômico, dispensa trava.

**Frente:** Sequências geradas com `$INCREMENT` podem pular números?
**Verso:** Podem, sobretudo se houver rollback. Elas garantem unicidade, não continuidade.

**Frente:** O que significa `concurrency = 0` no `%OpenId()`?
**Verso:** Abrir sem adquirir nenhuma trava. Valores maiores são progressivamente mais restritivos, até 4.

**Frente:** Quais métodos travam um objeto persistente?
**Verso:** `%LockId(id)` / `%UnlockId(id)` e, para a classe inteira, `%LockExtent()` / `%UnlockExtent()`.

---

Digite CONTINUAR para o próximo capítulo.
