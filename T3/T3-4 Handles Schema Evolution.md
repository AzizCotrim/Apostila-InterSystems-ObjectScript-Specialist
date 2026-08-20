# Apostila InterSystems ObjectScript Specialist
## Capítulo 11 — T3.4 Handles Schema Evolution (Evolução do modelo de dados)

> Ainda em **T3 — IRIS Features**. Até aqui você construiu classes. Agora você vai **mudá-las com dados dentro** — que é a situação real de todo sistema em produção, e onde mora a maior parte dos acidentes.

---

## 1. O que você vai saber fazer ao terminar

1. Explicar o que acontece com os dados já gravados quando você **acrescenta**, **remove**, **renomeia** ou **muda o tipo** de uma propriedade.
2. Ler a **definição de armazenamento** e entender o mapa de posições que o compilador mantém.
3. Reconhecer o risco de **reaproveitamento de posição** ao remover e acrescentar propriedades.
4. Prever o efeito de mudar **`Required`**, **`MAXLEN`**, **`VALUELIST`**, `MINVAL` e `SCALE` sobre linhas antigas.
5. Acrescentar e remover **índices** em classes populadas, usando **`%BuildIndices()`** e **`%PurgeIndices()`**.
6. Encontrar **linhas legadas inválidas** com uma varredura de `%ValidateObject()`.
7. Usar **DDL de SQL** (`ALTER TABLE`) e entender que ela altera a **classe**.
8. Migrar dados com o padrão **acrescentar → preencher → cortar → remover**.
9. Escrever **migrações idempotentes e versionadas**, que podem rodar mais de uma vez sem estragar nada.
10. Renomear propriedades e classes com segurança.
11. Levar o projeto à versão **1.2**, com um mecanismo de migração de verdade.

---

## 2. O conceito em linguagem de gente

### 2.1 Mudar o formulário sem jogar fora as fichas antigas

O laboratório usa o formulário de papel há três anos. São 40 mil fichas preenchidas no arquivo.

Agora a direção pede: *"acrescente um campo de e-mail"*.

Você manda imprimir o formulário novo. As fichas novas terão o campo. E as 40 mil antigas? Elas continuam lá, **sem o campo**. Isso não é um problema — é o comportamento esperado. Uma ficha antiga simplesmente não tem e-mail.

Agora um pedido mais delicado: *"remova o campo 'telefone comercial', ninguém usa"*.

Você tira do formulário novo. E nas fichas antigas? **O telefone continua escrito lá.** O papel não se apaga sozinho. Ele fica ocupando espaço, invisível para o formulário novo, mas fisicamente presente.

E agora o pedido perigoso: *"acrescente um campo 'observações'"*.

Se a gráfica, por economia, imprimir o novo campo **exatamente na posição onde ficava o telefone comercial**, algo terrível acontece: ao ler uma ficha antiga, o telefone antigo aparecerá no campo de observações. Ninguém escreveu aquilo ali. O dado migrou de significado sozinho.

**Esse é o risco central deste capítulo**, e no IRIS ele é literal, não uma metáfora: o armazenamento é posicional, e posições liberadas podem ser reaproveitadas.

### 2.2 O que o IRIS faz e o que ele não faz

O IRIS é muito bom em algumas coisas e deliberadamente omisso em outras. Saber a divisão é meio caminho:

**O IRIS faz sozinho:**

- Acrescentar uma propriedade nova, sem tocar nos dados existentes.
- Manter as posições das propriedades já existentes quando você acrescenta outra.
- Recompilar a projeção SQL e a definição de armazenamento.
- Aceitar dados antigos que não obedecem às regras novas — **enquanto ninguém os regravar**.

**O IRIS NÃO faz sozinho:**

- Preencher a propriedade nova nas linhas antigas.
- Apagar o dado da propriedade que você removeu.
- Converter valores quando você muda o tipo.
- Revalidar linhas antigas contra as regras novas.
- Popular índices novos com dados antigos.
- Renomear: para ele, renomear é **remover uma coisa e criar outra**.

Cada item da segunda lista é uma tarefa **sua**. É isso que se chama **migração**.

### 2.3 A regra da validação preguiçosa

Este é o comportamento que mais confunde, e é fácil de lembrar quando enunciado assim:

> **As regras de validação valem no momento da gravação, não no momento em que são declaradas.**

Você tem 40 mil pacientes gravados. Você acrescenta `[ Required ]` ao campo `Sex`. O que acontece?

- **Nada, imediatamente.** As 40 mil linhas continuam lá, muitas com sexo em branco.
- Consultas continuam funcionando e devolvendo essas linhas.
- Mas na primeira vez que alguém abrir uma dessas fichas e mandar gravar — mesmo que só para corrigir um telefone —, o `%Save()` **falha**, exigindo o sexo.

O resultado é um sistema que funciona perfeitamente até um dia começar a recusar gravações de registros antigos, aparentemente ao acaso. O usuário reporta "não consigo salvar a ficha da Maria" e ninguém entende.

**A lição:** ao apertar uma regra, você tem que **varrer o legado** e corrigi-lo, ou aceitar conscientemente que ele quebrará quando tocado.

### 2.4 O ciclo seguro de uma mudança

Toda mudança arriscada pode ser dividida em quatro passos que **nunca quebram o sistema**:

1. **Acrescentar** — crie o novo, sem mexer no velho. Nesse momento os dois coexistem.
2. **Preencher** — copie/converta os dados do velho para o novo, em lote.
3. **Cortar** — passe o código a usar apenas o novo. O velho continua lá, mas ninguém o lê.
4. **Remover** — só depois de tudo estável, elimine o velho.

Compare com a alternativa ingênua: mudar tudo de uma vez e torcer. A diferença é que, no ciclo de quatro passos, **cada etapa é reversível** e o sistema nunca fica num estado inconsistente.

Esse é o padrão usado por qualquer equipe séria, em qualquer banco de dados. Ele tem outros nomes — *expand and contract*, escrita dupla —, mas a ideia é sempre esta.

### 2.5 Migração precisa ser repetível

Uma migração vai rodar em vários lugares: na sua máquina, no ambiente de testes, no de homologação, na produção. E, na produção, pode ser interrompida no meio por uma queda.

Portanto ela precisa de duas propriedades:

**Idempotente** — rodar duas vezes produz o mesmo resultado que rodar uma vez. Se ela já preencheu o campo, rodar de novo não deve estragar o que foi preenchido depois à mão.

**Versionada** — o sistema registra em que versão de esquema ele está, e cada migração sabe se já foi aplicada.

Analogia: é a diferença entre *"cole uma etiqueta em toda ficha"* e *"cole uma etiqueta em toda ficha que ainda não tem etiqueta"*. A segunda pode ser interrompida e retomada. A primeira, não.

---

## 3. A sintaxe e os mecanismos

### 3.1 O mapa de posições

Recapitulando o Capítulo 8: os dados de um objeto ficam num nó como uma lista, e o bloco `Storage` guarda o mapa.

```objectscript
Storage Default
{
<Data name="PatientDefaultData">
<Value name="1"><Value>%%CLASSNAME</Value></Value>
<Value name="2"><Value>Name</Value></Value>
<Value name="3"><Value>BirthDate</Value></Value>
<Value name="4"><Value>RecordNumber</Value></Value>
</Data>
<DataLocation>^LabStudy.PatientD</DataLocation>
...
}
```

Três fatos que decorrem disso e explicam tudo neste capítulo:

1. **A propriedade nova entra no fim da lista.** As posições existentes não se movem, e por isso os dados antigos continuam sendo lidos corretamente.
2. **A propriedade removida deixa o valor gravado onde estava.** O mapa deixa de apontar para aquela posição, mas os bytes permanecem no nó.
3. **Uma posição liberada pode vir a ser ocupada por uma propriedade futura.** Quando isso acontece, os dados antigos daquela posição passam a ser lidos como se fossem da propriedade nova.

O comportamento exato de reaproveitamento depende da versão e da sequência de operações: **verificar na documentação oficial**. O que você precisa levar deste capítulo é a **postura**: nunca conte com o reaproveitamento não acontecer. Se você removeu uma propriedade de uma classe com dados, **limpe aquela posição** antes de acrescentar outra, ou não remova — apenas pare de usar.

### 3.2 Acrescentar uma propriedade

```objectscript
Property Email As %String(MAXLEN = 120);
```

Compile. Efeitos:

- Linhas antigas: a propriedade lê como **string vazia**.
- Nenhum dado é reescrito.
- A coluna aparece no SQL, com `NULL` para as linhas antigas.
- Se houver `InitialExpression`, ele vale **apenas para objetos novos** — linhas antigas **não** recebem o valor padrão.

Este último ponto engana muita gente. `InitialExpression` roda no `%New()`, e as linhas antigas não vão passar por `%New()` nunca mais.

### 3.3 Remover uma propriedade

Apague a linha da classe e compile. Efeitos:

- A coluna some do SQL.
- O código que a referenciava **não compila mais**.
- **O dado continua no nó da global.**

Para limpar de verdade, é preciso reescrever os dados. A forma mais simples e segura, quando o volume permite, é regravar todos os objetos:

```objectscript
set rs = ##class(%SQL.Statement).%ExecDirect(, "SELECT %ID AS Id FROM LabStudy.PATIENT")
while rs.%Next() {
    set obj = ##class(LabStudy.Patient).%OpenId(rs.%Get("Id"))
    continue:'$ISOBJECT(obj)
    do obj.%Save()
}
```

Ao regravar, o IRIS monta o nó de novo, **a partir do mapa atual** — e a posição órfã desaparece.

Cuidado: essa varredura dispara callbacks, triggers e validações. Num sistema com auditoria, ela gera 40 mil entradas de trilha. Planeje.

### 3.4 Renomear uma propriedade

**Não existe renomeação automática.** Trocar `Property Fone` por `Property Phone` e compilar é interpretado como: remova `Fone`, crie `Phone`. Resultado: `Phone` nasce vazia em todas as linhas, e o valor de `Fone` fica órfão.

O caminho correto é o ciclo de quatro passos:

