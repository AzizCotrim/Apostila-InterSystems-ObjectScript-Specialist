# Apostila InterSystems ObjectScript Specialist
## Capítulo 6 — T2.2 Tracks Application Data (Rastreando dados da aplicação)

> Ainda em **T2 — Basic Programming**. No capítulo anterior você garantiu que as mudanças aconteçam de forma íntegra. Agora você vai garantir que elas fiquem **registradas**: quem mudou, o quê, quando e a partir de qual valor.

---

## 1. Objetivo do capítulo

Ao terminar este capítulo, você será capaz de:

1. Explicar a diferença entre **journal**, **auditoria** e **trilha de aplicação** — três coisas que muita gente confunde.
2. Identificar **quem** e **quando** com as variáveis especiais `$USERNAME`, `$ROLES`, `$JOB`, `$HOROLOG`, `$ZDATETIME` e `$NAMESPACE`.
3. Escrever uma **trilha de auditoria própria**, em global ou em classe persistente.
4. Declarar e usar **triggers** de SQL: `Event`, `Time`, `Foreach`, `Order`.
5. Ler valores dentro de um trigger com `{Campo}`, `{Campo*O}` (antigo) e `{Campo*N}` (novo).
6. Usar os pseudocampos `{%%ID}`, `{%%TABLENAME}`, `{%%CLASSNAME}` e `{%%OPERATION}`.
7. **Cancelar** uma operação de dentro de um trigger `BEFORE` usando `%ok` e `%msg`.
8. Entender por que um trigger comum **não dispara** num `%Save()` de objeto, e como fazer com que dispare.
9. Comparar **trigger** e **callback** (`%OnAfterSave`) e escolher o certo para cada caso.
10. Escrever no **log do sistema** (`messages.log`) e registrar **eventos de auditoria** do IRIS.
11. Entender o papel do **journal** como registro de mudanças e onde consultá-lo.
12. Evoluir o projeto: trilha de auditoria completa de pacientes e histórico de resultados de exames.

---

## 2. O conceito em linguagem de gente

### 2.1 Três cadernos diferentes

No laboratório existem três cadernos de registro, com finalidades distintas. Confundi-los é o erro conceitual mais comum deste tópico.

**Caderno 1 — o do técnico de manutenção: o journal.**

É o registro técnico de toda alteração feita nos armários, escrito automaticamente pelo próprio IRIS. Ele existe para uma finalidade específica: **poder desfazer e poder recuperar**. É ele que permite o rollback do capítulo anterior e a restauração de um banco após uma queda.

Ele é técnico, volumoso, rotativo (arquivos antigos são apagados) e não foi feito para consulta humana no dia a dia. Ninguém pergunta ao journal "quem alterou a ficha da Maria em março?".

**Caderno 2 — o do segurança da portaria: a auditoria do IRIS.**

É o registro de **eventos de segurança e de sistema**: quem fez login, quem falhou ao logar, quem mudou uma senha, quem apagou um namespace. Ele é configurado pelo administrador e vive numa base própria do IRIS.

Você pode acrescentar eventos seus a esse caderno, e isso é útil quando o evento é de natureza institucional ("relatório confidencial exportado").

**Caderno 3 — o do próprio laboratório: a trilha da aplicação.**

É o registro que **você escreve**, com a linguagem do negócio: *"o usuário `recepcao02`, às 14h03, alterou o resultado do exame 4711 de 13,5 para 14,2"*.

Esse é o caderno que o auditor do laboratório quer ler. Ele é seu: você decide o formato, o que entra, quanto tempo fica guardado.

**A regra de decisão:**

| Preciso de... | Uso |
|---|---|
| desfazer/recuperar dados após falha | **journal** (automático, não se programa) |
| registrar login, permissões, ações administrativas | **auditoria do IRIS** |
| registrar mudanças de negócio, com significado para o usuário | **trilha da aplicação** (você escreve) |

Este capítulo trata sobretudo do terceiro caderno, que é o que aparece no dia a dia de um desenvolvedor.

### 2.2 O que uma boa trilha registra

Toda entrada de trilha responde a cinco perguntas. Se faltar uma, a trilha é fraca:

1. **Quem?** — o usuário e o processo.
2. **Quando?** — data e hora com precisão.
3. **O quê?** — qual registro, identificado sem ambiguidade.
4. **Que operação?** — criação, alteração ou exclusão.
5. **De que para quê?** — o valor antigo e o novo.

A quinta é a mais esquecida e a mais valiosa. Saber que "alguém alterou o resultado" ajuda pouco; saber que "alterou de 13,5 para 140" muda tudo.

### 2.3 O que é um trigger

Você já conhece os **callbacks** do Capítulo 3: métodos com nome combinado que o IRIS chama sozinho quando um objeto é gravado ou apagado.

O **trigger** (gatilho) é a mesma ideia, mas **do lado do SQL**. É um pedaço de código pendurado na **tabela**, que dispara quando uma linha é inserida, alterada ou apagada.

Analogia: o callback é uma instrução escrita no verso da ficha ("ao arquivar esta ficha, faça isto"). O trigger é uma campainha instalada **na gaveta**: toca sempre que alguém mexe ali, não importa quem seja nem por qual porta entrou.

E aqui está a diferença prática mais importante entre os dois:

- O **callback** só dispara quando o dado passa pelo **mundo dos objetos** (`%Save()`, `%DeleteId()`).
- O **trigger** dispara quando o dado passa pelo **mundo do SQL** (`INSERT`, `UPDATE`, `DELETE`).

E como o IRIS deixa você usar as duas portas para os mesmos dados, um `INSERT` de SQL **não** dispara o `%OnAfterSave`, e um `%Save()` de objeto **não** dispara o trigger — a menos que você peça explicitamente que dispare, como veremos.

Essa assimetria é uma das perguntas favoritas da prova, e é também uma fonte real de bugs: você escreve a auditoria no callback, alguém importa dados por SQL, e a auditoria fica com um buraco.

### 2.4 Antes ou depois

Um trigger pode disparar **antes** ou **depois** da operação, e a escolha muda o que ele consegue fazer:

- **`BEFORE`** — dispara antes. Ainda dá tempo de **impedir** a operação. É o lugar de uma regra de veto: *"resultado de exame não pode ser alterado depois de liberado"*.
- **`AFTER`** — dispara depois de a operação ter acontecido. Já não dá para impedir, mas você tem certeza de que ocorreu. É o lugar da **trilha de auditoria**: só registre o que de fato aconteceu.

Regra de bolso: **veto no `BEFORE`, registro no `AFTER`.**

---

## 3. A sintaxe explicada

### 3.1 Quem e quando: as variáveis de contexto

| Variável | O que devolve |
|---|---|
| `$USERNAME` | o nome do usuário conectado |
| `$ROLES` | os papéis (permissões) que o usuário tem no momento |
| `$JOB` | o número do processo do sistema operacional |
| `$HOROLOG` | data e hora internas, no formato `dias,segundos` |
| `$ZDATETIME($HOROLOG, 3)` | data e hora legíveis, `AAAA-MM-DD HH:MM:SS` |
| `$NAMESPACE` | o namespace atual |
| `$ZVERSION` | a versão do IRIS em execução |
| `$IO` | o dispositivo de entrada/saída atual |

```
LABSTUDY>WRITE $USERNAME, !
_SYSTEM

LABSTUDY>WRITE $JOB, !
12345

LABSTUDY>WRITE $HOROLOG, !
67352,52011

LABSTUDY>WRITE $ZDATETIME($HOROLOG, 3), !
2026-08-19 14:26:51

LABSTUDY>WRITE $ROLES, !
%All
```

Uma entrada de trilha mínima e decente combina `$USERNAME`, `$JOB` e `$ZDATETIME($HOROLOG, 3)`.

Um detalhe importante sobre precisão: `$HOROLOG` tem resolução de **um segundo**. Se você precisar distinguir dois eventos no mesmo segundo, use `$ZTIMESTAMP`, que traz fração de segundo e está em UTC, ou combine o carimbo com um contador `$INCREMENT`.

### 3.2 A forma geral de um trigger

```
Trigger NomeDoTrigger [ Event = EVENTO, Time = MOMENTO, Foreach = ESCOPO, Order = N ]
{
    ... código ObjectScript ...
}
```

- **`Trigger`** — palavra fixa, no lugar de `Property` ou `Method`. **Obrigatória.**
- **`NomeDoTrigger`** — o nome, único dentro da classe. **Obrigatório.**
- **`[ ... ]`** — as palavras-chave, entre colchetes.
- **`{ ... }`** — o corpo, em ObjectScript. **Obrigatório.**
- Triggers **não** terminam com ponto e vírgula.

As palavras-chave, uma a uma:

**`Event`** — quando dispara. Valores: `INSERT`, `UPDATE`, `DELETE`. Podem ser combinados com barra:

```objectscript
Trigger T1 [ Event = INSERT ]
Trigger T2 [ Event = INSERT/UPDATE ]
Trigger T3 [ Event = INSERT/UPDATE/DELETE ]
```

**`Time`** — `BEFORE` ou `AFTER`. O padrão, se você omitir, é `BEFORE`. Escreva sempre explicitamente: código que depende de um padrão implícito é código que confunde.

**`Foreach`** — o escopo do disparo. É a palavra-chave mais importante e a menos conhecida:

| Valor | Quando dispara |
|---|---|
| `row` | uma vez por linha, apenas em operações **SQL** |
| `row/object` | uma vez por linha, em operações SQL **e também** em `%Save()`/`%DeleteId()` de objeto |
| `statement` | uma vez por comando SQL, independentemente de quantas linhas ele afetou |

O padrão é **`row`**. Ou seja: **por padrão, um trigger NÃO dispara quando você grava um objeto com `%Save()`.**

Se a intenção é auditar o dado independentemente da porta de entrada — que é quase sempre a intenção —, você **precisa** de `Foreach = row/object`.

**`Order`** — quando há vários triggers para o mesmo evento e momento, define a ordem de execução (menor primeiro).

### 3.3 Lendo valores dentro do trigger

Dentro do corpo do trigger, campos da tabela são referenciados **entre chaves**:

| Notação | Significado |
|---|---|
| `{Nome}` | o valor do campo `Nome` |
| `{Nome*O}` | o valor **antigo** (*old*), disponível em `UPDATE` e `DELETE` |
| `{Nome*N}` | o valor **novo** (*new*), disponível em `INSERT` e `UPDATE` |
| `{Nome*C}` | indicador de que o campo **mudou** (*changed*) nesta operação |

E há pseudocampos, todos com dois sinais de porcentagem:

| Pseudocampo | Significado |
|---|---|
| `{%%ID}` | o identificador da linha |
| `{%%TABLENAME}` | o nome da tabela |
| `{%%CLASSNAME}` | o nome da classe |
| `{%%OPERATION}` | a operação em curso: `INSERT`, `UPDATE` ou `DELETE` |

Exemplo:

```objectscript
Trigger LogChange [ Event = UPDATE, Time = AFTER, Foreach = row/object ]
{
    if {Name*C} {
        set ^AuditLog($INCREMENT(^AuditLog)) =
            {%%ID}_" Name: "_{Name*O}_" -> "_{Name*N}
    }
}
```

Repare que `{Name*C}` permite registrar **só** o que mudou, em vez de anotar todos os campos a cada alteração.

O conjunto exato de notações disponíveis varia um pouco conforme o tipo de trigger e a versão: **verificar na documentação oficial** quando precisar de uma variação menos comum.

### 3.4 Cancelando a operação: `%ok` e `%msg`

Dentro de um trigger `BEFORE`, duas variáveis especiais estão disponíveis:

- **`%ok`** — comece valendo `1`. Se você atribuir `0`, a operação é **cancelada**.
- **`%msg`** — a mensagem de erro que será apresentada.

```objectscript
Trigger BlockNegative [ Event = INSERT/UPDATE, Time = BEFORE, Foreach = row/object ]
{
    if {ResultValue} < 0 {
        set %ok = 0
        set %msg = "ResultValue cannot be negative"
    }
}
```

Com `Foreach = row/object`, esse veto vale tanto para um `INSERT` de SQL quanto para um `%Save()` de objeto — e, neste segundo caso, o `%Save()` devolve um `%Status` de erro carregando o `%msg`.

Note que `%ok` e `%msg` **só fazem sentido em triggers `BEFORE`**. Num `AFTER`, a operação já aconteceu.

### 3.5 Trigger ou callback? A tabela de decisão

| Situação | Escolha |
|---|---|
| Regra que precisa valer para SQL **e** objetos | **Trigger** com `Foreach = row/object` |
| Regra que envolve outros objetos relacionados, coleções, streams | **Callback** (`%OnBeforeSave`, `%OnValidateObject`) |
| Auditoria de mudança de campo, com valor antigo e novo | **Trigger** `AFTER` (o valor antigo é fácil de obter) |
| Normalização de dados antes de gravar | **Callback** `%OnBeforeSave` |
| Bloquear a operação com mensagem | Ambos servem: `%ok`/`%msg` no trigger, `$$$ERROR` no callback |

O valor antigo merece um comentário. Num trigger, `{Campo*O}` entrega o valor anterior de graça. Num callback, obter o valor anterior exige reabrir o objeto do disco antes de gravar — mais trabalho e mais custo. É a razão principal para preferir trigger em auditoria de campos.

### 3.6 Escrevendo no log do sistema

O IRIS mantém um arquivo `messages.log` (também chamado de console log), onde ele registra eventos de operação. Você pode escrever nele:

```objectscript
do ##class(%SYS.System).WriteToConsoleLog("LabStudy: import finished")
```

Use com parcimônia. Esse arquivo é lido por administradores para diagnosticar o sistema; enchê-lo de mensagens de aplicação atrapalha quem precisa dele. Reserve para eventos raros e importantes: falha de integração, início e fim de processos longos, condições anormais.

Existem também argumentos adicionais para indicar severidade e origem da mensagem: **verificar na documentação oficial**.

### 3.7 Registrando um evento de auditoria do IRIS

Para gravar um evento no caderno do segurança, o caminho tem duas etapas:

**Etapa 1 — o administrador cria o evento** no Portal, em **System Administration → Security → Auditing → Configure User Audit Events**. Um evento é identificado por três partes: **Source**, **Type** e **Name**.

**Etapa 2 — a aplicação dispara o evento:**

```objectscript
do $SYSTEM.Security.Audit("LabStudy", "Report", "Exported", eventData, description)
```

O evento passa a aparecer em **System Administration → Security → Auditing → View Audit Database**.

A lista completa de argumentos e as opções de configuração variam por versão: **verificar na documentação oficial**. O que a prova cobra é o conceito: eventos de auditoria do IRIS precisam ser **declarados antes** de serem usados, e ficam numa base separada, com controle de acesso próprio.

Quando usar isso em vez da trilha própria? Quando o evento for de natureza institucional e precisar da proteção e da retenção controlada da base de auditoria. Para o registro de negócio do dia a dia, a trilha própria é mais prática e mais consultável.

### 3.8 O journal

O journal é escrito automaticamente para toda alteração em globais de bancos com journaling ativado. Você não o programa; você o consulta quando precisa.

No Portal: **System Operation → Journals**. Ali é possível ver os arquivos de journal e examinar os registros.

Programaticamente, existe o pacote `%SYS.Journal`, com classes para abrir um arquivo de journal e percorrer seus registros. Isso é território de administração e recuperação, não de aplicação. Para a prova, saiba:

- O journal existe para **rollback e recuperação**, não para auditoria de negócio.
- Alterações em bancos **sem journaling** não podem ser desfeitas por rollback.
- O journal é **rotativo**: arquivos antigos são removidos conforme a política configurada. Não conte com ele para guardar histórico de longo prazo.

---

## 4. Exemplo comentado

Uma classe com trilha completa, usando triggers:

Arquivo `src/LabStudy/Demo/Product2.cls`:

```objectscript
/// Demonstrates SQL triggers used to build an application audit trail.
Class LabStudy.Demo.Product2 Extends %Persistent [ SqlTableName = PRODUCT2 ]
{

Property Code As %String(MAXLEN = 20) [ Required ];

Property Description As %String(MAXLEN = 200);

Property Price As %Numeric(SCALE = 2) [ InitialExpression = 0 ];

Property Discontinued As %Boolean [ InitialExpression = 0 ];

/// Refuses a negative price, for SQL and for object saves alike.
Trigger CheckPrice [ Event = INSERT/UPDATE, Time = BEFORE, Foreach = row/object ]
{
    if {Price} < 0 {
        set %ok = 0
        set %msg = "Price cannot be negative"
    }
}

/// Records the creation of a product.
Trigger AuditInsert [ Event = INSERT, Time = AFTER, Foreach = row/object, Order = 1 ]
{
    do ##class(LabStudy.Demo.Product2).WriteAudit({%%TABLENAME}, {%%ID}, "INSERT", "", {Code})
}

/// Records every field that actually changed.
Trigger AuditUpdate [ Event = UPDATE, Time = AFTER, Foreach = row/object, Order = 1 ]
{
    if {Code*C} {
        do ##class(LabStudy.Demo.Product2).WriteAudit({%%TABLENAME}, {%%ID}, "UPDATE Code", {Code*O}, {Code*N})
    }
    if {Price*C} {
        do ##class(LabStudy.Demo.Product2).WriteAudit({%%TABLENAME}, {%%ID}, "UPDATE Price", {Price*O}, {Price*N})
    }
    if {Discontinued*C} {
        do ##class(LabStudy.Demo.Product2).WriteAudit({%%TABLENAME}, {%%ID}, "UPDATE Discontinued", {Discontinued*O}, {Discontinued*N})
    }
}

/// Records the removal, keeping the last known code.
Trigger AuditDelete [ Event = DELETE, Time = AFTER, Foreach = row/object, Order = 1 ]
{
    do ##class(LabStudy.Demo.Product2).WriteAudit({%%TABLENAME}, {%%ID}, "DELETE", {Code*O}, "")
}

/// Writes one entry in the application audit trail.
ClassMethod WriteAudit(table As %String, id As %String, operation As %String, oldValue As %String = "", newValue As %String = "") As %Status
{
    set seq = $INCREMENT(^LabStudyAudit)

    set ^LabStudyAudit(seq, "when")      = $ZDATETIME($HOROLOG, 3)
    set ^LabStudyAudit(seq, "user")      = $USERNAME
    set ^LabStudyAudit(seq, "job")       = $JOB
    set ^LabStudyAudit(seq, "namespace") = $NAMESPACE
    set ^LabStudyAudit(seq, "table")     = table
    set ^LabStudyAudit(seq, "id")        = id
    set ^LabStudyAudit(seq, "operation") = operation
    set ^LabStudyAudit(seq, "old")       = oldValue
    set ^LabStudyAudit(seq, "new")       = newValue

    quit $$$OK
}

/// Prints the audit trail in a readable form.
ClassMethod PrintAudit() As %Status
{
    set seq = ""
    for {
        set seq = $ORDER(^LabStudyAudit(seq))
        quit:seq=""

        write ^LabStudyAudit(seq, "when"), " | "
        write ^LabStudyAudit(seq, "user"), " | "
        write ^LabStudyAudit(seq, "table"), "#", ^LabStudyAudit(seq, "id"), " | "
        write ^LabStudyAudit(seq, "operation")

        if ^LabStudyAudit(seq, "old") '= "" {
            write " | from [", ^LabStudyAudit(seq, "old"), "]"
        }
        if ^LabStudyAudit(seq, "new") '= "" {
            write " | to [", ^LabStudyAudit(seq, "new"), "]"
        }
        write !
    }
    quit $$$OK
}

}
```