```objectscript
// Passo 1: acrescentar, mantendo a antiga
Property Fone As %String(MAXLEN = 20);      // legado, não usar
Property Phone As %String(MAXLEN = 20);     // novo
```

```objectscript
// Passo 2: preencher em lote
set rs = ##class(%SQL.Statement).%ExecDirect(, "SELECT %ID AS Id FROM LabStudy.PATIENT WHERE Phone IS NULL AND Fone IS NOT NULL")
while rs.%Next() {
    set obj = ##class(LabStudy.Patient).%OpenId(rs.%Get("Id"))
    continue:'$ISOBJECT(obj)
    set obj.Phone = obj.Fone
    do obj.%Save()
}
```

```objectscript
// Passo 3: o código passa a usar só Phone
// Passo 4: depois de estável, remover Fone e regravar
```

Repare no `WHERE Phone IS NULL AND Fone IS NOT NULL` do passo 2: é isso que torna a migração **idempotente**. Rodar de novo não sobrescreve o que já foi migrado nem o que foi editado depois.

### 3.5 Mudar o tipo de uma propriedade

```objectscript
Property Code As %String;      // antes
Property Code As %Integer;     // depois
```

Efeitos:

- **Os dados gravados não mudam.** Continuam sendo o que eram.
- A leitura devolve o que está lá — inclusive um texto num campo agora declarado como inteiro.
- A **validação** só age na próxima gravação. Uma linha com `"ABC"` num `%Integer` sobrevive até alguém tentar salvá-la.
- Índices existentes podem ficar **inconsistentes**, porque a ordenação de texto e a de número são diferentes.

Regra: **mudança de tipo com dados existentes exige conversão explícita e reconstrução de índices.** Não é uma mudança "só de declaração".

Mudanças especialmente perigosas, que exigem migração completa:

- de valor único para **coleção** (`list Of`, `array Of`);
- para ou de **`%SerialObject`** embutido;
- de propriedade simples para **`Relationship`**;
- alteração de **`IdKey`** — que muda a própria identidade dos objetos.

### 3.6 Apertar uma regra de validação

| Mudança | Efeito nas linhas antigas |
|---|---|
| acrescentar `Required` | continuam gravadas; falham ao regravar se estiverem vazias |
| reduzir `MAXLEN` | continuam gravadas com o texto longo; falham ao regravar |
| acrescentar/reduzir `VALUELIST` | valores fora da lista sobrevivem até a regravação |
| acrescentar `MINVAL`/`MAXVAL` | idem |
| reduzir `SCALE` | valores antigos mantêm as casas; são arredondados na regravação |
| acrescentar índice `Unique` | duplicatas existentes **não** são detectadas até a regravação, e o índice fica inconsistente |

O último caso é o mais perigoso: um índice único criado sobre dados que já têm duplicatas produz um índice que **mente**. Sempre verifique a unicidade **antes**:

```sql
SELECT RecordNumber, COUNT(*) AS Qtd
FROM LabStudy.PATIENT
GROUP BY RecordNumber
HAVING COUNT(*) > 1
```

Se essa consulta devolver linhas, resolva as duplicatas **antes** de declarar o índice.

### 3.7 Encontrar o legado inválido

O procedimento padrão depois de apertar qualquer regra:

```objectscript
ClassMethod FindInvalid(className As %String) As %Integer
{
    set sql = "SELECT %ID AS Id FROM "_className
    set rs = ##class(%SQL.Statement).%ExecDirect(, sql)

    set bad = 0
    while rs.%Next() {
        set obj = $CLASSMETHOD(className, "%OpenId", rs.%Get("Id"))
        continue:'$ISOBJECT(obj)

        set sc = obj.%ValidateObject()
        if $$$ISERR(sc) {
            set bad = bad + 1
            write "id ", rs.%Get("Id"), ": ", $SYSTEM.Status.GetErrorText(sc), !
        }
    }

    quit bad
}
```

`%ValidateObject()` valida **sem gravar**, como você viu no Capítulo 2. Isso permite fazer o levantamento completo do estrago sem alterar nada.

### 3.8 Índices em classes populadas

**Acrescentar um índice:**

```objectscript
Index EmailIdx On Email;
```

Compile e **reconstrua**:

```
LABSTUDY>DO ##class(LabStudy.Patient).%BuildIndices()
```

Sem isso, o índice existe vazio e as consultas que o usarem devolverão menos linhas do que deveriam — um erro silencioso e grave.

**Reconstruir apenas alguns índices:**

```objectscript
do ##class(LabStudy.Patient).%BuildIndices($LISTBUILD("EmailIdx"))
```

**Remover as entradas de um índice:**

```objectscript
do ##class(LabStudy.Patient).%PurgeIndices($LISTBUILD("EmailIdx"))
```

Útil quando você remove um índice da classe: apague as entradas antes, senão elas ficam órfãs na global de índices.

**Conferir consistência:**

```objectscript
set sc = ##class(LabStudy.Patient).%ValidateIndices()
```

Compara os índices com os dados e aponta divergências. Em bases grandes é uma operação cara; rode em janela de manutenção. Os parâmetros disponíveis e o comportamento exato variam por versão: **verificar na documentação oficial**.

> **Confirme os três nomes antes de escrever a sua rotina de manutenção.**
> `%BuildIndices`, `%PurgeIndices` e `%ValidateIndices` são métodos herdados de
> `%Persistent`, e a disponibilidade e a assinatura de cada um mudaram ao longo
> das versões. A verificação leva uma linha por método:
>
> ```
> LABSTUDY>WRITE ##class(%Dictionary.CompiledMethod).%ExistsId("LabStudy.Patient||%ValidateIndices"), !
> 1
> ```
>
> E, para ver a assinatura completa dos métodos herdados de `%Persistent`:
>
> ```
> LABSTUDY>DO $SYSTEM.OBJ.ShowClass("%Library.Persistent")
> ```
>
> O mesmo vale para `$SYSTEM.SQL.Stats.Table.GatherTableStats()`, da seção 3.4:
> em versões anteriores a operação chamava-se *Tune Table* e era invocada por
> outro caminho. Use `DO $SYSTEM.SQL.Help()` para listar o que existe na sua
> instalação. **Uma rotina de manutenção que chama um método inexistente falha
> exatamente quando você mais precisa dela.**

### 3.9 Evolução por DDL de SQL

O IRIS aceita DDL padrão, e ela **altera a definição da classe**:

```sql
ALTER TABLE LabStudy.PATIENT ADD COLUMN Email VARCHAR(120)
ALTER TABLE LabStudy.PATIENT DROP COLUMN Email
CREATE INDEX EmailIdx ON LabStudy.PATIENT (Email)
DROP INDEX EmailIdx
```

Isso é muito prático para quem vem do mundo SQL, mas há duas consequências:

1. A classe é **modificada e recompilada**. Se o seu código-fonte está sob controle de versão, ele fica desatualizado em relação ao servidor — e a próxima compilação a partir do repositório pode desfazer a mudança.
2. Classes criadas por DDL têm características ligeiramente diferentes das criadas por definição de classe (por exemplo, quanto ao gerenciamento do armazenamento).

**Recomendação:** escolha **uma** fonte da verdade. Se o projeto usa arquivos de classe versionados, faça a evolução nos arquivos e compile. Use DDL para correções pontuais e para integração com ferramentas que só falam SQL.

### 3.10 Renomear uma classe

Também não é automático. As opções:

- **Exportar e reimportar** com o nome novo, depois migrar os dados da global antiga para a nova.
- **Criar a classe nova, migrar, e apagar a antiga** — o ciclo de quatro passos aplicado a classes inteiras.
- Ferramentas de refatoração da extensão do VS Code podem ajudar com as **referências no código**, mas não movem dados.

Lembre do Capítulo 8: o nome da global vem do nome da classe. Renomear a classe significa que os dados antigos ficam numa global com o nome antigo, invisíveis para a classe nova.

### 3.11 Inspecionando o esquema por código

O IRIS descreve as próprias classes em tabelas do pacote `%Dictionary`:

```objectscript
set rs = ##class(%SQL.Statement).%ExecDirect(,
    "SELECT Name, Type, Required FROM %Dictionary.CompiledProperty "
    _"WHERE parent = ? ORDER BY SequenceNumber", "LabStudy.Patient")

while rs.%Next() {
    write rs.%Get("Name"), " : ", rs.%Get("Type"), !
}
```

Isso permite escrever migrações que **descobrem** o estado atual do esquema antes de agir — por exemplo, "só acrescente esta propriedade se ela ainda não existir". As classes principais são `%Dictionary.CompiledClass`, `%Dictionary.CompiledProperty`, `%Dictionary.CompiledIndex` e as versões `...Definition` correspondentes. A lista completa de colunas: **verificar na documentação oficial**.

---

## 4. Exemplo comentado

Um pequeno mecanismo de migração versionada:

Arquivo `src/LabStudy/Demo/Migrator.cls`:

```objectscript
/// A minimal, idempotent, versioned migration runner.
Class LabStudy.Demo.Migrator Extends %RegisteredObject
{

/// Highest migration step implemented in this class.
Parameter LATEST = 3;

/// Where the applied schema version is recorded.
Parameter VERSIONGLOBAL = "^LabStudyDemoSchema";

/// Current schema version of this namespace.
ClassMethod CurrentVersion() As %Integer
{
    quit $GET(@..#VERSIONGLOBAL@("version"), 0)
}

/// Records that a step was applied.
ClassMethod MarkApplied(step As %Integer, note As %String = "") As %Status [ Private ]
{
    set @..#VERSIONGLOBAL@("version") = step
    set @..#VERSIONGLOBAL@("history", step, "when") = $ZDATETIME($HOROLOG, 3)
    set @..#VERSIONGLOBAL@("history", step, "user") = $USERNAME
    set @..#VERSIONGLOBAL@("history", step, "note") = note
    quit $$$OK
}

/// Runs every step that has not been applied yet.
ClassMethod Upgrade() As %Status
{
    set from = ..CurrentVersion()

    write "current schema version: ", from, !
    write "latest available      : ", ..#LATEST, !

    if from >= ..#LATEST {
        write "nothing to do", !
        quit $$$OK
    }

    for step = (from + 1):1:..#LATEST {
        write !, "-- applying step ", step, " --", !

        set sc = $CLASSMETHOD($CLASSNAME(), "Step"_step)

        if $$$ISERR(sc) {
            write "step ", step, " FAILED, stopping here", !
            do $SYSTEM.Status.DisplayError(sc)
            quit
        }

        do ..MarkApplied(step, "ok")
        write "step ", step, " applied", !
    }

    write !, "schema version is now ", ..CurrentVersion(), !
    quit $$$OK
}

/// Step 1: nothing but a marker, to prove the mechanism works.
ClassMethod Step1() As %Status
{
    write "  (baseline)", !
    quit $$$OK
}

/// Step 2: backfill a status field that new rows get by default
/// but legacy rows were created without.
ClassMethod Step2() As %Status
{
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id FROM LabStudy.EXAM WHERE ResultStatus IS NULL OR ResultStatus = ''")

    set fixed = 0
    while rs.%Next() {
        set exam = ##class(LabStudy.Exam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)

        set exam.ResultStatus = $SELECT(exam.ResultValue '= "": "final", 1: "pending")

        set sc = exam.%Save()
        if $$$ISERR(sc) {
            write "  could not fix exam ", rs.%Get("Id"), ": ",
                  $SYSTEM.Status.GetErrorText(sc), !
            continue
        }
        set fixed = fixed + 1
    }

    write "  ", fixed, " exams backfilled", !
    quit $$$OK
}

/// Step 3: normalise record numbers and rebuild the affected index.
ClassMethod Step3() As %Status
{
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id, RecordNumber FROM LabStudy.PATIENT")

    set changed = 0
    while rs.%Next() {
        set current = rs.%Get("RecordNumber")
        set wanted = $ZCONVERT($ZSTRIP(current, "<>W"), "U")

        continue:current=wanted

        set patient = ##class(LabStudy.Patient).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(patient)

        set patient.RecordNumber = wanted
        set sc = patient.%Save()
        continue:$$$ISERR(sc)

        set changed = changed + 1
    }

    write "  ", changed, " record numbers normalised", !

    do ##class(LabStudy.Patient).%BuildIndices($LISTBUILD("RecordIdx"))
    write "  RecordIdx rebuilt", !

    quit $$$OK
}

/// Shows what has been applied and when.
ClassMethod History() As %Status
{
    write "schema version: ", ..CurrentVersion(), !
    write "------------------------------", !

    set step = ""
    for {
        set step = $ORDER(@..#VERSIONGLOBAL@("history", step))
        quit:step=""

        write "step ", step, " | ",
              $GET(@..#VERSIONGLOBAL@("history", step, "when")), " | ",
              $GET(@..#VERSIONGLOBAL@("history", step, "user")), " | ",
              $GET(@..#VERSIONGLOBAL@("history", step, "note")), !
    }
    quit $$$OK
}

/// Development helper: forgets everything. Never use in production.
ClassMethod ResetVersion() As %Status
{
    kill @..#VERSIONGLOBAL
    write "version history cleared", !
    quit $$$OK
}

}
```

Comentando as decisões:

- **A versão fica numa global**, não numa classe persistente. Motivo: a migração precisa funcionar mesmo quando as classes do sistema estão em estado inconsistente — inclusive antes de existirem. Uma global crua não depende de nada.
- **`$CLASSMETHOD($CLASSNAME(), "Step"_step)`** monta o nome do método a partir do número. Assim, acrescentar o passo 4 é escrever `Step4()` e subir o parâmetro `LATEST` — nenhum outro lugar muda. É a chamada dinâmica do Capítulo 3 resolvendo um problema estrutural.
- **O laço para no primeiro passo que falha.** Continuar depois de uma falha aplicaria passos sobre um estado que não é o esperado. Parar deixa o sistema numa versão intermediária conhecida, e o operador conserta e roda de novo.
- **`MarkApplied` só é chamado se o passo deu certo.** Se ele fosse chamado antes, uma falha marcaria como aplicado algo que não foi.
- **`Step2` usa `WHERE ResultStatus IS NULL OR ResultStatus = ''`** — só toca no que precisa. Rodar duas vezes não faz nada na segunda. Idempotência por construção.
- **`Step2` infere o estado a partir do valor**: se há valor, o exame era final; se não há, era pendente. Essa inferência é uma **decisão de negócio** documentada no código, e é o tipo de coisa que precisa ser combinada com quem entende o domínio, não escolhida pelo programador sozinho.
- **`Step3` compara antes de gravar** (`continue:current=wanted`). Sem isso, ele regravaria 40 mil pacientes toda vez, disparando 40 mil entradas de auditoria a cada execução.
- **`Step3` reconstrói o índice afetado**, e apenas ele. Reconstruir todos seria desperdício.
- **`History` existe porque migração sem rastro é migração que ninguém consegue diagnosticar.** Saber *quando* e *por quem* um passo rodou resolve metade dos mistérios de produção.
- **`ResetVersion` está marcado como ferramenta de desenvolvimento, com aviso explícito.** Deixar a porta aberta sem placa é convite ao acidente.

### 4.1 Usando no Terminal

```
LABSTUDY>DO ##class(LabStudy.Demo.Migrator).CurrentVersion()

LABSTUDY>WRITE ##class(LabStudy.Demo.Migrator).CurrentVersion(), !
0

LABSTUDY>DO ##class(LabStudy.Demo.Migrator).Upgrade()
current schema version: 0
latest available      : 3

-- applying step 1 --
  (baseline)
step 1 applied

-- applying step 2 --
  2 exams backfilled
step 2 applied

-- applying step 3 --
  0 record numbers normalised
  RecordIdx rebuilt
step 3 applied

schema version is now 3

LABSTUDY>DO ##class(LabStudy.Demo.Migrator).Upgrade()
current schema version: 3
latest available      : 3
nothing to do

LABSTUDY>DO ##class(LabStudy.Demo.Migrator).History()
schema version: 3
------------------------------
step 1 | 2026-08-19 17:04:11 | _SYSTEM | ok
step 2 | 2026-08-19 17:04:11 | _SYSTEM | ok
step 3 | 2026-08-19 17:04:12 | _SYSTEM | ok
```

A segunda execução não fez nada. Esse é o comportamento que permite colocar `Upgrade()` na inicialização do sistema sem medo.

---

## 5. Variações e detalhes

### 5.1 Compilando classes dependentes

Quando você altera uma classe da qual outras herdam, ou que outras referenciam, as dependentes também precisam ser recompiladas.

```
LABSTUDY>DO $SYSTEM.OBJ.CompilePackage("LabStudy", "ck")
```

A extensão do VS Code costuma cuidar disso ao salvar, mas em migrações executadas no servidor você deve compilar explicitamente o pacote inteiro. Sintomas de dependente não recompilada são erros estranhos e intermitentes, que somem depois de uma recompilação geral.

Há flags específicas para forçar a recompilação de subclasses e dependentes: **verificar na documentação oficial** para a letra correspondente na sua versão.

### 5.2 Migração dentro ou fora de transação

Envolver uma migração inteira numa transação parece prudente e quase sempre é **errado**:

- 40 mil gravações numa transação só acumulam um journal enorme e mantêm travas por muito tempo.
- Uma falha no registro 39.999 desfaz tudo, e você recomeça do zero.

O padrão recomendado é o oposto: **transação por unidade de trabalho**, com o progresso registrado. Cada registro (ou cada lote de mil) é uma transação própria. Se cair no meio, o que já foi feito está feito, e a migração idempotente retoma de onde parou.

Reserve a transação única para migrações **pequenas** ou para aquelas em que um estado intermediário seria realmente inaceitável.

### 5.3 Migração em base grande: janela e lotes

Para volumes altos, três cuidados práticos:

- **Processe em lotes** com um `TOP` e um marcador de progresso, em vez de abrir um cursor sobre milhões de linhas.
- **Registre o último ID processado**, para poder retomar.
- **Considere a carga**: uma varredura que dispara triggers e auditoria multiplica o custo por três ou quatro.

E uma decisão importante: se a migração só precisa **corrigir valores**, sem passar por regras de negócio, um `UPDATE` de SQL é ordens de grandeza mais rápido que abrir objeto por objeto — ao custo de não disparar callbacks. Como você viu no Capítulo 9, essa é uma escolha consciente, não um descuido.

### 5.4 Compatibilidade para trás durante a transição

Durante os passos 1 a 3 do ciclo, o sistema convive com dois formatos. Duas técnicas ajudam:

**Propriedade calculada de compatibilidade:**

```objectscript
/// Deprecated. Reads from the new field.
Property Fone As %String [ Calculated ];

Method FoneGet() As %String [ CodeMode = expression ]
{
..Phone
}
```

Assim, código antigo que ainda lê `Fone` continua funcionando, lendo o campo novo.

**Escrita dupla:**

```objectscript
Method PhoneSet(value As %String) As %Status
{
    set i%Phone = value
    set i%Fone = value        // mantém o legado sincronizado durante a transição
    quit $$$OK
}
```

*(A notação `i%Propriedade` acessa a variável de instância diretamente, sem passar pelo método `Set` — evitando recursão infinita. Ela é o mecanismo interno por trás das propriedades; use-a apenas dentro de métodos `Get`/`Set` da própria classe.)*

Ambas são **temporárias por definição**. Escreva a data de remoção no comentário, ou elas ficarão para sempre.

### 5.5 Evolução e a projeção JSON

Se a classe usa `%JSON.Adaptor`, acrescentar uma propriedade muda o JSON exportado — e isso é uma mudança de contrato com quem consome a API.

Regras de convivência que valem a pena conhecer:

- **Acrescentar um campo** costuma ser seguro: consumidores bem escritos ignoram campos desconhecidos.
- **Remover ou renomear um campo** quebra consumidores. Use `%JSONFIELDNAME` para manter o nome externo estável mesmo renomeando a propriedade interna — é exatamente para isso que esse parâmetro existe.
- **Mudar o tipo** de um campo (número para texto, por exemplo) quebra consumidores silenciosamente.