Comentando as decisões:

- **`Foreach = row/object` em todos os triggers.** Sem isso, gravar um produto com `%Save()` não dispararia nada, e a trilha teria buracos exatamente onde a aplicação escreve mais.
- **Veto no `BEFORE`, registro no `AFTER`.** O `CheckPrice` precisa poder impedir; os de auditoria precisam registrar o que de fato aconteceu.
- **Um trigger por evento, em vez de um só com `Event = INSERT/UPDATE/DELETE`.** Cada operação precisa de tratamento diferente — no `DELETE` só existe valor antigo, no `INSERT` só existe novo. Separar deixa cada trigger simples e legível.
- **`{Campo*C}` antes de registrar.** Sem isso, um `UPDATE` que só mexeu no preço geraria três entradas de auditoria, duas delas dizendo "mudou de X para X". Ruído em trilha de auditoria é tão nocivo quanto ausência.
- **O trigger chama um `ClassMethod` em vez de escrever direto.** Isso mantém o corpo do trigger curto e concentra o formato da trilha num lugar só. Se amanhã a trilha virar uma classe persistente em vez de uma global, muda-se um método.
- **A global de auditoria usa subscritos nomeados** (`"when"`, `"user"`...), em vez de campos posicionais separados por vírgula. Fica autoexplicativa ao inspecionar com `ZWRITE` — e você agradecerá numa madrugada de diagnóstico.
- **`$INCREMENT(^LabStudyAudit)`** garante numeração única mesmo com muitos processos escrevendo ao mesmo tempo, sem precisar de trava.

### 4.1 Usando no Terminal

```
LABSTUDY>KILL ^LabStudyAudit
LABSTUDY>DO ##class(LabStudy.Demo.Product2).%KillExtent()

LABSTUDY>SET p = ##class(LabStudy.Demo.Product2).%New()
LABSTUDY>SET p.Code = "GLV", p.Description = "Gloves", p.Price = 25.50
LABSTUDY>WRITE $$$ISOK(p.%Save()), !
1

LABSTUDY>SET p.Price = 29.90
LABSTUDY>WRITE $$$ISOK(p.%Save()), !
1

LABSTUDY>SET p.Price = -5
LABSTUDY>DO $SYSTEM.Status.DisplayError(p.%Save())
ERROR #5001: Price cannot be negative

LABSTUDY>SET p.Price = 31.00, p.Discontinued = 1
LABSTUDY>WRITE $$$ISOK(p.%Save()), !
1

LABSTUDY>DO ##class(LabStudy.Demo.Product2).PrintAudit()
2026-08-19 14:31:02 | _SYSTEM | PRODUCT2#1 | INSERT | to [GLV]
2026-08-19 14:31:19 | _SYSTEM | PRODUCT2#1 | UPDATE Price | from [25.5] | to [29.9]
2026-08-19 14:31:44 | _SYSTEM | PRODUCT2#1 | UPDATE Price | from [29.9] | to [31]
2026-08-19 14:31:44 | _SYSTEM | PRODUCT2#1 | UPDATE Discontinued | from [0] | to [1]
```

O que observar:

- **A tentativa com preço negativo não deixou entrada nenhuma.** O trigger `BEFORE` cancelou antes que a operação existisse, e portanto os triggers `AFTER` nunca dispararam. Trilha de auditoria registra fatos, não intenções.
- **A quarta gravação alterou dois campos e gerou duas entradas**, com o mesmo carimbo de tempo. A descrição está por campo, o que é o formato mais útil para consulta depois.
- **O campo `Description` nunca apareceu na trilha** — porque não escrevemos um trigger para ele. Trilha é uma escolha deliberada de o que vale a pena registrar, não um despejo automático de tudo.

### 4.2 Provando que o trigger dispara por SQL também

No Portal, em **System Explorer → SQL**, namespace `LABSTUDY`:

```sql
UPDATE LabStudy.PRODUCT2 SET Price = 35 WHERE ID = 1
```

E de volta ao Terminal:

```
LABSTUDY>DO ##class(LabStudy.Demo.Product2).PrintAudit()
...
2026-08-19 14:35:10 | _SYSTEM | PRODUCT2#1 | UPDATE Price | from [31] | to [35]
```

A entrada apareceu, embora nenhum código ObjectScript de aplicação tenha sido chamado. Essa é a vantagem do trigger sobre o callback: **a campainha está na gaveta, não na ficha.**

---

## 5. Variações e detalhes

### 5.1 Trilha em classe persistente, em vez de global

Guardar a trilha numa classe persistente traz vantagens grandes: você ganha SQL, índices e consulta pelo Portal sem escrever nada.

```objectscript
Class LabStudy.AuditEntry Extends %Persistent [ SqlTableName = AUDIT_ENTRY ]
{
Property EventTime As %TimeStamp [ InitialExpression = {$ZDATETIME($HOROLOG, 3)} ];
Property UserName As %String(MAXLEN = 64) [ InitialExpression = {$USERNAME} ];
Property JobNumber As %String(MAXLEN = 20) [ InitialExpression = {$JOB} ];
Property TableName As %String(MAXLEN = 100);
Property RecordId As %String(MAXLEN = 50);
Property Operation As %String(MAXLEN = 30);
Property FieldName As %String(MAXLEN = 100);
Property OldValue As %String(MAXLEN = 500);
Property NewValue As %String(MAXLEN = 500);

Index TableRecordIdx On (TableName, RecordId);
Index EventTimeIdx On EventTime;
}
```

Repare que `EventTime`, `UserName` e `JobNumber` usam `InitialExpression`: quem grava a entrada não precisa se lembrar de preenchê-los.

O custo dessa escolha é desempenho: gravar um objeto persistente é mais caro que gravar uma global crua. Em sistemas com volume muito alto de auditoria, a global vence. Para a maioria dos casos, a classe persistente é a escolha certa, porque o valor de uma trilha está em **conseguir consultá-la**.

**Um cuidado sério:** se a classe de auditoria tiver ela própria triggers de auditoria, você cria um laço infinito. A classe de trilha nunca deve ser auditada por si mesma.

### 5.2 Trilha e transações

Este ponto é sutil e importante.

Se a trilha é escrita **dentro** da mesma transação da operação, um rollback **apaga a trilha junto**. Isso pode ser exatamente o que você quer (operação cancelada não deve aparecer como se tivesse acontecido) — ou exatamente o que você não quer (registrar tentativas fracassadas é justamente o interesse de uma auditoria de segurança).

Decida conscientemente:

- **Trilha do que aconteceu** → dentro da transação. Se desfez, a trilha some junto e a história fica coerente.
- **Trilha do que foi tentado** → precisa sobreviver ao rollback, e para isso o registro deve ir para um lugar que não participe da transação, ou ser escrito depois dela.

Não existe resposta única. Existe decisão consciente.

### 5.3 Guardar o objeto inteiro em JSON

Uma técnica prática, agora que você conhece o Capítulo 4: em vez de auditar campo a campo, guarde uma fotografia completa do objeto em JSON a cada mudança.

```objectscript
Method %OnAfterSave(insert As %Boolean) As %Status [ Private, ServerOnly = 1 ]
{
    set snapshot = ""
    set sc = ..%JSONExportToString(.snapshot)

    set entry = ##class(LabStudy.AuditEntry).%New()
    set entry.TableName = ..%ClassName(1)
    set entry.RecordId = ..%Id()
    set entry.Operation = $SELECT(insert: "INSERT", 1: "UPDATE")
    set entry.NewValue = snapshot
    quit entry.%Save()
}
```

Vantagem: você não precisa prever quais campos serão interessantes no futuro; a fotografia tem tudo. Desvantagem: ocupa muito mais espaço e comparar duas versões exige trabalho extra.

Escolha por volume: poucos registros críticos e muito valiosos → fotografia. Muitos registros com poucos campos sensíveis → campo a campo.

### 5.4 Trigger em classe herdada

Triggers são herdados pelas subclasses, como qualquer outro membro. Uma subclasse pode declarar um trigger de mesmo nome para substituir o herdado.

Se você tem uma hierarquia de classes auditáveis, uma boa estratégia é colocar os triggers numa classe base e fazer todas herdarem dela. Com `{%%TABLENAME}` e `{%%CLASSNAME}` no corpo, o mesmo trigger funciona corretamente para todas as filhas, identificando cada uma.