Ou seja: `%JSONFIELDNAME` é uma ferramenta de evolução de esquema, não apenas de estilo.

### 5.6 O que fazer quando a mudança é grande demais

Às vezes o modelo novo é incompatível com o antigo a ponto de não haver caminho gradual. Nesse caso, o padrão é:

1. Criar uma **classe nova**, com o modelo correto.
2. Escrever uma migração que **lê da antiga e escreve na nova**.
3. Rodar, conferir os totais, comparar amostras.
4. Cortar o sistema para a classe nova.
5. Manter a antiga somente leitura por um período de segurança.
6. Só então remover.

É mais trabalho do que alterar a classe existente — e é a única forma de fazer isso sem uma janela de indisponibilidade e sem risco de perder dados.

---

## 6. Pegadinhas e erros comuns

**1) Achar que `InitialExpression` preenche linhas antigas.**
Ele roda no `%New()`. Linhas antigas ficam vazias.

**2) Renomear uma propriedade editando o nome e compilando.**
Isso é remover e criar. O dado antigo fica órfão e o novo campo nasce vazio.

**3) Remover uma propriedade e acrescentar outra em seguida.**
Risco de a nova ocupar a posição liberada e passar a ler dados que não são dela.

**4) Acrescentar `Required` e achar que o sistema continua igual.**
Continua — até alguém tentar regravar uma linha antiga incompleta, e aí falha.

**5) Reduzir `MAXLEN` sem varrer o legado.**
Textos longos sobrevivem até a próxima gravação, quando começam a falhar.

**6) Criar índice `Unique` sobre dados que já têm duplicatas.**
O índice fica inconsistente. Verifique com `GROUP BY ... HAVING COUNT(*) > 1` antes.

**7) Criar índice numa classe populada e não rodar `%BuildIndices()`.**
Consultas devolvem menos linhas do que deveriam, sem erro.

**8) Remover índice sem `%PurgeIndices()`.**
Entradas órfãs permanecem na global de índices.

**9) Mudar o tipo de uma propriedade e achar que os dados foram convertidos.**
Não foram. É preciso converter explicitamente e reconstruir índices.

**10) Envolver uma migração de milhões de linhas numa única transação.**
Journal gigante, travas longas, e uma falha no fim desfaz tudo.

**11) Escrever migração não idempotente.**
Rodar duas vezes estraga dados ou sobrescreve correções manuais feitas depois.

**12) Marcar o passo como aplicado antes de ele terminar.**
Uma falha deixa registrado que algo foi feito, quando não foi.

**13) Misturar DDL de SQL com arquivos de classe versionados sem definir a fonte da verdade.**
A próxima compilação a partir do repositório desfaz a alteração feita por DDL.

**14) Renomear uma classe e esperar que os dados venham junto.**
Não vêm: a global tem o nome da classe antiga.

**15) Rodar uma migração que regrava tudo sem pensar na auditoria.**
Uma varredura de 40 mil registros gera 40 mil entradas de trilha e dispara todos os triggers.

**16) Não conferir o resultado da migração.**
Compare contagens antes e depois. Uma migração sem verificação é uma esperança, não um procedimento.

---

## 7. MÃO NA MASSA

---

### Exercício 11.1 — Acrescentar propriedade a uma classe com dados

**a) Enunciado:** Crie `LabStudy.Demo.Ev1` com `Code` e `Name`, grave três linhas, e **depois** acrescente `Category As %String [ InitialExpression = "general" ]`.

Responda, com evidência no Terminal:
1. As linhas antigas ganharam a categoria padrão?
2. O que aparece no SQL para elas?
3. E numa linha criada **depois** da mudança?
4. Inspecione a global de dados antes e depois. O que mudou?

**b) Dica:** Rode `ZWRITE ^LabStudy.Demo.Ev1D` antes de acrescentar a propriedade e de novo depois.

**c) Como testar:** As linhas antigas devem continuar com categoria vazia.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Primeiro, `src/LabStudy/Demo/Ev1.cls`:

```objectscript
/// Step 1: the original class.
Class LabStudy.Demo.Ev1 Extends %Persistent
{
Property Code As %String(MAXLEN = 20);
Property Name As %String(MAXLEN = 100);
}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Ev1).%KillExtent()

LABSTUDY>FOR pair="A:Alpha","B:Beta","C:Gamma" { SET o=##class(LabStudy.Demo.Ev1).%New() SET o.Code=$PIECE(pair,":",1),o.Name=$PIECE(pair,":",2) DO o.%Save() }

LABSTUDY>ZWRITE ^LabStudy.Demo.Ev1D
^LabStudy.Demo.Ev1D=3
^LabStudy.Demo.Ev1D(1)=$lb("","A","Alpha")
^LabStudy.Demo.Ev1D(2)=$lb("","B","Beta")
^LabStudy.Demo.Ev1D(3)=$lb("","C","Gamma")
```

Agora acrescente a propriedade e recompile:

```objectscript
Property Category As %String(MAXLEN = 30) [ InitialExpression = "general" ];
```

```
LABSTUDY>ZWRITE ^LabStudy.Demo.Ev1D
^LabStudy.Demo.Ev1D=3
^LabStudy.Demo.Ev1D(1)=$lb("","A","Alpha")
^LabStudy.Demo.Ev1D(2)=$lb("","B","Beta")
^LabStudy.Demo.Ev1D(3)=$lb("","C","Gamma")

LABSTUDY>SET o = ##class(LabStudy.Demo.Ev1).%OpenId(1)
LABSTUDY>WRITE "[", o.Category, "]", !
[]

LABSTUDY>SET novo = ##class(LabStudy.Demo.Ev1).%New()
LABSTUDY>WRITE "[", novo.Category, "]", !
[general]
LABSTUDY>SET novo.Code = "D", novo.Name = "Delta"
LABSTUDY>DO novo.%Save()

LABSTUDY>ZWRITE ^LabStudy.Demo.Ev1D
^LabStudy.Demo.Ev1D=4
^LabStudy.Demo.Ev1D(1)=$lb("","A","Alpha")
^LabStudy.Demo.Ev1D(2)=$lb("","B","Beta")
^LabStudy.Demo.Ev1D(3)=$lb("","C","Gamma")
^LabStudy.Demo.Ev1D(4)=$lb("","D","Delta","general")

LABSTUDY>DO ##class(%SQL.Statement).%ExecDirect(,"SELECT %ID, Code, Category FROM LabStudy.Demo.Ev1").%Display()
```

**Por que cada resultado:**

- **A global não mudou nem um byte** ao acrescentar a propriedade. Compilar altera a **definição**, não os **dados**. Isso é o que torna a operação segura e instantânea, mesmo com milhões de linhas.
- **A linha 4, criada depois, tem um elemento a mais** na lista: `"general"`. A propriedade nova entrou **no fim**, exatamente como a seção 3.1 previu.
- **As linhas antigas leem `Category` como vazio.** Não há erro: uma lista mais curta do que o mapa simplesmente devolve vazio para as posições ausentes — o mesmo comportamento de `$LISTGET` que você viu no Capítulo 10.
- **`InitialExpression` não alcançou o legado.** Ele roda no `%New()`, e as linhas 1 a 3 não vão passar por `%New()` de novo. Se você quiser `"general"` nelas, precisa de uma migração — que é o exercício 11.5.
- **No SQL, as três primeiras aparecem com `Category` nula.** Do ponto de vista do SQL, esse é exatamente o significado correto: valor desconhecido.

---

### Exercício 11.2 — Remover propriedade e ver o dado órfão

**a) Enunciado:** Em `LabStudy.Demo.Ev1`, acrescente `Temp As %String`, grave um valor nela em todas as linhas, e depois **remova** a propriedade da classe.

Responda com evidência:
1. O valor sumiu da global?
2. O que acontece se você tentar `SELECT Temp`?
3. Regrave um dos objetos. O que muda na global?
4. Acrescente uma propriedade nova. Onde ela é gravada?

**b) Dica:** Inspecione a global após cada passo. É o único jeito de ver o que está acontecendo de verdade.

**c) Como testar:** No passo 1, o valor deve continuar visível na global.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Acrescente `Property Temp As %String(MAXLEN = 30);` e compile.

```
LABSTUDY>SET rs=##class(%SQL.Statement).%ExecDirect(,"SELECT %ID AS Id FROM LabStudy.Demo.Ev1")
LABSTUDY>WHILE rs.%Next() { SET o=##class(LabStudy.Demo.Ev1).%OpenId(rs.%Get("Id")) SET o.Temp="TEMP-"_o.Code DO o.%Save() }

LABSTUDY>ZWRITE ^LabStudy.Demo.Ev1D
^LabStudy.Demo.Ev1D=4
^LabStudy.Demo.Ev1D(1)=$lb("","A","Alpha","","TEMP-A")
^LabStudy.Demo.Ev1D(2)=$lb("","B","Beta","","TEMP-B")
^LabStudy.Demo.Ev1D(3)=$lb("","C","Gamma","","TEMP-C")
^LabStudy.Demo.Ev1D(4)=$lb("","D","Delta","general","TEMP-D")
```

Agora **remova** `Property Temp` da classe e recompile.

```
LABSTUDY>ZWRITE ^LabStudy.Demo.Ev1D
^LabStudy.Demo.Ev1D=4
^LabStudy.Demo.Ev1D(1)=$lb("","A","Alpha","","TEMP-A")
^LabStudy.Demo.Ev1D(2)=$lb("","B","Beta","","TEMP-B")
^LabStudy.Demo.Ev1D(3)=$lb("","C","Gamma","","TEMP-C")
^LabStudy.Demo.Ev1D(4)=$lb("","D","Delta","general","TEMP-D")

LABSTUDY>SET rs=##class(%SQL.Statement).%ExecDirect(,"SELECT Temp FROM LabStudy.Demo.Ev1")
LABSTUDY>WRITE rs.%SQLCODE, " ", rs.%Message, !
-29 Field 'Temp' not found in the applicable tables

LABSTUDY>SET o = ##class(LabStudy.Demo.Ev1).%OpenId(1)
LABSTUDY>DO o.%Save()

LABSTUDY>ZWRITE ^LabStudy.Demo.Ev1D(1)
^LabStudy.Demo.Ev1D(1)=$lb("","A","Alpha","")
```