### 5.5 Quando NÃO auditar

Auditoria tem custo: espaço em disco, tempo em cada gravação e ruído na consulta. Vale a pena listar o que normalmente **não** se audita:

- Tabelas de configuração raramente alteradas e sem relevância legal.
- Dados temporários e de rascunho.
- Campos calculados e derivados (audite a origem, não a consequência).
- Leituras — a menos que haja exigência legal específica, como no caso de dados de saúde em algumas jurisdições.

E o oposto: campos que **sempre** merecem trilha são resultados clínicos, valores financeiros, permissões e status que mudam o que o sistema permite fazer.

---

## 6. Pegadinhas e erros comuns

**1) Esquecer `Foreach = row/object`.**
O trigger não dispara em `%Save()`, e você só descobre quando alguém pergunta por que metade das alterações não aparece na trilha. **O padrão é `row`, que é só SQL.**

**2) Achar que `%OnAfterSave` cobre tudo.**
Não cobre operações feitas por SQL. Se o dado tem duas portas, a auditoria tem que estar na gaveta.

**3) Usar `%ok` e `%msg` num trigger `AFTER`.**
Não têm efeito: a operação já aconteceu. Veto só no `BEFORE`.

**4) Não conferir `{Campo*C}` num trigger de `UPDATE`.**
Gera entradas de auditoria dizendo "mudou de X para X". Ruído puro.

**5) Confundir `{Campo*O}` com `{Campo*N}`.**
`O` de *old* (antigo), `N` de *new* (novo). Em `INSERT` não existe antigo; em `DELETE` não existe novo.

**6) Auditar a própria classe de auditoria.**
Laço infinito. A classe de trilha jamais deve ter triggers que escrevem em si mesma.

**7) Escrever a trilha em `messages.log`.**
Esse arquivo é do administrador. Trilha de negócio vai para a sua estrutura própria.

**8) Confundir journal com auditoria.**
Journal existe para rollback e recuperação, é rotativo e técnico. Não é histórico de negócio.

**9) Esperar que um evento de auditoria do IRIS funcione sem ter sido declarado.**
Eventos de usuário precisam ser criados antes no Portal.

**10) Guardar valor antigo sem guardar quem mudou.**
Uma trilha sem `$USERNAME` responde metade das perguntas.

**11) Confiar na precisão de `$HOROLOG` para ordenar eventos do mesmo segundo.**
A resolução é de um segundo. Para ordenação fina, mantenha um número de sequência com `$INCREMENT`.

**12) Escrever trilha dentro da transação sem pensar nas consequências.**
Um rollback apaga a trilha junto. Às vezes é o desejado; decida, não deixe acontecer por acaso.

**13) Auditar tudo, sempre.**
Trilha inflada é trilha que ninguém lê. Escolha o que importa.

**14) Colocar lógica pesada dentro de um trigger.**
O trigger roda em toda gravação. Código lento ali degrada o sistema inteiro. Mantenha triggers curtos.

---

## 7. MÃO NA MASSA

---

### Exercício 6.1 — Uma função de trilha do zero

**a) Enunciado:** Crie `LabStudy.Demo.Tracker` com:

1. `ClassMethod Log(area, message)` — grava uma entrada em `^DemoTrack` contendo sequência, data e hora, usuário, processo, namespace, área e mensagem.
2. `ClassMethod Print(area)` — imprime as entradas; se `area` vier vazia, imprime todas.
3. `ClassMethod Purge()` — limpa a trilha.

Depois, no Terminal, registre cinco eventos em duas áreas diferentes e imprima filtrando por área.

**b) Dica:** Use `$INCREMENT(^DemoTrack)` para a sequência e subscritos nomeados para os campos. Para percorrer, use o padrão `$ORDER` que já apareceu no Capítulo 3.

**c) Como testar:** `Print("import")` deve mostrar só as entradas dessa área.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Tracker.cls`:

```objectscript
/// A minimal, hand written application audit trail.
Class LabStudy.Demo.Tracker Extends %RegisteredObject
{

/// Writes one entry into the trail.
ClassMethod Log(area As %String, message As %String) As %Status
{
    set seq = $INCREMENT(^DemoTrack)

    set ^DemoTrack(seq, "when")      = $ZDATETIME($HOROLOG, 3)
    set ^DemoTrack(seq, "user")      = $USERNAME
    set ^DemoTrack(seq, "job")       = $JOB
    set ^DemoTrack(seq, "namespace") = $NAMESPACE
    set ^DemoTrack(seq, "area")      = area
    set ^DemoTrack(seq, "message")   = message

    quit $$$OK
}

/// Prints the trail, optionally filtered by area.
ClassMethod Print(area As %String = "") As %Status
{
    set shown = 0
    set seq = ""

    for {
        set seq = $ORDER(^DemoTrack(seq))
        quit:seq=""

        if (area '= "") && (^DemoTrack(seq, "area") '= area) {
            continue
        }

        write seq, " | "
        write ^DemoTrack(seq, "when"), " | "
        write ^DemoTrack(seq, "user"), "@", ^DemoTrack(seq, "job"), " | "
        write ^DemoTrack(seq, "area"), " | "
        write ^DemoTrack(seq, "message"), !

        set shown = shown + 1
    }

    write "-- ", shown, " entries --", !
    quit $$$OK
}

/// Clears the whole trail.
ClassMethod Purge() As %Status
{
    kill ^DemoTrack
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Purge()

LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Log("import","started file A")
LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Log("import","3 records read")
LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Log("login","user recepcao02 signed in")
LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Log("import","finished file A")
LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Log("login","user recepcao02 signed out")

LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Print("import")
1 | 2026-08-19 14:40:01 | _SYSTEM@12345 | import | started file A
2 | 2026-08-19 14:40:01 | _SYSTEM@12345 | import | 3 records read
4 | 2026-08-19 14:40:02 | _SYSTEM@12345 | import | finished file A
-- 3 entries --

LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Print()
1 | 2026-08-19 14:40:01 | _SYSTEM@12345 | import | started file A
2 | 2026-08-19 14:40:01 | _SYSTEM@12345 | import | 3 records read
3 | 2026-08-19 14:40:01 | _SYSTEM@12345 | login | user recepcao02 signed in
4 | 2026-08-19 14:40:02 | _SYSTEM@12345 | import | finished file A
5 | 2026-08-19 14:40:02 | _SYSTEM@12345 | login | user recepcao02 signed out
-- 5 entries --

LABSTUDY>ZWRITE ^DemoTrack(3)
^DemoTrack(3,"area")="login"
^DemoTrack(3,"job")=12345
^DemoTrack(3,"message")="user recepcao02 signed in"
^DemoTrack(3,"namespace")="LABSTUDY"
^DemoTrack(3,"user")="_SYSTEM"
^DemoTrack(3,"when")="2026-08-19 14:40:01"
```

**Por que cada decisão:**

- **`$INCREMENT` para a sequência** garante ordem e unicidade mesmo com vários processos escrevendo ao mesmo tempo — e, como você viu no capítulo anterior, sem custar uma trava.
- **Repare nos carimbos de tempo:** as entradas 1, 2 e 3 têm exatamente o mesmo segundo. Se você ordenasse por tempo, a ordem entre elas seria indefinida. A sequência numérica é o que preserva a ordem real. Este é o motivo prático da pegadinha 11 da seção anterior.
- **`Print` sem filtro percorre tudo e com filtro pula com `continue`.** Para uma trilha grande isso seria lento — a solução correta seria um índice por área, ou uma classe persistente com índice, como discutimos na seção 5.1. Para o exercício, a varredura é honesta e simples.
- **O `ZWRITE` final mostra a vantagem dos subscritos nomeados**: a estrutura se explica sozinha, em ordem alfabética, sem precisar de documentação.
- **A contagem no final do `Print`** parece um detalhe, mas é o que permite dizer "não há entradas" em vez de deixar a tela em branco e o usuário na dúvida.

---

### Exercício 6.2 — Triggers de `INSERT`, `UPDATE` e `DELETE`

**a) Enunciado:** Crie `LabStudy.Demo.Employee2`, persistente, com `Name As %String(MAXLEN = 100)`, `Department As %String(MAXLEN = 50)` e `Salary As %Numeric(SCALE = 2)`.

Declare três triggers, todos com `Foreach = row/object` e `Time = AFTER`, que escrevam na trilha do exercício anterior (`LabStudy.Demo.Tracker`):

1. `INSERT` — registra a criação com nome e departamento.
2. `UPDATE` — registra **apenas** os campos que mudaram, com valor antigo e novo.
3. `DELETE` — registra a exclusão com o nome antigo.

Teste criando um funcionário, alterando só o departamento, alterando salário e nome juntos, e apagando.

**b) Dica:** `{Campo*C}` diz se aquele campo mudou. Em `DELETE`, use `{Campo*O}`.

**c) Como testar:** A alteração só de departamento deve gerar **uma** entrada, não três.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Employee2.cls`:

```objectscript
/// Audited employee, using SQL triggers for the trail.
Class LabStudy.Demo.Employee2 Extends %Persistent [ SqlTableName = EMPLOYEE2 ]
{

Property Name As %String(MAXLEN = 100) [ Required ];

Property Department As %String(MAXLEN = 50);

Property Salary As %Numeric(SCALE = 2) [ InitialExpression = 0 ];

Trigger AuditInsert [ Event = INSERT, Time = AFTER, Foreach = row/object ]
{
    do ##class(LabStudy.Demo.Tracker).Log("employee",
        "INSERT #"_{%%ID}_" name="_{Name}_" dept="_{Department})
}

Trigger AuditUpdate [ Event = UPDATE, Time = AFTER, Foreach = row/object ]
{
    if {Name*C} {
        do ##class(LabStudy.Demo.Tracker).Log("employee",
            "UPDATE #"_{%%ID}_" Name: ["_{Name*O}_"] -> ["_{Name*N}_"]")
    }
    if {Department*C} {
        do ##class(LabStudy.Demo.Tracker).Log("employee",
            "UPDATE #"_{%%ID}_" Department: ["_{Department*O}_"] -> ["_{Department*N}_"]")
    }
    if {Salary*C} {
        do ##class(LabStudy.Demo.Tracker).Log("employee",
            "UPDATE #"_{%%ID}_" Salary: ["_{Salary*O}_"] -> ["_{Salary*N}_"]")
    }
}

Trigger AuditDelete [ Event = DELETE, Time = AFTER, Foreach = row/object ]
{
    do ##class(LabStudy.Demo.Tracker).Log("employee",
        "DELETE #"_{%%ID}_" name was ["_{Name*O}_"]")
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Purge()
LABSTUDY>DO ##class(LabStudy.Demo.Employee2).%KillExtent()

LABSTUDY>SET e = ##class(LabStudy.Demo.Employee2).%New()
LABSTUDY>SET e.Name = "Carla Nunes", e.Department = "Hematology", e.Salary = 5200
LABSTUDY>WRITE $$$ISOK(e.%Save()), !
1

LABSTUDY>SET e.Department = "Biochemistry"
LABSTUDY>WRITE $$$ISOK(e.%Save()), !
1

LABSTUDY>SET e.Salary = 5800, e.Name = "Carla Nunes Prado"
LABSTUDY>WRITE $$$ISOK(e.%Save()), !
1

LABSTUDY>SET id = e.%Id()
LABSTUDY>WRITE $$$ISOK(##class(LabStudy.Demo.Employee2).%DeleteId(id)), !
1

LABSTUDY>DO ##class(LabStudy.Demo.Tracker).Print("employee")
1 | 2026-08-19 14:45:10 | _SYSTEM@12345 | employee | INSERT #1 name=Carla Nunes dept=Hematology
2 | 2026-08-19 14:45:22 | _SYSTEM@12345 | employee | UPDATE #1 Department: [Hematology] -> [Biochemistry]
3 | 2026-08-19 14:45:35 | _SYSTEM@12345 | employee | UPDATE #1 Name: [Carla Nunes] -> [Carla Nunes Prado]
4 | 2026-08-19 14:45:35 | _SYSTEM@12345 | employee | UPDATE #1 Salary: [5200] -> [5800]
5 | 2026-08-19 14:45:47 | _SYSTEM@12345 | employee | DELETE #1 name was [Carla Nunes Prado]
```

**Por que cada decisão:**

- **A segunda gravação produziu uma única entrada**, apesar de o objeto inteiro ter sido regravado. Foi o `{Department*C}` fazendo o trabalho. Sem ele, haveria três entradas, duas delas inúteis.
- **A terceira gravação produziu duas entradas, na ordem em que os `if` aparecem no trigger** — não na ordem em que você fez os `SET`. Isso é esperado: o trigger vê apenas o estado antes e depois, não a sequência de atribuições.
- **O `DELETE` usou `{Name*O}`.** Depois de apagada, a linha não tem valor "atual"; só existe o antigo. Tentar `{Name}` ali não faria sentido.
- **Todos os triggers têm `Foreach = row/object`.** Faça o teste: remova essa palavra-chave de um deles, recompile, e refaça a sequência. Aquela operação deixará de aparecer na trilha, silenciosamente. É a demonstração mais direta da pegadinha número 1.
- **Note que o `%DeleteId()` também disparou o trigger** — porque `row/object` cobre também o apagamento pelo mundo dos objetos.

---

### Exercício 6.3 — Vetando com `%ok` e `%msg`

**a) Enunciado:** Acrescente a `LabStudy.Demo.Employee2` dois triggers `BEFORE`:

1. `CheckSalary` — recusa salário negativo, com mensagem clara.
2. `BlockDepartmentChange` — recusa **alterar** o departamento de um funcionário cujo nome comece com `"Dr"`, com mensagem explicando o motivo. (Regra artificial, só para exercitar a leitura do valor antigo num `BEFORE`.)

Teste os dois vetos, pelo objeto e pelo SQL.

**b) Dica:** Num `BEFORE UPDATE`, você tem tanto `{Campo*O}` quanto `{Campo*N}`. Para testar o início de um texto, `$EXTRACT(texto, 1, 2) = "Dr"`.