**Por que cada resultado:**

- **O dado sobreviveu à remoção da propriedade.** A global continuou com `"TEMP-A"` no quinto elemento, apesar de a classe não conhecer mais aquele campo. Compilar não reescreve dados — este é o mesmo fato do exercício anterior, agora com consequência oposta.
- **O SQL recusou a coluna**, com `SQLCODE` negativo e a mensagem indicando campo não encontrado. A projeção SQL acompanha a definição da classe, não o conteúdo da global. Os dois divergiram, e só a global sabe a verdade. *(O número exato do `SQLCODE` varia por versão e não vale a pena decorar — o que resolve é sempre `%Message`, como visto no Capítulo 9.)*
- **Regravar o objeto limpou o resto.** O nó da linha 1 foi remontado a partir do mapa atual, que tem quatro posições. O quinto elemento simplesmente deixou de ser escrito. **Regravar é a forma de faxina.**
- **As linhas 2, 3 e 4 continuam com o dado órfão** — porque ninguém as regravou. Numa base real, você teria uma mistura: linhas limpas e linhas com resíduo, dependendo de quem foi editado desde a mudança.
- **E aqui está o risco da seção 3.1:** se agora você acrescentar uma propriedade nova e o compilador atribuir a ela a quinta posição, as linhas 2, 3 e 4 passarão a exibir `TEMP-B`, `TEMP-C` e `TEMP-D` nesse campo novo — dados que ninguém escreveu ali. Faça o teste na sua instalação e observe o que acontece; o comportamento pode variar por versão, e é exatamente por isso que a recomendação é **regravar tudo antes de acrescentar qualquer coisa depois de uma remoção**.

---

### Exercício 11.3 — Apertar uma regra e caçar o legado inválido

**a) Enunciado:** Em `LabStudy.Demo.Ev1`, os nomes atuais têm até 100 caracteres. Faça o seguinte:

1. Crie uma linha com um nome de 60 caracteres.
2. Reduza `MAXLEN` de `Name` para 30 e recompile.
3. Verifique se as consultas continuam funcionando.
4. Escreva `ClassMethod FindInvalid()` que varre a extensão com `%ValidateObject()` e lista os problemas.
5. Tente regravar a linha problemática e observe o erro.
6. Corrija-a e confirme.

**b) Dica:** Para gerar um texto longo: `$TRANSLATE($JUSTIFY("", 60), " ", "x")`.

**c) Como testar:** No passo 3, o `SELECT` deve devolver a linha longa normalmente. No passo 5, o `%Save()` deve falhar.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>SET long = $TRANSLATE($JUSTIFY("", 60), " ", "x")
LABSTUDY>SET o = ##class(LabStudy.Demo.Ev1).%New()
LABSTUDY>SET o.Code = "L", o.Name = long
LABSTUDY>WRITE $$$ISOK(o.%Save()), " id=", o.%Id(), !
1 id=5
```

Agora altere para `Property Name As %String(MAXLEN = 30);` e recompile.

Acrescente à classe:

```objectscript
/// Sweeps the extent looking for rows that no longer satisfy the rules.
ClassMethod FindInvalid() As %Integer
{
    set rs = ##class(%SQL.Statement).%ExecDirect(, "SELECT %ID AS Id FROM LabStudy.Demo.Ev1")

    set bad = 0
    while rs.%Next() {
        set obj = ..%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(obj)

        set sc = obj.%ValidateObject()
        continue:$$$ISOK(sc)

        set bad = bad + 1
        write "id ", rs.%Get("Id"), " : ", $SYSTEM.Status.GetErrorText(sc), !
    }

    write "-- ", bad, " invalid rows --", !
    quit bad
}
```

```
LABSTUDY>SET rs=##class(%SQL.Statement).%ExecDirect(,"SELECT %ID, Code, Name FROM LabStudy.Demo.Ev1 WHERE Code='L'")
LABSTUDY>DO rs.%Display()
   (a linha aparece normalmente, com os 60 caracteres)

LABSTUDY>WRITE ##class(LabStudy.Demo.Ev1).FindInvalid(), !
id 5 : ERROR #7207: Datatype value 'xxxxx...' length longer than MAXLEN allowed of 30
-- 1 invalid rows --
1

LABSTUDY>SET o = ##class(LabStudy.Demo.Ev1).%OpenId(5)
LABSTUDY>DO $SYSTEM.Status.DisplayError(o.%Save())
ERROR #7207: Datatype value 'xxxxx...' length longer than MAXLEN allowed of 30

LABSTUDY>SET o.Name = $EXTRACT(o.Name, 1, 30)
LABSTUDY>WRITE $$$ISOK(o.%Save()), !
1

LABSTUDY>WRITE ##class(LabStudy.Demo.Ev1).FindInvalid(), !
-- 0 invalid rows --
0
```

**Por que cada resultado:**

- **A consulta continuou funcionando com o valor de 60 caracteres.** Leitura não valida. O `MAXLEN` é uma regra de **entrada**, não de saída.
- **O `%Save()` falhou sem que ninguém tivesse mexido no nome.** O objeto foi aberto e regravado igual — e a regra nova o rejeitou. Este é o cenário do "não consigo salvar a ficha da Maria" descrito na seção 2.3, reproduzido em laboratório.
- **`FindInvalid` é o procedimento que deveria ter rodado ANTES da mudança**, não depois. Numa base real, você aperta a regra em desenvolvimento, roda a varredura em cópia da produção, mede o estrago e só então decide.
- **A correção foi truncar**, o que aqui é aceitável porque o dado era artificial. Numa situação real, truncar dados de pacientes seria inaceitável — você precisaria decidir com o negócio: aumentar o limite de volta, criar um campo de texto longo, ou tratar caso a caso.
- **`%ValidateObject` não gravou nada** durante a varredura. Você levantou o problema inteiro sem alterar uma linha.

---

### Exercício 11.4 — Índices em classe populada

**a) Enunciado:** Em `LabStudy.Demo.Ev1`:

1. Acrescente `Index CodeIdx On Code;` e compile, **sem** reconstruir.
2. Rode uma consulta com `WHERE Code = 'A'` e observe. Inspecione a global de índices.
3. Rode `%BuildIndices()` e repita.
4. Verifique a consistência com `%ValidateIndices()`.
5. Tente acrescentar `Index CodeUnique On Code [ Unique ]` depois de criar uma **duplicata** de `Code`. O que acontece?
6. Remova o índice `CodeIdx` da classe e limpe as entradas com `%PurgeIndices()`.

**b) Dica:** No passo 2, inspecione `^LabStudy.Demo.Ev1I`.

**c) Como testar:** Antes do `%BuildIndices()`, a global de índices deve estar vazia ou incompleta para `CodeIdx`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Acrescente `Index CodeIdx On Code;` e compile.

```
LABSTUDY>ZWRITE ^LabStudy.Demo.Ev1I
   (nada, ou apenas entradas de outros índices — CodeIdx está vazio)

LABSTUDY>DO ##class(LabStudy.Demo.Ev1).%BuildIndices()

LABSTUDY>ZWRITE ^LabStudy.Demo.Ev1I
^LabStudy.Demo.Ev1I("CodeIdx"," A",1)=""
^LabStudy.Demo.Ev1I("CodeIdx"," B",2)=""
^LabStudy.Demo.Ev1I("CodeIdx"," C",3)=""
^LabStudy.Demo.Ev1I("CodeIdx"," D",4)=""
^LabStudy.Demo.Ev1I("CodeIdx"," L",5)=""

LABSTUDY>SET sc = ##class(LabStudy.Demo.Ev1).%ValidateIndices()
LABSTUDY>WRITE $$$ISOK(sc), !
1
```

Agora crie a duplicata e tente o índice único:

```
LABSTUDY>SET o=##class(LabStudy.Demo.Ev1).%New() SET o.Code="A",o.Name="Alpha duplicado" DO o.%Save()

LABSTUDY>SET rs=##class(%SQL.Statement).%ExecDirect(,"SELECT Code, COUNT(*) AS Qtd FROM LabStudy.Demo.Ev1 GROUP BY Code HAVING COUNT(*)>1")
LABSTUDY>DO rs.%Display()
Code  Qtd
A     2
```

Acrescente `Index CodeUnique On Code [ Unique ];` e compile:

```
LABSTUDY>DO ##class(LabStudy.Demo.Ev1).%BuildIndices($LISTBUILD("CodeUnique"))
   (a reconstrução acusa a violação de unicidade, ou produz um índice
    inconsistente em que apenas um dos dois registros é alcançável)

LABSTUDY>SET rs=##class(%SQL.Statement).%ExecDirect(,"SELECT %ID FROM LabStudy.Demo.Ev1 WHERE Code='A'")
LABSTUDY>DO rs.%Display()
```

Limpando:

```
LABSTUDY>DO ##class(LabStudy.Demo.Ev1).%PurgeIndices($LISTBUILD("CodeIdx"))

LABSTUDY>ZWRITE ^LabStudy.Demo.Ev1I
   (as entradas de CodeIdx sumiram)
```

**Por que cada resultado:**

- **O índice nasceu vazio.** A compilação criou a definição, não os dados. Uma consulta que o otimizador decidisse usar devolveria zero linhas — o erro mais insidioso possível, porque não há mensagem alguma.
- **`%BuildIndices()` populou tudo de uma vez.** Repare no formato das entradas: `("CodeIdx", " A", 1) = ""`, com o valor normalizado em maiúsculas e precedido de um espaço, exatamente como você viu no Capítulo 8.
- **A verificação de duplicatas ANTES do índice único é o passo que ninguém lembra de fazer.** A consulta com `GROUP BY ... HAVING COUNT(*) > 1` custa segundos e evita horas de confusão. Faça dela um hábito.
- **Criar índice único sobre dados duplicados coloca o sistema num estado incoerente**: a declaração afirma que não há repetição, e há. A partir daí, buscas pelo índice podem devolver apenas um dos registros, e o outro fica inalcançável por aquele caminho, embora continue existindo na tabela.
- **`%PurgeIndices()` é a contrapartida de `%BuildIndices()`.** Remover o índice da classe sem purgar deixa lixo na global de índices, ocupando espaço e podendo confundir uma verificação futura.

---

### Exercício 11.5 — PROJETO CONTÍNUO: migração de verdade

**a) Enunciado:** O sistema do laboratório está na versão 1.1 e tem um passivo real acumulado ao longo da apostila:

- Exames criados **antes** do Capítulo 10 estão com `ResultStatus` vazio.
- Alguns `RecordNumber` podem ter sido gravados fora do padrão.
- Falta um campo de contato do paciente.

Construa o mecanismo de migração do projeto:

1. Crie `LabStudy.Schema` com:
   - `ClassMethod Version()` e `ClassMethod SetVersion(v)`, guardando em `^LabStudySchema`;
   - `ClassMethod Upgrade()` que aplica os passos pendentes, para no primeiro erro, e registra histórico com data, usuário, duração e contagem afetada;
   - `ClassMethod Report()` que mostra a versão atual e o histórico;
   - `ClassMethod Check()` que **verifica** o estado do esquema sem alterar nada: quantos exames sem estado, quantos números de registro fora do padrão, quantos pacientes inválidos.
2. Implemente os passos:
   - **Step1** — linha de base: apenas registra a versão inicial.
   - **Step2** — preenche `ResultStatus` dos exames legados, inferindo `final` quando há valor e `pending` quando não há.
   - **Step3** — normaliza `RecordNumber` (maiúsculas, sem espaços) e reconstrói `RecordIdx`.
   - **Step4** — acrescenta o campo `Email` a `LabStudy.Patient` (você faz isso editando a classe) e a migração apenas **verifica** que ele existe, usando `%Dictionary`.
3. Em `LabStudy.App`, suba para `"1.2"` e faça `Run()` chamar `LabStudy.Schema.Report()` no início.

**b) Dica:** Cada passo deve tocar **apenas** no que ainda precisa ser tocado, e deve devolver a contagem de registros afetados.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Schema).Check()
LABSTUDY>DO ##class(LabStudy.Schema).Upgrade()
LABSTUDY>DO ##class(LabStudy.Schema).Upgrade()
LABSTUDY>DO ##class(LabStudy.Schema).Report()
```

A segunda execução do `Upgrade()` não deve fazer nada.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Primeiro, acrescente a `src/LabStudy/Patient.cls`:

```objectscript
/// Contact e-mail. Added in schema version 4.
Property Email As %String(MAXLEN = 120, %JSONFIELDNAME = "email");
```

`src/LabStudy/Schema.cls`:

```objectscript
/// Versioned, idempotent schema migration for the LabStudy system.
/// Every step must be safe to run more than once.
Class LabStudy.Schema Extends %RegisteredObject
{

/// Highest step implemented here.
Parameter LATEST = 4;

/// Global that records the applied version and the history.
Parameter VERSIONGLOBAL = "^LabStudySchema";

/// Current schema version of this namespace.
ClassMethod Version() As %Integer [ CodeMode = expression ]
{
$GET(@..#VERSIONGLOBAL@("version"), 0)
}

/// Forces a version. Development helper only.
ClassMethod SetVersion(v As %Integer) As %Status
{
    set @..#VERSIONGLOBAL@("version") = v
    write "schema version forced to ", v, !
    quit $$$OK
}

/// Records a successful step.
ClassMethod Record(step As %Integer, affected As %Integer, seconds As %Numeric) As %Status [ Private ]
{
    set @..#VERSIONGLOBAL@("version") = step
    set @..#VERSIONGLOBAL@("history", step, "when")     = $ZDATETIME($HOROLOG, 3)
    set @..#VERSIONGLOBAL@("history", step, "user")     = $USERNAME
    set @..#VERSIONGLOBAL@("history", step, "affected") = affected
    set @..#VERSIONGLOBAL@("history", step, "seconds")  = seconds
    quit $$$OK
}

/// Read only inspection: what still needs to be migrated.
ClassMethod Check() As %Status
{
    new SQLCODE, %msg, a, b, c

    write "==============================", !
    write "Schema check", !
    write "==============================", !
    write "current version : ", ..Version(), !
    write "latest available: ", ..#LATEST, !
    write "------------------------------", !

    &sql(SELECT COUNT(*) INTO :a FROM LabStudy.EXAM
         WHERE ResultStatus IS NULL OR ResultStatus = '')
    write "exams without status      : ", +$GET(a), !

    &sql(SELECT COUNT(*) INTO :b FROM LabStudy.PATIENT
         WHERE RecordNumber <> UPPER(RecordNumber))
    write "record numbers not upper  : ", +$GET(b), !

    &sql(SELECT COUNT(*) INTO :c FROM %Dictionary.CompiledProperty
         WHERE parent = 'LabStudy.Patient' AND Name = 'Email')
    write "Email property present    : ", $SELECT(+$GET(c): "yes", 1: "NO"), !

    write "==============================", !
    quit $$$OK
}

/// Applies every pending step, stopping at the first failure.
ClassMethod Upgrade() As %Status
{
    set from = ..Version()

    write "schema version ", from, " -> target ", ..#LATEST, !

    if from >= ..#LATEST {
        write "already up to date", !
        quit $$$OK
    }

    for step = (from + 1):1:..#LATEST {
        write !, "--- step ", step, " ---", !

        set start = $ZHOROLOG
        set affected = 0

        set sc = $CLASSMETHOD($CLASSNAME(), "Step"_step, .affected)

        set seconds = $FNUMBER($ZHOROLOG - start, "", 3)

        if $$$ISERR(sc) {
            write "step ", step, " FAILED after ", seconds, "s", !
            do $SYSTEM.Status.DisplayError(sc)
            write "stopping. schema stays at version ", ..Version(), !
            quit
        }

        do ..Record(step, affected, seconds)
        write "step ", step, " ok: ", affected, " rows in ", seconds, "s", !
    }

    write !, "schema version is now ", ..Version(), !
    quit $$$OK
}

/// Step 1: baseline. Nothing to change.
ClassMethod Step1(Output affected As %Integer) As %Status
{
    set affected = 0
    write "  baseline recorded", !
    quit $$$OK
}

/// Step 2: legacy exams have no ResultStatus. Infer it from the value.
ClassMethod Step2(Output affected As %Integer) As %Status
{
    set affected = 0

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id FROM LabStudy.EXAM "
        _"WHERE ResultStatus IS NULL OR ResultStatus = ''")

    while rs.%Next() {
        set exam = ##class(LabStudy.Exam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)

        // Business rule agreed with the laboratory:
        // a legacy exam with a value was already finished.
        set exam.ResultStatus = $SELECT(exam.ResultValue '= "": "final", 1: "pending")

        if exam.ResultStatus = "final", exam.ResultDate = "" {
            set exam.ResultDate = exam.CollectedOn
        }

        set sc = exam.%Save()
        if $$$ISERR(sc) {
            write "  exam ", rs.%Get("Id"), " failed: ",
                  $SYSTEM.Status.GetErrorText(sc), !
            continue
        }

        set affected = affected + 1
    }

    quit $$$OK
}

/// Step 3: normalise record numbers and rebuild the unique index.
ClassMethod Step3(Output affected As %Integer) As %Status
{
    set affected = 0

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id, RecordNumber FROM LabStudy.PATIENT")

    while rs.%Next() {
        set current = rs.%Get("RecordNumber")
        set wanted = $ZCONVERT($ZSTRIP(current, "<>W"), "U")

        continue:current=wanted

        set patient = ##class(LabStudy.Patient).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(patient)

        set patient.RecordNumber = wanted

        set sc = patient.%Save()
        if $$$ISERR(sc) {
            write "  patient ", rs.%Get("Id"), " failed: ",
                  $SYSTEM.Status.GetErrorText(sc), !
            continue
        }

        set affected = affected + 1
    }

    do ##class(LabStudy.Patient).%BuildIndices($LISTBUILD("RecordIdx"))
    write "  RecordIdx rebuilt", !

    quit $$$OK
}

/// Step 4: confirms that the Email property was added to the class.
/// The class change itself is done in source; this step only verifies.
ClassMethod Step4(Output affected As %Integer) As %Status
{
    new SQLCODE, %msg, found

    set affected = 0

    &sql(SELECT COUNT(*) INTO :found FROM %Dictionary.CompiledProperty
         WHERE parent = 'LabStudy.Patient' AND Name = 'Email')

    if '+$GET(found) {
        quit $$$ERROR($$$GeneralError,
            "Property Email is missing from LabStudy.Patient. Add it and recompile before running this step.")
    }

    write "  Email property confirmed", !
    quit $$$OK
}

/// Shows the version and the full history.
ClassMethod Report() As %Status
{
    write "schema version: ", ..Version(), " (latest ", ..#LATEST, ")", !
    write "------------------------------------------------", !

    set step = ""
    for {
        set step = $ORDER(@..#VERSIONGLOBAL@("history", step))
        quit:step=""

        write "step ", step, " | ",
              $GET(@..#VERSIONGLOBAL@("history", step, "when")), " | ",
              $GET(@..#VERSIONGLOBAL@("history", step, "user")), " | ",
              $GET(@..#VERSIONGLOBAL@("history", step, "affected")), " rows | ",
              $GET(@..#VERSIONGLOBAL@("history", step, "seconds")), "s", !
    }
    write "------------------------------------------------", !
    quit $$$OK
}

}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.2";

ClassMethod Run() As %Status
{
    do ..About()
    write !
    do ##class(LabStudy.Schema).Report()
    write !
    do ##class(LabStudy.Reports).Dashboard()
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.Schema).Check()
==============================
Schema check
==============================
current version : 0
latest available: 4
------------------------------
exams without status      : 4
record numbers not upper  : 0
Email property present    : yes
==============================

LABSTUDY>DO ##class(LabStudy.Schema).Upgrade()
schema version 0 -> target 4

--- step 1 ---
  baseline recorded
step 1 ok: 0 rows in 0.001s

--- step 2 ---
step 2 ok: 4 rows in 0.018s

--- step 3 ---
  RecordIdx rebuilt
step 3 ok: 0 rows in 0.009s

--- step 4 ---
  Email property confirmed
step 4 ok: 0 rows in 0.002s

schema version is now 4

LABSTUDY>DO ##class(LabStudy.Schema).Upgrade()
schema version 4 -> target 4
already up to date

LABSTUDY>DO ##class(LabStudy.Schema).Check()
==============================
Schema check
==============================
current version : 4
latest available: 4
------------------------------
exams without status      : 0
record numbers not upper  : 0
Email property present    : yes
==============================

LABSTUDY>DO ##class(LabStudy.Schema).Report()
schema version: 4 (latest 4)
------------------------------------------------
step 1 | 2026-08-19 17:30:02 | _SYSTEM | 0 rows | 0.001s
step 2 | 2026-08-19 17:30:02 | _SYSTEM | 4 rows | 0.018s
step 3 | 2026-08-19 17:30:02 | _SYSTEM | 0 rows | 0.009s
step 4 | 2026-08-19 17:30:02 | _SYSTEM | 0 rows | 0.002s
------------------------------------------------
```