**c) Como testar:** O `%Save()` deve devolver `%Status` de erro carregando o `%msg`. O `UPDATE` no SQL deve ser recusado com a mesma mensagem.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Acrescente à classe:

```objectscript
/// Refuses negative salaries, for SQL and for object saves.
Trigger CheckSalary [ Event = INSERT/UPDATE, Time = BEFORE, Foreach = row/object ]
{
    if {Salary} < 0 {
        set %ok = 0
        set %msg = "Salary cannot be negative (got "_{Salary}_")"
    }
}

/// Artificial rule: doctors keep their department.
Trigger BlockDepartmentChange [ Event = UPDATE, Time = BEFORE, Foreach = row/object ]
{
    if {Department*C} && ($EXTRACT({Name*O}, 1, 2) = "Dr") {
        set %ok = 0
        set %msg = "Department of a doctor cannot be changed"
    }
}
```

```
LABSTUDY>SET e = ##class(LabStudy.Demo.Employee2).%New()
LABSTUDY>SET e.Name = "Dr House", e.Department = "Diagnostics", e.Salary = 9000
LABSTUDY>WRITE $$$ISOK(e.%Save()), " id=", e.%Id(), !
1 id=2

LABSTUDY>SET e.Salary = -1
LABSTUDY>DO $SYSTEM.Status.DisplayError(e.%Save())
ERROR #5001: Salary cannot be negative (got -1)

LABSTUDY>SET e.Salary = 9500
LABSTUDY>SET e.Department = "Nephrology"
LABSTUDY>DO $SYSTEM.Status.DisplayError(e.%Save())
ERROR #5001: Department of a doctor cannot be changed

LABSTUDY>SET e = ##class(LabStudy.Demo.Employee2).%OpenId(2)
LABSTUDY>SET e.Salary = 9500
LABSTUDY>WRITE $$$ISOK(e.%Save()), !
1
```

E no SQL, pelo Portal:

```sql
UPDATE LabStudy.EMPLOYEE2 SET Department = 'Cardiology' WHERE ID = 2
```

O SQL devolve erro com a mesma mensagem `Department of a doctor cannot be changed`.

**Por que cada decisão:**

- **A mesma regra vale nas duas portas**, e foi escrita **uma vez**. Esse é o argumento mais forte a favor de triggers para regras de integridade: eles não podem ser contornados escolhendo outro caminho de acesso.
- **`{Name*O}` no trigger de `UPDATE`** lê o nome **antes** da alteração. Se o usuário estivesse renomeando e mudando o departamento ao mesmo tempo, a regra ainda se aplicaria com base em quem a pessoa era. Se você quisesse a regra baseada no nome novo, usaria `{Name*N}`. Perceber que existe essa escolha é o aprendizado do exercício.
- **Repare na terceira operação:** ela tinha **duas** mudanças (salário e departamento), e o veto do departamento cancelou **as duas**. É por isso que, depois, foi preciso reabrir o objeto do disco: a OREF na memória ainda carregava a mudança recusada. É o mesmo fenômeno do rollback que você viu no capítulo anterior — memória e banco divergindo.
- **`$EXTRACT(texto, 1, 2)`** pega os caracteres de 1 a 2. Funções de texto são o Capítulo 10; aqui, receita.

---

### Exercício 6.4 — Log do sistema e evento de auditoria

**a) Enunciado:**

1. Escreva uma mensagem no `messages.log` usando `##class(%SYS.System).WriteToConsoleLog()`.
2. Localize a mensagem no Portal, em **System Operation → System Logs → Messages Log**.
3. **(Parte opcional, exige perfil administrativo)** No Portal, crie um evento de auditoria de usuário com Source `LabStudy`, Type `Report` e Name `Exported`. Depois dispare-o pelo Terminal e localize-o em **View Audit Database**.

**b) Dica:** Se você não tiver permissão para configurar auditoria, ou se a opção não estiver disponível na sua instalação, faça apenas a primeira parte. A segunda é para você conhecer o caminho.

**c) Como testar:** A mensagem deve aparecer no arquivo de log com carimbo de data e hora.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Parte 1:

```
LABSTUDY>DO ##class(%SYS.System).WriteToConsoleLog("LabStudy: nightly import finished, 128 records")

LABSTUDY>WRITE "written", !
written
```

No Portal, **System Operation → System Logs → Messages Log**, a linha aparece no final, com data, hora e número do processo.

Parte 2, depois de criar o evento no Portal:

```
LABSTUDY>DO $SYSTEM.Security.Audit("LabStudy", "Report", "Exported", "patient=1", "Full patient report exported")

LABSTUDY>WRITE "audit event fired", !
audit event fired
```

E em **System Administration → Security → Auditing → View Audit Database**, o evento aparece com usuário, processo, data e os dados que você passou.

**Por que cada decisão:**

- **A mensagem no console log tem um prefixo com o nome do sistema** (`LabStudy:`). Num arquivo compartilhado por todo o IRIS, identificar a origem é o mínimo de educação com quem vai ler.
- **A mensagem é factual e quantificada** ("128 records"), não decorativa. Uma linha de log que não permite decidir nada é lixo.
- **A parte 2 depende de configuração prévia.** Isso não é um detalhe burocrático: é o desenho do recurso. Eventos de auditoria são controlados pelo administrador justamente para que a aplicação não possa inundar ou contaminar a base de auditoria. A prova pode perguntar isso: **eventos de auditoria de usuário precisam ser declarados antes de serem usados.**
- **A ordem exata dos argumentos de `$SYSTEM.Security.Audit` e a existência de argumentos adicionais variam por versão: verificar na documentação oficial.** Não invente; consulte.

---

### Exercício 6.5 — PROJETO CONTÍNUO: trilha completa do laboratório

**a) Enunciado:** Evolua o sistema:

1. Crie `LabStudy.AuditEntry`, persistente, conforme o modelo da seção 5.1, com `SqlTableName = AUDIT_ENTRY` e os índices `TableRecordIdx` e `EventTimeIdx`. Acrescente:
   - `ClassMethod Record(tableName, recordId, operation, fieldName, oldValue, newValue) As %Status` — cria e grava a entrada;
   - `ClassMethod PrintFor(tableName, recordId) As %Status` — imprime a trilha de um registro específico.
2. Em `LabStudy.Patient`, acrescente triggers com `Foreach = row/object`:
   - `AuditInsert` (`AFTER INSERT`) — registra a criação;
   - `AuditUpdate` (`AFTER UPDATE`) — registra alterações de `Name`, `RecordNumber` e `Active`, campo a campo;
   - `AuditDelete` (`AFTER DELETE`) — registra a exclusão com o número de registro antigo;
   - `ProtectRecordNumber` (`BEFORE UPDATE`) — **impede** que o `RecordNumber` de um paciente já cadastrado seja alterado, pois ele é a chave de negócio do laboratório.
3. Em `LabStudy.Exam`, acrescente um trigger `AFTER UPDATE` que registre alterações de `ResultValue` — o dado mais crítico do sistema.
4. Suba `LabStudy.App` para `"0.7"` e acrescente `ClassMethod AuditReport(recordNumber)` que localiza o paciente pelo número de registro e imprime a trilha dele.

**b) Dica:** No trigger `BEFORE UPDATE`, `{RecordNumber*C}` diz se houve tentativa de mudança.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.AuditEntry).%KillExtent()
LABSTUDY>SET id = ##class(LabStudy.Patient).CreateWithExams("Maria Silva","1990-05-17","F",[{"testCode":"HGB","value":13.5,"unit":"g/dL"}])
LABSTUDY>SET p = ##class(LabStudy.Patient).%OpenId(id)
LABSTUDY>SET p.Name = "Maria Silva Souza"
LABSTUDY>WRITE $$$ISOK(p.%Save()), !
LABSTUDY>SET p.RecordNumber = "REG-999999"
LABSTUDY>DO $SYSTEM.Status.DisplayError(p.%Save())
LABSTUDY>DO ##class(LabStudy.App).AuditReport("REG-000001")
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/AuditEntry.cls`:

```objectscript
/// One entry of the LabStudy application audit trail.
/// This class must never audit itself.
Class LabStudy.AuditEntry Extends %Persistent [ SqlTableName = AUDIT_ENTRY ]
{

Property EventTime As %TimeStamp [ InitialExpression = {$ZDATETIME($HOROLOG, 3)} ];

Property Sequence As %Integer [ InitialExpression = {$INCREMENT(^LabStudyAuditSeq)} ];

Property UserName As %String(MAXLEN = 64) [ InitialExpression = {$USERNAME} ];

Property JobNumber As %String(MAXLEN = 20) [ InitialExpression = {$JOB} ];

Property TableName As %String(MAXLEN = 100);

Property RecordId As %String(MAXLEN = 50);

Property Operation As %String(MAXLEN = 30);

Property FieldName As %String(MAXLEN = 100);

Property OldValue As %String(MAXLEN = 500);

Property NewValue As %String(MAXLEN = 500);

/// Fast lookup of the trail of one record.
Index TableRecordIdx On (TableName, RecordId);

/// Chronological browsing.
Index EventTimeIdx On EventTime;

/// Creates one trail entry.
ClassMethod Record(tableName As %String, recordId As %String, operation As %String, fieldName As %String = "", oldValue As %String = "", newValue As %String = "") As %Status
{
    set entry = ..%New()
    set entry.TableName = tableName
    set entry.RecordId = recordId
    set entry.Operation = operation
    set entry.FieldName = fieldName
    set entry.OldValue = $EXTRACT(oldValue, 1, 500)
    set entry.NewValue = $EXTRACT(newValue, 1, 500)

    quit entry.%Save()
}

/// Prints the trail of one record, in order.
ClassMethod PrintFor(tableName As %String, recordId As %String) As %Status
{
    write "Audit trail for ", tableName, " #", recordId, !
    write "--------------------------------------------------", !

    set found = 0
    set id = ""
    for {
        set id = $ORDER(^LabStudy.AuditEntryD(id))
        quit:id=""

        set entry = ..%OpenId(id)
        continue:'$ISOBJECT(entry)
        continue:entry.TableName'=tableName
        continue:entry.RecordId'=recordId

        write entry.EventTime, " | ", entry.UserName, " | ", entry.Operation
        if entry.FieldName '= "" {
            write " ", entry.FieldName
        }
        if (entry.OldValue '= "") || (entry.NewValue '= "") {
            write " | [", entry.OldValue, "] -> [", entry.NewValue, "]"
        }
        write !
        set found = found + 1
    }

    write "--------------------------------------------------", !
    write found, " entries", !
    quit $$$OK
}

}
```

Acrescente a `src/LabStudy/Patient.cls`:

```objectscript
/// The record number is the business key: it must never change.
Trigger ProtectRecordNumber [ Event = UPDATE, Time = BEFORE, Foreach = row/object ]
{
    if {RecordNumber*C} {
        set %ok = 0
        set %msg = "RecordNumber cannot be changed once assigned (was "_{RecordNumber*O}_")"
    }
}

Trigger AuditInsert [ Event = INSERT, Time = AFTER, Foreach = row/object ]
{
    do ##class(LabStudy.AuditEntry).Record({%%TABLENAME}, {%%ID}, "INSERT", "", "", {RecordNumber}_" / "_{Name})
}

Trigger AuditUpdate [ Event = UPDATE, Time = AFTER, Foreach = row/object ]
{
    if {Name*C} {
        do ##class(LabStudy.AuditEntry).Record({%%TABLENAME}, {%%ID}, "UPDATE", "Name", {Name*O}, {Name*N})
    }
    if {Active*C} {
        do ##class(LabStudy.AuditEntry).Record({%%TABLENAME}, {%%ID}, "UPDATE", "Active", {Active*O}, {Active*N})
    }
}

Trigger AuditDelete [ Event = DELETE, Time = AFTER, Foreach = row/object ]
{
    do ##class(LabStudy.AuditEntry).Record({%%TABLENAME}, {%%ID}, "DELETE", "", {RecordNumber*O}, "")
}
```

Acrescente a `src/LabStudy/Exam.cls`:

```objectscript
/// The result value is the most critical field of the system.
Trigger AuditResultChange [ Event = UPDATE, Time = AFTER, Foreach = row/object ]
{
    if {ResultValue*C} {
        do ##class(LabStudy.AuditEntry).Record({%%TABLENAME}, {%%ID}, "UPDATE", "ResultValue", {ResultValue*O}, {ResultValue*N})
    }
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "0.7";

/// Prints the audit trail of one patient, found by record number.
ClassMethod AuditReport(recordNumber As %String) As %Status
{
    set patient = ##class(LabStudy.Patient).FindByRecord(recordNumber)
    if '$ISOBJECT(patient) {
        write "No patient with record number ", recordNumber, !
        quit $$$OK
    }

    do ##class(LabStudy.AuditEntry).PrintFor("PATIENT", patient.%Id())

    write !
    write "Exams of this patient:", !
    for i = 1:1:patient.Exams.Count() {
        set exam = patient.Exams.GetAt(i)
        do ##class(LabStudy.AuditEntry).PrintFor("EXAM", exam.%Id())
    }
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.AuditEntry).%KillExtent()
LABSTUDY>DO ##class(LabStudy.Patient).%KillExtent()
LABSTUDY>DO ##class(LabStudy.Exam).%KillExtent()
LABSTUDY>KILL ^LabStudySeq

LABSTUDY>SET id = ##class(LabStudy.Patient).CreateWithExams("Maria Silva","1990-05-17","F",[{"testCode":"HGB","value":13.5,"unit":"g/dL"}])

LABSTUDY>SET p = ##class(LabStudy.Patient).%OpenId(id)
LABSTUDY>SET p.Name = "Maria Silva Souza"
LABSTUDY>WRITE $$$ISOK(p.%Save()), !
1