**Por que cada decisão:**

- **`Check()` existe separado de `Upgrade()`, e não altera nada.** Antes de rodar uma migração em produção, alguém precisa poder olhar e dizer "vai mexer em 4 exames". Migração que não pode ser inspecionada antes é migração que ninguém autoriza.
- **Cada passo devolve a contagem afetada por `Output`.** Isso vai para o histórico, e é o que permite auditar depois: "no dia 19, o passo 2 tocou em 4 exames". Sem essa contagem, o histórico só diz que algo aconteceu.
- **A duração é medida com `$ZHOROLOG`**, que devolve segundos com fração desde o início do dia. Numa migração de produção, saber que um passo levou 40 minutos da última vez é o que permite planejar a janela.
- **O passo 2 documenta a regra de negócio no comentário.** "Exame legado com valor já estava concluído" é uma **decisão**, não um fato técnico. Ela foi combinada com o laboratório, e o código registra isso. Daqui a dois anos, alguém vai perguntar por que os exames antigos estão como `final`, e a resposta estará ali.
- **O passo 2 também preenche `ResultDate` a partir de `CollectedOn`** quando ela está vazia. É uma aproximação declarada: não sabemos quando o resultado saiu, mas sabemos que não foi antes da coleta. Melhor um valor documentadamente aproximado do que um campo vazio que ninguém sabe interpretar.
- **`if exam.ResultStatus = "final", exam.ResultDate = ""`** usa vírgula como "e" — forma compacta e idiomática do ObjectScript para condições encadeadas.
- **O passo 3 compara antes de gravar.** Na execução acima ele afetou **zero** registros, porque os números já estavam normalizados. Uma migração que faz zero alterações e ainda assim é executada é o resultado **normal** e desejado: ela confirma que a condição já está satisfeita.
- **O passo 3 reconstrói o índice mesmo tendo afetado zero linhas.** É barato e garante o estado final independentemente do caminho percorrido. Migrações devem garantir o **estado**, não apenas executar ações.
- **O passo 4 não altera o esquema: ele verifica.** A alteração da classe é feita no código-fonte e versionada junto com o resto. A migração apenas **falha ruidosamente** se alguém esquecer de aplicar o código antes de rodar a migração. Essa separação — mudança de esquema no fonte, mudança de dados na migração — é a prática recomendada, porque mantém uma única fonte da verdade para a definição das classes, como discutido na seção 3.9.
- **`%Dictionary.CompiledProperty` é consultado por SQL comum.** O IRIS descreve a si mesmo em tabelas, e isso permite migrações que se adaptam ao estado real do servidor.
- **`SetVersion` está marcado como ferramenta de desenvolvimento.** Ele existe para você poder testar: `SetVersion(1)` e rodar `Upgrade()` de novo para ver os passos 2 a 4 executarem. Em produção, forçar versão é falsificar a história.

---

## 8. Quiz do capítulo

**Q1.** Você acrescenta uma propriedade com `InitialExpression = "X"` a uma classe com 1000 linhas gravadas. O que acontece com essas linhas?

- A) Todas recebem `"X"` automaticamente.
- B) Continuam com a propriedade vazia; `InitialExpression` só vale no `%New()`.
- C) A compilação falha.
- D) As linhas são apagadas.

---

**Q2.** Onde o compilador coloca uma propriedade nova no mapa de armazenamento?

- A) No início, empurrando as demais.
- B) No fim da lista, sem mover as existentes.
- C) Em ordem alfabética, reorganizando tudo.
- D) Numa global separada.

---

**Q3.** Você remove uma propriedade da classe e recompila. O que acontece com os dados dela já gravados?

- A) São apagados imediatamente.
- B) Permanecem no nó da global, órfãos, até que o objeto seja regravado.
- C) São movidos para outra global.
- D) A compilação é recusada enquanto houver dados.

---

**Q4.** Qual é a forma correta de renomear uma propriedade preservando os dados?

- A) Editar o nome e recompilar.
- B) Usar `ALTER TABLE RENAME COLUMN`.
- C) Acrescentar a nova, migrar os dados, cortar o código, e só então remover a antiga.
- D) Renomear a classe inteira.

---

**Q5.** Você acrescenta `[ Required ]` a uma propriedade numa classe com dados incompletos. O que acontece?

- A) A compilação falha.
- B) Nada muda até alguém tentar regravar uma linha incompleta, que então falha.
- C) As linhas incompletas são apagadas.
- D) As linhas incompletas recebem um valor padrão.

---

**Q6.** Você acrescenta um índice a uma classe populada e compila. Consultas que usam esse índice devolvem menos linhas do que deveriam. Por quê?

- A) O índice está corrompido.
- B) O índice foi criado vazio; falta rodar `%BuildIndices()`.
- C) O otimizador precisa ser reiniciado.
- D) É necessário apagar e recriar a classe.

---

**Q7.** Qual método remove as entradas de um índice sem apagar os dados?

- A) `%KillExtent()`
- B) `%PurgeIndices()`
- C) `%BuildIndices()`
- D) `%ValidateIndices()`

---

**Q8.** Antes de declarar um índice `Unique` sobre uma coluna de uma tabela populada, o que se deve fazer?

- A) Nada; o IRIS resolve as duplicatas sozinho.
- B) Verificar se já existem duplicatas, com `GROUP BY ... HAVING COUNT(*) > 1`.
- C) Apagar todos os índices existentes.
- D) Converter a coluna para `%Integer`.

---

**Q9.** Você muda o tipo de uma propriedade de `%String` para `%Integer` numa classe com dados de texto. O que acontece com os dados existentes?

- A) São convertidos automaticamente.
- B) Permanecem como estão; a validação só age na próxima gravação, e índices podem ficar inconsistentes.
- C) São apagados.
- D) A compilação falha.

---

**Q10.** O que significa dizer que uma migração é **idempotente**?

- A) Que ela roda dentro de uma transação.
- B) Que rodá-la duas vezes produz o mesmo resultado que rodá-la uma vez.
- C) Que ela pode ser desfeita.
- D) Que ela só roda uma vez por instalação.

---

**Q11.** Por que a versão do esquema costuma ser guardada numa global crua, e não numa classe persistente?

- A) Por desempenho.
- B) Porque a migração precisa funcionar mesmo quando as classes do sistema estão inconsistentes ou ainda não existem.
- C) Porque globais não podem ser apagadas.
- D) Porque classes persistentes não aceitam números.

---

**Q12.** Qual é o problema de envolver uma migração de milhões de registros numa única transação?

- A) Transações não funcionam em migração.
- B) Journal enorme, travas mantidas por muito tempo, e uma falha no fim desfaz todo o trabalho.
- C) O IRIS limita transações a 1000 operações.
- D) Não há problema; é a prática recomendada.

---

**Q13.** Um `ALTER TABLE ... ADD COLUMN` no IRIS faz o quê?

- A) Cria apenas uma coluna SQL, sem afetar a classe.
- B) Altera a definição da classe e a recompila.
- C) É recusado em classes definidas por código.
- D) Cria uma tabela separada.

---

**Q14.** Você renomeia uma classe persistente. O que acontece com os dados?

- A) São movidos automaticamente para a global de nome novo.
- B) Continuam na global de nome antigo, invisíveis para a classe nova.
- C) São apagados.
- D) São duplicados nas duas globais.

---

**Q15.** Qual método permite descobrir linhas legadas que violam as regras atuais, **sem** alterá-las?

- A) `%Save()`
- B) `%ValidateObject()`
- C) `%BuildIndices()`
- D) `%KillExtent()`

---

**Q16.** Qual é a ordem correta do ciclo seguro de mudança de esquema?

- A) Remover → acrescentar → preencher → cortar
- B) Acrescentar → preencher → cortar → remover
- C) Cortar → remover → acrescentar → preencher
- D) Preencher → acrescentar → remover → cortar

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** `InitialExpression` é avaliado no `%New()`, e linhas antigas não passam mais por ele.
- **A está errada:** seria necessária uma migração de preenchimento.
- **C está errada:** a compilação funciona normalmente.
- **D está errada:** nenhum dado é tocado.

**Q2 — Resposta: B.**
- **B está certa:** a propriedade nova entra no fim, preservando as posições existentes — é isso que mantém os dados antigos legíveis.
- **A e C estão erradas:** reorganizar posições invalidaria todos os dados gravados.
- **D está errada:** ela vai para a mesma global de dados.

**Q3 — Resposta: B.**
- **B está certa:** compilar altera a definição, não os dados. O valor fica órfão no nó até uma regravação remontá-lo.
- **A está errada:** não há limpeza automática.
- **C está errada:** os dados não são movidos.
- **D está errada:** a compilação não verifica dados.