LABSTUDY>SET p.RecordNumber = "REG-999999"
LABSTUDY>DO $SYSTEM.Status.DisplayError(p.%Save())
ERROR #5001: RecordNumber cannot be changed once assigned (was REG-000001)

LABSTUDY>SET e = ##class(LabStudy.Exam).%OpenId(1)
LABSTUDY>SET e.ResultValue = 14.2
LABSTUDY>WRITE $$$ISOK(e.%Save()), !
1

LABSTUDY>DO ##class(LabStudy.App).AuditReport("REG-000001")
Audit trail for PATIENT #1
--------------------------------------------------
2026-08-19 15:02:11 | _SYSTEM | INSERT | [] -> [REG-000001 / Maria Silva]
2026-08-19 15:02:30 | _SYSTEM | UPDATE Name | [Maria Silva] -> [Maria Silva Souza]
--------------------------------------------------
2 entries

Exams of this patient:
Audit trail for EXAM #1
--------------------------------------------------
2026-08-19 15:03:02 | _SYSTEM | UPDATE ResultValue | [13.5] -> [14.2]
--------------------------------------------------
1 entries
```

**Por que cada decisão:**

- **`ProtectRecordNumber` é uma regra de integridade, não de auditoria — e por isso vive num trigger `BEFORE`.** O número de registro é a chave de negócio do laboratório; se ele pudesse mudar, toda referência externa em papel, em outro sistema ou na memória de uma pessoa apontaria para o lugar errado. Note que a mensagem informa o valor antigo, o que ajuda muito quem recebeu o erro.
- **A tentativa de mudar o `RecordNumber` não gerou entrada de auditoria**, porque o `BEFORE` cancelou antes de os `AFTER` rodarem. Se o requisito fosse registrar **tentativas** de violação, o registro teria que ir para outro lugar, escrito pelo próprio `BEFORE`. É uma decisão de projeto, e vale a pena você refletir sobre ela.
- **`LabStudy.AuditEntry` não tem trigger nenhum.** Se tivesse, cada entrada de auditoria geraria outra entrada, indefinidamente. Isso é intencional e está anotado no comentário da classe, para o próximo desenvolvedor não cair na armadilha.
- **A propriedade `Sequence` com `$INCREMENT` no `InitialExpression`** resolve o problema de ordenação dentro do mesmo segundo que apareceu no exercício 6.1. O `EventTime` é para humanos; a `Sequence` é para ordenar.
- **`$EXTRACT(valor, 1, 500)` no `Record`.** Os campos têm `MAXLEN = 500`; um valor maior faria o `%Save()` da própria auditoria falhar — e uma auditoria que quebra a operação que ela deveria apenas observar é um desastre. Cortar é preferível a falhar, **neste caso específico**, porque a trilha é acessória. Repare que essa é uma exceção consciente ao princípio geral de nunca truncar dados silenciosamente.
- **Só três campos do paciente são auditados** (`Name`, `Active` e o `RecordNumber` na criação). `BirthDate` e `Sex` não mudam na prática; `Address` e `Allergies` mudariam com frequência e gerariam ruído. Auditoria é curadoria.
- **O trigger do `Exam` audita apenas `ResultValue`.** É o campo pelo qual alguém pode ser processado. Unidade e código de exame são estruturais, não clínicos.
- **`AuditReport` percorre também os exames do paciente**, montando uma visão completa. Esse é o formato que um auditor de verdade pediria: tudo que aconteceu com aquele atendimento, num lugar só.
- **O `PrintFor` varre a global de dados com `$ORDER`, sem usar o índice que declaramos.** Isso é propositalmente ineficiente e está aqui para incomodar você: temos `TableRecordIdx` justamente para essa consulta, e não o estamos usando. No Capítulo 15 (SQL a partir do ObjectScript) reescreveremos isso com uma consulta que aproveita o índice, e a diferença será visível.

---

## 8. Quiz do capítulo

**Q1.** Qual é a finalidade principal do **journal** do IRIS?

- A) Registrar mudanças de negócio para consulta pelos usuários.
- B) Permitir rollback de transações e recuperação após falhas.
- C) Registrar logins e tentativas de acesso.
- D) Substituir a trilha de auditoria da aplicação.

---

**Q2.** Qual é o valor padrão da palavra-chave `Foreach` de um trigger?

- A) `row/object`
- B) `statement`
- C) `row`
- D) Não existe padrão; é obrigatória.

---

**Q3.** Você escreveu um trigger `[ Event = UPDATE, Time = AFTER ]` sem especificar `Foreach`. Um `%Save()` de objeto dispara esse trigger?

- A) Sim, sempre.
- B) Não; com o padrão `row`, ele só dispara em operações SQL.
- C) Só se o objeto tiver índices.
- D) Só dentro de uma transação.

---

**Q4.** Como cancelar uma operação de dentro de um trigger?

- A) `quit $$$ERROR(...)` num trigger `AFTER`.
- B) `set %ok = 0` e `set %msg = "motivo"` num trigger `BEFORE`.
- C) `trollback` no corpo do trigger.
- D) Não é possível cancelar por trigger.

---

**Q5.** Num trigger de `UPDATE`, o que representa `{Price*O}`?

- A) O valor novo.
- B) O valor antigo, anterior à alteração.
- C) Um indicador de que o campo mudou.
- D) O nome da coluna.

---

**Q6.** Você quer que a auditoria registre apenas os campos que realmente mudaram num `UPDATE`. Qual notação usar?

- A) `{Campo*N}`
- B) `{Campo*O}`
- C) `{Campo*C}`
- D) `{%%OPERATION}`

---

**Q7.** Qual pseudocampo devolve o identificador da linha dentro de um trigger?

- A) `{ID}`
- B) `{%%ID}`
- C) `{%Id}`
- D) `{%%RECORD}`

---

**Q8.** Qual variável especial devolve o nome do usuário conectado?

- A) `$JOB`
- B) `$ROLES`
- C) `$USERNAME`
- D) `$NAMESPACE`

---

**Q9.** Duas entradas de trilha gravadas no mesmo segundo têm o mesmo `$HOROLOG`. Como garantir a ordem correta entre elas?

- A) Ordenando por `$USERNAME`.
- B) Mantendo um número de sequência com `$INCREMENT`.
- C) Usando `$JOB` como critério.
- D) Não é possível.

---

**Q10.** Você quer auditar a alteração de um campo registrando o valor antigo. Qual abordagem exige menos trabalho?

- A) Callback `%OnBeforeSave`, reabrindo o objeto do disco para comparar.
- B) Trigger `AFTER UPDATE`, usando `{Campo*O}` e `{Campo*N}`.
- C) Consultar o journal.
- D) Usar `%OnValidateObject`.

---

**Q11.** Qual afirmação sobre eventos de auditoria de usuário do IRIS está correta?

- A) Podem ser disparados sem configuração prévia.
- B) Precisam ser declarados antes no Portal, com Source, Type e Name.
- C) São gravados no `messages.log`.
- D) São desfeitos por rollback.

---

**Q12.** Por que uma classe de auditoria nunca deve ter triggers de auditoria sobre si mesma?

- A) Porque triggers não funcionam em classes persistentes.
- B) Porque cada entrada geraria outra entrada, criando um laço infinito.
- C) Porque a auditoria é somente leitura.
- D) Porque o IRIS proíbe isso na compilação.

---

**Q13.** A trilha de auditoria foi escrita dentro da mesma transação da operação, e a transação sofreu rollback. O que acontece com a trilha?

- A) Permanece gravada.
- B) É desfeita junto com a operação.
- C) É movida para o journal.
- D) Gera erro.

---

**Q14.** Qual chamada escreve uma mensagem no log do sistema (`messages.log`)?

- A) `write "mensagem"`
- B) `do ##class(%SYS.System).WriteToConsoleLog("mensagem")`
- C) `do $SYSTEM.Security.Audit("mensagem")`
- D) `set ^messages = "mensagem"`

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** o journal existe para desfazer transações e recuperar o banco após falhas.
- **A está errada:** ele é técnico, volumoso e rotativo; não é histórico de negócio.
- **C está errada:** logins ficam na base de auditoria.
- **D está errada:** ele não substitui a trilha da aplicação, que tem outro propósito e outra retenção.

**Q2 — Resposta: C.**
- **C está certa:** o padrão é `row`, ou seja, dispara apenas em operações SQL.
- **A está errada:** `row/object` precisa ser pedido explicitamente.
- **B está errada:** `statement` é um escopo diferente, também não padrão.
- **D está errada:** a palavra-chave é opcional e tem padrão.

**Q3 — Resposta: B.**
- **B está certa:** sem `Foreach = row/object`, o `%Save()` não dispara o trigger. É a origem clássica de buracos na trilha.
- **A está errada:** só seria sempre com `row/object`.
- **C está errada:** índices não têm relação com isso.
- **D está errada:** transações não têm relação com isso.

**Q4 — Resposta: B.**
- **B está certa:** `%ok = 0` cancela e `%msg` fornece a mensagem, num trigger `BEFORE`.
- **A está errada:** num `AFTER` a operação já aconteceu, e o retorno não cancela nada.
- **C está errada:** dar rollback dentro do trigger interferiria na transação do chamador de forma imprevisível.
- **D está errada:** é justamente o que `%ok` permite.