**Q4 — Resposta: C.**
- **C está certa:** é o ciclo de quatro passos, e é o único caminho que preserva os dados sem janela de indisponibilidade.
- **A está errada:** isso equivale a remover e criar, perdendo o vínculo com o dado antigo.
- **B está errada:** ainda que exista sintaxe DDL, o mapeamento de armazenamento e os dados exigem cuidado próprio.
- **D está errada:** renomear a classe é um problema maior, não a solução.

**Q5 — Resposta: B.**
- **B está certa:** validação é preguiçosa — vale no momento da gravação.
- **A está errada:** a compilação não valida dados.
- **C e D estão erradas:** o IRIS não altera dados por causa de uma mudança de declaração.

**Q6 — Resposta: B.**
- **B está certa:** o índice é criado vazio e precisa ser populado com `%BuildIndices()`.
- **A está errada:** não há corrupção; há ausência.
- **C está errada:** não existe esse procedimento.
- **D está errada:** seria destrutivo e desnecessário.

**Q7 — Resposta: B.**
- **B está certa:** `%PurgeIndices()` remove as entradas de índice indicadas.
- **A está errada:** `%KillExtent()` apaga os **dados**.
- **C está errada:** `%BuildIndices()` popula.
- **D está errada:** `%ValidateIndices()` apenas confere.

**Q8 — Resposta: B.**
- **B está certa:** duplicatas existentes tornam o índice único incoerente. Verifique antes.
- **A está errada:** o IRIS não resolve isso sozinho.
- **C e D estão erradas:** não têm relação com o problema.

**Q9 — Resposta: B.**
- **B está certa:** os dados permanecem; a validação age na gravação seguinte, e a ordenação de índices pode divergir.
- **A está errada:** não há conversão automática.
- **C está errada:** nada é apagado.
- **D está errada:** a compilação aceita a mudança.

**Q10 — Resposta: B.**
- **B está certa:** idempotência é poder rodar de novo sem efeito adicional — essencial para retomar após falha.
- **A está errada:** isso é atomicidade, outro conceito.
- **C está errada:** isso é reversibilidade.
- **D está errada:** o controle de "só uma vez" é o versionamento, não a idempotência.

**Q11 — Resposta: B.**
- **B está certa:** a global não depende de nenhuma classe estar compilada nem íntegra.
- **A está errada:** desempenho é irrelevante aqui.
- **C está errada:** globais podem ser apagadas normalmente.
- **D está errada:** classes persistentes aceitam números sem problema.

**Q12 — Resposta: B.**
- **B está certa:** o custo em journal e travas é alto, e uma falha desfaz todo o progresso.
- **A está errada:** transações funcionam; a questão é escala.
- **C está errada:** não existe esse limite.
- **D está errada:** a prática recomendada é transação por unidade de trabalho.

**Q13 — Resposta: B.**
- **B está certa:** no IRIS, DDL altera a definição da classe e recompila.
- **A está errada:** não existe coluna sem propriedade correspondente.
- **C está errada:** é aceito, embora possa conflitar com o código-fonte versionado.
- **D está errada:** nenhuma tabela nova é criada.

**Q14 — Resposta: B.**
- **B está certa:** o nome da global deriva do nome da classe; renomear deixa os dados para trás.
- **A está errada:** não há movimentação automática.
- **C e D estão erradas:** nada é apagado nem duplicado.

**Q15 — Resposta: B.**
- **B está certa:** `%ValidateObject()` aplica as regras e devolve `%Status` sem gravar.
- **A está errada:** `%Save()` grava — ou falha, alterando o estado da tentativa.
- **C e D estão erradas:** tratam de índices e de exclusão de dados.

**Q16 — Resposta: B.**
- **B está certa:** acrescentar, preencher, cortar o uso do antigo e só então remover — cada etapa reversível.
- **A, C e D estão erradas:** todas removem ou cortam antes de o novo estar pronto, criando janela de inconsistência.

---

## 9. Resumo relâmpago

1. Compilar altera a **definição**; **não** altera os dados gravados.
2. Propriedade nova entra **no fim** do mapa de armazenamento; as posições existentes não se movem.
3. Linhas antigas leem a propriedade nova como **vazio**. **`InitialExpression` não alcança o legado.**
4. Remover uma propriedade **deixa o dado órfão** no nó da global. Só uma regravação limpa.
5. **Cuidado com o reaproveitamento de posição**: remover e depois acrescentar pode fazer a propriedade nova ler dados da antiga.
6. **Não existe renomeação automática.** Renomear é remover mais criar. Use o ciclo de quatro passos.
7. **Ciclo seguro: acrescentar → preencher → cortar → remover.** Cada etapa é reversível.
8. **Validação é preguiçosa**: regras novas valem na próxima **gravação**, não retroativamente.
9. Apertar `Required`, `MAXLEN`, `VALUELIST`, `MINVAL` ou `SCALE` cria um passivo silencioso no legado.
10. Antes de um índice `Unique`, **procure duplicatas** com `GROUP BY ... HAVING COUNT(*) > 1`.
11. Índice novo em classe populada exige **`%BuildIndices()`**; índice removido exige **`%PurgeIndices()`**; consistência se confere com **`%ValidateIndices()`**.
12. Mudança de tipo **não converte dados** e pode deixar índices inconsistentes.
13. Mudanças que exigem migração completa: para coleção, para `%SerialObject`, para `Relationship`, e alteração de `IdKey`.
14. Use **`%ValidateObject()`** numa varredura para levantar o legado inválido **sem alterar nada**.
15. DDL de SQL **altera a classe** e a recompila. Defina **uma** fonte da verdade: fonte versionado ou servidor.
16. Renomear a classe deixa os dados na global de nome antigo.
17. Migração precisa ser **idempotente** (rodar de novo não estraga) e **versionada** (sabe o que já aplicou).
18. Marque o passo como aplicado **só depois** de ele terminar com sucesso, e **pare no primeiro erro**.
19. **Transação por unidade de trabalho**, não uma transação única para milhões de registros.
20. `%Dictionary.CompiledClass`, `%Dictionary.CompiledProperty` e `%Dictionary.CompiledIndex` permitem inspecionar o esquema por SQL.
21. Migração que regrava tudo dispara **callbacks, triggers e auditoria**. Planeje o custo.
22. **Sempre confira o resultado**: compare contagens antes e depois.

---

## 10. Cartões de memorização

**Frente:** Compilar uma classe altera os dados já gravados?
**Verso:** Não. Altera apenas a definição. Dados só mudam quando alguém os regrava.

**Frente:** Onde entra uma propriedade nova no mapa de armazenamento?
**Verso:** No fim da lista, sem mover as posições existentes.

**Frente:** `InitialExpression` preenche linhas antigas?
**Verso:** Não. Ele roda no `%New()`, e linhas antigas não passam mais por ele.

**Frente:** O que acontece com os dados de uma propriedade removida?
**Verso:** Ficam órfãos no nó da global até que o objeto seja regravado.

**Frente:** Qual o risco de remover uma propriedade e depois acrescentar outra?
**Verso:** A nova pode ocupar a posição liberada e passar a ler dados antigos que não são dela.

**Frente:** Como renomear uma propriedade sem perder dados?
**Verso:** Ciclo de quatro passos: acrescentar a nova, preencher, cortar o uso da antiga, remover.

**Frente:** O que significa "validação preguiçosa"?
**Verso:** Regras novas valem na próxima gravação. Linhas antigas inválidas sobrevivem até serem tocadas.

**Frente:** Acrescentei `Required`. O que quebra e quando?
**Verso:** Nada quebra na hora. Quebra quando alguém regravar uma linha antiga que está vazia naquele campo.

**Frente:** Criei um índice numa classe com dados. O que falta?
**Verso:** `%BuildIndices()`. Sem isso o índice fica vazio e as consultas devolvem menos linhas, sem erro.

**Frente:** Como remover as entradas de um índice?
**Verso:** `%PurgeIndices($LISTBUILD("NomeDoIndice"))`.

**Frente:** O que fazer antes de declarar um índice `Unique` numa tabela populada?
**Verso:** Procurar duplicatas: `GROUP BY coluna HAVING COUNT(*) > 1`.

**Frente:** Mudei o tipo de uma propriedade. Os dados foram convertidos?
**Verso:** Não. Eles permanecem como estavam; converta explicitamente e reconstrua os índices.

**Frente:** Como listar linhas legadas inválidas sem alterá-las?
**Verso:** Varrer a extensão chamando `%ValidateObject()` em cada objeto.

**Frente:** O que faz `ALTER TABLE ... ADD COLUMN` no IRIS?
**Verso:** Altera a definição da classe e a recompila.

**Frente:** Renomeei a classe. Onde ficaram os dados?
**Verso:** Na global de nome antigo — invisíveis para a classe nova.

**Frente:** O que é uma migração idempotente?
**Verso:** Uma que pode ser executada mais de uma vez sem efeito adicional. Faça-a filtrar pelo que ainda falta.

**Frente:** Quando marcar um passo de migração como aplicado?
**Verso:** Só depois de ele terminar com sucesso. E pare no primeiro erro.

**Frente:** Por que guardar a versão do esquema numa global crua?
**Verso:** Porque a migração precisa funcionar mesmo com as classes do sistema inconsistentes ou inexistentes.

**Frente:** Migração de milhões de linhas: uma transação ou várias?
**Verso:** Várias — transação por unidade de trabalho, com progresso registrado e retomada possível.

**Frente:** Onde o IRIS descreve as próprias classes?
**Verso:** No pacote `%Dictionary`: `CompiledClass`, `CompiledProperty`, `CompiledIndex`, consultáveis por SQL.

**Frente:** Qual o ciclo seguro de mudança de esquema?
**Verso:** Acrescentar → preencher → cortar → remover.

**Frente:** Que custo escondido tem uma migração que regrava tudo?
**Verso:** Ela dispara callbacks, triggers e auditoria — multiplicando o tempo e gerando milhares de entradas de trilha.

---

Digite CONTINUAR para o próximo capítulo.