**Q5 — Resposta: B.**
- **B está certa:** `*O` é de *old*, o valor anterior à alteração.
- **A está errada:** o valor novo é `*N`.
- **C está errada:** o indicador de mudança é `*C`.
- **D está errada:** o nome da coluna não se obtém assim.

**Q6 — Resposta: C.**
- **C está certa:** `{Campo*C}` indica se aquele campo foi alterado nesta operação.
- **A está errada:** `*N` só dá o valor novo, sem dizer se mudou.
- **B está errada:** `*O` só dá o valor antigo.
- **D está errada:** `%%OPERATION` informa se é INSERT, UPDATE ou DELETE.

**Q7 — Resposta: B.**
- **B está certa:** os pseudocampos usam dois sinais de porcentagem: `{%%ID}`, `{%%TABLENAME}`, `{%%CLASSNAME}`, `{%%OPERATION}`.
- **A está errada:** faltam os sinais de porcentagem.
- **C está errada:** `%Id()` é método de objeto, não notação de trigger.
- **D está errada:** esse pseudocampo não existe.

**Q8 — Resposta: C.**
- **C está certa:** `$USERNAME` traz o usuário conectado.
- **A está errada:** `$JOB` é o número do processo.
- **B está errada:** `$ROLES` traz os papéis/permissões.
- **D está errada:** `$NAMESPACE` traz o namespace atual.

**Q9 — Resposta: B.**
- **B está certa:** `$HOROLOG` tem resolução de um segundo; um contador atômico preserva a ordem real.
- **A está errada:** o usuário não indica ordem.
- **C está errada:** o número do processo tampouco.
- **D está errada:** é perfeitamente possível, com sequência.

**Q10 — Resposta: B.**
- **B está certa:** o trigger entrega o valor antigo de graça em `{Campo*O}`.
- **A está errada:** funciona, mas exige reabrir o objeto do disco: mais código e mais custo.
- **C está errada:** o journal é técnico e não serve como fonte de auditoria de negócio.
- **D está errada:** `%OnValidateObject` valida; não tem acesso natural ao valor anterior gravado.

**Q11 — Resposta: B.**
- **B está certa:** eventos de usuário são declarados no Portal, identificados por Source, Type e Name, e só então podem ser disparados.
- **A está errada:** sem declaração prévia, o evento não é registrado.
- **C está errada:** vão para a base de auditoria, não para o console log.
- **D está errada:** a base de auditoria é independente das transações da aplicação.

**Q12 — Resposta: B.**
- **B está certa:** gravar uma entrada dispararia o trigger que grava outra entrada, indefinidamente.
- **A está errada:** triggers funcionam normalmente em classes persistentes.
- **C está errada:** a classe é gravável; o problema é a recursão.
- **D está errada:** o compilador não impede; a responsabilidade é sua.

**Q13 — Resposta: B.**
- **B está certa:** a trilha participa da mesma transação e é desfeita junto.
- **A está errada:** só permaneceria se estivesse fora da transação.
- **C está errada:** o journal não é destino de trilha.
- **D está errada:** não há erro; há reversão.

**Q14 — Resposta: B.**
- **B está certa:** `##class(%SYS.System).WriteToConsoleLog()` escreve no `messages.log`.
- **A está errada:** `write` escreve no dispositivo atual, normalmente a tela.
- **C está errada:** essa chamada registra evento de auditoria, e com outra assinatura.
- **D está errada:** não existe essa global de log.

---

## 9. Resumo relâmpago

1. Três registros diferentes: **journal** (rollback e recuperação), **auditoria do IRIS** (segurança e sistema) e **trilha da aplicação** (negócio, escrita por você).
2. Uma boa trilha responde: **quem, quando, o quê, qual operação, de que valor para qual valor**.
3. Contexto: `$USERNAME`, `$ROLES`, `$JOB`, `$HOROLOG`, `$ZDATETIME($HOROLOG,3)`, `$NAMESPACE`, `$ZVERSION`.
4. `$HOROLOG` tem precisão de **um segundo**. Para ordenar eventos do mesmo segundo, use `$INCREMENT`.
5. **Trigger** é código pendurado na tabela; **callback** é código pendurado na classe. Trigger pega SQL; callback pega objeto.
6. Sintaxe: `Trigger Nome [ Event = ..., Time = ..., Foreach = ..., Order = n ] { ... }`. Sem ponto e vírgula no fim.
7. `Event`: `INSERT`, `UPDATE`, `DELETE`, combináveis com barra.
8. `Time`: `BEFORE` (pode vetar) ou `AFTER` (registra o que aconteceu). **Veto no BEFORE, registro no AFTER.**
9. **`Foreach` padrão é `row` — só SQL.** Para pegar também `%Save()` e `%DeleteId()`, use **`row/object`**.
10. Valores: `{Campo}`, `{Campo*O}` (antigo), `{Campo*N}` (novo), `{Campo*C}` (mudou).
11. Pseudocampos: `{%%ID}`, `{%%TABLENAME}`, `{%%CLASSNAME}`, `{%%OPERATION}`.
12. Cancelar operação: `set %ok = 0` e `set %msg = "motivo"`, apenas em trigger `BEFORE`.
13. Trigger entrega o valor antigo de graça; callback exigiria reabrir o objeto. Por isso trigger vence em auditoria de campos.
14. Trilha em classe persistente ganha SQL, índices e Portal; em global ganha desempenho.
15. **A classe de auditoria nunca é auditada por si mesma** — laço infinito.
16. Trilha dentro da transação é desfeita pelo rollback. Decida conscientemente se isso é o desejado.
17. `##class(%SYS.System).WriteToConsoleLog(texto)` escreve no `messages.log` — use com parcimônia.
18. Eventos de auditoria do IRIS precisam ser **declarados antes** no Portal (Source/Type/Name) e são disparados com `$SYSTEM.Security.Audit(...)`.
19. Triggers rodam em toda gravação: **mantenha-os curtos**.

---

## 10. Cartões de memorização

**Frente:** Para que serve o journal do IRIS?
**Verso:** Rollback de transações e recuperação após falhas. Não é histórico de negócio.

**Frente:** Onde ficam os eventos de login e ações administrativas?
**Verso:** Na base de auditoria do IRIS, configurada pelo administrador.

**Frente:** Quais cinco perguntas uma boa entrada de trilha responde?
**Verso:** Quem, quando, o quê, qual operação, e de que valor para qual valor.

**Frente:** Qual variável dá o usuário conectado? E o processo?
**Verso:** `$USERNAME` e `$JOB`.

**Frente:** Como obter data e hora legíveis?
**Verso:** `$ZDATETIME($HOROLOG, 3)` — formato `AAAA-MM-DD HH:MM:SS`.

**Frente:** Qual a resolução de `$HOROLOG` e como contorná-la?
**Verso:** Um segundo. Para ordenar eventos simultâneos, mantenha um contador com `$INCREMENT`.

**Frente:** Qual o valor padrão de `Foreach` num trigger, e o que isso implica?
**Verso:** `row` — o trigger dispara **apenas** em operações SQL, não em `%Save()`.

**Frente:** Como fazer um trigger disparar também em `%Save()` e `%DeleteId()`?
**Verso:** `Foreach = row/object`.

**Frente:** Diferença entre `Time = BEFORE` e `Time = AFTER`.
**Verso:** `BEFORE` pode cancelar a operação; `AFTER` registra o que já aconteceu.

**Frente:** Como cancelar uma operação num trigger?
**Verso:** `set %ok = 0` e `set %msg = "motivo"`, num trigger `BEFORE`.

**Frente:** O que significam `{X*O}`, `{X*N}` e `{X*C}`?
**Verso:** Valor antigo (*old*), valor novo (*new*) e indicador de que o campo mudou (*changed*).

**Frente:** Como obter o ID da linha dentro de um trigger?
**Verso:** `{%%ID}`. Há também `{%%TABLENAME}`, `{%%CLASSNAME}` e `{%%OPERATION}`.

**Frente:** Trigger ou callback para auditar valor antigo de um campo?
**Verso:** Trigger `AFTER`, porque `{Campo*O}` entrega o valor anterior sem custo extra.

**Frente:** Trigger ou callback para normalizar dados antes de gravar?
**Verso:** Callback `%OnBeforeSave` — ele trabalha no mundo dos objetos, onde a normalização é natural.

**Frente:** Por que a classe de auditoria não pode ter triggers de auditoria?
**Verso:** Cada entrada geraria outra entrada: laço infinito.

**Frente:** O que acontece com a trilha se ela é escrita dentro da transação e há rollback?
**Verso:** Ela é desfeita junto. Se você precisa registrar tentativas, escreva fora da transação.

**Frente:** Como escrever no `messages.log`?
**Verso:** `do ##class(%SYS.System).WriteToConsoleLog("texto")`. Com parcimônia: o arquivo é do administrador.

**Frente:** O que é preciso antes de disparar um evento de auditoria de usuário?
**Verso:** Declará-lo no Portal, com Source, Type e Name.

**Frente:** Trilha em global ou em classe persistente?
**Verso:** Classe persistente dá SQL, índices e consulta pelo Portal; global é mais rápida. Escolha por volume e por necessidade de consulta.

**Frente:** Qual o risco de colocar lógica pesada num trigger?
**Verso:** Ele roda em toda gravação; código lento ali degrada o sistema inteiro.

---

Digite CONTINUAR para o próximo capítulo.
